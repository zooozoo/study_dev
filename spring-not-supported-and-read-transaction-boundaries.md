# 출석 API 커넥션 고갈로 배우는 NOT_SUPPORTED와 조회 트랜잭션 경계

트랜잭션을 일부러 끈 메서드(`NOT_SUPPORTED`) 안에서 조회를 하면, 그 조회가 잡은 DB 커넥션이 메서드가 끝날 때까지 남을 수 있다.
이 문서는 그 이유를 Spring·Hibernate 소스로 따라가고, 조회마다 짧은 트랜잭션을 씌워 커넥션을 바로 돌려주는 방법을 정리한다.

- 작성일: 2026-11-17
- 계기: 업무 티켓 MORACARE-3196 (홈 출석 API 커넥션 중첩 점유 개선)
- 확인 버전: Spring Boot 3.5.6 · Spring Framework 6.2.11 · Hibernate ORM 6.6.29
- 함께 읽기: 전파·프록시·커넥션 풀 기초는 [Spring 트랜잭션 경계 노트](spring-transaction-boundaries-and-batch.md)의 2장·3장·부록 A.1에 있다.

## 0. 이번 사건 한 문단 요약

"오늘 출석"을 기록하고 이번 주 출석 현황을 돌려주는 API가 있다. 이 API는 먼저 회원의 이용권을 조회하고, 같은 회원의 동시 요청을 막는 분산락을 잡은 뒤 출석을 저장한다. 커밋을 락 해제보다 먼저 끝내려고, 바깥 메서드는 `@Transactional(propagation = NOT_SUPPORTED)`로 트랜잭션을 끄고 저장만 안쪽 `@Transactional`로 감쌌다. 그런데 바깥에서 트랜잭션 없이 실행한 이용권 조회가 커넥션 하나를 **메서드가 끝날 때까지** 쥐고 있었다. 저장을 시작하면 커넥션을 하나 더 잡는다. 그래서 요청 하나가 커넥션 두 개를 썼고, 동시 요청이 몰리자 커넥션 풀이 바닥나 30초 대기 뒤 HTTP 500이 났다. 해결은 조회 메서드에 짧은 `@Transactional`을 씌워 "조회가 끝나면 커넥션 반납"이 되게 만드는 것이었다.

> 이 문단에 모르는 단어(전파, `NOT_SUPPORTED`, EntityManager, 커넥션 반납 시점, 읽기 전용 라우팅)가 있으면 이 문서가 도움이 된다.

## 핵심 요약

