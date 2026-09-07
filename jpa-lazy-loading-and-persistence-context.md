# 한 건을 고치려다 N건을 읽는 코드로 배우는 JPA 영속성 컨텍스트와 지연 로딩

ORM(Object-Relational Mapping — 객체와 관계형 DB 테이블을 자동으로 오가게 해주는 기술)을 쓰면 SQL이 코드에서 사라진다. 편한 대신, **어떤 코드 한 줄이 언제 몇 번의 쿼리를 일으키는지 눈에 안 보인다.** 이 문서는 그 보이지 않는 부분 — 영속성 컨텍스트, 1차 캐시, 지연 로딩 — 을 정리하고, 그것이 트랜잭션 경계와 어떻게 맞물려 성능 사고로 이어지는지 본다.

작성일: 2026-09-07
기준 버전: Hibernate ORM 6.6.29.Final · Spring Boot 3.5.6 · Spring Data JPA 3.5.4 (문서 안의 측정값은 모두 이 조합에서 직접 실행해 얻었다)

---

## 이 문서를 쓰게 만든 상황

배치 작업이 예약된 메시지를 수신자들에게 하나씩 보내고 있었다. 수신자가 254명. 249명까지 보낸 뒤 컨테이너가 메모리 한도에 붙은 채 22분간 아무 진행 없이 멈췄고 결국 강제 종료됐다.

범인은 **한 항목의 상태를 바꾸는 메서드**였다. 항목 하나를 `발송됨`으로 표시하려고 부모(예약) 전체를 다시 읽고, 그 부모의 자식 목록에서 원하는 항목을 찾는 코드였다. 수신자가 N명이면 이 메서드가 N번 호출되고, 매번 `1 + N`개의 엔티티가 새로 메모리에 올라온다.

더 이상한 건 **이 코드가 며칠 전까지는 아무 문제가 없었다는 점**이다. 바뀐 것은 이 메서드가 아니라 **트랜잭션 경계**였다.

> 이 문단에서 "영속성 컨텍스트", "지연 로딩", "트랜잭션 경계" 중 하나라도 흐릿하다면 이 문서가 도움이 된다.

---

## 질문 → 어디를 볼 것인가

