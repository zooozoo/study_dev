# 배치 커넥션 고갈 사건으로 배우는 Spring 트랜잭션 경계와 배치 트랜잭션 매니저

> 회사 배치 잡의 운영 장애(MORACARE-3153) 수정 과정에서 나온 개념들을 배경 지식 없이도 읽을 수 있게 정리한 학습 문서.
> 코드 예시는 전부 회사 코드가 아닌 **최소 재현 예제**로 일반화했다.
> 작성일: 2026-09-02

---

## 0. 이번 사건 한 문단 요약

하루 한 번 도는 배치 잡이 있다. 예약된 알림을 회원 250명에게 보내는 일인데, 회원 한 명마다 "메시지를 DB에 저장하고 → 외부 푸시 API를 호출"하는 걸 루프로 반복한다. 어느 날 이 배치가 도는 시간에, **배치와 아무 상관 없는 일반 사용자 API가 HTTP 500을 뱉었다.** 원인은 이랬다. 배치 잡의 스텝 전체가 **DB 트랜잭션 하나**로 묶여 있었고, 그 안에서 외부 푸시 API를 250번 순차 호출하느라 트랜잭션이 96초 동안 열려 있었다. 그동안 DB 커넥션 1개와 테이블 행 잠금이 계속 붙잡혀 있었고, 같은 테이블을 건드리는 앱 요청들이 잠금 대기에 걸려 **커넥션 풀(4개)이 말라버렸다.** 고치는 방법은 "스텝 전체 = 트랜잭션 1개" 구조를 깨고, 트랜잭션을 **회원 1명 단위(약 0.4초)** 로 쪼개는 것이었다.

이 문단 안에 모르는 단어가 하나라도 있으면 이 문서가 도움이 된다. 아래 순서대로 읽으면 위 문단이 전부 이해된다.

---

## 1. 내가 물어본 것들 → 어디를 보면 되나

