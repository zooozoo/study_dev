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

## JPA · 영속성 컨텍스트
- 영속성 컨텍스트란 무엇인가 (동일성 보장 · 변경 감지 · 쓰기 지연) — [2장](jpa-lazy-loading-and-persistence-context.md#2-orm은-무엇을-대신해-주나--엔티티와-영속성-컨텍스트)
- 1차 캐시와 그 수명 — [3장](jpa-lazy-loading-and-persistence-context.md#3-영속성-컨텍스트는-1차-캐시다)
- 지연 로딩과 프록시 — 무엇이 로딩을 발동시키나 — [4장](jpa-lazy-loading-and-persistence-context.md#4-지연-로딩--프록시는-언제-진짜가-되나)
- **단건을 고치려는 코드가 N건을 읽는 순간** — [5장](jpa-lazy-loading-and-persistence-context.md#5-핵심--한-건을-고치려는-코드가-n건을-읽는-순간)
- **트랜잭션 경계를 좁히면 캐시 히트가 DB 왕복이 된다** — [6장](jpa-lazy-loading-and-persistence-context.md#6-왜-어제까지는-문제가-아니었나--캐시의-수명은-트랜잭션의-수명이다)
- N+1 문제와의 차이, 쿼리 수가 지표가 안 되는 이유 — [7장](jpa-lazy-loading-and-persistence-context.md#7-이게-n1-문제와-같은-것인가)
- 비용을 숫자로 확인하기 (Hibernate 통계 · SQL 로그) — [8장](jpa-lazy-loading-and-persistence-context.md#8-눈으로-확인하는-법--통계와-sql-로그) · [부록 A.4](jpa-lazy-loading-and-persistence-context.md#a4-통계-지표-읽는-법과-켤-때의-부작용)
- 고치는 네 가지 방법과 대가 (전용 조회 · fetch join · 벌크 UPDATE) — [9장](jpa-lazy-loading-and-persistence-context.md#9-고치는-네-가지-방법과-각각의-대가)
- 단방향 `@OneToMany`의 벽과 읽기 전용 FK 매핑 — [10장](jpa-lazy-loading-and-persistence-context.md#10-자식으로-직접-조회하기--단방향-연관의-벽과-읽기-전용-매핑)
- 성능 회귀를 테스트로 잡기 (진짜 DB로 비용을 단언하기) — [11장](jpa-lazy-loading-and-persistence-context.md#11-회귀를-테스트로-잡는-법)
- `LazyInitializationException` — [부록 A.1](jpa-lazy-loading-and-persistence-context.md#a1-lazyinitializationexception--세션이-닫힌-뒤에-건드리면)
- 연관관계별 `fetch` 기본값 — [부록 A.2](jpa-lazy-loading-and-persistence-context.md#a2-연관관계별-fetch-기본값)
- `open-in-view` — [부록 A.3](jpa-lazy-loading-and-persistence-context.md#a3-open-in-view--지연-로딩의-수명을-http-요청까지-늘리는-스위치)

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

## AWS 계정 · 조직 · 권한
- AWS 조직의 전체 구조 (Organization · 조직의 Root · OU · 관리 계정 · 멤버 계정) — [2장](aws-organizations-accounts-and-access.md#2-aws-조직의-전체-구조--무엇이-무엇을-담고-있나)
- **AWS 계정과 로그인하는 사람은 서로 다른 축이다** — [3장](aws-organizations-accounts-and-access.md#3-계정과-사람은-서로-다른-축이다)
- 로그인 방법 네 가지 (루트 사용자 · IAM 사용자 · IAM 역할 · Identity Center와 Permission Set) — [4장](aws-organizations-accounts-and-access.md#4-로그인하는-방법은-네-가지뿐이다)
- 같은 이메일인데 왜 다른 신원인가, 계정 초대 vs 사용자 초대 — [5장](aws-organizations-accounts-and-access.md#5-같은-이메일인데-왜-다른-신원인가)
- 계정 안의 관리자 권한 vs 조직을 관리하는 권한 — [6장](aws-organizations-accounts-and-access.md#6-어디를-관리하는-권한인가--계정-안과-조직-위)
- SCP는 권한을 주지 않고 상한만 정한다 — [6.3](aws-organizations-accounts-and-access.md#63-scp는-권한을-주지-않는다--상한만-정한다)
- 조직 규모에 따른 계정·OU 구성과 관리 부담 — [8장](aws-organizations-accounts-and-access.md#8-조직-규모에-따라-어떻게-나누나)
- Control Tower를 언제 고려하나 — [부록 A.1](aws-organizations-accounts-and-access.md#a1-control-tower--언제-고려하나)
- 멤버 계정의 루트 자격증명을 중앙에서 없애는 기능 — [부록 A.2](aws-organizations-accounts-and-access.md#a2-루트-접근-중앙-관리--멤버-계정의-루트를-없애는-기능)

## 클라우드 계정 수명 관리
- 임시 계정을 만들고 정리하기까지의 전체 흐름 — [7장](aws-organizations-accounts-and-access.md#7-인증용-계정을-만들고-정리하기까지)
- **리소스 삭제 · 계정 이동 · OU 삭제 · 조직에서 제거 · 계정 폐쇄 · 조직 삭제는 서로 다른 것을 지운다** — [7.3](aws-organizations-accounts-and-access.md#73-지우는-방법은-여섯-가지고-서로-다른-것을-지운다)
- 루트 이메일 재사용 제한과 폐쇄 전 이메일 변경 — [7.4](aws-organizations-accounts-and-access.md#74-루트-이메일은-폐쇄-전에-바꿔야-한다)
- 폐쇄 직후(90일) vs 영구 폐쇄, 실제 정리 순서와 대기 기간 — [7.5](aws-organizations-accounts-and-access.md#75-실제-정리-순서와-대기-기간)

## 운영 관측성
- 사후 조사를 위해 미리 남겨야 하는 것 (GC 로그, 알람 조건, 오케스트레이터 이벤트 보관 기간) — [11장](jvm-memory-and-container-limits.md#11-사후에-알-수-있으려면-무엇을-남겨야-하나)

## DB 스키마 변경 · 온라인 DDL
- DDL과 온라인 DDL이란 무엇인가 — [1장](mysql-online-ddl-and-schema-migration.md#1-ddl과-온라인-ddl--용어부터)
- 테이블은 파일이고 행은 특정 물리 레이아웃으로 놓인다 — [2장](mysql-online-ddl-and-schema-migration.md#2-테이블은-파일이고-행은-그-안에-특정-모양으로-놓여-있다)
- **테이블 rebuild가 물리적으로 하는 일과 그 비용** — [3장](mysql-online-ddl-and-schema-migration.md#3-rebuild란-무엇인가--새-파일을-만들어-전-행을-옮겨-적는-일)
- 세 가지 알고리즘 (COPY · INPLACE · INSTANT), INPLACE도 rebuild 한다 — [4장](mysql-online-ddl-and-schema-migration.md#4-세-가지-알고리즘--copy-inplace-instant)
- INSTANT는 기존 행을 안 건드린다 — 데이터 딕셔너리와 행 버전 — [5장](mysql-online-ddl-and-schema-migration.md#5-instant는-기존-행을-안-건드린다--그럼-읽을-때는-어떻게-되나)
- **알고리즘을 안 적으면 조용히 내려간다 — 안전 단언으로서의 ALGORITHM** — [6장](mysql-online-ddl-and-schema-migration.md#6-안-적으면-조용히-내려간다--그래서-굳이-적는다)
- 인덱스 추가를 별도 문장으로 빼는 이유, 나눌 때의 원자성 대가 — [7장](mysql-online-ddl-and-schema-migration.md#7-인덱스는-왜-별도-문장으로-빼야-하나)
- 버전 의존 (8.0.12 INSTANT 도입, 8.0.29 임의 위치), AFTER를 피하는 이유 — [8장](mysql-online-ddl-and-schema-migration.md#8-버전에-따라-달라지는-것--8012와-8029)
- INSTANT의 64회 한도와 rebuild로 리셋 — [9장](mysql-online-ddl-and-schema-migration.md#9-instant에는-64번이라는-한도가-있다)
- 마이그레이션 체크리스트 — [10장](mysql-online-ddl-and-schema-migration.md#10-정리--마이그레이션-체크리스트)
- 연산별 rebuild 여부 확인법 (공식 온라인 DDL 표 읽기) — [부록 A.1](mysql-online-ddl-and-schema-migration.md#a1-연산별로-rebuild-하는지-확인하는-법)
- INSTANT가 불가능한 조건들 — [부록 A.2](mysql-online-ddl-and-schema-migration.md#a2-instant가-아예-불가능한-조건들)
- ALGORITHM과 LOCK은 서로 다른 축이다 — [부록 A.3](mysql-online-ddl-and-schema-migration.md#a3-알고리즘과-잠금은-서로-다른-축이다)
- 실험 재현 스크립트 — [부록 A.4](mysql-online-ddl-and-schema-migration.md#a4-이-노트의-실험을-재현하는-스크립트)

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
| [jpa-lazy-loading-and-persistence-context.md](jpa-lazy-loading-and-persistence-context.md) | JPA 영속성 컨텍스트 · 지연 로딩 · ORM 성능 | 2026-09-07 |
| [aws-organizations-accounts-and-access.md](aws-organizations-accounts-and-access.md) | AWS 계정·조직(Organizations) · 로그인/권한 · 계정 수명 관리 | 2026-09-10 |
| [mysql-online-ddl-and-schema-migration.md](mysql-online-ddl-and-schema-migration.md) | DB 스키마 변경 · MySQL 온라인 DDL · 마이그레이션 실무 | 2026-09-11 |