| 질문 | 섹션 |
|---|---|
| 단건을 반환하는 메서드가 왜 N건을 읽나 | [5장](#5-핵심--한-건을-고치려는-코드가-n건을-읽는-순간) |
| 영속성 컨텍스트가 정확히 뭔가 | [2장](#2-orm은-무엇을-대신해-주나--엔티티와-영속성-컨텍스트) · [3장](#3-영속성-컨텍스트는-1차-캐시다) |
| `LAZY`인데 왜 로딩이 일어나나. 언제 발동되나 | [4장](#4-지연-로딩--프록시는-언제-진짜가-되나) |
| 코드를 안 바꿨는데 왜 갑자기 느려졌나 | [6장](#6-왜-어제까지는-문제가-아니었나--캐시의-수명은-트랜잭션의-수명이다) |
| 이게 N+1 문제인가 | [7장](#7-이게-n1-문제와-같은-것인가) |
| 실제로 몇 건을 읽는지 어떻게 확인하나 | [8장](#8-눈으로-확인하는-법--통계와-sql-로그) |
| 어떻게 고치나. 벌크 UPDATE는 왜 함부로 쓰면 안 되나 | [9장](#9-고치는-네-가지-방법과-각각의-대가) |
| 자식 테이블만 조회하려는데 자식에 부모 ID 필드가 없다 | [10장](#10-자식으로-직접-조회하기--단방향-연관의-벽과-읽기-전용-매핑) |
| 이 회귀를 테스트로 어떻게 막나 | [11장](#11-회귀를-테스트로-잡는-법) |
| `LazyInitializationException`은 왜 나나 | [부록 A.1](#a1-lazyinitializationexception--세션이-닫힌-뒤에-건드리면) |
| `fetch` 기본값이 뭐였더라 | [부록 A.2](#a2-연관관계별-fetch-기본값) |
| `open-in-view`가 뭔가 | [부록 A.3](#a3-open-in-view--지연-로딩의-수명을-http-요청까지-늘리는-스위치) |
| 통계를 켰더니 로그가 폭주한다 | [부록 A.4](#a4-통계-지표-읽는-법과-켤-때의-부작용) |

---

## 1. 이 문서가 쓰는 예제

실제 사건의 도메인 대신 **주문(Order)과 주문 항목(OrderLine)** 으로 바꿔 쓴다. 구조는 같다 — 부모 하나에 자식이 여럿이고, 자식의 상태를 하나씩 바꿔 나간다.

```kotlin
@Entity
@Table(name = "orders")
class Order(
    @Id val id: UUID,

    // 단방향 @OneToMany: 부모가 자식 테이블의 FK 컬럼(order_id)을 직접 관리한다.
    // 자식(OrderLine)에는 부모를 가리키는 필드가 없다 — 10장에서 이게 벽이 된다.
    @OneToMany(cascade = [CascadeType.ALL], orphanRemoval = true, fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)
    val lines: MutableList<OrderLine> = mutableListOf(),
)

@Entity
@Table(name = "order_lines")
class OrderLine(
    @Id val id: UUID,
) {
    @Enumerated(EnumType.STRING)
    var status: LineStatus = LineStatus.PENDING
        private set

    // 도메인 가드: 이미 처리된 항목을 두 번 처리하지 못하게 막는다.
    // 이 가드가 코드에 남아 있느냐가 9장 선택의 핵심 기준이다.
    fun markShipped() {
        if (status != LineStatus.PENDING) throw IllegalStateException("이미 처리된 항목")
        status = LineStatus.SHIPPED
    }
}
```

용어 세 개를 먼저 풀어둔다.

- **엔티티(Entity)** — DB 테이블의 한 행에 대응하는 객체. 위의 `Order`, `OrderLine`.
- **연관관계(association)** — 엔티티끼리의 참조. `Order.lines`가 그것이다.
- **FK(Foreign Key, 외래 키)** — 자식 행이 어느 부모에 속하는지 가리키는 컬럼. 여기서는 `order_lines.order_id`.

---

## 2. ORM은 무엇을 대신해 주나 — 엔티티와 영속성 컨텍스트

SQL을 직접 쓰면 이렇게 된다.

```sql
SELECT status FROM order_lines WHERE id = ?;
-- 애플리케이션에서 판단
UPDATE order_lines SET status = 'SHIPPED' WHERE id = ?;
```

JPA를 쓰면 이렇게 된다.

```kotlin
val line = repository.findById(lineId).orElseThrow()
line.markShipped()
// UPDATE 문을 쓰지 않는다
```

`UPDATE`가 어디로 갔나. **영속성 컨텍스트(persistence context)** 가 대신 써준다.

영속성 컨텍스트는 **"지금 이 작업 단위에서 내가 다루고 있는 엔티티들"을 담아두는 상자**다. Java에서는 `EntityManager` 객체가 그 상자를 들고 있다. 하는 일이 세 가지다.

1. **동일성 보장** — 같은 ID의 행을 두 번 읽으면 **같은 객체**를 돌려준다.
2. **변경 감지(dirty checking)** — 상자에 들어올 때의 값을 스냅샷으로 기억해 두고, 작업이 끝날 때 지금 값과 비교해 **달라진 필드만** `UPDATE`로 내보낸다. 위 코드에 `UPDATE`가 없는 이유가 이것이다.
3. **쓰기 지연** — `INSERT`/`UPDATE`를 모아뒀다가 flush 시점에 한꺼번에 보낸다.

**이 상자의 수명이 이 문서 전체의 열쇠다.** 상자는 트랜잭션이 시작될 때 만들어지고 끝날 때 버려진다. 6장에서 다시 나온다.

---

## 3. 영속성 컨텍스트는 1차 캐시다

동일성 보장(1번)을 뒤집어 보면 캐시다. 그래서 **1차 캐시(first-level cache)** 라고 부른다.

```kotlin
@Transactional
fun example(id: UUID) {
    val a = repository.findById(id).orElseThrow()   // SELECT 실행
    val b = repository.findById(id).orElseThrow()   // SELECT 안 함 — 상자에서 꺼냄
    println(a === b)                                 // true (동일 객체)
}
```

두 번째 `findById`는 **DB에 가지 않는다.** 상자 안에 이미 그 ID의 객체가 있으면 그것을 그대로 돌려준다.

여기에 두 가지 성질이 따라온다.

- **범위가 좁다.** 상자는 트랜잭션 하나에 하나다. 다른 트랜잭션, 다른 스레드와 공유되지 않는다. (여러 트랜잭션이 공유하는 캐시는 **2차 캐시**라고 부르는 별개 기능이고, 기본적으로 꺼져 있다.)
- **수명이 짧다.** 트랜잭션이 끝나면 통째로 버려진다. 다음 트랜잭션은 빈 상자로 시작한다.

> **왜 실전에서 중요한가.** "같은 걸 여러 번 읽는 코드"가 캐시 히트라서 공짜인지, 매번 DB 왕복인지는 **코드가 아니라 트랜잭션 경계가 결정한다.** 같은 코드가 경계에 따라 공짜였다가 비싸진다.

---

## 4. 지연 로딩 — 프록시는 언제 진짜가 되나

`Order`를 읽으면 `lines`도 같이 읽힐까? `fetch = FetchType.LAZY`면 **아니다.**

부모를 읽는 시점에 Hibernate는 `lines` 자리에 진짜 목록 대신 **프록시(proxy — 진짜인 척하는 껍데기 객체)** 를 꽂아둔다. 이 프록시는 자기 안에 아무 데이터도 갖고 있지 않고, "필요해지면 그때 DB에서 가져오겠다"는 약속만 들고 있다.

**그럼 언제 "필요해지나".** 컬렉션의 **내용에 접근하는 거의 모든 동작**이 방아쇠다.

| 동작 | 로딩 발동 | 이유 |
|---|---|---|
| `order.lines` (참조만) | ❌ | 프록시 객체를 가리키기만 함 |
| `order.lines.size` | ✅ | 크기를 알려면 내용이 있어야 함 |
| `order.lines.isEmpty()` | ✅ | 위와 같음 |
| `for (l in order.lines)` | ✅ | 순회 = 내용 접근 |
| `order.lines.firstOrNull { ... }` | ✅ | **순회한다** — 이게 사건의 방아쇠였다 |
| `order.lines.map { ... }` | ✅ | 순회 |
| `order.lines.add(x)` | 상황에 따라 | `PersistentBag`이면 발동 안 할 수 있음 |

핵심은 이것이다. **`firstOrNull`, `find`, `filter` 같은 Kotlin 컬렉션 함수는 DB를 모른다.** 이들은 평범한 메모리 컬렉션 함수라서, "이 조건에 맞는 것 하나만 DB에서 가져와" 같은 요청을 만들 수 없다. **전부 가져온 다음 메모리에서 거른다.**

```kotlin
order.lines.firstOrNull { it.id == lineId }
//         └── 이 순간 SELECT ... WHERE order_id = ? 가 나가고 자식이 전부 올라온다
//                                              └── 그 뒤에야 조건 검사가 시작된다
```

`WHERE id = ?`가 아니라 `WHERE order_id = ?`라는 데 주목. 조건은 SQL이 아니라 **로딩이 끝난 다음** 적용된다.

---

## 5. 핵심 — 한 건을 고치려는 코드가 N건을 읽는 순간

이제 사건의 코드를 볼 수 있다.

```kotlin
@Transactional(propagation = Propagation.REQUIRES_NEW)
fun markLineShipped(orderId: UUID, lineId: UUID) {
    val order = orderRepository.findById(orderId).orElse(null) ?: return
    val line = order.lines.firstOrNull { it.id == lineId } ?: return
    line.markShipped()
}
```

읽어보면 "주문 하나 찾아서, 그 안의 항목 하나 찾아서, 상태 바꾼다"이다. 어디에도 "여러 건"이라는 말이 없다.

**`findById`는 정말로 단건만 읽는다.** 반환 타입이 `Optional<Order>`이고 PK 조회니 여러 건이 나올 수도 없다. 문제는 그 다음 줄이다.

실제로 나가는 SQL은 두 개다.

```sql
-- ① findById — 부모 1건
select o.id, ... from orders o where o.id = ?

-- ② firstOrNull 이 순회를 시작하는 순간 — 그 주문의 자식 전부
select l.order_id, l.id, l.status, ... from order_lines l where l.order_id = ?
```

**직접 측정한 값** (자식 5건짜리 주문, 위 메서드 1회 호출):

| | 로드된 엔티티 | 실행된 쿼리 | 컬렉션 초기화 |
|---|---|---|---|
| 위 코드 | **6** (부모 1 + 자식 5) | 2 | 1 |
| 자식만 직접 조회 (9장) | **1** | 1 | 0 |

자식이 5건이라 6이다. 자식이 254건이면 **255개 엔티티**가 한 번의 호출에 올라온다. 그리고 이 메서드는 **자식마다 한 번씩** 불린다.

$$\text{총 로드} = N \times (1 + N)$$

254건이면 약 6만 5천 개다. 엔티티는 스냅샷(변경 감지용 복사본)까지 함께 만들어지므로 실제 메모리 압력은 그보다 크다. 부모에 **암복호화 컬럼이나 큰 텍스트 컬럼**이 있으면 그 비용도 매번 반복된다.

> **왜 실전에서 중요한가.** 코드 리뷰에서 이 메서드는 정상으로 보인다. "단건 조회 + 단건 수정"이다. 비용은 `firstOrNull`이라는 **평범한 컬렉션 함수 한 개**에 숨어 있고, 그 함수가 얼마나 비싼지는 `lines`의 fetch 전략을 알아야만 보인다.

---

## 6. 왜 어제까지는 문제가 아니었나 — 캐시의 수명은 트랜잭션의 수명이다

사건의 진짜 교훈은 여기다. **이 메서드는 바뀌지 않았다.** 바뀐 것은 트랜잭션 경계였다.

### 이전 — 작업 전체가 한 트랜잭션

```
[트랜잭션 시작] ─ 영속성 컨텍스트(상자) 생성
  작업 시작 시 부모 + 자식 전부 로드 → 상자에 들어감
  markLineShipped(1번째) → findById? 상자에 있음. lines? 이미 초기화됨.  DB 왕복 0
  markLineShipped(2번째) → 마찬가지                                      DB 왕복 0
  ...
  markLineShipped(N번째) →                                              DB 왕복 0
[트랜잭션 끝] ─ 상자 폐기
```

루프 안의 조회는 **전부 1차 캐시 히트**였다. 추가 로딩 0건. 코드가 비효율적으로 생겼어도 대가를 치르지 않았다.

### 이후 — 항목마다 독립 트랜잭션

커넥션을 오래 붙들지 않으려고(그 자체로는 옳은 목적) 트랜잭션을 항목 단위로 쪼갰다. `REQUIRES_NEW`는 **매번 새 트랜잭션**을 열고, 새 트랜잭션은 **새 영속성 컨텍스트**를 얻는다.

```
markLineShipped(1번째) [새 트랜잭션 → 빈 상자] → SELECT 부모, SELECT 자식 전부   (1+N)
markLineShipped(2번째) [새 트랜잭션 → 빈 상자] → SELECT 부모, SELECT 자식 전부   (1+N)
...
```

**캐시가 매번 비어 있으니 매 호출이 실제 DB 조회가 된다.**

정리하면 이렇다.

| | 이전 | 이후 |
|---|---|---|
| 상자의 수명 | 작업 전체 | 항목 하나 |
| 루프 안 조회의 정체 | 캐시 히트 | 실제 DB 조회 |
| 총 로드 엔티티 | 1 + N | N × (1 + N) |
| 커넥션 점유 | 길다 (문제였음) | 짧다 (개선됨) |

> **왜 실전에서 중요한가.** 트랜잭션 경계를 바꾸는 변경은 보통 "커넥션 점유 시간"이나 "부분 커밋"의 관점에서 검토된다. 그런데 **1차 캐시의 수명도 함께 바뀐다.** 경계를 좁히면 캐시 히트에 기대던 코드가 전부 실제 조회로 바뀐다. 경계를 건드릴 때는 **"이 경계 안에서 캐시 히트로 공짜였던 조회가 무엇인가"** 를 반드시 같이 묻는다.
>
> 관련: 트랜잭션 경계 자체에 대해서는 [Spring 트랜잭션 노트의 8장](spring-transaction-boundaries-and-batch.md#8-경계를-바꾸면-딸려오는-것들)에 readOnly 리플리카 라우팅과 커밋 후 콜백 시점이 정리돼 있다.

---

## 7. 이게 N+1 문제와 같은 것인가

가깝지만 다르다. 둘을 구분해 두면 진단이 빨라진다.

**전형적인 N+1 문제** — 목록을 읽고 각 원소의 연관을 건드릴 때.

```kotlin
val orders = orderRepository.findAll()        // 쿼리 1번 (부모 N건)
orders.forEach { it.lines.size }              // 각 부모마다 쿼리 1번 → N번
// 총 1 + N 쿼리
```

**이 문서의 문제** — 단건을 반복해서 읽되, 매번 컬렉션을 통째로 초기화할 때.

```kotlin
repeat(N) { markLineShipped(orderId, lineIds[it]) }   // 호출마다 쿼리 2번
// 총 2N 쿼리, 총 N × (1+N) 엔티티
```

| | 전형적 N+1 | 이 문서의 경우 |
|---|---|---|
| 방아쇠 | 부모 여러 건 × 연관 접근 | 부모 한 건 × 반복 호출 |
| 쿼리 수 | 1 + N | 2N |
| 로드 엔티티 | N + (자식 총량) | **N × (1 + N)** |
| 눈에 띄는가 | 쿼리 수가 튀어 잘 보인다 | **쿼리 수는 2N으로 완만해 잘 안 보인다** |
| 흔한 처방 | fetch join, `@BatchSize`, `@EntityGraph` | 자식 직접 조회 (9장) |

**중요한 함정 하나.** 이 경우 **쿼리 수는 진단 지표로 부적합하다.** 고치기 전 2N, 고친 뒤 N — 겨우 절반이다. 반면 **로드된 엔티티 수는 N×(1+N)에서 N으로** 떨어진다. 메모리 문제를 쫓을 때는 쿼리 수가 아니라 **로드된 행 수**를 봐야 한다.

`@BatchSize`도 여기서는 도움이 안 된다. `@BatchSize`는 **여러 프록시를 한 번에** 초기화해 왕복 횟수를 줄이는 기능인데, 여기서는 초기화 대상 컬렉션이 매번 하나뿐이다. 줄일 왕복이 없다.

---

## 8. 눈으로 확인하는 법 — 통계와 SQL 로그

추측하지 말고 센다. 두 가지 방법이 있다.

### 방법 1 — Hibernate 통계 (숫자로 센다)

```yaml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
```

```kotlin
val stats = entityManagerFactory.unwrap(SessionFactory::class.java).statistics
stats.isStatisticsEnabled = true   // 코드에서 켜도 된다 (설정 없이 이 줄만으로도 동작)
stats.clear()

markLineShipped(orderId, lineId)

println(stats.entityLoadCount)       // 로드된 엔티티 인스턴스 수  ← 이 문제의 지표
println(stats.prepareStatementCount) // 실행된 쿼리 수
println(stats.collectionLoadCount)   // 초기화된 컬렉션 수
```

읽는 법:

| 지표 | 의미 | 이 문제에서 |
|---|---|---|
| `entityLoadCount` | 하이드레이트된 엔티티 인스턴스 수. **컬렉션 원소도 각각 1로 센다** | 6 → 1 |
| `prepareStatementCount` | 실행된 SQL 문 수 | 2 → 1 |
| `collectionLoadCount` | 초기화된 컬렉션 수. **0이면 지연 로딩이 안 터졌다는 뜻** | 1 → 0 |

`collectionLoadCount`가 특히 좋은 지표다. "컬렉션 전량 로딩이 사라졌는가"를 **0 아니면 1**로 이진 판정해 준다.

주의: 통계는 `SessionFactory` 단위 **누계**다. 트랜잭션마다 새 세션이 열려도 값은 계속 쌓인다. 그래서 재기 직전에 `clear()`가 필요하다. 그리고 **통계를 켜면 세션 메트릭 로그가 함께 켜진다** — [부록 A.4](#a4-통계-지표-읽는-법과-켤-때의-부작용) 참조.

### 방법 2 — SQL 로그 (눈으로 본다)

```yaml
logging:
  level:
    org.hibernate.SQL: DEBUG
```

찾을 것은 **루프 안에서 `where <fk_컬럼> = ?` 형태의 SELECT가 반복되는지**다. 그게 컬렉션 전량 로딩의 지문이다.

```
select l.order_id, l.id, l.status from order_lines l where l.order_id=?   ← 이게 반복되면 문제
select l.id, l.status, l.order_id from order_lines l where l.id=? and l.order_id=?   ← 고친 뒤
```

숫자로 회귀를 잠그려면 방법 1, 운영 환경에서 빠르게 확인하려면 방법 2가 낫다.

---

## 9. 고치는 네 가지 방법과 각각의 대가

목표는 **"항목 하나를 고칠 때 항목 하나만 읽는다"** 이다.

### (a) 자식을 직접 조회한다 — 이 경우의 정답

```kotlin
interface OrderLineRepository : JpaRepository<OrderLine, UUID> {
    fun findByIdAndOrderId(lineId: UUID, orderId: UUID): OrderLine?
}
```

```kotlin
@Transactional(propagation = Propagation.REQUIRES_NEW)
fun markLineShipped(orderId: UUID, lineId: UUID) {
    val line = lineRepository.findByIdAndOrderId(lineId = lineId, orderId = orderId) ?: return
    line.markShipped()
}
```

부모를 아예 읽지 않는다. 쿼리 1번, 엔티티 1건.

`orderId`를 조건에 넣는 이유는 성능이 아니라 **안전**이다. 다른 주문의 항목 ID가 잘못 흘러 들어와도 조회가 비어 남의 데이터를 건드리지 못한다. 조건을 SQL에 두면 이 보장이 **DB 수준에서** 걸린다.

`markShipped()`를 그대로 호출하므로 **도메인 가드가 코드에 남는다.** 변경 감지가 `UPDATE`를 만들어 준다.

> 다만 이 방법에는 선행 조건이 있다 — 자식이 `orderId`로 조회될 수 있어야 한다. 단방향 연관에서는 그게 안 된다. [10장](#10-자식으로-직접-조회하기--단방향-연관의-벽과-읽기-전용-매핑)에서 푼다.

### (b) fetch join / `@EntityGraph`로 한 번에 읽는다

```kotlin
@Query("select o from Order o join fetch o.lines where o.id = :id")
fun findWithLines(@Param("id") id: UUID): Order?
```

쿼리 2번이 1번이 된다. 하지만 **읽는 행 수는 그대로 1+N**이다. 이 문제의 지표는 행 수이므로 근본 해결이 아니다. 부모와 자식 전부가 실제로 필요할 때(작업 시작 시 1회 로드 같은) 쓰는 도구다.

### (c) 벌크 UPDATE

```kotlin
@Modifying
@Query("update OrderLine l set l.status = 'SHIPPED' where l.id = :lineId and l.status = 'PENDING'")
fun markShipped(@Param("lineId") lineId: UUID): Int
```

가장 빠르다. 읽지 않고 바로 쓴다. 대가가 넷이다.

1. **도메인 가드가 WHERE 절로 축소된다.** `markShipped()`의 검사 로직이 SQL 문자열 안으로 사라진다. 다음 사람이 이 쿼리를 고칠 때 그 규칙을 못 본다.
2. **영속성 컨텍스트를 우회한다.** 상자 안에 그 엔티티가 이미 있으면 **상자의 값이 낡은 채로 남는다.** `@Modifying(clearAutomatically = true, flushAutomatically = true)`로 완화하지만, 그러면 상자 전체가 비워져 다른 엔티티의 변경 감지까지 영향을 받는다.
3. **엔티티 생명주기 기능이 건너뛰어진다.** `@PreUpdate` 같은 콜백, 낙관적 락 버전 증가, 그리고 **감사 로그(Envers 등)를 쓰는 엔티티라면 이력이 남지 않는다.** ← 단, 이건 **그 엔티티가 실제로 감사 대상일 때만** 해당한다. 확인하지 않고 "감사 이력이 사라진다"를 근거로 삼으면 틀린 근거가 된다(실제로 이 판단을 잘못한 적이 있다).
4. 반환값이 `Int`(영향받은 행 수)라 호출부가 "왜 0건인지"를 알 수 없다.

**쓸 자리는 있다.** 도메인 규칙이 없는 대량 상태 전이(예: 만료 처리 배치)에는 (c)가 맞다. 규칙이 있는 건별 전이에는 (a)가 맞다.

### (d) 트랜잭션 경계를 되돌린다

캐시 히트로 돌아가니 증상은 사라진다. 하지만 경계를 넓힌 이유(커넥션 점유, 부분 커밋 허용)가 함께 사라진다. **증상의 원인과 경계 변경의 목적은 별개** — 되돌리는 건 보통 오답이다.

### 정리

| 방법 | 쿼리 | 읽는 행 | 도메인 가드 | 언제 |
|---|---|---|---|---|
| (a) 자식 직접 조회 | 1 | **1** | 유지 | **건별 상태 전이 — 기본 선택** |
| (b) fetch join | 1 | 1+N | 유지 | 부모+자식이 다 필요할 때 |
| (c) 벌크 UPDATE | 1 | 0 | WHERE로 축소 | 규칙 없는 대량 전이 |
| (d) 경계 되돌리기 | — | — | 유지 | 대개 오답 |

---

## 10. 자식으로 직접 조회하기 — 단방향 연관의 벽과 읽기 전용 매핑

9장 (a)를 쓰려면 `findByIdAndOrderId`가 성립해야 한다. Spring Data는 **메서드 이름을 파싱해서** 쿼리를 만들므로, `OrderLine` 엔티티에 `orderId`라는 **속성이 존재해야** 한다.

그런데 1장의 매핑에는 없다.

```kotlin
class OrderLine(
    @Id val id: UUID,
)   // orderId 필드가 없다
```

**단방향 `@OneToMany` + `@JoinColumn`** 이기 때문이다. FK 컬럼 `order_lines.order_id`는 DB에 분명히 있지만, **그 컬럼을 관리하는 주체는 부모** 다. 자식은 자기가 어느 부모에 속하는지 객체 수준에서 모른다.

### 해법 — 같은 컬럼을 읽기 전용으로 한 번 더 매핑한다

```kotlin
class OrderLine(
    @Id val id: UUID,
) {
    /**
     * 부모가 관리하는 FK 컬럼을 읽기 전용으로만 매핑한다.
     * 쓰기는 계속 부모 컬렉션이 담당하므로 DDL 변경이 없다.
     */
    @Column(name = "order_id", nullable = false, insertable = false, updatable = false)
    val orderId: UUID? = null
}
```

**왜 충돌하지 않나.** 같은 컬럼을 두 곳에서 매핑하면 보통 "중복 매핑" 오류가 난다. 그런데 Hibernate의 중복 컬럼 검사는 **`insertable` 또는 `updatable` 중 하나라도 `true`인 속성만** 대상으로 한다. 둘 다 `false`면 검사에서 제외된다. 즉 **쓰기 주체는 여전히 하나뿐**이라 충돌이 아니다.

부모가 자식을 저장할 때는 Hibernate가 내부적으로 만든 **backref**(쓰기 전용 내부 속성)가 FK를 채운다. 새 매핑은 여기에 끼어들지 않는다.

### 주의할 점 세 가지

**1. 타입 매핑을 관례대로 맞춘다.** 특히 UUID처럼 DB 표현이 갈리는 타입은 명시가 필요하다. 예를 들어 컬럼이 `varchar(36)`인데 매핑에서 타입을 지정하지 않으면 Hibernate가 binary로 잡아 **`ddl-auto: validate` 환경에서 기동이 실패**한다.

```kotlin
@JdbcTypeCode(SqlTypes.VARCHAR)
@Column(name = "order_id", nullable = false, length = 36, insertable = false, updatable = false)
val orderId: UUID? = null
```

참고로 Hibernate 6.6의 스키마 검증기는 **컬럼 존재 여부와 JDBC 타입 코드만** 비교하고 `length`·nullability는 보지 않는다. 기동을 실제로 좌우하는 건 타입 지정 쪽이고, `length`는 관례를 맞추는 의미다.

**2. `nullable = false`인데 왜 `UUID?`인가.** DDL 상으로는 NOT NULL이 맞다. 하지만 **부모가 FK를 쓰기 전** — 즉 새 자식 객체를 만들어 부모 컬렉션에 넣은 직후 — 메모리 상의 이 필드는 비어 있다. 그래서 nullable 타입으로 선언한다. 조회 술어로만 쓰고 값을 역참조하지 않으니 `!!` 같은 강제 언래핑을 넣을 이유가 없다.

**3. 같은 타입 인자 두 개의 순서.** `findByIdAndOrderId(lineId: UUID, orderId: UUID)`는 인자가 둘 다 `UUID`다. **순서를 바꿔 써도 컴파일된다.** 바뀌면 조회가 조용히 비고, 호출부가 그걸 "없음"으로 처리하면 아무 로그 없이 아무 일도 안 일어난다. 호출부에서 **named argument**를 쓰는 게 값싼 방어다.

```kotlin
lineRepository.findByIdAndOrderId(lineId = lineId, orderId = orderId)
```

(Spring Data는 술어를 **메서드 이름**에서 파싱하므로 파라미터 이름은 자유롭게 지어도 된다. `id` 같은 모호한 이름 대신 도메인 의미가 드러나는 이름을 쓰는 편이 낫다.)

---

## 11. 회귀를 테스트로 잡는 법

이 종류의 문제는 **기능 테스트로는 안 잡힌다.** 고치기 전에도 결과는 정확했다 — 느리고 메모리를 많이 썼을 뿐이다. 그래서 **비용 자체를 단언**해야 한다.

```kotlin
@Test
fun `항목 1건 갱신에 엔티티를 1건만 읽는다`() {
    val order = saveOrderWith(lineCount = 5)
    val stats = entityManagerFactory.unwrap(SessionFactory::class.java).statistics
    stats.isStatisticsEnabled = true
    stats.clear()

    markLineShipped(order.id, order.lines.first().id)

    assertEquals(
        1L, stats.entityLoadCount,
        "항목 1건만 읽어야 한다. 6이면 부모 전체 재조회가 되살아난 것이다",
    )
}
```

이 테스트가 값을 갖는 조건이 둘 있다.

**진짜 DB를 쓸 것.** 저장소를 목(mock)으로 스텁하면 통계가 안 잡히는 건 물론이고, "다른 부모의 항목은 갱신되지 않는다" 같은 단언이 **스텁이 null을 돌려주는 자기충족 테스트**가 된다. 실제 쿼리 술어를 고정하려면 인메모리 DB든 컨테이너든 진짜 DB가 필요하다.

**옛 구현에서 실제로 실패하는지 확인할 것.** 위 예제의 기대값 `1`과 주석의 `6`은 추론이 아니라 [8장의 측정](#8-눈으로-확인하는-법--통계와-sql-로그)에서 나온 값이다. 자식 수를 바꾸면 기대값도 바뀌므로(자식 M건이면 옛 구현은 `1 + M`), 테스트 픽스처의 자식 수를 고정하고 실패 메시지에 그 수를 적어두는 게 좋다.

---

## 12. 정리 — 다음에 이걸 만나면

**증상으로 알아채기**

- 배치·반복 작업의 메모리가 처리 건수의 **제곱**에 비례해 보인다
- 쿼리 수는 크게 안 늘었는데 응답이 느리고 메모리가 는다 → **쿼리 수 말고 로드된 행 수를 봐라**
- 코드를 안 바꿨는데 갑자기 느려졌다 → **트랜잭션 경계가 바뀌었는지** 봐라

**코드에서 찾을 패턴**

```kotlin
val parent = parentRepository.findById(parentId)...
val child = parent.children.firstOrNull { it.id == childId }   // ← 여기
```

부모를 읽고 **컬렉션에서 자식을 찾는** 모양. 컬렉션이 `LAZY`면 전량 로딩이다.

**확인**

`generate_statistics`를 켜고 `collectionLoadCount`를 본다. 루프 안에서 0이 아니면 컬렉션 전량 로딩이 일어나고 있다.

**고치기**

건별 상태 전이면 **자식 직접 조회**(9장 a). FK를 조회 축으로 쓰려면 **읽기 전용 매핑**을 추가한다(10장). 도메인 가드가 있으면 엔티티 메서드를 계속 호출해 가드를 코드에 남긴다.

**막기**

`entityLoadCount` 또는 `collectionLoadCount`를 단언하는 테스트를 **진짜 DB로** 붙인다(11장).

**한 문장으로**

> 1차 캐시의 수명은 트랜잭션의 수명이다. 트랜잭션 경계를 좁히면, 캐시 히트라서 공짜였던 조회가 전부 실제 DB 왕복으로 바뀐다.

---

# 부록

## A.1 `LazyInitializationException` — 세션이 닫힌 뒤에 건드리면

지연 로딩의 반대편 함정이다. 프록시는 **자기를 만든 영속성 컨텍스트가 살아 있을 때만** 데이터를 가져올 수 있다.

```kotlin
fun broken(id: UUID): Int {
    val order = tx.execute { orderRepository.findById(id).orElseThrow() }  // 트랜잭션 끝 → 상자 폐기
    return order.lines.size    // LazyInitializationException
}
```

메시지는 보통 이렇게 나온다.

```
could not initialize proxy [Order.lines] - no Session
```

해결은 셋 중 하나다.

1. 필요한 걸 **트랜잭션 안에서** 다 읽어둔다 (fetch join / `@EntityGraph` / 명시적 초기화)
2. 트랜잭션 경계를 그 지점까지 넓힌다
3. 애초에 컬렉션이 필요 없게 쿼리를 바꾼다 (이 문서의 10장)

`open-in-view`(A.3)를 켜면 이 예외가 "사라지는" 것처럼 보이는데, 문제를 숨기는 쪽에 가깝다.

## A.2 연관관계별 `fetch` 기본값

JPA 명세가 정하는 기본값이다. **자주 헷갈리므로 외우기보다 매번 확인하는 게 낫다.**

| 연관 | 기본 `fetch` | 감각 |
|---|---|---|
| `@OneToMany` | `LAZY` | 여럿 → 미루는 게 안전 |
| `@ManyToMany` | `LAZY` | 여럿 → 미루는 게 안전 |
| `@ManyToOne` | **`EAGER`** | 하나 → 같이 읽음. **N+1의 흔한 출처** |
| `@OneToOne` | **`EAGER`** | 하나 → 같이 읽음 |

`@ManyToOne`이 기본 `EAGER`라는 게 실무에서 잘 물린다. 목록 조회에서 각 행마다 부모를 한 번씩 더 읽게 되기 쉬워서, 명시적으로 `LAZY`로 두고 필요할 때 fetch join 하는 스타일이 흔하다.

## A.3 `open-in-view` — 지연 로딩의 수명을 HTTP 요청까지 늘리는 스위치

Spring Boot의 `spring.jpa.open-in-view`는 **영속성 컨텍스트를 HTTP 요청이 끝날 때까지 열어두는** 옵션이다. 켜져 있으면 컨트롤러나 뷰에서 지연 로딩을 해도 `LazyInitializationException`이 안 난다.

**Spring Boot의 기본값은 `true`** 이고, 기동할 때 경고 로그를 남긴다. 끄는 쪽이 일반적으로 권장된다.

| | 켜짐 (기본) | 꺼짐 |
|---|---|---|
| 뷰에서 지연 로딩 | 된다 | `LazyInitializationException` |
| DB 커넥션 점유 | **요청 끝까지** | 서비스 트랜잭션 동안만 |
| 숨은 쿼리 | 컨트롤러·뷰에서도 발생 | 서비스 계층에 갇힘 |

끄면 "어디서 쿼리가 나가는지"가 서비스 계층으로 모여 이 문서 같은 문제를 찾기 쉬워진다. 대신 필요한 데이터를 트랜잭션 안에서 다 읽어두는 규율이 필요하다.

## A.4 통계 지표 읽는 법과 켤 때의 부작용

### 주요 지표

| 지표 | 센다 |
|---|---|
| `entityLoadCount` | 로드된 엔티티 인스턴스 수 (컬렉션 원소 포함) |
| `entityInsertCount` / `entityUpdateCount` / `entityDeleteCount` | 실행된 쓰기 |
| `collectionLoadCount` | 초기화된 컬렉션 수 |
| `prepareStatementCount` | 실행된 SQL 문 수 |
| `queryExecutionMaxTime` | 가장 느렸던 쿼리 시간 |

### 부작용 — 세션 메트릭 로그가 함께 켜진다

`generate_statistics = true`로 켜면 **세션이 닫힐 때마다 세션 메트릭이 INFO로 출력된다.**

Hibernate 6.6 소스에서 기본값이 이렇게 결정된다.

```java
// SessionFactoryOptionsBuilder
final boolean logSessionMetrics =
        configurationService.getSetting( LOG_SESSION_METRICS, BOOLEAN, statisticsEnabled );
//                                                            ↑ 기본값이 "통계 활성 여부"
```

즉 `hibernate.session.events.log`를 명시하지 않으면 **통계를 켠 것만으로 로그가 따라 켜진다.** 트랜잭션이 잘게 쪼개진 코드일수록 세션이 자주 열리고 닫혀서 출력이 급격히 늘어난다. 실제로 테스트 스위트에 통계를 전역으로 켰다가 로그 블록이 수백 개 쌓여 **진짜 실패 로그가 묻힌** 적이 있다.

대응은 둘 중 하나다.

```yaml
# (a) 로그만 끈다
spring.jpa.properties.hibernate.session.events.log: false
```

```kotlin
// (b) 전역 설정을 두지 않고, 필요한 테스트 안에서만 켠다 — 더 좁은 방법
stats.isStatisticsEnabled = true
```

측정이 필요한 곳이 테스트 한두 개뿐이라면 (b)가 낫다. 카운트 여부는 런타임에 판정되므로 설정 파일 없이 이 한 줄만으로 동작한다.
