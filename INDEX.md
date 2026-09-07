# Study Dev Index

개발 일반 학습 노트 — **카테고리로 찾는다.** 각 문서는 배경 지식 없이 읽을 수 있게 쓴다(작성 규칙: [AGENTS.md](AGENTS.md)).

> 규모 확장 규칙: 카테고리는 처음엔 한 파일의 **섹션**으로 시작 → 내용이 쌓이면 **독립 파일**로 분리 → 파일이 여럿 되면 **폴더**로 승격. 분리·승격 시 이 인덱스의 링크만 갱신하면 된다.

## 네트워크 · 실시간 통신
- HTTP의 한계와 keep-alive — [2장](realtime-web-networking-cors-infra.md#2-먼저-http부터--왜-실시간-통신이-어려운가) · [부록 A.1~A.2](realtime-web-networking-cors-infra.md#a1-http는-응답이-끝나면-연결을-닫는다)
- WebSocket (Upgrade 핸드셰이크, ws/wss, 하트비트) — [3장](realtime-web-networking-cors-infra.md#3-websocket--한-번-연결해두고-계속-쓰는-방식)
- STOMP (WebSocket 위의 메시징 규약) — [4장](realtime-web-networking-cors-infra.md#4-stomp--websocket-위에-얹는-대화-규칙)
- SockJS 폴백 사다리 (XHR streaming/polling, 폴백의 대가) — [6장](realtime-web-networking-cors-infra.md#6-sockjs--websocket이-막히면-http로-흉내내기)

## 웹 보안
- CORS (동작 원리, 헤더 사전, WebSocket이 안 걸리는 이유) — [7장](realtime-web-networking-cors-infra.md#7-cors--브라우저가-남의-집-응답을-못-읽게-막는-규칙)

## 인프라
- 프록시 · TLS 인터셉션 — 사내망이 WebSocket을 막는 이유 — [5장](realtime-web-networking-cors-infra.md#5-왜-사내망에서-websocket이-막히나)
- 로드밸런서와 sticky session — [9장](realtime-web-networking-cors-infra.md#9-로드밸런서와-세션-고정sticky-session)
- WAF — [10장](realtime-web-networking-cors-infra.md#10-waf--웹-방화벽)

## 전송 계층 기초
- TCP 연결 종료(4-way, TIME_WAIT), 커널/앱 경계, 프록시 절단, 실패 판정 — [부록 A](realtime-web-networking-cors-infra.md#a1-http는-응답이-끝나면-연결을-닫는다)

## Spring · 트랜잭션
- 트랜잭션의 기초 (커넥션·행 잠금·커밋, 오래 열린 트랜잭션이 위험한 이유) — [2장](spring-transaction-boundaries-and-batch.md#2-트랜잭션이란-무엇인가--커넥션-잠금-커밋)
- `@Transactional`의 실체 (프록시, 전파, 참여자는 커밋하지 않는다) — [3장](spring-transaction-boundaries-and-batch.md#3-transactional의-실체--프록시와-전파)
- 트랜잭션 매니저 (전략 인터페이스와 구현체들, Boot 자동설정) — [4장](spring-transaction-boundaries-and-batch.md#4-트랜잭션-매니저--전략-인터페이스와-구현체들)
- 트랜잭션 매니저를 빈으로 올리면 조용히 깨지는 함정 — [7장](spring-transaction-boundaries-and-batch.md#7-조용히-깨지는-함정--트랜잭션-매니저를-빈으로-올리지-말-것)
- 경계를 바꾸면 딸려오는 것 (readOnly 리플리카 라우팅, 커밋 후 콜백 시점) — [8장](spring-transaction-boundaries-and-batch.md#8-경계를-바꾸면-딸려오는-것들)
- self-invocation 함정, `private` 메서드에 안 먹는 이유 — [부록 A.3](spring-transaction-boundaries-and-batch.md#a3-self-invocation--같은-클래스-안에서-부르면-왜-안-먹나)

## Spring Batch
- 청크 스텝 vs 태스클릿 스텝 — [6.1](spring-transaction-boundaries-and-batch.md#61-먼저-스텝에는-두-종류가-있다)
- `ResourcelessTransactionManager` (소스로 보는 실체, 버전별 패키지 이동) — [5장](spring-transaction-boundaries-and-batch.md#5-resourcelesstransactionmanager--아무것도-잡지-않는-트랜잭션-매니저)
- 배치 트랜잭션 매니저 선택 기준, 메타데이터 원자성이라는 대가 — [6장](spring-transaction-boundaries-and-batch.md#6-배치에서-어떤-트랜잭션-매니저를-고를-것인가)
- 트랜잭션 경계 변경을 테스트로 잡는 법 (판별력 있는 통합 테스트, 실 DB A/B) — [10장](spring-transaction-boundaries-and-batch.md#10-트랜잭션-경계를-테스트로-잡는-법)

## JVM 메모리 · 컨테이너
- 종료 코드 읽는 법 (128+N, 137이 특별한 이유) — [2장](jvm-memory-and-container-limits.md#2-프로세스가-죽는다는-것--종료-코드부터) · [부록 A.2](jvm-memory-and-container-limits.md#a2-종료-코드-빠른-참조)
- 힙 밖의 메모리 (Metaspace·code cache·스레드 스택·direct buffer) — [4장](jvm-memory-and-container-limits.md#4-jvm이-쓰는-메모리는-힙만이-아니다) · [부록 A.4](jvm-memory-and-container-limits.md#a4-왜-메모리-사용량rss이-힙보다-큰가)
- 컨테이너 안의 JVM (UseContainerSupport, MaxRAMPercentage와 그 기본값) — [5장](jvm-memory-and-container-limits.md#5-컨테이너-안의-jvm은-자기-몫을-어떻게-아나)
- **메모리 고갈의 두 가지 방식 — 안에서 터지나 밖에서 죽나** — [6장](jvm-memory-and-container-limits.md#6-메모리가-떨어지는-두-가지-방식--이-문서의-핵심)
- 조용한 정체의 해부 (힙 상한 > 컨테이너 여유, GC 스래싱, 페이지 회수) — [7장](jvm-memory-and-container-limits.md#7-왜-조용한-정체가-생기나)
- 즉시 실패하게 만들기 (`ExitOnOutOfMemoryError`, OOM의 흔적이 바뀐다) — [8장](jvm-memory-and-container-limits.md#8-정체-대신-즉시-실패하게-만드는-법)
- JVM 옵션 우선순위 함정 (`JAVA_TOOL_OPTIONS` vs 커맨드라인) — [9장](jvm-memory-and-container-limits.md#9-같은-옵션을-두-군데서-주면-어느-쪽이-이기나)
- 컨테이너 크기가 GC 선택을 바꾼다 — [10장](jvm-memory-and-container-limits.md#10-컨테이너에서는-gc-종류도-달라진다)
- cgroup 기초 — [부록 A.1](jvm-memory-and-container-limits.md#a1-cgroup--리눅스가-자원-한도를-거는-방법)
- Metaspace 두 옵션의 차이 — [부록 A.3](jvm-memory-and-container-limits.md#a3-metaspace의-두-옵션은-이름만-비슷하고-역할이-다르다)

## 운영 관측성
- 사후 조사를 위해 미리 남겨야 하는 것 (GC 로그, 알람 조건, 오케스트레이터 이벤트 보관 기간) — [11장](jvm-memory-and-container-limits.md#11-사후에-알-수-있으려면-무엇을-남겨야-하나)

## DB 동시성 · 자원
- 커넥션 풀이 마른다는 것 (Hikari, 연쇄 장애) — [부록 A.1](spring-transaction-boundaries-and-batch.md#a1-커넥션-풀이-마른다는-것)
- 행 잠금의 수명, lock wait timeout, 교착 — [부록 A.2](spring-transaction-boundaries-and-batch.md#a2-행-잠금은-언제-잡히고-언제-풀리나)
- 트랜잭션을 쪼갤 때 생기는 부분 커밋 위험과 멱등성 — [9장](spring-transaction-boundaries-and-batch.md#9-트랜잭션을-쪼개면-새로-생기는-위험--부분-커밋)

---
| 문서 | 다루는 카테고리 | 작성/갱신 |
|---|---|---|
| [realtime-web-networking-cors-infra.md](realtime-web-networking-cors-infra.md) | 실시간 통신 · 웹 보안(CORS) · 인프라 · TCP 기초 | 2026-08-18 |
| [spring-transaction-boundaries-and-batch.md](spring-transaction-boundaries-and-batch.md) | Spring 트랜잭션 · Spring Batch · DB 동시성/자원 | 2026-09-02 |
| [jvm-memory-and-container-limits.md](jvm-memory-and-container-limits.md) | JVM 메모리 · 컨테이너 한도 · 운영 관측성 | 2026-09-07 |