| 질문 | 섹션 |
|---|---|
| 트랜잭션이란 게 정확히 뭔가 | [2장 트랜잭션의 기초](#2-트랜잭션이란-무엇인가--커넥션-잠금-커밋) |
| 트랜잭션이 오래 열려 있으면 왜 남한테 피해가 가나 | [2.4 오래 열린 트랜잭션이 위험한 이유](#24-오래-열린-트랜잭션이-왜-위험한가) |
| `@Transactional`은 어떻게 동작하나 | [3장 @Transactional의 실체](#3-transactional의-실체--프록시와-전파) |
| 트랜잭션이 "합쳐진다"는 게 무슨 뜻인가 | [3.3 전파 REQUIRED](#33-required--있으면-얹히고-없으면-내가-연다) |
| 안쪽 메서드에 `@Transactional`이 있는데 왜 따로 커밋이 안 되나 | [3.4 참여자는 커밋하지 않는다](#34-참여자는-커밋하지-않는다--실제-커밋은-바깥-하나뿐) |
| `REQUIRES_NEW`는 뭐가 다른가 | [3.5 REQUIRES_NEW](#35-requires_new--무조건-새로-연다) |
| `PlatformTransactionManager`가 뭔가 | [4장 트랜잭션 매니저](#4-트랜잭션-매니저--전략-인터페이스와-구현체들) |
| Spring Batch의 `ResourcelessTransactionManager`는 뭐고 뭐가 다른가 | [5장 ResourcelessTransactionManager](#5-resourcelesstransactionmanager--아무것도-잡지-않는-트랜잭션-매니저) |
| 배치에는 보통 어떤 트랜잭션 매니저를 쓰나, 선택 기준은 | [6장 배치에서 무엇을 고를 것인가](#6-배치에서-어떤-트랜잭션-매니저를-고를-것인가) |
| 청크 스텝과 태스클릿 스텝은 뭐가 다른가 | [6.1 두 가지 스텝](#61-먼저-스텝에는-두-종류가-있다) |
| 스텝 트랜잭션 매니저를 바꾸면 뭘 잃나 | [6.4 대가: 메타데이터 원자성](#64-대가--배치-메타데이터와의-원자성을-잃는다) |
| 트랜잭션 매니저를 빈으로 등록하면 왜 위험한가 | [7장 조용히 깨지는 함정](#7-조용히-깨지는-함정--트랜잭션-매니저를-빈으로-올리지-말-것) |
| 트랜잭션 경계를 바꾸면 뭐가 딸려서 바뀌나 | [8장 경계를 바꾸면 딸려오는 것들](#8-경계를-바꾸면-딸려오는-것들) |
| readOnly 플래그로 읽기 전용 DB로 보내는 건 어떻게 되나 | [8.1 읽기 전용 라우팅](#81-읽기-전용-라우팅이-갑자기-동작하기-시작한다) |
| 커밋 후 이벤트는 언제 터지나 | [8.2 커밋 후 콜백](#82-커밋-후-콜백이-터지는-시점이-달라진다) |
| 트랜잭션을 쪼개면 새로 생기는 위험은 | [9장 쪼갤 때 생기는 새 위험](#9-트랜잭션을-쪼개면-새로-생기는-위험--부분-커밋) |
| 이런 변경을 어떻게 테스트하나 | [10장 트랜잭션 경계 테스트](#10-트랜잭션-경계를-테스트로-잡는-법) |
| 커넥션 풀이 마른다는 게 정확히 무슨 일인가 | [부록 A.1 커넥션 풀](#a1-커넥션-풀이-마른다는-것) |
| 행 잠금은 언제 잡히고 언제 풀리나 | [부록 A.2 행 잠금의 수명](#a2-행-잠금은-언제-잡히고-언제-풀리나) |
| 프록시 기반이라 self-invocation이 안 된다는 게 뭔가 | [부록 A.3 self-invocation 함정](#a3-self-invocation--같은-클래스-안에서-부르면-왜-안-먹나) |

---

## 2. 트랜잭션이란 무엇인가 — 커넥션, 잠금, 커밋

### 2.1 트랜잭션은 "묶음 처리"다

데이터베이스에 여러 번 쓰기를 하는데, **전부 성공하거나 전부 실패해야** 하는 경우가 있다. 계좌 이체가 교과서 예시다.

```
A 계좌에서 1만원 빼기
B 계좌에 1만원 넣기
```

첫 줄만 성공하고 둘째 줄에서 서버가 죽으면 돈이 증발한다. 그래서 이 둘을 하나로 묶어서 "둘 다 되든지, 둘 다 안 되든지"를 보장하게 만드는 게 **트랜잭션(transaction)** 이다.

```sql
BEGIN;                                   -- 트랜잭션 시작
UPDATE accounts SET balance = balance - 10000 WHERE id = 'A';
UPDATE accounts SET balance = balance + 10000 WHERE id = 'B';
COMMIT;                                  -- 여기서 비로소 "확정"
```

- **BEGIN** — 트랜잭션을 연다
- **COMMIT** — 지금까지의 변경을 확정한다. 이 순간부터 다른 사람에게 보인다
- **ROLLBACK** — 지금까지의 변경을 전부 취소한다. 아무 일도 없었던 게 된다

### 2.2 트랜잭션은 DB 커넥션 위에서 산다

여기가 핵심인데, 자주 건너뛰는 부분이다.

**트랜잭션은 "DB 커넥션 하나"에 묶여 있다.** BEGIN을 보낸 그 커넥션으로 이후 SQL을 계속 보내야 같은 트랜잭션이고, COMMIT도 그 커넥션으로 보내야 한다.

```
애플리케이션 ──[커넥션 #3]──▶ DB
   BEGIN            (커넥션 #3에서)
   UPDATE ...       (커넥션 #3에서)
   UPDATE ...       (커넥션 #3에서)
   COMMIT           (커넥션 #3에서)
                    ← 여기서야 커넥션 #3이 풀려난다
```

따라서 **트랜잭션이 열려 있는 동안 그 커넥션은 다른 누구도 쓸 수 없다.** 트랜잭션이 96초 열려 있으면, 커넥션 하나가 96초 동안 묶여 있는 것이다. → [부록 A.1](#a1-커넥션-풀이-마른다는-것)

### 2.3 쓰기를 하면 행 잠금이 걸린다

트랜잭션 안에서 어떤 행(row)을 `UPDATE`하면, DB는 그 행에 **배타 잠금(exclusive lock)** 을 건다. "지금 내가 고치는 중이니 아무도 건드리지 마"라는 표시다.

이 잠금은 **커밋(또는 롤백) 시점에 풀린다.** 중간에 자동으로 풀리지 않는다.

```
트랜잭션 T1:  BEGIN → UPDATE rows(id=5) → ...... 96초 ...... → COMMIT
                          ↑ 잠금 획득                            ↑ 잠금 해제

트랜잭션 T2:            UPDATE rows(id=5)  ← 여기서 멈춰서 대기한다
                        (T1이 커밋할 때까지 96초를 기다린다)
```

기다리다 못해 설정된 시간(MySQL 기본 `innodb_lock_wait_timeout` = 50초)을 넘기면 T2는 에러로 죽는다. → [부록 A.2](#a2-행-잠금은-언제-잡히고-언제-풀리나)

### 2.4 오래 열린 트랜잭션이 왜 위험한가

2.2와 2.3을 합치면 답이 나온다. 트랜잭션이 오래 열려 있으면 **두 가지 자원을 동시에 오래 붙잡는다.**

| 붙잡는 것 | 결과 |
|---|---|
| DB 커넥션 1개 | 커넥션 풀에서 하나가 빠진다 |
| 건드린 행들의 잠금 | 같은 행을 쓰려는 다른 요청이 전부 대기 |

그리고 여기서 **연쇄 반응**이 일어난다. 대기하는 쪽도 자기 커넥션을 붙잡은 채로 기다리기 때문이다.

```
배치 트랜잭션      : 커넥션 1개를 96초 점유 + 행 잠금 보유
앱 요청 A (대기)   : 잠금 대기하며 커넥션 1개를 41초 점유
앱 요청 B (대기)   : 잠금 대기하며 커넥션 1개를 46초 점유
앱 요청 C (대기)   : 잠금 대기하며 커넥션 1개를 50초 점유
                     ─────────────────────────────────
                     풀에 커넥션이 4개뿐이면 여기서 고갈
앱 요청 D~         : 커넥션을 못 받아 타임아웃 → HTTP 500
                     ↑ 잠금과 아무 관계도 없는 무관한 API까지 죽는다
```

**이게 0장 사건의 정확한 메커니즘이다.** 배치는 자기 일만 했는데, 트랜잭션을 오래 잡은 탓에 무관한 사용자 요청까지 무너뜨렸다.

> **교훈 한 줄:** 트랜잭션 안에 **외부 네트워크 호출**(HTTP API, 푸시, 메일, S3)을 넣지 마라. 외부 응답 시간이 그대로 DB 자원 점유 시간이 된다. 상대가 느려지면 우리 DB가 마른다.

---

## 3. `@Transactional`의 실체 — 프록시와 전파

Spring에서는 `BEGIN`/`COMMIT`을 직접 쓰지 않고 애너테이션을 단다.

```kotlin
@Service
class OrderService(private val repository: OrderRepository) {

    @Transactional
    fun placeOrder(command: PlaceOrderCommand) {
        repository.save(...)      // 이 메서드 전체가 트랜잭션 안
        repository.save(...)
    }   // ← 메서드가 정상 종료하면 COMMIT, 예외가 나가면 ROLLBACK
}
```

### 3.1 애너테이션이 어떻게 트랜잭션을 열까 — 프록시

Spring은 `@Transactional`이 붙은 빈을 **프록시(proxy — 대리인 객체)** 로 감싼다. 우리가 주입받는 건 원본이 아니라 이 대리인이다.

```
호출자 ──▶ [프록시]  ──▶ [진짜 OrderService]
             │
             ├─ 메서드 진입 전:  트랜잭션 BEGIN
             ├─ 진짜 메서드 실행
             └─ 메서드 종료 후:  COMMIT (예외면 ROLLBACK)
```

이 구조 때문에 **프록시를 거치지 않는 호출에는 트랜잭션이 안 걸린다.** → [부록 A.3](#a3-self-invocation--같은-클래스-안에서-부르면-왜-안-먹나)

### 3.2 트랜잭션이 이미 열려 있으면? — 전파(propagation)

`@Transactional`이 붙은 메서드가 **다른 `@Transactional` 메서드를 호출**하면 어떻게 될까? 트랜잭션이 두 개 생길까?

이걸 정하는 게 **전파 속성(propagation)** 이다. 자주 쓰는 것만 보면 된다.

| 전파 | 바깥 트랜잭션이 있을 때 | 없을 때 |
|---|---|---|
| `REQUIRED` (**기본값**) | 그 트랜잭션에 **얹혀 탄다** | **새로 연다** |
| `REQUIRES_NEW` | 바깥을 **잠시 멈추고**(suspend) 새로 연다 | 새로 연다 |
| `SUPPORTS` | 얹혀 탄다 | 트랜잭션 없이 실행 |
| `MANDATORY` | 얹혀 탄다 | **예외를 던진다** |
| `NOT_SUPPORTED` | 바깥을 멈추고 트랜잭션 없이 실행 | 트랜잭션 없이 실행 |

### 3.3 `REQUIRED` — 있으면 얹히고, 없으면 내가 연다

이 문서의 주제에서 가장 중요한 게 이 기본값이다. **똑같은 코드가 바깥 상황에 따라 다르게 동작한다.**

Spring 소스(`AbstractPlatformTransactionManager.getTransaction()`)를 보면 분기가 이렇게 생겼다.

```java
if (isExistingTransaction(transaction)) {          // 바깥 트랜잭션이 있나?
    return handleExistingTransaction(...);         // → 있으면 참여 처리로
}
...
else if (def.getPropagationBehavior() == PROPAGATION_REQUIRED || ...) {
    SuspendedResourcesHolder suspendedResources = suspend(null);
    // → 없으면 여기서 새 트랜잭션을 만든다
}
```

그리고 참여할 때는 로그 한 줄만 남기고 **아무것도 새로 열지 않는다.**

```java
// PROPAGATION_REQUIRED, PROPAGATION_SUPPORTS, PROPAGATION_MANDATORY:
// regular participation in existing transaction.
logger.debug("Participating in existing transaction");
```

### 3.4 참여자는 커밋하지 않는다 — 실제 커밋은 바깥 하나뿐

여기가 "안쪽 메서드에 `@Transactional`이 있는데 왜 따로 커밋이 안 되나"의 답이다.

커밋 처리(`processCommit`)의 분기를 보면, **`isNewTransaction()`이 참인 경우에만** 진짜 `doCommit()`을 호출한다.

```java
else if (status.isNewTransaction()) {
    logger.debug("Initiating transaction commit");
    doCommit(status);              // ← 여기서만 실제 COMMIT이 DB로 나간다
}
```

참여자(`isNewTransaction() == false`)는 이 분기를 못 타고 그냥 지나간다. 즉:

```
바깥 @Transactional  BEGIN ─────────────────────────────┐
  안쪽 @Transactional      (참여만. 커밋 안 함)          │
  안쪽 @Transactional      (참여만. 커밋 안 함)          │
                     COMMIT ◀──────────────────────────┘  ← 여기서 한 번에
```

**이것이 0장 사건의 구조다.** 안쪽 서비스들은 각자 `@Transactional`을 달고 있었지만, 바깥에 배치 스텝의 큰 트랜잭션이 있어서 전부 거기에 흡수됐다. 그래서 250명분 쓰기가 잡이 끝나야 한꺼번에 커밋됐다.

> 반대로 말하면, **바깥 트랜잭션만 없애면** 안쪽 `@Transactional`들이 "원래 선언대로" 각자 열고 각자 커밋한다. 쪼개는 코드를 따로 쓸 필요가 없다. 5장~6장이 이 이야기다.

한 가지 더. 참여자가 예외를 던지면 바깥 트랜잭션이 **rollback-only**로 마킹된다(`globalRollbackOnParticipationFailure` 기본값 `true`). 즉 안쪽에서 예외를 `try-catch`로 삼켜도, 바깥은 이미 "이 트랜잭션은 롤백만 가능"으로 오염된 뒤라 마지막에 커밋이 실패한다. **트랜잭션 안에서 예외를 삼키는 코드는 생각만큼 안전하지 않다.**

### 3.5 `REQUIRES_NEW` — 무조건 새로 연다

`REQUIRES_NEW`는 바깥 트랜잭션을 **잠시 멈추고**(suspend) 완전히 별개의 트랜잭션을 연다. 별개라는 건 **다른 DB 커넥션을 쓴다**는 뜻이다.

```
바깥 트랜잭션 (커넥션 #3)  BEGIN ──[suspend]──────────[resume]── COMMIT
  안쪽 REQUIRES_NEW (커넥션 #7)      BEGIN ... COMMIT
                                     ↑ 여기서 즉시 확정된다
```

용도는 "바깥이 롤백돼도 이건 남아야 한다"는 경우다. 감사 로그, 실패 기록, 진행 상태 표시 같은 것.

**주의할 함정 두 가지:**

1. **커넥션을 2개 쓴다.** 커넥션 풀이 작으면 이것만으로도 고갈 위험이 있다.
2. **자기 자신과 교착(deadlock)이 날 수 있다.** 바깥 트랜잭션이 어떤 행에 잠금을 건 채 멈춰 있는데, 안쪽 `REQUIRES_NEW`가 다른 커넥션에서 **같은 행**을 건드리면 영원히 서로를 기다린다. 바깥은 안쪽이 끝나길 기다리고, 안쪽은 바깥이 잠금을 풀길 기다린다.

```
커넥션 #3 (바깥):  UPDATE rows(id=5)   ← 잠금 보유, suspend 상태
커넥션 #7 (안쪽):  UPDATE rows(id=5)   ← 잠금 대기... 영원히
```

---

## 4. 트랜잭션 매니저 — 전략 인터페이스와 구현체들

### 4.1 `PlatformTransactionManager`는 인터페이스다

지금까지 "트랜잭션을 연다/닫는다"고 했는데, 실제로 그 일을 하는 객체가 **트랜잭션 매니저**다. Spring은 이걸 인터페이스로 추상화했다.

```java
package org.springframework.transaction;   // spring-tx

public interface PlatformTransactionManager extends TransactionManager {
    TransactionStatus getTransaction(TransactionDefinition definition);
    void commit(TransactionStatus status);
    void rollback(TransactionStatus status);
}
```

`@Transactional`은 이 인터페이스에 대고 일을 시킨다. **어떤 구현체가 꽂혀 있는지는 모른다.** 그래서 JPA를 쓰든 JDBC를 쓰든 Kafka를 쓰든 애너테이션 코드는 똑같다. 전형적인 **전략 패턴(strategy pattern)** 이다.

### 4.2 구현체들 — 무엇을 "붙잡느냐"가 다르다

| 구현체 | 소속 라이브러리 | 무엇을 관리하나 |
|---|---|---|
| `DataSourceTransactionManager` | spring-jdbc | JDBC `Connection` 하나 |
| `JpaTransactionManager` | spring-orm | JPA `EntityManager` + 그 밑의 JDBC 커넥션 |
| `JtaTransactionManager` | spring-tx | 여러 리소스에 걸친 분산 트랜잭션(XA) |
| `KafkaTransactionManager` | spring-kafka | Kafka 프로듀서 트랜잭션 |
| `ResourcelessTransactionManager` | **spring-batch-infrastructure** | **아무것도 관리하지 않음** ← 5장 |

패키지가 여기저기 흩어져 있는 건 **"인터페이스는 코어에, 구현체는 해당 기술 모듈에"** 라는 Spring의 일관된 방식 때문이다. 헷갈리기 쉬운 지점이 하나 있다.

```kotlin
// 선언 타입은 spring-tx의 인터페이스
private val transactionManager: PlatformTransactionManager

// 하지만 실제로 주입되는 객체는 spring-orm의 JpaTransactionManager
```

즉 "spring-tx의 매니저를 쓴다"는 표현은 정확하지 않다. **선언은 spring-tx 인터페이스, 실제 구현체는 다른 모듈**인 경우가 대부분이다.

### 4.3 누가 구현체를 정하나 — Spring Boot 자동설정

Spring Boot는 클래스패스와 빈 상황을 보고 알아서 하나 꽂아준다. JPA를 쓰면 `JpaBaseConfiguration`이 이렇게 등록한다.

```java
@Bean
@ConditionalOnMissingBean(TransactionManager.class)     // ← 7장의 함정이 여기 있다
public PlatformTransactionManager transactionManager(...) {
    return new JpaTransactionManager();
}
```

`@ConditionalOnMissingBean(TransactionManager.class)`는 **"`TransactionManager` 타입 빈이 하나도 없을 때만 이걸 등록한다"** 는 뜻이다. 7장에서 이게 왜 위험한지 다룬다.

---

## 5. `ResourcelessTransactionManager` — 아무것도 잡지 않는 트랜잭션 매니저

### 5.1 소스가 전부 말해준다

Spring Batch가 제공하는 이 구현체는 이름 그대로다. **resource-less — 관리할 리소스가 없다.** 핵심 부분만 옮기면 이렇다.

```java
package org.springframework.batch.support.transaction;   // spring-batch-infrastructure 5.x

public class ResourcelessTransactionManager extends AbstractPlatformTransactionManager {

    @Override
    protected void doBegin(Object transaction, TransactionDefinition definition) {
        ((ResourcelessTransaction) transaction).begin();     // boolean active = true 로 세팅
    }

    @Override
    protected void doCommit(DefaultTransactionStatus status) {
        logger.debug("Committing resourceless transaction on [" + ... + "]");   // 로그가 전부
    }

    @Override
    protected void doRollback(DefaultTransactionStatus status) {
        logger.debug("Rolling back resourceless transaction on [" + ... + "]"); // 로그가 전부
    }

    @Override
    protected void doSetRollbackOnly(DefaultTransactionStatus status) { }        // 비어 있음
}
```

`doCommit`과 `doRollback`이 **로그 한 줄이 전부**다. DataSource도 EntityManager도 스레드에 바인딩하지 않는다. `PlatformTransactionManager` 계약만 형식적으로 만족시키는 **빈 껍데기**다.

### 5.2 그래서 어떤 효과가 나나

이걸 스텝의 트랜잭션 매니저로 넣으면 이렇게 된다.

| | `JpaTransactionManager` | `ResourcelessTransactionManager` |
|---|---|---|
| 스텝 본문을 감싸는 DB 트랜잭션 | **있다** | **없다** |
| DB 커넥션 확보 | 한다 | 안 한다 |
| 본문 안의 `@Transactional`(REQUIRED) | 참여만 하고 커밋 안 함 | **바깥이 없으니 스스로 열고 스스로 커밋** |
| 커밋/롤백 | 실제 SQL 발행 | 로그만 |

세 번째 줄이 핵심이다. 3.3에서 본 대로 `REQUIRED`는 **바깥이 없으면 스스로 연다.** 스텝의 껍데기 트랜잭션을 없애는 것만으로, 안쪽 메서드들이 자기 경계를 되찾는다.

```
[이전]  스텝 트랜잭션 있음
        BEGIN ────────────────────────────────────── (전체 96초)
          회원1 저장+푸시   회원2 저장+푸시  ... 회원N
        COMMIT

[이후]  스텝 트랜잭션 없음(resourceless)
        회원1: [BEGIN 저장 COMMIT] → [BEGIN 푸시·상태갱신 COMMIT] → [BEGIN 완료표시 COMMIT]
        회원2: ...
               └ 각 트랜잭션이 0.4초 남짓. 커넥션과 잠금이 그 길이만큼만 붙잡힌다
```

**"트랜잭션을 쪼개는 코드"를 한 줄도 쓰지 않았다.** 껍데기를 걷어냈더니 원래 선언이 살아난 것뿐이다.

### 5.3 버전 주의 — Spring Batch 6에서 패키지가 옮겨진다

jar를 열어 확인한 실제 경로다.

```
Spring Batch 5.2.3 : org/springframework/batch/support/transaction/ResourcelessTransactionManager.class
Spring Batch 6.0.2 : org/springframework/batch/infrastructure/support/transaction/ResourcelessTransactionManager.class
```

Batch 6로 올릴 때 import 수정이 필요하다. 이런 건 기억에 의존하지 말고 **쓰고 있는 버전의 jar나 공식 문서로 확인**하는 게 맞다.

---

## 6. 배치에서 어떤 트랜잭션 매니저를 고를 것인가

### 6.1 먼저, 스텝에는 두 종류가 있다

Spring Batch의 잡(job)은 스텝(step)의 나열이고, 스텝은 두 가지 방식 중 하나로 만든다.

**(1) 청크 지향 스텝(chunk-oriented)** — "읽고 → 가공하고 → 묶어서 쓰기"를 반복

```kotlin
StepBuilder("importUsersStep", jobRepository)
    .chunk<UserCsvRow, User>(100, transactionManager)   // 100건마다 커밋
    .reader(csvReader)
    .processor(rowToEntityProcessor)
    .writer(jpaItemWriter)
    .build()
```

100건을 모아 쓰고 커밋, 다음 100건 쓰고 커밋 — 이 **"청크 단위 커밋"이 Spring Batch의 간판 기능**이다.

**(2) 태스클릿 스텝(tasklet)** — "그냥 이 코드 한 덩어리를 실행해라"

```kotlin
StepBuilder("cleanupStep", jobRepository)
    .tasklet({ _, _ ->
        doSomething()
        RepeatStatus.FINISHED
    }, transactionManager)
    .build()
```

두 방식 모두 `transactionManager`를 인자로 받고, 그게 **스텝 본문을 감싸는 트랜잭션**을 결정한다.

### 6.2 기본값은 데이터소스 트랜잭션 매니저다

**청크 스텝에서는 선택지가 없다.** 청크 커밋 자체가 스텝 트랜잭션 위에 서 있기 때문이다. 예를 들어 `JpaItemWriter`는 트랜잭션에 묶인 `EntityManager`를 요구하고, 없으면 이렇게 죽는다.

```java
// JpaItemWriter.doWrite()
EntityManager entityManager = EntityManagerFactoryUtils.getTransactionalEntityManager(entityManagerFactory);
if (entityManager == null) {
    throw new DataAccessResourceFailureException("Unable to obtain a transactional EntityManager");
}
```

즉 **청크 스텝에 `ResourcelessTransactionManager`를 넣으면 안 된다.** 커밋이 안 되는 정도가 아니라 writer가 예외로 죽는다.

### 6.3 선택 기준

| 스텝이 하는 일 | 골라야 할 것 | 이유 |
|---|---|---|
| **청크 지향** + DB 쓰기 | **데이터소스 TM** (선택지 없음) | 청크 커밋과 writer가 트랜잭션을 요구 |
| 태스클릿이 **DB 쓰기를 직접** 하고 "전부 아니면 전무"가 필요 | **데이터소스 TM** | 롤백 단위가 태스클릿 전체여야 함 |
| 태스클릿이 **DB를 안 씀** (파일 이동, S3 업로드, 외부 API 호출, 알림 발송) | **Resourceless** | 잡을 리소스가 없는데 커넥션을 여는 건 낭비 |
| 태스클릿이 DB를 쓰지만 **안쪽 서비스가 자기 `@Transactional`로 경계를 소유** | **Resourceless** | 경계 주인이 이미 안쪽에 있음 |
| 루프 안에 **장시간 외부 호출**이 섞임 | **Resourceless** + 항목 단위 경계 | 외부 응답 시간을 DB 점유에서 분리 |

> **판단 규칙 한 줄:** **"이 스텝의 롤백 단위가 스텝 전체인가?"**
> 그렇다 → 데이터소스 TM. 아니다(안쪽이 경계를 갖거나, 애초에 DB를 안 쓴다) → Resourceless.

마지막 두 줄이 0장 사건이다. 태스클릿 안의 서비스들이 이미 각자 `@Transactional`을 갖고 있었는데 바깥에 껍데기를 하나 더 씌운 상태였고, 그 껍데기가 안쪽 경계를 전부 흡수해 96초짜리 트랜잭션을 만들었다.

### 6.4 대가 — 배치 메타데이터와의 원자성을 잃는다

Spring Batch는 잡·스텝의 실행 이력을 **메타데이터 테이블**(`BATCH_JOB_EXECUTION`, `BATCH_STEP_EXECUTION` 등)에 기록한다. 이 기록을 담당하는 게 `JobRepository`다.

`TaskletStep`은 이 갱신을 **스텝 트랜잭션 안에서** 호출한다.

```java
// TaskletStep.ChunkTransactionCallback 내부
getJobRepository().update(stepExecution);   // ← 스텝 트랜잭션 안
```

스텝 트랜잭션이 진짜 DB 트랜잭션이면 "비즈니스 데이터 + 메타데이터"가 한 덩어리로 커밋/롤백된다. Resourceless로 바꾸면 이 원자성이 **사라진다.** 잡이 중간에 죽었을 때 "메타데이터는 진행 중이라 하는데 실제 데이터는 절반 처리됨" 같은 어긋남이 가능해진다.

다만 두 가지를 알아둘 것.

1. **메타데이터 자체는 정상 기록된다.** `JobRepository`는 자기 트랜잭션 매니저를 따로 갖는다(Spring Boot가 컨텍스트의 TM을 주입). 스텝 TM이 Resourceless여도 `JobRepository`의 `@Transactional`이 자기 트랜잭션을 열고 커밋한다.
2. **애초에 원자성이 없던 잡도 많다.** 진행 상태를 `REQUIRES_NEW`로 즉시 커밋하는 구조(3.5)라면, 그 잡은 이미 메타데이터 원자성을 포기한 상태다. 이 경우 Resourceless로 바꿔도 새로 잃는 게 없다.

---

## 7. 조용히 깨지는 함정 — 트랜잭션 매니저를 빈으로 올리지 말 것

4.3에서 본 자동설정 조건을 다시 보자.

```java
@Bean
@ConditionalOnMissingBean(TransactionManager.class)
public PlatformTransactionManager transactionManager(...) {
    return new JpaTransactionManager();
}
```

`ResourcelessTransactionManager`를 편의상 이렇게 등록했다고 하자.

```kotlin
@Bean
fun resourcelessTransactionManager() = ResourcelessTransactionManager()   // ⚠️ 절대 금지
```

이 순간 컨텍스트에 `TransactionManager` 타입 빈이 **생겼으므로**, 자동설정이 백오프해서 **`JpaTransactionManager`가 아예 만들어지지 않는다.** 그러면 애플리케이션의 모든 `@Transactional`이 이 빈 껍데기 매니저를 받는다.

```
결과: 모든 @Transactional 이 "커밋 = 로그 한 줄" 이 된다
      → 쓰기가 커밋되지 않거나, 롤백이 동작하지 않거나, 예상 밖으로 자동커밋된다
```

**가장 나쁜 점은 아무 예외도 안 난다는 것이다.** 빈이 하나뿐이라 `NoUniqueBeanDefinitionException`도 없고, 기동 로그도 깨끗하다. 조용히 잘못 동작한다.

**대응 두 가지:**

```kotlin
// 1) 빈으로 올리지 말고 필요한 스텝 안에서 인라인 생성
@Bean
fun myTasklerStep(): Step =
    StepBuilder("myTaskletStep", jobRepository)
        .tasklet(myTasklet(), ResourcelessTransactionManager())   // ← 이 스텝 전용
        .build()
```

```kotlin
// 2) 회귀를 테스트로 잠근다 (다음 사람이 되돌리는 걸 막는다)
@Test
fun `컨텍스트의 PlatformTransactionManager 는 여전히 JpaTransactionManager 다`() {
    assertInstanceOf(JpaTransactionManager::class.java, transactionManager)
}
```

> 꼭 빈으로 등록해야 한다면 `@Qualifier`로 구분하고 자동설정 쪽을 `@Primary`로 지정하는 방법도 있지만, **"스텝 하나에만 필요한 걸 전역 빈으로 올리지 않는다"** 가 더 단순하고 안전하다.

---

## 8. 경계를 바꾸면 딸려오는 것들

트랜잭션 경계는 생각보다 많은 것의 **전제**다. 경계를 바꾸면 직접 건드리지 않은 동작이 함께 바뀐다. 대표적인 두 가지.

### 8.1 읽기 전용 라우팅이 갑자기 동작하기 시작한다

읽기 부하를 분산하려고 **읽기 전용 복제본(read replica)** 으로 SELECT를 보내는 구성이 흔하다. Spring에서는 보통 `AbstractRoutingDataSource`로 이렇게 만든다.

```kotlin
class ReplicationRoutingDataSource : AbstractRoutingDataSource() {
    override fun determineCurrentLookupKey(): Any =
        if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) "READ_ONLY" else "READ_WRITE"
}
```

이 "현재 트랜잭션이 읽기 전용인가" 플래그는 **새 트랜잭션을 시작할 때** 세팅된다.

```java
// AbstractPlatformTransactionManager.prepareSynchronization()
TransactionSynchronizationManager.setCurrentTransactionReadOnly(definition.isReadOnly());
```

여기서 중요한 게 **참여자는 이 코드를 안 탄다**는 점이다. 3.4에서 본 것처럼 참여자는 새 트랜잭션을 만들지 않으니까.

```
[바깥 쓰기 트랜잭션이 있을 때]
  @Transactional(readOnly = true) 메서드 → 쓰기 트랜잭션에 참여 → 플래그 안 바뀜 → 라이터로 감

[바깥 트랜잭션이 없어졌을 때]
  @Transactional(readOnly = true) 메서드 → 자기 트랜잭션을 새로 엶 → 플래그 걸림 → 리플리카로 감
```

즉 `readOnly = true` 선언은 그대로인데, **바깥 트랜잭션을 걷어낸 순간 처음으로 선언대로 동작하기 시작한다.** 코드를 한 글자도 안 바꿨는데 SQL이 다른 DB 인스턴스로 나간다.

이게 좋은 일인지 나쁜 일인지는 상황에 따른다. 확인할 것:

- **복제 지연** — 방금 쓴 걸 곧바로 읽는 경로가 있으면 못 읽을 수 있다. "쓰고 → 리플리카에서 읽는" 구간이 코드 흐름에 실제로 존재하는지 본다
- **가용성 의존** — 리플리카가 죽거나 페일오버 중이면 그 읽기가 실패한다. 없던 실패 지점이 하나 생긴다

### 8.2 커밋 후 콜백이 터지는 시점이 달라진다

"트랜잭션이 커밋된 뒤에만 실행하라"는 이벤트 리스너가 있다.

```kotlin
@TransactionalEventListener   // 기본 phase = AFTER_COMMIT
fun on(event: OrderPlacedEvent) {
    mailSender.send(...)      // 커밋이 확정된 뒤에만 메일을 보낸다
}
```

`@TransactionalEventListener`의 기본 phase는 `AFTER_COMMIT`이다(소스: `TransactionPhase phase() default TransactionPhase.AFTER_COMMIT`). 그리고 **"커밋"은 실제로 커밋하는 그 트랜잭션 하나**를 말한다(3.4).

```
[스텝 전체가 한 트랜잭션]  항목1 이벤트 ┐
                          항목2 이벤트 ├─ 전부 대기 → 잡이 끝날 때 한꺼번에 발화
                          항목N 이벤트 ┘

[항목 단위 트랜잭션]       항목1 이벤트 → 항목1 커밋 직후 발화
                          항목2 이벤트 → 항목2 커밋 직후 발화
```

리스너가 메일·푸시·외부 연동을 한다면 **발송 타이밍과 묶음 크기가 통째로 바뀐다.** 리스너 코드는 그대로인데 관측되는 동작이 달라지므로, 경계를 바꿀 때 반드시 확인할 항목이다.

---

## 9. 트랜잭션을 쪼개면 새로 생기는 위험 — 부분 커밋

트랜잭션을 쪼개면 좋기만 한 게 아니다. **"전부 아니면 전무"를 포기하는 것**이므로, 중간 상태가 새로 생긴다.

항목 하나를 처리하는 흐름이 이렇다고 하자.

```
① [BEGIN 데이터 저장 COMMIT]
② [BEGIN 외부 API 호출 COMMIT]
③ [BEGIN 처리완료 표시 COMMIT]
```

**① 이후 ③ 이전에 프로세스가 죽으면** 그 항목은 "데이터는 저장됐는데 상태는 미처리"로 남는다. 재실행하면 미처리 항목을 다시 집어서 **중복 처리**한다.

쪼개기 전과 비교하면 실패의 **방향과 범위**가 이렇게 뒤바뀐다.

| | 쪼개기 전 (스텝 = 1 트랜잭션) | 쪼갠 후 (항목 단위) |
|---|---|---|
| 취약 창 | 스텝 전체 (예: 96초) | 항목 1건 안 (예: 0.4초) |
| 크래시 시 영향 범위 | 그때까지 처리한 K건 전부 | **최대 1건** |
| 실패 방향 | 데이터 유실 (롤백되는데 외부 호출은 이미 나감) | 중복 (데이터·호출 둘 다 재실행) |
| 복구 | 어려움 (상태는 완료인데 데이터가 없음) | 불필요하거나 쉬움 |

**대체로 쪼개는 쪽이 낫다.** 창이 짧아지고 범위가 좁아지며, "유실"이 "중복"으로 바뀐다. 유실은 아무도 모르게 지나가지만 중복은 눈에 띈다.

그래도 중복이 곤란하면 선택지는 이렇다.

1. **멱등성(idempotency) 확보** — 처리 키에 유니크 제약을 걸어 재실행 시 중복 삽입이 실패하게 한다. 가장 견고하다
2. **저장과 완료표시를 한 트랜잭션으로** — ①과 ③을 묶으면 중복 창이 사라지고 대신 "외부 호출만 유실"로 바뀐다. 단 이러면 배치가 다른 모듈의 트랜잭션 경계를 소유하게 되어 계층 경계가 흐려진다
3. **수용하고 문서화** — 창이 충분히 좁고 재실행이 드물면 합리적인 선택이다. 다만 **"중복 없음"이라고 잘못 적어두지 말 것.** 운영 문서에 들어간 거짓 안전 주장은 나중에 판단을 망친다

---

## 10. 트랜잭션 경계를 테스트로 잡는 법

트랜잭션 경계 변경은 **단위 테스트로 안 잡힌다.** 목(mock)을 쓰면 트랜잭션 자체가 없기 때문이다. 두 층으로 나눠서 잡는다.

### 10.1 설정 회귀를 잡는 테스트 (값싸다)

"다음 사람이 되돌리는 것"을 막는다. 컨텍스트만 띄우면 되므로 빠르다.

```kotlin
@Test
fun `스텝의 트랜잭션 매니저가 ResourcelessTransactionManager 인가`() {
    // TaskletStep 에는 트랜잭션 매니저 getter 가 없어(private 필드 + setter 만) 리플렉션으로 꺼낸다
    val tm = ReflectionTestUtils.getField(step, "transactionManager")
    assertInstanceOf(ResourcelessTransactionManager::class.java, tm)
}

@Test
fun `컨텍스트의 PlatformTransactionManager 는 여전히 JpaTransactionManager 인가`() {
    assertInstanceOf(JpaTransactionManager::class.java, transactionManager)   // 7장의 함정 방어
}
```

### 10.2 경계 자체를 잡는 통합 테스트 (판별력이 핵심)

**"중간에 죽었을 때 앞선 항목의 데이터가 살아남는가"** 를 본다. 쪼개기 전이면 롤백돼 0건, 쪼갠 후면 N건이다.

여기서 놓치기 쉬운 함정이 하나 있다. 대부분의 배치 루프는 항목별 예외를 `catch`해서 삼킨다.

```kotlin
items.forEach { item ->
    try {
        process(item)
    } catch (e: Exception) {      // ← 여기서 다 삼킨다
        markFailed(item, e)
    }
}
```

그래서 **"실패하는 항목을 하나 넣는" 테스트는 판별력이 없다.** 스텝이 예외를 밖으로 안 던지니 쪼개기 전에도 바깥 트랜잭션이 정상 커밋되고, 테스트가 양쪽 다 통과한다.

가르려면 `catch (Exception)`을 **통과하는** 것을 던져야 한다. 자바/코틀린에서 `Exception`과 `Error`는 형제이므로(둘 다 `Throwable`의 자식), `Error`를 던지면 catch에 안 걸린다.

```kotlin
@TestConfiguration
class CrashInjectingConfig {
    @Bean @Primary
    fun failingPort(real: RealPort): Port = object : Port {
        var calls = 0
        override fun process(item: Item) {
            real.process(item)                 // 진짜 처리를 먼저 시킨다 (커밋이 진짜여야 하므로)
            if (++calls == FAIL_AT) {
                throw AssertionError("주입된 크래시")   // Exception 이 아니라 Error
            }
        }
    }
}

@Test
fun `루프 도중 죽어도 앞선 항목의 데이터는 커밋돼 남는다`() {
    seed(itemCount = 3)
    failingPort.failAtCall = 2

    val stepExecution = runStep()

    assertEquals(BatchStatus.FAILED, stepExecution.status)
    assertEquals(2, repository.count())   // 쪼개기 전이면 0건이 되어 반드시 실패한다
}
```

**꼭 확인할 것: 이 테스트가 실제로 "쪼개기 전"에서 실패하는가.** 설정을 잠시 되돌려 돌려보고, 빨간불이 뜨는 걸 눈으로 본 다음 복구한다. 이걸 안 하면 아무것도 검증하지 못하는 테스트를 갖게 될 수 있다.

한 가지 더 — 스텝의 트랜잭션 매니저가 검증 대상이므로 **서비스를 직접 호출하면 안 되고 반드시 스텝을 태워야** 한다.

```kotlin
private fun runStep(): StepExecution {
    val jobExecution = jobRepository.createJobExecution("testJob", uniqueParams())
    val stepExecution = jobExecution.createStepExecution("myStep")
    jobRepository.add(stepExecution)
    step.execute(stepExecution)      // AbstractStep.execute 는 Throwable 을 잡아 FAILED 로 기록하고 재던지지 않는다
    return stepExecution
}
```

### 10.3 실제 DB에서 확인하는 법 (A/B)

테스트 DB(H2 등)로 안 잡히거나 확신이 필요하면, **잡이 도는 중에 별도 커넥션으로 관찰**한다.

```bash
# 잡 실행과 동시에 다른 세션에서 폴링
while :; do
  echo "$(date +%T) $(mysql -N -B -e 'SELECT COUNT(*) FROM target_table')"
  sleep 0.1
done
```

| 관측 | 쪼갠 후 | 쪼개기 전 |
|---|---|---|
| 다른 세션에서 본 행 수 | `0 → 1 → 2` **단계적** | `0 → 2` 잡 종료 후 한 번에 |

**중간값이 다른 커넥션에서 보인다는 것 자체가 결정적 증거다.** 커밋되지 않은 데이터는 다른 커넥션에서 안 보이므로, 중간값이 보였다는 건 그 시점에 이미 커밋됐다는 뜻이다. 그리고 커밋됐다는 건 **행 잠금도 그때 풀렸다**는 뜻이라(2.3), 잠금 보유 시간까지 같이 증명된다.

---

## 부록 A

### A.1 커넥션 풀이 마른다는 것

애플리케이션은 DB 커넥션을 요청마다 새로 만들지 않는다. TCP 연결과 인증에 수십 ms가 들기 때문에, 미리 몇 개 만들어두고 돌려 쓴다. 이 보관함이 **커넥션 풀(connection pool)** 이다(Spring Boot 기본은 HikariCP).

```
[커넥션 풀]  #1 #2 #3 #4        ← 크기 4로 설정된 경우

요청 A ──▶ #1 빌림 ──▶ 쿼리 ──▶ #1 반납
요청 B ──▶ #2 빌림 ...
```

핵심 성질 두 가지.

1. **풀 크기는 유한하다.** 크게 잡는다고 좋아지지 않는다. DB 쪽 최대 커넥션 수와 CPU가 한계이고, 너무 크면 DB가 컨텍스트 스위칭으로 느려진다. 웹 애플리케이션에서 인스턴스당 10~20 정도가 흔하고, 배치나 소규모 서비스는 더 작다
2. **다 빌려간 상태에서 더 요청하면 기다린다.** 설정된 시간(Hikari 기본 `connectionTimeout` 30초) 안에 못 받으면 예외가 나고, 보통 그대로 HTTP 500이 된다

```
SQLTransientConnectionException: HikariPool-1 - Connection is not available,
request timed out after 30000ms
```

여기서 무서운 건 **연쇄**다. 커넥션을 기다리는 요청은 자기 스레드를 붙잡고 있고, 그 스레드가 쌓이면 결국 **DB와 무관한 API까지 응답하지 못한다.** 하나의 느린 트랜잭션이 서비스 전체 장애로 번지는 전형적인 경로다.

### A.2 행 잠금은 언제 잡히고 언제 풀리나

**잡히는 시점** — 해당 SQL이 실행되는 순간이다. 커밋 때가 아니다.

```sql
BEGIN;
UPDATE rows SET x = 1 WHERE id = 5;   -- ← 여기서 id=5 행에 배타 잠금
-- ... 다른 작업 ...                    -- 그동안 계속 잠금 보유
COMMIT;                                -- ← 여기서 해제
```

**풀리는 시점** — 커밋 또는 롤백. **중간에 자동으로 풀리지 않는다.** 이게 2.4의 근거다. 트랜잭션 안에서 UPDATE를 일찍 하고 외부 API를 오래 호출하면, 그 잠금이 API 응답 시간만큼 유지된다.

**대기와 타임아웃** — 다른 트랜잭션이 같은 행을 건드리면 대기한다. MySQL InnoDB는 `innodb_lock_wait_timeout`(기본 50초)을 넘기면 에러를 던진다.

```
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```

**교착(deadlock)** — 두 트랜잭션이 서로가 가진 잠금을 기다리면 영원히 안 끝난다. DB가 이를 감지해 한쪽을 죽인다(MySQL 에러 1213). 3.5의 `REQUIRES_NEW` 자기 교착도 같은 원리인데, 이건 **한 요청 안에서** 두 커넥션이 서로를 기다리는 형태라 더 헷갈린다.

**실전 원칙:** 잠금을 잡는 SQL은 트랜잭션의 **가능한 뒤쪽**에 두고, 잡은 뒤에는 **빨리 커밋한다.** 특히 잠금 보유 상태에서 외부 호출을 하지 않는다.

### A.3 self-invocation — 같은 클래스 안에서 부르면 왜 안 먹나

3.1에서 본 대로 `@Transactional`은 프록시가 구현한다. 그래서 **프록시를 거치지 않는 호출에는 안 걸린다.**

```kotlin
@Service
class ReportService {

    fun generateAll(ids: List<Long>) {
        ids.forEach { generateOne(it) }      // ⚠️ 프록시를 안 거친다 → 트랜잭션 없음
    }

    @Transactional
    fun generateOne(id: Long) { ... }
}
```

`generateAll` 안의 `generateOne(it)`은 `this.generateOne(it)`이다. `this`는 **프록시가 아니라 진짜 객체**라서 애너테이션이 무시된다.

```
호출자 ──▶ [프록시] ──▶ [진짜 ReportService]
                          generateAll()
                            └─ this.generateOne()   ← 프록시를 우회
```

**해결책:**

1. **클래스를 분리한다** (가장 깔끔) — `generateOne`을 다른 빈으로 옮기고 주입받는다
2. **자기 자신을 주입받는다** — `@Lazy private val self: ReportService`로 받아 `self.generateOne()` 호출. 동작하지만 의도가 안 드러나 권장하지 않는다
3. **`TransactionTemplate`을 직접 쓴다** — 애너테이션 대신 프로그래밍 방식으로 경계를 만든다

```kotlin
fun generateAll(ids: List<Long>) {
    ids.forEach { id ->
        transactionTemplate.execute { generateOne(id) }   // 경계가 코드에 명시적으로 보인다
    }
}
```

**같은 이유로 `private` 메서드에도 `@Transactional`이 안 먹는다.** 프록시가 오버라이드할 수 없기 때문이다.
