# Study Dev Index

개발 일반 학습 노트 — **카테고리로 찾는다.** 각 문서는 배경 지식 없이 읽을 수 있게 쓴다(작성 규칙: [AGENTS.md](AGENTS.md)).

> 규모 확장 규칙: 카테고리는 처음엔 한 파일의 **섹션**으로 시작 → 내용이 쌓이면 **독립 파일**로 분리 → 파일이 여럿 되면 **폴더**로 승격. 분리·승격 시 이 인덱스의 링크만 갱신하면 된다.

## 네트워크 · 실시간 통신
- HTTP의 한계와 keep-alive — [2장](realtime-web-networking-cors-infra.md#2-먼저-http부터--왜-실시간-통신이-어려운가) · [부록 A.1~A.2](realtime-web-networking-cors-infra.md#a1-http는-응답이-끝나면-연결을-닫는다)
- WebSocket (Upgrade 핸드셰이크, ws/wss, 하트비트) — [3장](realtime-web-networking-cors-infra.md#3-websocket--한-번-연결해두고-계속-쓰는-방식)
- STOMP (WebSocket 위의 메시징 규약) — [4장](realtime-web-networking-cors-infra.md#4-stomp--websocket-위에-얹는-대화-규칙)
- SockJS 폴백 사다리 (XHR streaming/polling, 폴백의 대가) — [6장](realtime-web-networking-cors-infra.md#6-sockjs--websocket이-막히면-http로-흉내내기)

## 암호학 기초
- **암호학이 지키려는 네 가지** (기밀성·무결성·인증·부인방지)와 기술 매핑 — [1장](cryptography-fundamentals-for-backend.md#1-암호학은-무엇을-지키려고-있나--네-가지-목적)
- "양방향 키 / 단방향 키"라는 구분이 정확하지 않은 이유 — [2장](cryptography-fundamentals-for-backend.md#2-먼저-용어를-바로잡자--양방향-키--단방향-키는-정확한-구분이-아니다)
- 대칭키 암호 (AES·ChaCha20, AEAD, 속도 실측) 와 키 전달 문제 — [3장](cryptography-fundamentals-for-backend.md#3-대칭키-암호--빠르지만-키를-건네줄-방법이-없다)
- 공개키 암호 (RSA 작은 숫자 예제, 공개키 암호화 → 개인키 복호화) — [4장](cryptography-fundamentals-for-backend.md#4-공개키-암호--잠그는-열쇠와-여는-열쇠를-분리한다)
- 해시 (SHA-256, 눈사태 효과, 왜 복호화라는 개념이 없나, 비밀번호 해싱은 별개) — [5장](cryptography-fundamentals-for-backend.md#5-해시--되돌릴-수-없는-지문)
- **전자서명 — 개인키로 서명, 공개키로 검증** — [6장](cryptography-fundamentals-for-backend.md#6-전자서명--이-문서에서-가장-중요한-primitive)
- **"개인키로 암호화한다"가 왜 부정확한가** (EC·Ed25519 에는 암호화 연산이 없다) — [6.4](cryptography-fundamentals-for-backend.md#64-개인키로-암호화한다가-왜-부정확한가) · [부록 A.1](cryptography-fundamentals-for-backend.md#a1-rsa-에서-서명과-암호화가-왜-닮아-보이나)
- RSA · ECDSA · Ed25519 비교, ECDSA 난수 재사용 위험 — [6.5](cryptography-fundamentals-for-backend.md#65-rsa--ecdsa--ed25519-는-어떻게-다른가)
- HMAC — 해시·HMAC·전자서명의 차이와 부인방지 — [7장](cryptography-fundamentals-for-backend.md#7-hmac--공유-비밀로-만드는-무결성-태그)
- **JWT 의 HS256 vs RS256, `alg` 혼동 취약점** — [7.4](cryptography-fundamentals-for-backend.md#74-jwt-의-hs256-과-rs256)
- 키 교환 (DH·ECDH·ECDHE) 과 전방 비밀성, TLS 1.3 에서 키 교환이 스위트에서 분리된 것 — [8장](cryptography-fundamentals-for-backend.md#8-키-교환--비밀을-보내지-않고-비밀을-공유하는-법)
- 하이브리드 암호 — 왜 RSA 로 전부 암호화하지 않나 — [9장](cryptography-fundamentals-for-backend.md#9-하이브리드-암호--왜-rsa-로-전부-암호화하지-않나)
- **primitive 가 TLS 안에서 조립되는 큰 그림** — [10장](cryptography-fundamentals-for-backend.md#10-여기까지를-tls-한-장으로)
- 스스로 답해 보기 10문항 — [11장](cryptography-fundamentals-for-backend.md#11-스스로-답해-보기)
- 실습 명령 모음 — [부록 A.2](cryptography-fundamentals-for-backend.md#a2-실습-명령-모음)

## TLS · 인증서 · 신뢰 저장소
- 이 주제를 읽는 데 필요한 암호학 요약 (도구 한 표, 전자서명, 인증 ≠ 기밀성) — [2장](tls-certificates-and-trust-stores.md#2-여기서-필요한-암호학)
- 인증서란 무엇인가 (X.509 필드 읽기, `sha256WithRSAEncryption` 이름의 오해) — [3장](tls-certificates-and-trust-stores.md#3-인증서란-무엇인가--공개키에-이름표를-붙인-것)
- CA와 서명 체인, 루트가 자기서명인 이유 — [4장](tls-certificates-and-trust-stores.md#4-ca-와-서명-체인--보증인을-세운다)
- **신뢰 저장소 = 검증이 끝나는 지점** (신뢰 앵커, 경로 검증 알고리즘) — [5장](tls-certificates-and-trust-stores.md#5-신뢰-저장소--검증이-끝나는-지점)
- 왜 서버는 루트를 보내주지 않나 — [5.4](tls-certificates-and-trust-stores.md#54-왜-서버는-루트를-보내주지-않나)
- keystore ≠ truststore, `cacerts` 비밀번호가 공개돼도 괜찮은 이유, mTLS — [6장](tls-certificates-and-trust-stores.md#6-keystore-와-truststore-는-정반대다)
- **런타임마다 신뢰 목록을 따로 들고 다니는 이유와 그 부작용** — [7장](tls-certificates-and-trust-stores.md#7-왜-런타임마다-신뢰-목록을-따로-들고-있나)
- 체인 말고 더 검사하는 것들 (SAN 호스트명, 유효기간, CRL/OCSP/stapling, 용도) — [8장](tls-certificates-and-trust-stores.md#8-검증은-체인-확인만이-아니다)
- 오류 메시지 읽는 법 — `PKIX path building failed` 는 **차단이 아니다** — [9장](tls-certificates-and-trust-stores.md#9-오류-메시지-읽는-법)
- 신뢰 저장소를 고칠 때의 대가 비교 (추가 vs 대체, `KeychainStore`, `JAVA_TOOL_OPTIONS`) — [11.3](tls-certificates-and-trust-stores.md#113-해결-수단과-각각의-대가)
- 재사용 가능한 진단 절차 6단계 — [12장](tls-certificates-and-trust-stores.md#12-다시-만났을-때의-진단-절차)
- 미니 PKI 실습 (루트→중간→서버 3단 체인 직접 만들기) — [13장](tls-certificates-and-trust-stores.md#13-직접-해보기--미니-pki-실습)
- `openssl` · `keytool` 명령 사전 — [부록 A.1](tls-certificates-and-trust-stores.md#a1-openssl--keytool-명령-사전)
- 조사 중 걸린 함정들 (LibreSSL의 `-CAfile`, CA `keyUsage` 누락) — [부록 A.3](tls-certificates-and-trust-stores.md#a3-조사-중에-걸린-함정들)

## 웹 보안
- CORS (동작 원리, 헤더 사전, WebSocket이 안 걸리는 이유) — [7장](realtime-web-networking-cors-infra.md#7-cors--브라우저가-남의-집-응답을-못-읽게-막는-규칙)

## 인프라
- 프록시 · TLS 인터셉션 — 사내망이 WebSocket을 막는 이유 — [5장](realtime-web-networking-cors-infra.md#5-왜-사내망에서-websocket이-막히나)
- TLS 가로채기가 작동하는 원리 (연결을 둘로 쪼개고 자기 인증서를 내민다) — [10장](tls-certificates-and-trust-stores.md#10-tls-가로채기가-작동하는-원리)
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

## JPA · 영속성 컨텍스트 · N+1
- **JPA · Hibernate · Spring Data JPA의 역할 분담** (표준 스펙 vs 구현체 vs Repository 추상화) — [1장](jpa-hibernate-fundamentals-and-n-plus-one.md#1-jpa--hibernate--spring-data-jpa--누가-무엇을-하나)
- SQL과 JPQL의 차이 (테이블/컬럼 vs 엔티티/프로퍼티, Hibernate가 사이에서 하는 번역, JPQL은 HQL의 부분집합) — [2장](jpa-hibernate-fundamentals-and-n-plus-one.md#2-sql과-jpql--테이블을-보느냐-엔티티를-보느냐)
- LAZY vs EAGER — 왜 기본은 LAZY이고 EAGER는 목록 조회의 N+1을 못 막나 — [3장](jpa-hibernate-fundamentals-and-n-plus-one.md#3-lazy와-eager--연관-엔티티를-언제-읽을-것인가)
- **N+1 문제 — 정의 · 재현 · "LAZY라서"가 아닌 진짜 원인** — [4장](jpa-hibernate-fundamentals-and-n-plus-one.md#4-n1-문제--목록-하나가-쿼리-101번이-되는-순간)
- Fetch Join과 API별 조회 전략 (매핑은 LAZY, 판단은 쿼리에서) — [5장](jpa-hibernate-fundamentals-and-n-plus-one.md#5-fetch-join--이번-쿼리에서는-팀이-꼭-필요하다)
- EntityGraph — Fetch Join과 같은 목적의 선언형 표현 — [6장](jpa-hibernate-fundamentals-and-n-plus-one.md#6-entitygraph--같은-목적을-어노테이션으로-선언하기) · FETCH vs LOAD [부록 A.3](jpa-hibernate-fundamentals-and-n-plus-one.md#a3-entitygraph의-fetch와-load-타입)
- **Batch Fetching (`default_batch_fetch_size`) 과 Fetch Join의 차이** — [7장](jpa-hibernate-fundamentals-and-n-plus-one.md#7-batch-fetching--나중에-읽되-묶어서-읽는다) · [7.3절](jpa-hibernate-fundamentals-and-n-plus-one.md#73-fetch-join과-batch-fetching은-무엇이-다른가)
- 영속성 컨텍스트와 1차 캐시 — 정의, 동작, "1차 캐시가 아닌 것" — [8장](jpa-hibernate-fundamentals-and-n-plus-one.md#8-persistence-context--jpa가-엔티티를-들고-있는-공간)
- Dirty Checking — 스냅샷 비교, flush 시점, `save()` 없이 UPDATE가 안 나가는 경우 — [9장](jpa-hibernate-fundamentals-and-n-plus-one.md#9-dirty-checking--바꾸기만-하면-update가-나간다)
- 전체 개념 연결 한 장 · 핵심 요약 10문장 · 면접 질문 체크리스트 — [10장](jpa-hibernate-fundamentals-and-n-plus-one.md#10-전체-개념-연결) · [11장](jpa-hibernate-fundamentals-and-n-plus-one.md#11-핵심-요약-10문장) · [12장](jpa-hibernate-fundamentals-and-n-plus-one.md#12-면접-질문-체크리스트)
- Kotlin 엔티티 설정 — `kotlin-jpa`(no-arg)와 `allOpen`(프록시용 open) — [부록 A.1](jpa-hibernate-fundamentals-and-n-plus-one.md#a1-kotlin-엔티티-설정--no-arg-생성자와-open-클래스)
- Fetch Join의 함정 — 컬렉션 페이징이 메모리에서 되는 것(HHH90003004), 컬렉션 둘 fetch의 카테시안 곱 — [부록 A.2](jpa-hibernate-fundamentals-and-n-plus-one.md#a2-fetch-join의-두-가지-함정--페이징과-컬렉션-둘)
- SQL 로그로 쿼리 수 세어 보기 — [부록 A.4](jpa-hibernate-fundamentals-and-n-plus-one.md#a4-sql-로그로-직접-확인하기)
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
- Dev Weekly 한 장 요약 (계정·조직 축 vs 로그인·권한 축) — [요약과 구조도](aws-organizations-accounts-and-access.md#dev-weekly-한-장-요약--계정과-로그인-주체를-두-축으로-보기)
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

## Kotlin 언어
- 함수 타입과 람다 — 함수도 값이다 — [1장](kotlin-callable-references.md#1-함수도-값이다--함수-타입과-람다)
- **`::` 호출 가능 참조 — 호출하지 않고 이름으로 가리킨다** — [2장](kotlin-callable-references.md#2-는-이미-있는-함수를-이름으로-가리킨다)
- `::`의 네 가지 모양 (최상위 · `타입::멤버` · `객체::멤버` · `::생성자`, 프로퍼티 참조) — [3장](kotlin-callable-references.md#3-의-네-가지-모양)
- unbound vs bound — 수신자가 인자로 남나 묶이나, 묶이는 시점 — [4장](kotlin-callable-references.md#4-수신자를-묶느냐-마느냐--unbound와-bound)
- **클래스 안의 `::member` = `this::member`, 오버라이드가 불리는 이유, `protected` 함수를 넘기는 설계** — [5장](kotlin-callable-references.md#5-클래스-안의-member는-thismember다--계기-코드-해석)
- 오버로드는 기대 타입으로 고른다 — [6장](kotlin-callable-references.md#6-같은-이름이-여러-개면--기대-타입으로-고른다)
- Kotlin 1.4 적응(기본 인자 · `Unit` · `vararg` · `suspend`)은 인자 위치에서만 된다 — [7장](kotlin-callable-references.md#7-kotlin-14의-적응adaptation--인자-위치에서만-된다)
- 람다냐 참조냐 고르는 기준 — [8장](kotlin-callable-references.md#8-람다냐-참조냐--고르는-기준)
- `::class` / `::class.java` — [9장](kotlin-callable-references.md#9-class--같은-기호의-다른-쓰임)
- Java 메서드 참조와 비교 — [10장](kotlin-callable-references.md#10-java-메서드-참조와-비교)
- **함수 타입 프로퍼티 `(A) -> B` — 함수를 담는 칸, 부르는 법(`invoke`), 채우는 법(참조·람다·`it`·고정값)** — [11장](kotlin-callable-references.md#11-받는-쪽--함수-타입-프로퍼티는-계산-방법을-끼워-넣는-칸이다)
- 저장 시점과 실행 시점이 다르다 — 값이 흐르는 순서 — [11.4절](kotlin-callable-references.md#114-계기-코드에서-값이-흐르는-순서)
- 기본값 · null 허용 함수 타입 · `typealias` · `fun interface` 고르는 법 — [11.5절](kotlin-callable-references.md#115-자주-만나는-변형들)
- JVM의 `Function1`과 클로저 — [11.6절](kotlin-callable-references.md#116-jvm에서의-실체--function1-객체와-클로저)
- `KFunction` · `KProperty` — [부록 A.1](kotlin-callable-references.md#a1-참조의-실제-타입--kfunction과-kproperty)

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
| [aws-organizations-accounts-and-access.md](aws-organizations-accounts-and-access.md) | AWS 계정·조직(Organizations) · 로그인/권한 · 계정 수명 관리 | 2026-09-11 |
| [mysql-online-ddl-and-schema-migration.md](mysql-online-ddl-and-schema-migration.md) | DB 스키마 변경 · MySQL 온라인 DDL · 마이그레이션 실무 | 2026-09-11 |
| [tls-certificates-and-trust-stores.md](tls-certificates-and-trust-stores.md) | TLS · X.509 인증서 · 신뢰 저장소(PKI) · 런타임별 CA 관리 | 2026-09-16 |
| [cryptography-fundamentals-for-backend.md](cryptography-fundamentals-for-backend.md) | 암호학 기초 — 대칭키·공개키·해시·전자서명·HMAC·키 교환·하이브리드 · JWT | 2026-09-16 |
| [kotlin-callable-references.md](kotlin-callable-references.md) | Kotlin 언어 — 함수 타입·함수 타입 프로퍼티·호출 가능 참조(`::`)·bound/unbound·가상 디스패치·1.4 적응 | 2026-09-18 |
| [jpa-hibernate-fundamentals-and-n-plus-one.md](jpa-hibernate-fundamentals-and-n-plus-one.md) | JPA 기초 — JPA/Hibernate/Spring Data JPA · JPQL · LAZY/EAGER · N+1 · Fetch Join/EntityGraph/Batch Fetching · 영속성 컨텍스트 · Dirty Checking · 면접 체크리스트 | 2026-09-20 |