1. `NOT_SUPPORTED`는 "이 메서드는 트랜잭션 없이 실행한다"는 전파 방식이다. 바깥에 트랜잭션이 있으면 잠시 멈춰 둔다. → [2장](#2-not_supported는-트랜잭션-없이-실행하라는-전파-방식이다)
2. 트랜잭션이 없어도 Spring은 그 메서드 범위 동안 쓸 EntityManager(JPA에서 DB 작업을 맡는 객체)를 하나 열어 두고, 메서드가 끝날 때 닫는다. → [3장](#3-트랜잭션이-없어도-entitymanager는-메서드-범위로-열린다)
3. Spring은 Hibernate의 커넥션 처리 방식을 "처음 쓸 때 잡고, 세션을 닫을 때까지 쥔다"로 정한다. 그래서 트랜잭션 없는 조회가 잡은 커넥션은 메서드 끝까지 남는다. → [4장](#4-커넥션은-첫-쿼리에서-잡히고-entitymanager가-닫힐-때-풀린다)
4. 그 상태에서 안쪽 `@Transactional`이 시작되면 새 EntityManager와 새 커넥션을 쓴다. 요청 하나가 커넥션 두 개를 쥔다. → [5장](#5-요청-하나가-커넥션-두-개를-쥐는-순간)
5. 모든 요청이 첫 커넥션을 쥔 채 두 번째를 기다리면 풀 안에서 서로를 기다리는 교착이 생긴다. 타임아웃까지 아무도 진행하지 못한다. → [5.2절](#52-풀이-작으면-요청끼리-서로를-기다리다-멈춘다)
6. 바깥 메서드를 트랜잭션으로 감싸면 "락 해제 → 커밋" 순서가 되어 중복 저장이 생길 수 있다. 그래서 `NOT_SUPPORTED`를 쓴 설계 자체는 맞다. → [6장](#6-바깥을-트랜잭션으로-묶으면-안-되는-이유--커밋과-락-해제-순서)
7. 해법은 바깥은 그대로 두고, 그 안의 조회 메서드마다 짧은 `@Transactional`을 씌우는 것이다. 조회가 끝나면 커밋과 함께 커넥션이 풀로 돌아간다. → [7장](#7-해법--조회마다-짧은-트랜잭션을-씌운다)
8. `readOnly = true`를 붙여도 반납 시점은 같다. 다만 읽기 전용 DB로 라우팅될 수 있다. 쓰기 대상을 정하는 조회는 복제 지연 때문에 원본 DB를 써야 한다. → [8장](#8-readonly를-붙일지는-어느-db를-읽어야-하는지로-정한다)
9. 이 경계는 Hikari의 "사용 중 커넥션 수"를 직접 세는 테스트로 확인할 수 있다. → [9장](#9-커넥션-반납을-테스트로-확인하는-법)

## 면접 질문 체크리스트

- [ ] `REQUIRED`·`REQUIRES_NEW`·`NOT_SUPPORTED`는 각각 무엇이 다른가? → [2장](#2-not_supported는-트랜잭션-없이-실행하라는-전파-방식이다)
- [ ] 트랜잭션이 없는 메서드에서 JPA 조회를 하면 커넥션은 언제 풀리나? → [3장](#3-트랜잭션이-없어도-entitymanager는-메서드-범위로-열린다), [4장](#4-커넥션은-첫-쿼리에서-잡히고-entitymanager가-닫힐-때-풀린다)
- [ ] 커넥션 풀 크기가 충분해 보여도 풀이 바닥나는 구조는 무엇인가? → [5장](#5-요청-하나가-커넥션-두-개를-쥐는-순간)
- [ ] 분산락과 트랜잭션을 함께 쓸 때 커밋과 락 해제는 어떤 순서여야 하나? 이유는? → [6장](#6-바깥을-트랜잭션으로-묶으면-안-되는-이유--커밋과-락-해제-순서)
- [ ] 조회 메서드에 `@Transactional`을 붙이면 커넥션 점유가 왜 줄어드나? → [7장](#7-해법--조회마다-짧은-트랜잭션을-씌운다)
- [ ] `readOnly = true`를 붙이면 안 되는 조회는 어떤 조회인가? → [8장](#8-readonly를-붙일지는-어느-db를-읽어야-하는지로-정한다)
- [ ] 트랜잭션 경계 변경을 목(mock) 단위 테스트로 검증하면 왜 부족한가? → [9장](#9-커넥션-반납을-테스트로-확인하는-법)

## 1. 내가 물어본 것들 → 어디를 보면 되나

| 질문 | 볼 곳 |
|---|---|
| `NOT_SUPPORTED`에서 트랜잭션은 어떻게 할당되나 | [2장](#2-not_supported는-트랜잭션-없이-실행하라는-전파-방식이다), [3장](#3-트랜잭션이-없어도-entitymanager는-메서드-범위로-열린다) |
| 트랜잭션이 없는데 왜 커넥션을 계속 쥐고 있나 | [4장](#4-커넥션은-첫-쿼리에서-잡히고-entitymanager가-닫힐-때-풀린다) |
| 왜 하필 커넥션 "두 개"인가 | [5장](#5-요청-하나가-커넥션-두-개를-쥐는-순간) |
| 그럼 그냥 메서드 전체에 `@Transactional`을 붙이면 안 되나 | [6장](#6-바깥을-트랜잭션으로-묶으면-안-되는-이유--커밋과-락-해제-순서) |
| `@Transactional(readOnly = true)`를 붙이면 트랜잭션이 어떻게 시작하고 반납되나 | [7장](#7-해법--조회마다-짧은-트랜잭션을-씌운다), [8장](#8-readonly를-붙일지는-어느-db를-읽어야-하는지로-정한다) |
| 실제로 반납되는지 어떻게 확인하나 | [9장](#9-커넥션-반납을-테스트로-확인하는-법) |
| 이 설명의 근거가 되는 Spring 소스는 어디인가 | [부록 A](#부록-a-spring-소스로-보는-근거) |

---

## 2. NOT_SUPPORTED는 트랜잭션 없이 실행하라는 전파 방식이다

**전파(propagation)** 는 "이 메서드가 불릴 때 이미 트랜잭션이 있으면 어떻게 할까"를 정하는 규칙이다. 자세한 설명은 [Spring 트랜잭션 경계 노트 3장](spring-transaction-boundaries-and-batch.md#3-transactional의-실체--프록시와-전파)에 있다. 여기서는 이 문서에 필요한 세 가지만 비교한다.

| 전파 방식 | 바깥에 트랜잭션이 있을 때 | 바깥에 트랜잭션이 없을 때 |
|---|---|---|
| `REQUIRED` (기본값) | 그 트랜잭션에 참여한다 | 새로 연다 |
| `REQUIRES_NEW` | 바깥 것을 멈춰 두고 새로 연다 | 새로 연다 |
| `NOT_SUPPORTED` | 바깥 것을 멈춰 두고 **트랜잭션 없이** 실행한다 | 트랜잭션 없이 실행한다 |

Spring의 `Propagation.NOT_SUPPORTED` 주석도 같은 뜻이다. 트랜잭션 없이 실행하고, 현재 트랜잭션이 있으면 멈춰 둔다(suspend).

식당에 비유하면 이렇다.

- DB 커넥션은 **테이블**이다. 커넥션 풀은 테이블이 정해진 수만큼 있는 식당이다.
- 트랜잭션은 **테이블 예약 기간**이다. 시작하면 테이블을 잡고, 커밋이나 롤백으로 끝나면 테이블을 돌려준다.
- `NOT_SUPPORTED`는 "이 메서드는 예약 없이 진행"이라는 표시다.

예약이 없으면 테이블을 돌려줄 정해진 시점도 없다. 그러면 테이블은 언제 돌려줄까? 이것이 3장과 4장의 질문이다.

> **왜 실전에서 중요한가.** `NOT_SUPPORTED`는 "트랜잭션 비용을 아끼는 설정"으로 오해하기 쉽다. 하지만 이 설정은 **커넥션을 언제 돌려줄지 정하는 경계도 함께 없앤다.** 커넥션이 언제 풀리는지는 다른 규칙(3장, 4장)이 정한다.

## 3. 트랜잭션이 없어도 EntityManager는 메서드 범위로 열린다

**EntityManager**는 JPA에서 엔티티를 읽고 쓰는 작업을 맡는 객체다. Hibernate에서는 이것이 곧 **세션(Session)** 이다. 보통은 트랜잭션 하나에 EntityManager 하나가 붙는다.

그런데 Spring은 "빈(empty) 트랜잭션"에서도 EntityManager를 열어 둔다. 빈 트랜잭션은 `NOT_SUPPORTED`·`SUPPORTS`처럼 진짜 DB 트랜잭션은 없지만 Spring이 범위만 관리하는 구간이다.

순서대로 보면 이렇다.

1. Spring 트랜잭션 매니저의 기본 동기화 설정은 `SYNCHRONIZATION_ALWAYS`다. 소스 주석은 이 설정이 `NOT_SUPPORTED` 같은 빈 트랜잭션에서도 **트랜잭션 동기화**(트랜잭션이 끝날 때 할 일을 등록해 두는 장치)를 켠다고 설명한다.
2. 그 구간에서 코드가 EntityManager로 쿼리를 보내면 Spring(`EntityManagerFactoryUtils`)이 EntityManager를 새로 만든다.
3. Spring은 그 EntityManager를 현재 스레드에 묶고, "범위가 끝나면 닫기"를 동기화에 등록한다.
4. 그 구간 안의 다른 조회들도 같은 EntityManager를 다시 쓴다.
5. `NOT_SUPPORTED` 메서드가 끝나면 동기화가 실행되어 EntityManager가 닫힌다.

```
notSupportedMethod() 시작         ← 빈 트랜잭션 범위 시작, 동기화 켜짐
  ├ 첫 조회                        ← EntityManager 생성 → 스레드에 묶임
  ├ 두 번째 조회                   ← 같은 EntityManager 재사용
  ...
notSupportedMethod() 끝           ← 여기서 EntityManager 닫힘
```

> **왜 실전에서 중요한가.** "트랜잭션이 없으니 아무것도 붙잡지 않는다"는 직관은 틀렸다. **EntityManager는 메서드가 끝날 때까지 산다.** 다음 4장에서 보듯이 커넥션도 이 EntityManager를 따라간다.

## 4. 커넥션은 첫 쿼리에서 잡히고 EntityManager가 닫힐 때 풀린다

커넥션의 수명은 두 장치가 정한다.

### 4.1 잡는 시점 — 첫 쿼리를 보낼 때 잡는다

- **Hibernate의 DELAYED_ACQUISITION**: EntityManager를 만들었다고 바로 커넥션을 잡지 않는다. 첫 SQL을 보낼 때 잡는다.
- **Spring `LazyConnectionDataSourceProxy`**: 읽기 DB와 쓰기 DB를 나누는 구성에서 자주 쓴다. 이 프록시는 실제 JDBC 커넥션을 **첫 Statement를 만들 때** 풀에서 꺼낸다. 덕분에 "어느 DB로 갈지"를 트랜잭션 설정(예: readOnly)을 다 본 뒤에 고를 수 있다. 8장의 라우팅이 이 덕분에 동작한다.

### 4.2 놓는 시점 — 세션을 닫을 때까지 쥔다

Hibernate는 커넥션을 언제 놓을지 정하는 설정(`hibernate.connection.handling_mode`)이 있다.

| 모드 | 커넥션을 놓는 시점 |
|---|---|
| `DELAYED_ACQUISITION_AND_RELEASE_AFTER_STATEMENT` | SQL 한 문장이 끝날 때마다 |
| `DELAYED_ACQUISITION_AND_RELEASE_AFTER_TRANSACTION` | 트랜잭션이 끝날 때 |
| `DELAYED_ACQUISITION_AND_HOLD` | 세션(EntityManager)을 닫을 때 |

Spring의 `HibernateJpaVendorAdapter`는 따로 설정하지 않으면 **`DELAYED_ACQUISITION_AND_HOLD`를 강제로 넣는다.** Spring Boot는 이 어댑터를 쓴다. 근거 소스는 [부록 A](#부록-a-spring-소스로-보는-근거)에 있다.

3장과 합치면 결론이 나온다.

```
NOT_SUPPORTED 메서드
  ├ 첫 조회 → EntityManager 생성 → 첫 SQL에서 커넥션 A 획득
  │          (HOLD 모드: 세션이 닫힐 때까지 A를 놓지 않음)
  ├ ... 무엇을 하든 A는 계속 잡혀 있음 ...
  └ 메서드 끝 → EntityManager 닫힘 → 이때 A 반납
```

> **왜 실전에서 중요한가.** 트랜잭션이 있는 메서드에서는 "커밋 = 커넥션 반납"이라 경계가 눈에 보인다. 트랜잭션이 없는 구간에서는 **반납 시점이 바깥 메서드의 끝으로 밀린다.** 그 사이에 락 대기나 외부 호출이 끼면 커넥션을 쓰지도 않으면서 붙잡고 있게 된다.

참고로 `open-in-view`(HTTP 요청 전체 동안 EntityManager를 열어 두는 설정)를 켜 두면 수명이 요청 끝까지 더 늘어난다. 이 사건의 서비스는 꺼 두었기 때문에 수명이 "`NOT_SUPPORTED` 메서드 끝"까지였다. 이 설정은 [JPA 지연 로딩 노트 부록 A.3](jpa-lazy-loading-and-persistence-context.md#a3-open-in-view--지연-로딩의-수명을-http-요청까지-늘리는-스위치)에 정리돼 있다.

## 5. 요청 하나가 커넥션 두 개를 쥐는 순간

### 5.1 예제 — 출석 기록 API

중립적인 예제로 구조만 옮긴다.

```kotlin
@Service
class AttendanceService(
    private val subscriptionReader: SubscriptionReader,   // 조회 전용
    private val attendanceWriter: AttendanceWriter,       // @Transactional 쓰기
    private val lock: DistributedLock,
) {
    @Transactional(propagation = Propagation.NOT_SUPPORTED)
    fun checkIn(memberId: Long): WeeklyAttendance {
        val plan = subscriptionReader.findPlan(memberId)          // ① 조회 (트랜잭션 없음)
        lock.withLock("attendance:$memberId") {                   // ② 락 대기
            attendanceWriter.markToday(memberId, plan)            // ③ 쓰기 트랜잭션
        }
        return subscriptionReader.findWeekly(memberId)            // ④ 조회
    }
}

@Component
class AttendanceWriter(private val repository: AttendanceRepository) {
    @Transactional
    fun markToday(memberId: Long, plan: Plan) { /* upsert */ }
}
```

시간 순서로 커넥션을 세면 이렇다.

```
checkIn() 시작 (NOT_SUPPORTED, 빈 트랜잭션 범위)
 │
 ├ ① findPlan          → 범위용 EntityManager 생성 → 커넥션 A 획득 (HOLD)
 │
 ├ ② 락 대기            → A를 쥔 채 기다림                  [사용 중: 1]
 │
 ├ ③ markToday 시작     → 진짜 트랜잭션 시작
 │                        → 새 EntityManager → 커넥션 B 획득 [사용 중: 2]
 │   저장 → 커밋        → B 반납                           [사용 중: 1]
 │   락 해제
 │
 ├ ④ findWeekly        → 범위용 EntityManager(A) 재사용
 └ checkIn() 끝        → EntityManager 닫힘 → A 반납       [사용 중: 0]
```

③에서 왜 A를 재사용하지 않고 B를 새로 잡을까? `JpaTransactionManager`는 트랜잭션을 시작할 때 스레드에 묶인 EntityManager를 찾는다. 그 EntityManager가 이미 다른 범위의 동기화에 묶여 있으면 **새 EntityManager를 만든다.** 3장의 범위용 EntityManager가 바로 그런 경우다. 그래서 A는 그대로 두고 B를 새로 연다. 근거는 [부록 A](#부록-a-spring-소스로-보는-근거)에 있다.

### 5.2 풀이 작으면 요청끼리 서로를 기다리다 멈춘다

커넥션 풀 크기가 4개라고 해 보자. 서로 다른 회원 4명이 동시에 요청하면 이렇게 된다.

```
요청 1: A1 잡음 → B 기다림 ─┐
요청 2: A2 잡음 → B 기다림  │  풀 4개가 전부 A로 잡혀 있음
요청 3: A3 잡음 → B 기다림  │  아무도 B를 얻지 못함
요청 4: A4 잡음 → B 기다림 ─┘  → 커넥션 대기 타임아웃까지 전원 정지
```

모두가 "하나를 쥔 채 하나를 더 기다린다." 누구도 먼저 놓지 않으니 풀 안에서 교착이 생긴다. 커넥션 대기 타임아웃(사건에서는 30초)이 지나면 요청이 실패한다. 실제 사건에서는 다른 기능의 긴 락 대기도 겹쳐 풀이 더 빨리 말랐다.

> **왜 실전에서 중요한가.** 요청 하나가 커넥션을 **두 개** 필요로 하면 풀 크기를 늘려도 안전하지 않다. 동시 요청이 "풀 크기 이상"이면 언제든 같은 교착이 재현된다. 풀 크기 문제가 아니라 **요청당 최대 점유 수** 문제로 봐야 한다. 커넥션 풀이 마르는 일반적인 과정은 [Spring 트랜잭션 경계 노트 부록 A.1](spring-transaction-boundaries-and-batch.md#a1-커넥션-풀이-마른다는-것)에 있다.

## 6. 바깥을 트랜잭션으로 묶으면 안 되는 이유 — 커밋과 락 해제 순서

"그럼 `checkIn()` 전체에 `@Transactional`을 붙이면 커넥션 하나로 끝나지 않나?" 맞다. 하지만 다른 문제가 생긴다.

`REQUIRED`라면 안쪽 `markToday()`는 바깥 트랜잭션에 **참여**만 한다. 참여자는 커밋하지 않는다. 실제 커밋은 바깥 메서드가 끝날 때 한 번 일어난다([Spring 트랜잭션 경계 노트 3.4절](spring-transaction-boundaries-and-batch.md#34-참여자는-커밋하지-않는다--실제-커밋은-바깥-하나뿐)). 그러면 순서가 이렇게 바뀐다.

| 바깥이 `NOT_SUPPORTED` (의도한 순서) | 바깥이 `@Transactional` (깨진 순서) |
|---|---|
| 1. 락 획득 | 1. 락 획득 |
| 2. 저장 | 2. 저장 (아직 커밋 안 됨) |
| 3. **커밋** | 3. **락 해제** |
| 4. **락 해제** | 4. ④ 조회 등 나머지 실행 |
| | 5. 메서드 끝에서 **커밋** |

깨진 순서에서는 3번과 5번 사이에 같은 회원의 다른 요청이 락을 잡을 수 있다. 그 요청은 아직 커밋되지 않은 "오늘 출석"을 보지 못한다. 그래서 다시 INSERT하고, 유일 키 충돌이나 중복 저장이 생긴다.

> **왜 실전에서 중요한가.** 락은 "커밋된 결과를 다음 사람이 보게 하는 것"까지 보장해야 의미가 있다. **커밋이 락 해제보다 먼저** 끝나야 한다. 바깥을 `NOT_SUPPORTED`로 비운 것은 이 순서를 지키려는 올바른 선택이다. 문제는 그 안의 조회였다.

## 7. 해법 — 조회마다 짧은 트랜잭션을 씌운다

바깥 `NOT_SUPPORTED`와 쓰기 트랜잭션은 그대로 둔다. 조회 메서드에만 `@Transactional`을 붙인다.

```kotlin
@Component
@Transactional            // 조회마다 짧은 트랜잭션 (readOnly 여부는 8장)
class SubscriptionReader(private val queryFactory: JPAQueryFactory) {
    fun findPlan(memberId: Long): Plan? = /* 조회 */
    fun findWeekly(memberId: Long): WeeklyAttendance = /* 조회 */
}
```

이제 `findPlan()` 하나를 부를 때 일어나는 일은 이렇다.

1. Spring 프록시가 호출을 가로채 트랜잭션을 시작한다. 바깥은 `NOT_SUPPORTED`라 참여할 트랜잭션이 없으므로 새로 연다.
2. 이 트랜잭션 전용 EntityManager가 생긴다.
3. 첫 SQL에서 커넥션을 잡는다(4.1절).
4. 조회를 실행한다.
5. 메서드가 리턴하면 프록시가 커밋한다.
6. 트랜잭션이 끝나면서 EntityManager가 닫히고 커넥션이 풀로 돌아간다.

```
checkIn() 시작 (NOT_SUPPORTED)
 ├ findPlan()   → 트랜잭션 시작 → A 획득 → 조회 → 커밋 → A 반납   [최대 1]
 ├ 락 대기       → 쥔 커넥션 없음                                  [0]
 ├ markToday()  → B 획득 → 저장 → 커밋 → B 반납 → 락 해제          [최대 1]
 ├ findWeekly() → C 획득 → 조회 → 커밋 → C 반납                   [최대 1]
 └ checkIn() 끝
```

어느 순간에도 커넥션은 **최대 1개**다. 핵심은 `readOnly` 플래그가 아니라 **"트랜잭션 경계 = 커넥션 반납 시점"을 메서드 단위로 명확히 만든 것**이다.

주의할 점이 두 가지 있다.

- **범위 안의 조회를 하나도 빠뜨리면 안 된다.** 트랜잭션 없이 실행되는 조회가 하나라도 있으면 3장의 범위용 EntityManager가 생기고, 그 커넥션은 다시 메서드 끝까지 남는다. 바깥 메서드에서 부르는 모든 조회 경로를 확인해야 한다.
- **프록시를 거쳐야 한다.** 같은 클래스 안에서 `this.findPlan()`처럼 부르면 프록시를 지나지 않아 트랜잭션이 열리지 않는다([Spring 트랜잭션 경계 노트 부록 A.3](spring-transaction-boundaries-and-batch.md#a3-self-invocation--같은-클래스-안에서-부르면-왜-안-먹나)).

> **왜 실전에서 중요한가.** "조회에 트랜잭션은 필요 없다"는 말은 데이터 일관성만 보면 맞을 때가 많다. 하지만 **자원 관리** 관점에서는 트랜잭션이 커넥션 반납 시점을 정해 주는 장치다. 트랜잭션을 꺼 둔 구간 안에서는 오히려 조회마다 트랜잭션을 씌워야 자원이 빨리 돌아온다.

## 8. readOnly를 붙일지는 어느 DB를 읽어야 하는지로 정한다

`@Transactional(readOnly = true)`와 `@Transactional`은 **커넥션 반납 시점이 같다.** 둘 다 메서드가 끝날 때 커밋하고 반납한다. 차이는 다음과 같다.

| | 커넥션 반납 | 읽기·쓰기 분리 구성에서 가는 DB | Hibernate flush |
|---|---|---|---|
| `@Transactional(readOnly = true)` | 메서드 끝 | 읽기 복제본(replica) | 하지 않음 |
| `@Transactional` | 메서드 끝 | 원본(primary) | 커밋 때 함 |

읽기·쓰기 분리 구성은 보통 이렇게 라우팅한다. 이 방식은 [Spring 트랜잭션 경계 노트 8.1절](spring-transaction-boundaries-and-batch.md#81-읽기-전용-라우팅이-갑자기-동작하기-시작한다)에 있다.

```kotlin
class ReplicationRoutingDataSource : AbstractRoutingDataSource() {
    override fun determineCurrentLookupKey() =
        if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) "replica" else "primary"
}
// LazyConnectionDataSourceProxy(routingDataSource)로 감싸 첫 쿼리 때 고르게 한다
```

복제본은 원본을 **조금 늦게** 따라온다. 이것을 **복제 지연(replication lag)** 이라고 한다. 예를 들면 이렇다.

1. 회원이 방금 유료 이용권을 결제했다. 원본에는 반영됐다.
2. 복제본에는 아직 반영되지 않았다.
3. `findPlan()`이 복제본을 읽고 "무료 플랜"이라고 판단한다.
4. 출석이 **잘못된 플랜으로 저장된다.**

그래서 기준은 이렇다.

- **화면에 보여 주기만 하는 조회**: `readOnly = true`로 복제본을 읽어도 된다. 잠깐 옛값이 보여도 다음 조회에서 맞춰진다.
- **쓰기 대상을 정하는 조회**: 원본을 읽어야 한다. 이 사건의 해결에서도 처음에 붙인 `readOnly = true`를 일반 `@Transactional`로 바꿨다.

> **왜 실전에서 중요한가.** `readOnly = true`는 "읽기만 하니까 붙이는 무해한 표시"가 아니다. 라우팅 구성이 있으면 **읽는 DB가 바뀐다.** 붙이기 전에 "이 조회 결과로 무엇을 쓰는가"를 먼저 묻는다.

## 9. 커넥션 반납을 테스트로 확인하는 법

트랜잭션 경계는 Spring 프록시와 실제 JPA가 있어야 생긴다. 목(mock) 단위 테스트는 프록시도 EntityManager도 없으므로 반납 여부를 판단할 수 없다.

작은 Hikari 풀과 메모리 DB(H2)로 실제 경계를 만들고, `HikariPoolMXBean.activeConnections`(지금 빌려 간 커넥션 수)를 세면 된다.

```kotlin
// 준비: maximumPoolSize = 2 인 HikariDataSource, JpaTransactionManager,
//       TransactionInterceptor로 감싼 reader 프록시
TransactionTemplate(txManager).apply {
    propagationBehavior = TransactionDefinition.PROPAGATION_NOT_SUPPORTED
}.executeWithoutResult {
    reader.findPlan(1L)                                       // 조회 실행
    assertEquals(0, pool.hikariPoolMXBean.activeConnections)  // 조회 직후 반납됐나

    TransactionTemplate(txManager).executeWithoutResult {     // 쓰기 트랜잭션
        em.createNativeQuery("SELECT 1").singleResult
        assertEquals(1, pool.hikariPoolMXBean.activeConnections) // 두 개가 아니라 하나인가
    }
}
assertEquals(0, pool.hikariPoolMXBean.activeConnections)
```

이 테스트는 판별력이 있다. `reader`에서 `@Transactional`을 지우면 조회 직후 값이 1이 되고, 쓰기 안에서는 2가 되어 실패한다. 조회 안에서 `TransactionSynchronizationManager.isCurrentTransactionReadOnly()`를 확인하면 8장의 라우팅 결정도 같이 고정할 수 있다.

실제 DB로 확인하려면 풀을 작게(예: 2~4개) 잡은 로컬 환경에서 변경 전후로 동시 요청을 보낸다. Hikari의 사용 중·대기 수와 타임아웃 발생 여부를 비교한다. 경계 테스트의 일반적인 방법은 [Spring 트랜잭션 경계 노트 10장](spring-transaction-boundaries-and-batch.md#10-트랜잭션-경계를-테스트로-잡는-법)에 있다.

---

## 부록 A. Spring 소스로 보는 근거

Spring Framework 6.2.11 기준이다(Spring Boot 3.5.6이 가져오는 버전).

| 이 문서의 주장 | 소스 위치 | 내용 |
|---|---|---|
| `NOT_SUPPORTED`는 트랜잭션 없이 실행하고 바깥 것을 멈춘다 | `spring-tx` `Propagation.NOT_SUPPORTED` | "Execute non-transactionally, suspend the current transaction if one exists." |
| 빈 트랜잭션에서도 동기화가 켜진다 | `spring-tx` `AbstractPlatformTransactionManager.SYNCHRONIZATION_ALWAYS` (기본값) | 주석이 `PROPAGATION_NOT_SUPPORTED`를 포함한 빈 트랜잭션에서도 동기화를 켠다고 설명한다 |
| 빈 범위에서도 EntityManager를 만들어 스레드에 묶고 범위 끝에 정리한다 | `spring-orm` `EntityManagerFactoryUtils.doGetTransactionalEntityManager` | 동기화가 켜져 있으면 새 EntityManager를 만들고 `registerSynchronization` 후 `bindResource` 한다 |
| Spring이 HOLD 모드를 강제한다 | `spring-orm` `HibernateJpaVendorAdapter` | 기본값에서 `CONNECTION_HANDLING`에 `DELAYED_ACQUISITION_AND_HOLD`를 넣는다 |
| 이미 범위에 묶인 EntityManager가 있으면 새 트랜잭션은 새 EntityManager를 연다 | `spring-orm` `JpaTransactionManager.doBegin` | 기존 holder가 없거나 `isSynchronizedWithTransaction()`이면 `createEntityManagerForTransaction()` |
| 빈 범위에서 새 트랜잭션을 시작해도 범위용 EntityManager는 풀리지 않는다 | `spring-tx` `AbstractPlatformTransactionManager.suspend` | 진행 중인 트랜잭션 객체가 없으면(`transaction == null`) `doSuspend`를 부르지 않는다. 동기화만 멈춰 둔다 |
| 커넥션은 첫 Statement 때 꺼낸다 | `spring-jdbc` `LazyConnectionDataSourceProxy` | "fetching actual JDBC Connections lazily, i.e. not until first creation of a Statement" |

## 부록 B. 다른 해법들과 그 대가

| 해법 | 효과 | 대가 |
|---|---|---|
| 조회 메서드에 짧은 `@Transactional` (이 문서의 선택) | 조회마다 커넥션 반납, 기존 순서 유지 | 범위 안의 모든 조회 경로를 빠짐없이 감싸야 한다 |
| 바깥 메서드 전체를 `@Transactional` | 커넥션 하나로 끝난다 | 커밋이 락 해제 뒤로 밀려 중복 저장 위험(6장) |
| `hibernate.connection.handling_mode`를 `..._RELEASE_AFTER_STATEMENT`로 변경 | 트랜잭션 없는 조회도 문장마다 반납 | 애플리케이션 전체 동작이 바뀐다. 영향 범위가 너무 넓다 |
| 커넥션 풀 크기 증가 | 교착이 생기는 동시 요청 수가 늘어난다 | 요청당 두 개를 쥐는 구조는 그대로라 근본 해결이 아니다(5.2절). DB 최대 연결 수도 소모한다 |
| 조회를 락 안쪽 쓰기 트랜잭션으로 옮김 | 커넥션 하나로 끝난다 | 락을 쥐는 시간이 길어지고, 서비스 구조를 크게 바꿔야 한다 |
