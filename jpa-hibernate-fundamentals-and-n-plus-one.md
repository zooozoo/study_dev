# 회원 목록 API 하나로 배우는 JPA · Hibernate 기초와 N+1 문제

Kotlin + Spring 프로젝트에서 `JpaRepository` 인터페이스 하나만 선언하면 DB 조회·저장이 된다. 편한데, **그 뒤에서 누가 SQL을 만들고, 언제 몇 번 실행되고, 왜 `save()`를 안 불렀는데 UPDATE가 나가는지**는 안 보인다. 이 문서는 그 보이지 않는 부분을 "왜 이 개념이 필요한가 → 어떤 문제가 생기나 → 어떻게 푸나" 순서로 한 줄로 꿰어 정리한다.

작성일: 2026-09-20
기준 버전: Hibernate ORM 6.6 · Spring Data JPA 3.5 (Spring Boot 3.5) · Kotlin 2.x. 이 문서의 SQL은 **Hibernate가 만드는 SQL의 대략적인 모양**이다. 실제 출력은 별칭·컬럼 나열 방식이 다르고, 정확한 문장은 [SQL 로그](#a4-sql-로그로-직접-확인하기)로 확인한다.

> 이 문서는 사건이 아니라 학습이 계기다. 사건에서 출발한 자매 문서 [한 건을 고치려다 N건을 읽는 코드로 배우는 JPA 영속성 컨텍스트와 지연 로딩](jpa-lazy-loading-and-persistence-context.md)이 영속성 컨텍스트와 트랜잭션 경계를 더 깊게 다룬다. 이 문서를 먼저 읽고 그쪽으로 가면 순서가 맞는다.

---

## 0. 이 문서 전체를 한 문단으로

`memberRepository.findAll()`로 회원 목록을 가져와 각 회원의 팀 이름을 찍는 코드가 있다. 회원이 100명이면 **SQL이 101번** 나간다. 회원 목록 1번 + 회원마다 팀 조회 1번씩. 이것이 **N+1 문제**다. 팀 연관을 **LAZY**로 두었기 때문에 팀은 "실제로 건드릴 때" 따로 조회되고, 100명을 순회하며 100번 건드리니 100번 조회된다. 고치는 방법은 처음 쿼리에서 팀을 같이 가져오거나(**Fetch Join**, **EntityGraph**), 나중에 조회하더라도 여러 팀을 `IN`으로 묶어 한 번에 가져오는 것(**Batch Fetching**)이다. 이렇게 조회된 엔티티는 **영속성 컨텍스트**가 들고 있어서 같은 트랜잭션 안에서 다시 찾으면 DB에 안 가고(**1차 캐시**), 값을 바꾸면 트랜잭션이 끝날 때 알아서 UPDATE가 나간다(**Dirty Checking**).

이 문단에 모르는 단어가 하나라도 있으면 이 문서가 도움이 된다. 아래 순서대로 읽으면 위 문단이 전부 이해된다.

---

## 질문 → 어디를 볼 것인가

| 질문 | 섹션 |
|---|---|
| JPA와 Hibernate는 뭐가 다른가. Spring Data JPA는 또 뭔가 | [1장](#1-jpa--hibernate--spring-data-jpa--누가-무엇을-하나) |
| JPQL은 SQL과 뭐가 다른가 | [2장](#2-sql과-jpql--테이블을-보느냐-엔티티를-보느냐) |
| LAZY와 EAGER는 뭐가 다르고 뭘 기본으로 둬야 하나 | [3장](#3-lazy와-eager--연관-엔티티를-언제-읽을-것인가) |
| N+1 문제가 뭔가. 왜 생기나 | [4장](#4-n1-문제--목록-하나가-쿼리-101번이-되는-순간) |
| Fetch Join은 뭔가 | [5장](#5-fetch-join--이번-쿼리에서는-팀이-꼭-필요하다) |
| EntityGraph는 언제 쓰나 | [6장](#6-entitygraph--같은-목적을-어노테이션으로-선언하기) |
| Fetch Join과 Batch Fetching은 뭐가 다른가 | [7장](#7-batch-fetching--나중에-읽되-묶어서-읽는다) · [7.3절](#73-fetch-join과-batch-fetching은-무엇이-다른가) |
| 영속성 컨텍스트가 뭔가. 1차 캐시가 뭔가 | [8장](#8-persistence-context--jpa가-엔티티를-들고-있는-공간) |
| `save()`를 안 불렀는데 왜 UPDATE가 나가나 | [9장](#9-dirty-checking--바꾸기만-하면-update가-나간다) |
| 지금까지가 어떻게 하나로 이어지나 | [10장](#10-전체-개념-연결) |
| 면접에서 한 줄로 답하려면 | [11장](#11-핵심-요약-10문장) · [12장](#12-면접-질문-체크리스트) |
| Kotlin 엔티티는 왜 `open`이어야 하나 | [부록 A.1](#a1-kotlin-엔티티-설정--no-arg-생성자와-open-클래스) |
| Fetch Join 쓸 때 조심할 것 (페이징, 컬렉션 두 개) | [부록 A.2](#a2-fetch-join의-두-가지-함정--페이징과-컬렉션-둘) |
| EntityGraph의 FETCH와 LOAD 타입 차이 | [부록 A.3](#a3-entitygraph의-fetch와-load-타입) |
| 쿼리가 몇 번 나가는지 눈으로 보려면 | [부록 A.4](#a4-sql-로그로-직접-확인하기) |

---

## 1. JPA · Hibernate · Spring Data JPA — 누가 무엇을 하나

### 1.1 왜 세 개나 있나

Kotlin 코드에서 DB 행을 다루는 가장 원시적인 방법은 **JDBC(Java Database Connectivity — Java에서 DB에 SQL을 보내고 결과를 받는 표준 API)** 다. `Connection`을 열고, SQL 문자열을 쓰고, `ResultSet`에서 컬럼을 하나씩 꺼내 객체에 옮겨 담는다. 테이블이 30개면 이 옮겨 담는 코드도 30벌이다.

이 "객체 ↔ 테이블 행" 변환을 자동으로 해주는 기술이 **ORM(Object-Relational Mapping — 객체와 관계형 DB 테이블을 서로 대응시켜 주는 기술)** 이다. 그런데 ORM 라이브러리가 여럿 나오자 "ORM은 이런 인터페이스로 쓰자"는 **표준**이 필요해졌고, 그 표준이 JPA다. 그리고 Spring은 그 표준을 더 적게 타이핑하고 쓰도록 한 겹 더 감쌌다. 그래서 세 층이다.

```text
Spring Application        ← 우리가 쓰는 코드 (Service, Controller)
       ↓
Spring Data JPA           ← Repository 인터페이스만 선언하면 구현체를 만들어 줌
       ↓
JPA                       ← 표준 인터페이스 (EntityManager, @Entity, JPQL …)
       ↓
Hibernate                 ← JPA 인터페이스의 실제 구현체. 엔티티 상태 관리, SQL 생성
       ↓
JDBC                      ← SQL 문자열을 DB 드라이버로 보내는 표준 API
       ↓
MySQL / PostgreSQL        ← 실제 DB
```

### 1.2 각 층의 역할

| 층 | 정체 | 하는 일 | 대표 이름 |
|---|---|---|---|
| **JPA** (Jakarta Persistence API) | **표준 스펙**. 인터페이스와 어노테이션의 묶음 | "엔티티는 `@Entity`로 표시한다", "`EntityManager.find()`는 이렇게 동작한다"를 **정의만** 한다. 직접 실행되는 코드가 아니다 | `@Entity`, `@Id`, `EntityManager`, JPQL |
| **Hibernate** | JPA 스펙을 **구현한 라이브러리** | 엔티티 객체의 상태를 추적하고, JPQL을 SQL로 번역하고, JDBC로 DB와 통신한다. 실제로 일하는 쪽 | `Session`(= `EntityManager` 구현), `SessionFactory` |
| **Spring Data JPA** | JPA 위의 **편의 계층** | Repository 인터페이스를 보고 구현 클래스를 **런타임에 만들어 준다**. 메서드 이름 → JPQL 변환, `@Query`, 페이징, `@Transactional` 기본 적용 | `JpaRepository`, `@Query`, `@EntityGraph` |

비유하면 JPA는 **콘센트 규격**, Hibernate는 규격에 맞게 만든 **실제 콘센트**, Spring Data JPA는 콘센트에 꽂기만 하면 되도록 미리 배선해 둔 **멀티탭**이다. Spring Boot에서 `spring-boot-starter-data-jpa`를 넣으면 세 층이 한 번에 딸려 온다. 그래서 "JPA 쓴다"는 말과 "Hibernate 쓴다"는 말이 실무에서 섞여 쓰인다. 정확히는 **JPA 인터페이스로 코드를 쓰고, 실행은 Hibernate가 한다.**

### 1.3 Repository 인터페이스 한 줄이 주는 것

```kotlin
interface UserRepository : JpaRepository<User, Long>
```

이 한 줄을 쓰면 구현 클래스를 하나도 안 만들었는데 아래가 전부 된다.

```kotlin
userRepository.findById(id)     // Optional<User> — SELECT ... WHERE user_id = ?
userRepository.save(user)       // 새 엔티티면 INSERT, 이미 있는 엔티티면 병합
userRepository.delete(user)     // DELETE ... WHERE user_id = ?
userRepository.findAll()        // SELECT 전체
userRepository.count()          // SELECT COUNT(*)
userRepository.existsById(id)   // 존재 여부
```

어떻게 되는 걸까. 애플리케이션이 뜰 때 Spring Data JPA가 `UserRepository`를 구현하는 **프록시 객체**를 만들어 빈으로 등록한다. 그 프록시는 내부적으로 `EntityManager`(JPA)를 들고 있고, `findById`가 불리면 `entityManager.find(User::class.java, id)`를 호출한다. 그러면 Hibernate가 SQL을 만들어 JDBC로 보낸다. **우리 → Spring Data JPA → JPA → Hibernate → JDBC → DB** 순서 그대로다.

메서드 이름으로도 쿼리를 만든다.

```kotlin
interface UserRepository : JpaRepository<User, Long> {
    fun findByName(name: String): List<User>          // 이름 규칙 → JPQL 자동 생성
    @Query("SELECT u FROM User u WHERE u.name = :name")
    fun searchByName(name: String): List<User>        // JPQL을 직접 씀
}
```

두 방식 모두 결국 **JPQL**이 만들어지고, 그 JPQL을 Hibernate가 SQL로 바꾼다. 그래서 다음 장은 JPQL이다.

---

## 2. SQL과 JPQL — 테이블을 보느냐 엔티티를 보느냐

### 2.1 왜 SQL을 그대로 안 쓰고 JPQL이 따로 있나

ORM의 약속은 "개발자는 **객체**만 다루고, 테이블은 ORM이 신경 쓴다"는 것이다. 그런데 조회 조건은 결국 문장으로 써야 한다. 이때 SQL을 쓰면 다시 테이블·컬럼 이름이 코드에 들어오고, 결과도 행(row)으로 돌아와 객체로 다시 옮겨 담아야 한다. 그래서 JPA는 **엔티티와 프로퍼티를 대상으로 하는 조회 언어**를 따로 정의했다. 그것이 **JPQL(Jakarta Persistence Query Language)** 이다.

> Hibernate에는 자체 언어 **HQL(Hibernate Query Language)** 이 있고, Hibernate 문서는 "JPQL은 초기 HQL에서 영감을 받았고 현대 HQL의 진부분집합(proper subset)"이라고 설명한다. 즉 JPQL로 쓴 문장은 전부 HQL로도 유효하고, HQL에는 표준에 없는 확장이 더 있다. Spring Data JPA의 `@Query`에 쓰는 문장은 Hibernate가 HQL 파서로 읽는다.

### 2.2 같은 조회를 SQL과 JPQL로

DB 테이블이 이렇고,

```sql
CREATE TABLE users (
    user_id   BIGINT PRIMARY KEY,
    user_name VARCHAR(100)
);
```

엔티티가 이렇다고 하자. 테이블 이름과 클래스 이름, 컬럼 이름과 프로퍼티 이름이 **일부러 다르게** 되어 있다.

```kotlin
@Entity
@Table(name = "users")
class User(

    @Id
    @Column(name = "user_id")
    val id: Long,

    @Column(name = "user_name")
    val name: String
)
```

**SQL**은 DB가 아는 이름을 쓴다.

```sql
SELECT *
FROM users
WHERE user_name = 'John';
```

**JPQL**은 Kotlin 코드가 아는 이름을 쓴다.

```jpql
SELECT u
FROM User u
WHERE u.name = :name
```

차이를 나란히 놓으면 이렇다.

```text
SQL                           JPQL

users                         User
user_name                     u.name

→ DB의 테이블 / 컬럼 이름     → Kotlin/Java 엔티티 클래스 / 프로퍼티 이름
→ 결과: 행(row)의 집합        → 결과: User 객체의 리스트
→ SELECT * 가 자연스러움      → SELECT u — "엔티티 u를 통째로"
```

| 관점 | SQL | JPQL |
|---|---|---|
| 대상 | 테이블, 컬럼 | 엔티티, 프로퍼티 |
| `FROM` 뒤 | `users` (테이블) | `User` (엔티티 이름 — 기본은 클래스 이름) |
| 조건의 왼쪽 | `user_name` (컬럼) | `u.name` (프로퍼티) |
| 결과 | `ResultSet` 행 | 엔티티 객체 (영속성 컨텍스트가 관리 → [8장](#8-persistence-context--jpa가-엔티티를-들고-있는-공간)) |
| 연관 표현 | `JOIN team t ON m.team_id = t.id` | `JOIN m.team t` — FK 컬럼 이름을 몰라도 됨 |
| DB 종속성 | 방언(dialect)에 묶임 | 없음. Hibernate가 DB별 SQL로 번역 |
| 틀린 이름을 쓰면 | 실행 시 DB 오류 | JPQL 파싱 시 오류 (Spring Data JPA는 **기동 시점**에 `@Query`를 검증) |

Hibernate 문서의 표현을 빌리면, JPQL/HQL에서 `Book`은 Java 클래스를, `book.title`은 그 클래스의 필드를 가리키며 **DB 테이블과 컬럼을 직접 참조하는 것은 허용되지 않는다.** 그래서 `WHERE u.user_name = :name`이라고 쓰면 "User 엔티티에 `user_name`이라는 프로퍼티가 없다"며 실패한다. 컬럼 이름이 아니라 **프로퍼티 이름**을 써야 한다.

### 2.3 Hibernate가 사이에서 하는 일

```text
JPQL  "SELECT u FROM User u WHERE u.name = :name"
  ↓
엔티티 매핑 확인
  User    → @Table(name = "users")      → users 테이블
  u.name  → @Column(name = "user_name") → user_name 컬럼
  ↓
SQL 생성 (DB 방언에 맞춰)
  SELECT u1_0.user_id, u1_0.user_name FROM users u1_0 WHERE u1_0.user_name = ?
  ↓
JDBC로 실행, 파라미터 바인딩 (:name → ?)
  ↓
결과 행 → User 객체로 조립 → 영속성 컨텍스트에 등록 → 반환
```

이 번역 덕분에 **컬럼 이름이 바뀌어도 `@Column(name=...)` 한 곳만 고치면** 되고, MySQL에서 PostgreSQL로 옮겨도 JPQL은 그대로다. 대신 "이 JPQL이 어떤 SQL이 되나"는 눈에 안 보인다. 그 안 보이는 부분에서 이 문서의 나머지 문제들이 생긴다.

### 2.4 왜 이게 실전에서 중요한가

- `@Query`를 쓸 때 컬럼 이름을 넣어 기동 실패하는 일이 잦다. 항상 **프로퍼티 이름**을 쓴다고 기억하면 된다.
- Spring Data JPA의 메서드 이름 쿼리(`findByName`)도 프로퍼티 이름 기준이다. `findByUserName`은 `userName` 프로퍼티를 찾는다.
- 정말 DB 고유 문법이 필요하면 `@Query(nativeQuery = true)`로 SQL을 직접 쓴다. 이때는 테이블·컬럼 이름을 쓴다. 다만 결과를 엔티티로 받으려면 컬럼이 매핑과 맞아야 한다.

---

## 3. LAZY와 EAGER — 연관 엔티티를 언제 읽을 것인가

### 3.1 왜 이 선택지가 필요한가

이 문서의 나머지 예제는 **회원(Member)과 팀(Team)** 이다. 회원 여럿이 팀 하나에 속한다.

```kotlin
@Entity
class Team(
    @Id
    @GeneratedValue
    val id: Long = 0,

    val name: String
)

@Entity
class Member(
    @Id
    @GeneratedValue
    val id: Long = 0,

    var name: String,                       // 9장에서 바꿔 볼 것이라 var

    @ManyToOne(fetch = FetchType.LAZY)
    val team: Team
)
```

> Kotlin에서 이 클래스가 그대로 동작하려면 `open` 클래스와 인자 없는 생성자가 필요하다. Spring Initializr가 만들어 주는 설정이 그 일을 해 준다. → [부록 A.1](#a1-kotlin-엔티티-설정--no-arg-생성자와-open-클래스)

`Member`를 DB에서 읽을 때 `team` 필드를 어떻게 채울지가 문제다. Member 테이블에는 `team_id` 컬럼만 있고, 팀 이름은 Team 테이블에 있다. 선택지는 둘이다.

- **지금 같이 읽는다** — 회원을 읽는 김에 팀도 조회해서 `team` 필드에 진짜 `Team` 객체를 넣는다. → **EAGER**
- **나중에 필요하면 읽는다** — 일단 `team` 자리에 "필요해지면 그때 가져오겠다"는 **프록시(proxy — 진짜인 척하는 대리 객체)** 를 꽂아 둔다. → **LAZY**

### 3.2 LAZY의 동작 흐름

```text
Member 조회
   ↓
Team은 아직 조회하지 않음 (team 필드에는 프록시)
   ↓
member.team.name 처럼 실제로 팀 내용에 접근
   ↓
그 순간 Team SELECT 실행 → 프록시가 진짜 데이터로 채워짐
```

코드로 보면 이렇다.

```kotlin
val member = memberRepository.findById(1L).get()
```

```sql
SELECT *
FROM member
WHERE id = 1;
-- team 테이블은 건드리지 않았다
```

이후에

```kotlin
println(member.team.name)
```

를 실행하는 순간,

```sql
SELECT *
FROM team
WHERE id = ?;
```

가 **추가로** 나간다. `member.team` 자체는 프록시라 접근해도 쿼리가 안 나가고, `.name`처럼 **내용을 요구하는 순간** 나간다. (프록시의 `id`만 읽는 것은 보통 쿼리 없이 된다. FK 값은 이미 알고 있으니까.)

중요한 조건이 하나 있다. 이 "나중에 읽기"는 **영속성 컨텍스트가 아직 열려 있을 때**만 된다. 트랜잭션이 끝나 컨텍스트가 닫힌 뒤에 `member.team.name`을 건드리면 `LazyInitializationException`이 난다. 자세한 건 자매 문서 [부록 A.1](jpa-lazy-loading-and-persistence-context.md#a1-lazyinitializationexception--세션이-닫힌-뒤에-건드리면)에 있다.

### 3.3 EAGER는 왜 기본으로 두면 안 되나

EAGER는 반대로 **Member를 조회할 때 Team도 같이 준비**한다. `findById(1L)` 한 번에 팀까지 채워져 있으니 편해 보인다. 문제는 두 가지다.

**첫째, 모든 API가 팀을 필요로 하지 않는다.** 회원 이름 목록만 내려주는 API(`GET /members/names`)도 팀을 읽게 된다. 연관이 셋이면 셋 다, 그 연관의 연관까지 EAGER면 그것까지 딸려 온다. 안 쓰는 데이터를 위해 쿼리와 메모리를 쓴다.

**둘째, EAGER는 N+1을 막아 주지 않는다.** 이게 자주 오해되는 지점이다. `findById` 같은 단건 조회는 Hibernate가 JOIN으로 한 번에 읽어 줄 수 있지만, `findAll()`이나 JPQL로 **목록**을 조회하면 Hibernate는 일단 JPQL에 적힌 대로 회원만 SELECT한 뒤, EAGER 약속을 지키려고 **회원마다 팀을 즉시 따로 조회**한다. 결과는 4장의 N+1과 똑같고, 심지어 팀을 쓰지 않는 API에서도 발생한다. LAZY였다면 안 건드리니 안 나갔을 쿼리다.

JPA 표준의 기본값은 `@ManyToOne`·`@OneToOne`이 EAGER, `@OneToMany`·`@ManyToMany`가 LAZY다. 그런데 Hibernate 문서는 명시적으로 **"모든 연관을 정적으로 LAZY로 두고, 즉시 로딩이 필요하면 동적 fetch 전략을 쓰라"** 고 권한다. 기본값 표는 자매 문서 [부록 A.2](jpa-lazy-loading-and-persistence-context.md#a2-연관관계별-fetch-기본값)에 있다.

### 3.4 설계 방향

```text
기본적으로 LAZY
      +
필요한 조회에서 Fetch 전략을 별도로 선택
(Fetch Join · EntityGraph · Batch Fetching)
```

엔티티의 `fetch` 속성은 **"이 연관을 평소에 어떻게 다룰까"** 를 정하는 정적 기본값이고, **"이번 API에서는 팀이 꼭 필요하다"** 는 판단은 쿼리 쪽에서 내린다. 이 분리가 이 문서의 핵심 설계 원칙이고, 5~7장은 그 "쿼리 쪽에서 내리는 방법" 세 가지다.

---

## 4. N+1 문제 — 목록 하나가 쿼리 101번이 되는 순간

### 4.1 정의

> **N개의 엔티티를 한 번의 쿼리로 조회한 뒤, 각 엔티티의 연관 엔티티를 조회하면서 추가 쿼리가 N번 발생하는 문제.**

### 4.2 재현

`GET /members` API가 회원과 소속 팀 이름을 함께 내려준다고 하자.

```kotlin
val members = memberRepository.findAll()

members.forEach {
    println(it.team.name)          // 응답 DTO로 옮기는 자리라고 생각하면 된다
}
```

처음 `findAll()`에서:

```sql
SELECT *
FROM member;
```

→ **1번**. 회원 N명이 올라오고, 각 회원의 `team`은 프록시다.

이후 `forEach` 안에서 `it.team.name`을 건드릴 때마다:

```sql
SELECT * FROM team WHERE id = ?;   -- 1번째 회원의 팀
SELECT * FROM team WHERE id = ?;   -- 2번째 회원의 팀
SELECT * FROM team WHERE id = ?;   -- 3번째 회원의 팀
...
```

→ **최대 N번**. 따라서:

```text
   1        (회원 목록)
+  N        (회원마다 팀 하나)
= N + 1 queries
```

"최대"인 이유는 8장에서 나온다. 같은 팀을 이미 읽었으면 1차 캐시에서 꺼내므로 실제 횟수는 **서로 다른 팀의 수**만큼이다. 회원 100명이 팀 3개에 속해 있으면 1+3이고, 100명이 100개 팀이면 1+100이다. 어느 쪽이든 **데이터 양에 비례해 쿼리가 늘어난다**는 게 문제의 본질이다. 개발 DB에서 회원 5명일 때는 6번이라 못 느끼고, 운영에서 회원 1만 명이 되면 1만 번이 된다.

### 4.3 왜 생기나 — "LAZY라서"가 아니다

> N+1은 단순히 "LAZY라서" 생기는 것이 아니다. **최초 쿼리에서 연관 데이터를 가져오지 않은 상태에서, N개의 엔티티 각각에 대해 연관 데이터에 접근하기 때문**에 생긴다.

이 구분이 중요한 이유:

- 3.3절에서 봤듯 **EAGER로 바꿔도** 목록 조회에서는 N+1이 그대로 난다. 첫 쿼리가 팀을 안 가져온 건 같기 때문이다. 오히려 팀이 필요 없는 API에서도 나게 된다.
- 반대로 LAZY여도 **첫 쿼리에서 팀을 같이 가져오면** N+1은 없다. 그게 5장의 Fetch Join이다.
- 그러니 해결의 방향은 "LAZY를 없앤다"가 아니라 **"이번 조회에 필요한 연관을 첫 쿼리에서(또는 묶어서) 가져온다"** 다.

```text
원인의 구조

  첫 쿼리가 연관을 안 가져옴      ×      N개 각각에서 연관에 접근
  (LAZY든 EAGER-목록조회든)              (forEach, DTO 변환, JSON 직렬화 …)
                        ↓
                  N번의 추가 쿼리
```

### 4.4 어디서 잘 나오나

| 상황 | 접근하는 코드 |
|---|---|
| 목록을 DTO로 변환 | `members.map { MemberResponse(it.name, it.team.name) }` |
| 엔티티를 그대로 JSON 응답 | Jackson이 `team` getter를 호출하며 프록시를 초기화 |
| 템플릿에서 `${member.team.name}` | 뷰 렌더링 시점 (`open-in-view`가 켜져 있을 때) |
| 배치에서 건별 처리 | 루프 안에서 연관 접근 |

공통점은 **"목록 + 연관 접근"** 이다. 이 조합을 코드에서 보면 반사적으로 "첫 쿼리에서 이 연관을 가져오고 있나?"를 물어야 한다. 실제로 몇 번 나가는지 세는 법은 [부록 A.4](#a4-sql-로그로-직접-확인하기)에 있다.

---

## 5. Fetch Join — "이번 쿼리에서는 팀이 꼭 필요하다"

### 5.1 무엇인가

N+1의 원인이 "첫 쿼리가 연관을 안 가져옴"이니, 가장 직접적인 해결은 **첫 쿼리에서 연관까지 JOIN으로 같이 가져오는 것**이다. JPQL에서 그 지시가 `JOIN FETCH`다.

```kotlin
interface MemberRepository : JpaRepository<Member, Long> {

    @Query("""
        SELECT m
        FROM Member m
        JOIN FETCH m.team
    """)
    fun findAllWithTeam(): List<Member>
}
```

Hibernate는 대략 이런 SQL을 만든다.

```sql
SELECT
    m.*,
    t.*
FROM member m
JOIN team t
    ON m.team_id = t.id;
```

한 번의 SQL로 회원과 팀 컬럼이 같이 돌아오고, Hibernate는 그 행으로 `Member` 객체와 `Team` 객체를 **둘 다 조립해서** `member.team`에 진짜 `Team`을 채워 넣는다. 그래서 이후 `it.team.name`을 건드려도 이미 있는 값이라 쿼리가 안 나간다.

```text
Fetch Join 적용 후

  SELECT m.*, t.* FROM member m JOIN team t ...   ← 1번
  forEach { it.team.name }                        ← 0번 (이미 로딩됨)
  = 1 query
```

일반 `JOIN`과의 차이를 짚고 가자. `SELECT m FROM Member m JOIN m.team t WHERE t.name = 'A'`처럼 **FETCH 없는 JOIN**은 조건을 걸기 위해 조인만 하고, 결과로는 `Member`만 조립한다. `team`은 여전히 프록시라 N+1이 그대로 난다. **연관을 결과에 채워 넣으라**는 지시는 `FETCH`가 붙어야 한다.

### 5.2 핵심 의미 — 매핑은 그대로, 판단은 쿼리에서

```text
Entity Mapping은 LAZY로 유지
        ↓
하지만 "이번 Query에서는 Team이 반드시 필요하다"
        ↓
그 쿼리에만 Fetch Join
```

`@ManyToOne(fetch = LAZY)`는 손대지 않는다. 대신 팀이 필요한 조회 메서드에만 `JOIN FETCH`를 쓴다. 그래서 같은 `Member` 엔티티를 두고 API마다 다른 조회 전략을 가질 수 있다.

### 5.3 API별 조회 전략

```text
GET /members/names          회원 이름 목록만 필요
  → memberRepository.findAll()          (일반 조회, 팀 안 읽음 — 쿼리 1번)

GET /members                회원 + 소속 팀 이름 필요
  → memberRepository.findAllWithTeam()  (Fetch Join — 쿼리 1번, 팀 포함)
```

```kotlin
@Service
class MemberQueryService(private val memberRepository: MemberRepository) {

    @Transactional(readOnly = true)
    fun names(): List<String> =
        memberRepository.findAll().map { it.name }              // team 접근 없음

    @Transactional(readOnly = true)
    fun listWithTeam(): List<MemberResponse> =
        memberRepository.findAllWithTeam()
            .map { MemberResponse(it.name, it.team.name) }      // 이미 로딩된 team
}
```

즉 **엔티티 자체를 EAGER로 고정하지 않고, 각 API 또는 유스케이스가 실제로 필요한 데이터에 맞춰 조회 전략을 선택한다.** 만약 `Member.team`을 EAGER로 박아 두었다면 `names()`도 팀을 읽었을 것이고, `listWithTeam()`에서는 3.3절대로 N+1이 났을 것이다. 둘 다 손해다.

### 5.4 알아 둘 것 두 가지

- **`JOIN FETCH`는 기본이 내부 조인(inner join)** 이다. 팀이 없는 회원(`team_id`가 NULL)은 결과에서 빠진다. 팀 없는 회원도 포함하려면 `LEFT JOIN FETCH m.team`을 쓴다. Hibernate 문서도 "inner join fetch는 연관이 있는 기준 엔티티만, left outer join fetch는 연관 유무와 무관하게 전부"라고 구분한다.
- **컬렉션(`@OneToMany`)을 Fetch Join 할 때는 함정이 둘** 있다. 페이징이 메모리에서 되어 버리는 문제와, 컬렉션 둘을 동시에 fetch하면 카테시안 곱이 되는 문제다. → [부록 A.2](#a2-fetch-join의-두-가지-함정--페이징과-컬렉션-둘)

---

## 6. EntityGraph — 같은 목적을 어노테이션으로 선언하기

### 6.1 무엇인가

EntityGraph는 JPA 표준 기능으로, **"이 조회에서는 이 연관들을 함께 로딩하라"** 는 계획을 쿼리 문장이 아니라 **선언**으로 지정한다. Spring Data JPA에서는 `@EntityGraph` 어노테이션으로 쓴다.

```kotlin
interface MemberRepository : JpaRepository<Member, Long> {

    @EntityGraph(attributePaths = ["team"])
    @Query("SELECT m FROM Member m")
    fun findAllWithTeam(): List<Member>

    // 메서드 이름 쿼리에도 붙는다
    @EntityGraph(attributePaths = ["team"])
    fun findByNameContaining(keyword: String): List<Member>

    // 기본 제공 메서드를 override 해서 붙일 수도 있다
    @EntityGraph(attributePaths = ["team"])
    override fun findAll(): List<Member>
}
```

의미:

```text
Member를 조회할 때
team도 함께 로딩
```

Spring Data JPA는 `attributePaths`를 JPA `EntityGraph` 객체로 바꿔 쿼리 힌트로 넘기고, Hibernate는 그 힌트를 보고 `team`을 JOIN으로 같이 가져온다. 결과 SQL은 5장의 Fetch Join과 사실상 같은 모양(LEFT JOIN)이 된다. 중첩 연관은 `"team.company"`처럼 점으로 잇는다.

### 6.2 Fetch Join과 비교

```text
Fetch Join

  JPQL Query 문장 안에
  JOIN FETCH m.team 을 직접 쓴다
  → 쿼리와 fetch 전략이 한 문장에 있음
```

```text
EntityGraph

  Query는 그대로 두고
  @EntityGraph(attributePaths = ["team"]) 로 fetch 전략만 따로 선언
  → 쿼리와 fetch 전략이 분리됨
```

| | Fetch Join | EntityGraph |
|---|---|---|
| 지정 위치 | JPQL 문장 | 어노테이션 (또는 `@NamedEntityGraph`) |
| JPQL 없이 사용 | 불가 (문장이 있어야 함) | 가능 — 메서드 이름 쿼리, `findAll` 등에도 붙음 |
| 조인 종류 | 직접 고름 (`JOIN FETCH` / `LEFT JOIN FETCH`) | Hibernate가 LEFT JOIN으로 처리 |
| 같은 쿼리의 변형 | 문장을 하나 더 씀 | 같은 문장에 그래프만 다르게 붙임 |
| 세밀한 제어 (조인 조건, 별칭 재사용) | 자유로움 | 제한적 |

**Fetch Join을 먼저 이해하고, EntityGraph는 그와 비슷한 목적의 다른 표현 방식으로 이해하면 된다.** 실무에서는 "JPQL을 직접 쓰는 메서드는 `JOIN FETCH`, 메서드 이름 쿼리나 기본 메서드에 연관을 얹고 싶을 때는 `@EntityGraph`" 정도로 나뉘는 경우가 많다. 둘 다 **"첫 쿼리에서 연관을 같이 가져온다"** 는 점은 같고, 그래서 컬렉션 페이징 함정([부록 A.2](#a2-fetch-join의-두-가지-함정--페이징과-컬렉션-둘))도 같이 적용된다.

`@EntityGraph`에는 `type` 속성이 있고 기본값은 `FETCH`다. `FETCH`와 `LOAD`의 차이는 [부록 A.3](#a3-entitygraph의-fetch와-load-타입)에 정리했다.

---

## 7. Batch Fetching — 나중에 읽되, 묶어서 읽는다

### 7.1 무엇인가

Fetch Join과 EntityGraph는 **첫 쿼리를 바꾸는** 방법이다. Batch Fetching은 다른 접근이다. **첫 쿼리는 그대로 두고, 나중에 LAZY 로딩이 일어날 때 한 건씩이 아니라 여러 건을 묶어서 가져온다.**

기존 LAZY 로딩:

```text
Member 1 → Team 조회
Member 2 → Team 조회
Member 3 → Team 조회
...
```

```sql
SELECT * FROM team WHERE id = 1;
SELECT * FROM team WHERE id = 2;
SELECT * FROM team WHERE id = 3;
```

Batch Fetching:

```text
Member 1의 team에 접근
   ↓
"지금 영속성 컨텍스트에 아직 초기화 안 된 Team 프록시가 또 있나?"
   ↓
있으면 그 id들을 모아서 한 번에
```

```sql
SELECT *
FROM team
WHERE id IN (1, 2, 3, ...);
```

첫 번째 회원의 팀을 건드리는 순간, Hibernate는 영속성 컨텍스트 안의 **아직 초기화되지 않은 같은 타입 프록시**들을 배치 크기만큼 모아 `IN` 절 하나로 조회한다. 그러면 두 번째, 세 번째 회원의 팀은 이미 로딩되어 있어 쿼리가 안 나간다.

### 7.2 설정

전역 기본값은 Spring Boot 설정으로 준다.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 100
```

Hibernate 설정 키는 `hibernate.default_batch_fetch_size`다. Hibernate 문서에 따르면 이 값을 주지 않으면 **`@BatchSize`를 명시한 엔티티·컬렉션에만** 배치 로딩이 적용된다. 즉 전역 설정 없이는 꺼져 있는 셈이다. 특정 연관에만 다른 크기를 주고 싶으면 엔티티 클래스나 컬렉션 필드에 `@BatchSize(size = 50)`을 붙인다. 이 방식은 `@ManyToOne` 프록시와 `@OneToMany` 컬렉션 **둘 다**에 적용된다.

> Hibernate 5에서 있던 `hibernate.batch_fetch_style` 옵션은 Hibernate 6에서 deprecated 됐고, 문서는 "적절한 배치 스타일이 자동으로 선택된다"고만 밝힌다. 그래서 `IN` 목록의 정확한 길이(예: 100개 미만이면 몇 개 단위로 끊는지)는 버전에 따라 다를 수 있다. 실제 문장은 SQL 로그로 확인한다.

### 7.3 Fetch Join과 Batch Fetching은 무엇이 다른가

```text
Fetch Join

  처음 Query에서
  JOIN으로 연관 Entity까지 한 번에 조회
  → 쿼리 1번
```

```text
Batch Fetching

  첫 Query에서는 부모 Entity만 조회

  나중에 Lazy Loading이 발생할 때
  여러 연관 Entity를 IN Query로 묶어서 조회
  → 쿼리 1번 + ceil(연관 수 / batch size)번
```

따라서 Batch Fetching은:

```text
1 + N
  ↓
1 + 몇 번의 Batch Query
```

로 **줄이는(완화하는)** 방법이다. 없애는 게 아니다. 회원 1000명이 서로 다른 팀 1000개에 속하고 batch size가 100이면 1 + 10번이다.

| | Fetch Join / EntityGraph | Batch Fetching |
|---|---|---|
| 적용 시점 | 첫 쿼리 | LAZY 로딩이 발생하는 순간 |
| 적용 단위 | 특정 조회 메서드 | 전역 설정 또는 엔티티/컬렉션 단위 |
| 쿼리 수 | 1 | 1 + (연관 수 ÷ 배치 크기) |
| 코드 변경 | 조회 메서드마다 지정 | 설정 한 줄로 전체에 적용 |
| 컬렉션 페이징 | 메모리 페이징 함정 있음 | **없음** — 부모는 DB에서 페이징, 자식은 나중에 IN |
| 컬렉션 여러 개 | 카테시안 곱 / 예외 위험 | 없음 — 컬렉션마다 별도 IN 쿼리 |
| 잘 맞는 상황 | "이 API는 이 연관이 반드시 필요" | 페이징 목록 + 컬렉션, 또는 코드 곳곳의 산발적 LAZY 접근을 한 번에 완화 |

실무의 흔한 조합은 이렇다. **`@ManyToOne`처럼 행 수가 안 늘어나는 연관은 Fetch Join**으로 첫 쿼리에 붙이고, **`@OneToMany` 컬렉션은 페이징과 충돌하므로 Batch Fetching**으로 처리하고, `default_batch_fetch_size`는 **안전망으로 항상 켜 둔다.** 안전망을 켜 두면 놓친 N+1이 최소한 1+N에서 1+몇 번으로는 내려온다.

Hibernate 문서는 배치 로딩을 소개하면서도 DTO 프로젝션이나 `JOIN FETCH`가 보통 더 나은 성능을 낸다고 덧붙인다. 배치는 만능이 아니라 **첫 쿼리를 손댈 수 없거나 손대면 다른 문제가 생기는 자리의 차선책**으로 보는 게 맞다.

---

## 8. Persistence Context — JPA가 엔티티를 들고 있는 공간

### 8.1 왜 이런 공간이 필요한가

지금까지 "조회된 엔티티"라고만 했는데, 그 엔티티는 조회된 뒤 어디에 있을까. 그냥 Kotlin 객체로 반환되고 끝이면 JPA는 다음 두 가지를 할 수 없다.

- 같은 행을 두 번 조회했을 때 **같은 객체**를 돌려주는 것 (안 그러면 한쪽을 고쳤는데 다른 쪽은 옛날 값인 상황이 생긴다)
- 객체의 값이 **바뀌었는지** 알아채는 것 (9장)

그래서 JPA는 조회하거나 저장한 엔티티를 **자기 관리 공간에 등록**해 두고 추적한다. 그 공간이 **영속성 컨텍스트(Persistence Context)** 다.

> **정의**: JPA가 현재 관리하고 있는 엔티티들의 공간. `EntityManager`가 하나씩 들고 있으며, Spring에서는 보통 **트랜잭션 하나 = 영속성 컨텍스트 하나**의 수명을 가진다.

```text
Application
     ↓  find / persist / JPQL
EntityManager
     ↓
Persistence Context        ← { id → 엔티티 인스턴스 } 를 들고 있는 공간
     ↓  필요할 때만 SQL
Database
```

### 8.2 동작 — 같은 id는 한 번만 읽는다

```kotlin
val member1 = entityManager.find(Member::class.java, 1L)
val member2 = entityManager.find(Member::class.java, 1L)
```

첫 번째 `find`에서:

```sql
SELECT *
FROM member
WHERE id = 1;
```

가 실행되고, 영속성 컨텍스트에는 개념적으로 이렇게 저장된다.

```text
Persistence Context

  (Member, id = 1)
       ↓
  Member(id=1, name="John")   ← 지금 메모리에 있는 그 객체
```

두 번째 `find`는 **DB에 가지 않는다.** 컨텍스트에 `(Member, 1)`이 이미 있으니 그 객체를 그대로 돌려준다. 그래서:

```kotlin
member1 === member2   // true — 같은 인스턴스
```

이것이 **1차 캐시(first-level cache)** 다. Hibernate 문서는 `Session`이 "대체로 repeatable read 성격의 영속성 컨텍스트(1차 캐시)"를 유지한다고 표현한다. 같은 트랜잭션 안에서는 같은 id를 몇 번 찾아도 처음 읽은 그 객체다.

### 8.3 1차 캐시가 4장·7장과 연결되는 지점

- 4장에서 N+1의 추가 쿼리가 "최대 N번"이었던 이유가 이것이다. 회원 100명이 팀 3개에 속하면, 팀 id 1을 두 번째로 만나는 순간 컨텍스트에 이미 있으니 쿼리가 안 나간다. 실제 추가 쿼리는 서로 다른 팀 수만큼이다.
- 7장의 Batch Fetching이 "컨텍스트 안의 초기화 안 된 프록시를 모은다"고 한 것도 이 공간이 있어서 가능하다.
- Fetch Join으로 읽은 `Team`도 컨텍스트에 등록된다. 그래서 그 뒤 `entityManager.find(Team::class.java, 1L)`은 DB에 안 간다.

### 8.4 1차 캐시가 아닌 것

캐시라는 말 때문에 생기는 오해를 미리 걷어내자.

| 오해 | 실제 |
|---|---|
| 애플리케이션 전체가 공유하는 캐시다 | **아니다.** 영속성 컨텍스트 하나(= 보통 트랜잭션 하나) 안에서만 유효하다. 트랜잭션이 끝나면 사라진다 |
| JPQL 쿼리도 캐시에서 답한다 | **아니다.** JPQL은 항상 DB로 나간다. 다만 돌아온 행의 id가 컨텍스트에 이미 있으면 **새 객체를 만들지 않고 기존 객체를 돌려준다** (동일성 유지). `find(id)`만 캐시를 먼저 본다 |
| 성능을 위한 캐시다 | 부수 효과일 뿐이고 본래 목적은 **동일성 보장과 변경 추적**이다 |

트랜잭션 경계를 좁히면 이 캐시의 수명도 짧아져 어제까지 캐시 히트였던 조회가 DB 왕복이 되는 사고가 실제로 있다. 그 이야기는 자매 문서 [6장](jpa-lazy-loading-and-persistence-context.md#6-왜-어제까지는-문제가-아니었나--캐시의-수명은-트랜잭션의-수명이다)에 있다.

---

## 9. Dirty Checking — 바꾸기만 하면 UPDATE가 나간다

### 9.1 현상

```kotlin
@Service
class MemberService(private val memberRepository: MemberRepository) {

    @Transactional
    fun changeName(id: Long) {

        val member = memberRepository.findById(id).get()

        member.name = "Alice"
        // memberRepository.save(member) 가 없다
    }
}
```

`save()`를 부르지 않았는데도 메서드가 끝나며 트랜잭션이 커밋되는 시점에:

```sql
UPDATE member
SET name = 'Alice'
WHERE id = ?;
```

가 나간다.

### 9.2 왜 되나 — 영속성 컨텍스트가 원본을 기억한다

8장에서 엔티티는 조회되는 순간 영속성 컨텍스트에 등록된다고 했다. 등록할 때 Hibernate는 **그 시점의 값을 스냅샷으로 따로 저장**해 둔다. 그리고 flush(플러시 — 컨텍스트의 변경 내용을 SQL로 만들어 DB에 반영하는 동작) 시점에 현재 값과 스냅샷을 비교한다.

```text
Entity 조회
  컨텍스트 등록 + 스냅샷 저장
  name = "John"         (스냅샷: name = "John")
        ↓
Entity 값 변경
  member.name = "Alice"
  name = "Alice"        (스냅샷: name = "John")   ← 아직 SQL 없음
        ↓
Transaction 종료 직전 → Flush
  JPA: "스냅샷과 현재 상태가 다르다" (dirty = 더러워졌다 = 바뀌었다)
        ↓
UPDATE SQL 생성 → 실행 → 커밋
```

이것이 **Dirty Checking(변경 감지)** 이다. "관리 중인 엔티티"라는 조건이 핵심이다. 컨텍스트가 관리하는 엔티티만 스냅샷이 있고, 그것만 비교된다.

### 9.3 언제 flush 되나

Hibernate 기본 flush 모드는 `AUTO`이고, 문서는 두 시점을 명시한다.

1. **트랜잭션 커밋 직전**
2. **JPQL/HQL 쿼리를 실행하기 직전**, 그 쿼리가 아직 반영 안 된 변경과 겹칠 때 (예: `Member`를 바꾼 뒤 `SELECT m FROM Member m ...`을 실행하면, 쿼리가 옛 값을 보지 않도록 먼저 UPDATE를 내보낸다)

즉 `member.name = "Alice"`라고 쓴 줄에서 UPDATE가 나가는 게 아니다. 컨텍스트는 변경을 **쌓아 두었다가(write-behind)** 필요한 순간 한꺼번에 내보낸다. 그래서 같은 엔티티의 필드 셋을 바꿔도 UPDATE는 한 번이다.

### 9.4 안 되는 경우 — 실전에서 물리는 지점

| 상황 | 결과 | 이유 |
|---|---|---|
| `@Transactional`이 없는 서비스 메서드에서 `findById` 후 값 변경 | **UPDATE 안 나감** | Spring Data JPA의 `findById`는 자기 트랜잭션(읽기 전용)을 열고 닫는다. 메서드가 반환되는 순간 컨텍스트가 닫혀 엔티티는 **준영속(detached)** 이 되고, 그 뒤 변경은 아무도 안 본다 |
| `val name`인 Kotlin 프로퍼티 | 컴파일 오류 | 바꿀 프로퍼티는 `var`여야 한다 (3장 예제에서 `name`을 `var`로 둔 이유) |
| 트랜잭션 밖에서 값을 바꾸고 `save()` 호출 | UPDATE 나감 | `save()`는 준영속 엔티티를 **merge** 해서 다시 관리 대상으로 만든 뒤 반영한다. 이때는 `save()`가 필요하다 |
| `@Transactional(readOnly = true)` 안에서 값 변경 | Hibernate flush 모드가 `MANUAL`로 바뀌어 **UPDATE 안 나감** | 읽기 전용 트랜잭션은 변경을 반영하지 않는다는 뜻이다. 조회 API에 붙이면 의도치 않은 UPDATE도 막아 준다 |

정리하면, **트랜잭션 안에서 조회한 엔티티를 그 트랜잭션 안에서 바꾸면 `save()` 없이 반영된다.** 반대로 그 조건이 하나라도 깨지면 `save()`가 필요하다. `@Transactional`이 프록시로 어떻게 동작하는지는 [Spring 트랜잭션 노트 3장](spring-transaction-boundaries-and-batch.md#3-transactional의-실체--프록시와-전파)에 있다.

---

## 10. 전체 개념 연결

```text
Entity 연관관계 (@ManyToOne Member.team)
        ↓
LAZY / EAGER  — 연관을 언제 읽을지 정하는 정적 기본값. 기본은 LAZY
        ↓
LAZY Loading  — 프록시를 꽂아 두고 실제 접근 시 SELECT
        ↓
여러 Entity에서 반복되면  (findAll() + forEach { it.team.name })
        ↓
N+1  — 첫 쿼리가 연관을 안 가져온 상태에서 N개 각각이 연관에 접근
        ↓
해결 / 완화  — "이번 조회에 필요한 연관을 어떻게 가져올까"를 쿼리 쪽에서 결정

    Fetch Join       첫 쿼리에서 JOIN FETCH 로 같이     → 1번
    EntityGraph      같은 목적을 @EntityGraph 선언으로   → 1번
    Batch Fetching   나중에 읽되 IN 으로 묶어서          → 1 + 몇 번
        ↓
조회된 Entity
        ↓
Persistence Context가 관리  — 트랜잭션 수명의 { id → 인스턴스 } 공간, 스냅샷 보관
        ↓
1차 캐시  — 같은 id 재조회는 DB 안 감, 같은 인스턴스 보장 (N+1이 "최대 N"인 이유)
        ↓
Entity 값 변경  (member.name = "Alice")
        ↓
Dirty Checking  — flush 시점에 스냅샷과 비교
        ↓
UPDATE SQL  — 커밋 직전(또는 JPQL 실행 직전) 자동 생성
```

한 줄로 다시 말하면: **연관을 LAZY로 두면 N+1이 생길 수 있고, 그건 조회마다 fetch 전략을 골라 해결하며, 그렇게 읽힌 엔티티는 영속성 컨텍스트가 관리하기 때문에 다시 읽지 않아도 되고 바꾸기만 해도 저장된다.**

---

## 11. 핵심 요약 10문장

1. **JPA는 ORM 표준**(인터페이스·어노테이션의 스펙)이고 **Hibernate는 그 대표적인 구현체**다. 코드는 JPA로 쓰고 실행은 Hibernate가 한다.
2. **Spring Data JPA**는 JPA를 Spring에서 편하게 쓰도록 Repository 추상화를 제공한다. 인터페이스만 선언하면 구현 프록시를 만들어 준다.
3. **SQL은 테이블/컬럼**을 대상으로 하고 **JPQL은 엔티티/프로퍼티**를 대상으로 한다. Hibernate가 매핑을 보고 JPQL을 SQL로 번역한다.
4. **LAZY**는 연관 엔티티를 실제 사용할 때 조회한다. 기본은 LAZY로 두고 필요한 조회에서 fetch 전략을 고른다.
5. **N+1**은 N개의 엔티티 각각에서 연관 데이터를 조회하면서 추가 쿼리가 반복되는 문제다. 첫 쿼리가 연관을 안 가져왔기 때문에 생기며, EAGER로 바꿔도 목록 조회에서는 그대로 난다.
6. **Fetch Join**은 특정 조회에서 필요한 연관 엔티티를 처음부터 JOIN으로 같이 가져온다. 쿼리 1번.
7. **EntityGraph**는 같은 목적을 `@EntityGraph` 어노테이션으로 선언적으로 지정하는 방법이다. 메서드 이름 쿼리와 기본 메서드에도 붙는다.
8. **Batch Fetching**은 LAZY 로딩이 일어날 때 여러 대상을 `IN` 쿼리로 묶어 1+N을 1+몇 번으로 완화한다. 첫 쿼리는 그대로다.
9. **Persistence Context**는 JPA가 엔티티를 관리하는 공간이다. 같은 id는 한 번만 읽고 같은 인스턴스를 돌려주며(**1차 캐시**), 수명은 보통 트랜잭션과 같다.
10. **Dirty Checking**은 관리 중인 엔티티의 변경을 스냅샷과 비교해 감지하고 flush 시점에 UPDATE SQL을 자동으로 생성하는 기능이다. `save()` 없이 반영된다.

---

## 12. 면접 질문 체크리스트

각 질문에 소리 내어 답해 보고, 막히면 오른쪽 섹션으로 돌아간다. 답 예시는 "한 문장 + 근거 한 문장" 길이다.

| 질문 | 답의 뼈대 | 다시 볼 곳 |
|---|---|---|
| **JPA와 Hibernate의 차이는?** | JPA는 ORM 표준 스펙(인터페이스), Hibernate는 그 구현체. `EntityManager`는 JPA 인터페이스이고 Hibernate의 `Session`이 그 구현이다. | [1.2절](#12-각-층의-역할) |
| **Spring Data JPA는 무엇인가?** | JPA 위의 편의 계층. `JpaRepository` 인터페이스만 선언하면 구현 프록시를 만들어 주고, 메서드 이름 쿼리·`@Query`·페이징·`@EntityGraph`를 제공한다. | [1.3절](#13-repository-인터페이스-한-줄이-주는-것) |
| **JPQL과 SQL의 차이는?** | SQL은 테이블·컬럼, JPQL은 엔티티·프로퍼티를 대상으로 한다. 결과도 행이 아니라 엔티티 객체이며 영속성 컨텍스트가 관리한다. Hibernate가 매핑을 보고 SQL로 번역한다. | [2.2절](#22-같은-조회를-sql과-jpql로) |
| **LAZY와 EAGER의 차이는?** | LAZY는 연관에 프록시를 꽂아 두고 실제 접근 시 조회, EAGER는 엔티티 조회 시 같이 준비. EAGER는 불필요한 조회를 만들고 목록 조회의 N+1도 못 막으므로 기본은 LAZY. | [3장](#3-lazy와-eager--연관-엔티티를-언제-읽을-것인가) |
| **N+1 문제란?** | 목록 1번 조회 후 각 엔티티의 연관을 접근하며 N번의 추가 쿼리가 나가는 문제. 데이터 양에 비례해 쿼리 수가 는다. | [4.1절](#41-정의) · [4.2절](#42-재현) |
| **N+1은 왜 발생하는가?** | "LAZY라서"가 아니라 첫 쿼리가 연관을 안 가져온 상태에서 N개 각각이 연관에 접근하기 때문. EAGER도 목록 조회에선 같은 이유로 발생한다. | [4.3절](#43-왜-생기나--lazy라서가-아니다) |
| **Fetch Join은 무엇인가?** | JPQL의 `JOIN FETCH`로 첫 쿼리에서 연관 엔티티까지 JOIN으로 같이 가져와 결과에 채우는 것. 매핑은 LAZY로 두고 이 조회에만 적용한다. 기본은 inner join. | [5장](#5-fetch-join--이번-쿼리에서는-팀이-꼭-필요하다) |
| **Fetch Join과 Batch Fetching의 차이는?** | Fetch Join은 첫 쿼리에서 JOIN으로 1번에, Batch Fetching은 첫 쿼리는 그대로 두고 LAZY 로딩 시 `IN`으로 묶어 1+몇 번. 컬렉션 페이징에는 Batch가 안전하다. | [7.3절](#73-fetch-join과-batch-fetching은-무엇이-다른가) |
| **EntityGraph는 언제 사용하는가?** | Fetch Join과 같은 목적(연관 함께 로딩)을 쿼리 문장 대신 어노테이션으로 선언하고 싶을 때. 특히 메서드 이름 쿼리나 `findAll` 같은 기본 메서드에 연관을 얹을 때. | [6장](#6-entitygraph--같은-목적을-어노테이션으로-선언하기) |
| **Persistence Context란?** | JPA가 관리 중인 엔티티를 담는 공간. `EntityManager`가 들고 있고 수명은 보통 트랜잭션. 동일성 보장·변경 감지·쓰기 지연의 근거가 된다. | [8.1절](#81-왜-이런-공간이-필요한가) |
| **1차 캐시란?** | 영속성 컨텍스트가 `{id → 인스턴스}`를 들고 있어 같은 id의 `find`는 DB에 안 가고 같은 인스턴스를 돌려주는 것. 트랜잭션 범위이며 JPQL은 캐시를 거치지 않고 DB로 간다. | [8.2절](#82-동작--같은-id는-한-번만-읽는다) · [8.4절](#84-1차-캐시가-아닌-것) |
| **Dirty Checking은 어떻게 동작하는가?** | 엔티티를 컨텍스트에 등록할 때 스냅샷을 저장하고, flush 시점(커밋 직전·JPQL 실행 직전)에 현재 값과 비교해 다르면 UPDATE를 만든다. 트랜잭션 밖의 준영속 엔티티에는 안 된다. | [9.2절](#92-왜-되나--영속성-컨텍스트가-원본을-기억한다) · [9.4절](#94-안-되는-경우--실전에서-물리는-지점) |

---

# 부록

## A.1 Kotlin 엔티티 설정 — no-arg 생성자와 open 클래스

3장의 `Member` 클래스에는 인자 없는 생성자가 없고, Kotlin 클래스는 기본이 `final`이다. JPA와 Hibernate 입장에서는 둘 다 문제다.

| 요구 | 이유 | Kotlin에서 해결 |
|---|---|---|
| **인자 없는 생성자** | Hibernate 문서: "엔티티 클래스는 인자 없는 생성자를 가져야 한다. Hibernate와 Jakarta Persistence 둘 다 요구한다." DB 행을 객체로 만들 때 일단 빈 객체를 만들고 필드를 채우기 때문 | `kotlin("plugin.jpa")` — `@Entity`·`@Embeddable`·`@MappedSuperclass` 클래스에 합성 no-arg 생성자를 만들어 준다. 코드에서는 안 보이고 리플렉션으로만 호출된다 |
| **`final`이 아닌 클래스** | Hibernate 문서: "지연 로딩은 런타임 프록시에 의존하며, 엔티티 클래스가 non-final이어야 한다. final 클래스도 저장은 되지만 **지연 연관을 프록시로 가져올 수 없다.**" 프록시는 엔티티를 **상속**해서 만드는데 `final`은 상속이 안 되기 때문 | `allOpen` 플러그인에 JPA 어노테이션을 등록 — 해당 클래스와 멤버를 컴파일 시 `open`으로 바꿔 준다 |

Spring Initializr(Kotlin + Spring Data JPA)가 생성하는 `build.gradle.kts`가 정확히 이 두 가지를 넣어 준다.

```kotlin
plugins {
    kotlin("jvm") version "2.x"
    kotlin("plugin.spring") version "2.x"   // @Component, @Transactional 등을 open
    kotlin("plugin.jpa") version "2.x"      // 엔티티에 no-arg 생성자
    // ...
}

allOpen {
    annotation("jakarta.persistence.Entity")
    annotation("jakarta.persistence.MappedSuperclass")
    annotation("jakarta.persistence.Embeddable")
}
```

`allOpen` 블록이 빠지면 어떻게 되나. 컴파일도 되고 기동도 된다. 그런데 `@ManyToOne(fetch = LAZY)`라고 써도 프록시를 못 만드니 **사실상 LAZY가 안 된다.** "LAZY로 했는데 왜 팀이 같이 조회되지?"의 원인 중 하나가 이것이다. `kotlin("plugin.jpa")`는 **no-arg 생성자만** 담당하고 `open`은 만들지 않으므로, `allOpen` 블록은 별도로 있어야 한다.

`data class`를 엔티티로 쓰는 것도 피하는 게 보통이다. `equals`/`hashCode`가 모든 프로퍼티를 쓰게 되어 LAZY 프록시와 컬렉션에서 예상 밖 동작이 생기고, `copy()`가 id까지 복사한다. 이 문서 예제처럼 일반 `class`를 쓴다.

## A.2 Fetch Join의 두 가지 함정 — 페이징과 컬렉션 둘

5장·6장의 방법은 `@ManyToOne`(회원 → 팀)처럼 **행 수가 안 늘어나는 연관**에서는 안전하다. `@OneToMany`(팀 → 회원들)처럼 **행 수가 늘어나는 컬렉션**을 fetch할 때 두 가지 함정이 있다.

**함정 1 — 페이징이 메모리에서 된다.**

```kotlin
@Query("SELECT t FROM Team t JOIN FETCH t.members")
fun findAllWithMembers(pageable: Pageable): Page<Team>
```

팀 1개에 회원 100명이면 JOIN 결과는 100행이다. SQL의 `LIMIT 10`은 **행 10개**를 자르므로 팀 1개의 회원 10명만 돌아와 팀 데이터가 깨진다. 그래서 Hibernate는 컬렉션 fetch join에 페이징이 걸리면 SQL에 LIMIT을 넣지 않고 **전체를 다 읽은 뒤 메모리에서 자른다.** 이때 다음 경고를 남긴다.

```text
HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory
```

Hibernate 쿼리 언어 문서도 "fetch join은 보통 limit이나 페이징이 있는 쿼리에서는 피해야 한다"고 적는다. 데이터가 커지면 페이징의 의미가 없어지고 메모리가 터진다. 이 경우의 정답이 7장의 Batch Fetching이다. 부모(팀)는 DB에서 정상 페이징하고, 컬렉션(회원들)은 나중에 `IN`으로 묶어 가져온다.

**함정 2 — 컬렉션 둘을 동시에 fetch하면 카테시안 곱.**

팀에 `members`(100명)와 `projects`(20개)가 있을 때 둘 다 `JOIN FETCH`하면 JOIN 결과는 100 × 20 = 2,000행이다. 데이터가 중복되어 부풀고, Hibernate는 두 컬렉션이 모두 `List`(bag)면 아예 `MultipleBagFetchException`("쿼리가 여러 bag을 동시에 fetch하려 한다")으로 막는다. 문서도 to-many 연관 여러 개를 병렬로 fetch하면 카테시안 곱이 되어 성능 위험이 있다고 경고한다. 해결은 **컬렉션은 하나만 fetch join하고 나머지는 Batch Fetching**에 맡기는 것이다.

정리:

```text
@ManyToOne / @OneToOne 연관   → Fetch Join / EntityGraph 로 첫 쿼리에
@OneToMany 컬렉션 + 페이징    → Batch Fetching (default_batch_fetch_size)
@OneToMany 컬렉션 여러 개     → 하나만 Fetch Join, 나머지는 Batch Fetching
```

## A.3 EntityGraph의 FETCH와 LOAD 타입

`@EntityGraph(type = ...)`의 기본값은 `EntityGraphType.FETCH`다. 두 타입은 **그래프에 적지 않은 나머지 연관**을 어떻게 다루느냐가 다르다. Hibernate 문서 기준:

| 타입 | 그래프에 적은 연관 | 적지 않은 연관 |
|---|---|---|
| **FETCH** (기본) | EAGER로 로딩 | **전부 LAZY** 취급 — 매핑에 EAGER라고 되어 있어도 이번 조회에선 미룸 |
| **LOAD** | EAGER로 로딩 | 매핑의 정적 설정을 그대로 따름 (EAGER면 EAGER) |

모든 연관을 LAZY로 두는 이 문서의 방침을 따르면 둘의 차이는 거의 없다. 레거시 코드에 EAGER 매핑이 섞여 있을 때만 "FETCH는 그것까지 잠시 꺼 준다"는 점이 의미가 있다.

## A.4 SQL 로그로 직접 확인하기

이 문서의 모든 "쿼리 N번"은 로그로 세어 봐야 몸에 붙는다. 로컬 프로필에서만 켠다.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        format_sql: true
logging:
  level:
    org.hibernate.SQL: debug                                  # 실행되는 SQL 문장
    org.hibernate.orm.jdbc.bind: trace                        # 바인딩된 파라미터 값
```

확인 순서는 이렇다.

1. `findAll()` + `forEach { it.team.name }` 을 실행하고 `select ... from team` 이 몇 번 찍히는지 센다. → N+1 확인
2. 같은 코드를 `findAllWithTeam()`(Fetch Join)으로 바꾸고 다시 센다. → `join team` 이 들어간 문장 1번
3. `default_batch_fetch_size: 100` 을 켜고 원래 `findAll()` 로 돌아가 센다. → `where id in (?, ?, ...)` 문장 몇 번
4. `changeName()` 을 실행하고 메서드 끝에서 `update member` 가 찍히는지 본다. → Dirty Checking 확인. `@Transactional`을 빼고 다시 실행해 안 찍히는 것도 본다.

쿼리 수를 숫자로 세고 싶으면 Hibernate 통계를 켠다. 켜는 법과 지표 읽는 법, 켤 때의 부작용은 자매 문서 [8장](jpa-lazy-loading-and-persistence-context.md#8-눈으로-확인하는-법--통계와-sql-로그)과 [부록 A.4](jpa-lazy-loading-and-persistence-context.md#a4-통계-지표-읽는-법과-켤-때의-부작용)에 있다.

## A.5 참고한 공식 문서

- Hibernate ORM 6.6 User Guide — Fetching(배치 로딩, 기본 fetch 전략과 LAZY 권장), Flushing(AUTO flush 시점), Entity(non-final 클래스, no-arg 생성자)
- Hibernate ORM 6.6 Query Language Guide — HQL과 JPQL의 관계, 엔티티·속성 참조, fetch join과 페이징
- Hibernate ORM 6.6 Javadoc — `FetchSettings.DEFAULT_BATCH_FETCH_SIZE`, `BATCH_FETCH_STYLE` deprecated, `MultipleBagFetchException`, `QueryLogging` HHH90003004
- Spring Data JPA Reference — Core concepts(`CrudRepository` 메서드), Query Methods(`@Query`, `@EntityGraph`), `EntityGraph` Javadoc(`type` 기본값 FETCH)
- Kotlin 문서 — no-arg 컴파일러 플러그인(`kotlin-jpa`), all-open 컴파일러 플러그인
- Spring Initializr가 생성한 Kotlin + Data JPA `build.gradle.kts` (2026-09-20 기준)
