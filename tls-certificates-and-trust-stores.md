# 로컬 개발 환경의 HTTPS 전면 실패로 배우는 TLS 인증서와 신뢰 저장소

작성 2026-09-16 · 갱신 2026-09-16 (암호학 기초를 별도 문서로 분리)

HTTPS 는 매일 쓰지만, 정작 "서버가 내민 인증서를 내 컴퓨터가 **무엇을 근거로** 믿는가"는 건드릴 일이 없다. 그러다 사내 보안장비가 있는 환경에서 어느 날 갑자기 **브라우저는 멀쩡한데 JDK·Python·Node 만 전부 HTTPS 에 실패**하면, 그 근거가 어디 저장돼 있고 왜 도구마다 다른지를 처음부터 알아야 진단이 된다. 이 노트는 그 하루를 개념 순서로 다시 정리한 것이다.

이 문서의 모든 실행 결과는 **로컬에서 직접 재현한 것**이다. 명령을 그대로 따라 하면 같은 출력이 나온다.

> **자매 문서** — 이 문서는 전자서명·해시·공개키라는 단어를 전제한다. 2장에 필요한 만큼만 요약해 두었고, 각 개념을 "어떤 문제를 풀려고 존재하는가"부터 제대로 쌓은 설명은 [백엔드 개발자를 위한 암호학 기초](cryptography-fundamentals-for-backend.md)에 있다.

---

## 0. 이번 사건 한 문단 요약

사내망에서 **TLS 검사 장비**(회사가 직원 PC 의 HTTPS 를 복호화해 검사하는 보안 제품)가 모든 HTTPS 연결을 중간에서 끊고 **자기 이름으로 다시 서명한 인증서**를 클라이언트에 내밀고 있었다. 그 장비의 **루트 CA 인증서**는 회사가 macOS **시스템 키체인**에 배포해 두었기 때문에 브라우저와 Apple 계열 도구는 정상 동작했다. 그런데 JDK 는 `cacerts`, Python 은 OpenSSL 번들, Node 는 내장 목록이라는 **각자의 신뢰 저장소**를 따로 들고 다니며, 거기에는 그 루트가 없었다. 결과적으로 Spring Boot 앱이 기동 중 클라우드 시크릿 저장소를 호출하는 단계에서 죽고, Gradle 은 캐시에 없는 의존성을 못 받고, CLI 도구들이 줄줄이 실패했다. 오류는 전부 `PKIX path building failed` 계열이었는데 이건 **네트워크 차단이 아니라 내 클라이언트가 스스로 거부한 것**이다.

이 문단에 모르는 단어가 하나라도 있으면 이 문서가 도움이 된다.

## 1. 내가 물어본 것들 → 어디를 보면 되나

| 질문 | 섹션 |
|---|---|
| 서명·해시·공개키가 뭔지 대충만 안다. 이 문서를 읽으려면 뭘 알아야 하나 | [2장](#2-여기서-필요한-암호학) |
| 인증서는 서버가 자기를 증명하려고 들고 있는 것 아닌가? 그럼 내 OS·JDK 가 들고 있는 인증서는 뭔가 | [3장](#3-인증서란-무엇인가--공개키에-이름표를-붙인-것) · [5장](#5-신뢰-저장소--검증이-끝나는-지점) |
| 루트 CA 는 누가 보증하나 | [4.2](#42-그럼-ca-는-누가-보증하나--루트는-자기가-자기를-서명한다) |
| 왜 서버가 루트까지 보내주지 않나 | [5.4](#54-왜-서버는-루트를-보내주지-않나) |
| keystore 와 truststore 는 뭐가 다른가 | [6장](#6-keystore-와-truststore-는-정반대다) |
| `cacerts` 비밀번호가 `changeit` 로 공개돼 있는데 괜찮은가 | [6.2](#62-cacerts-비밀번호가-공개돼-있어도-괜찮은-이유) |
| 왜 런타임마다 목록을 따로 들고 있나. OS 에 한 번 넣으면 안 되나 | [7장](#7-왜-런타임마다-신뢰-목록을-따로-들고-있나) |
| `PKIX path building failed` 는 차단당한 건가 | [9.3](#93-가장-중요한-구분--이건-차단이-아니다) |
| 나가는 트래픽에 회사 CA 서명이 덮여서 나가는 건가 | [10.2](#102-바뀌는-것은-내가-받는-신분증이지-내가-보내는-트래픽이-아니다) |
| "신뢰 목록에 루트를 추가한다"는 게 구체적으로 뭘 하는 건가 | [11.3](#113-해결-수단과-각각의-대가) · [부록 A.1](#a1-openssl--keytool-명령-사전) |
| 이건 내가 세팅할 일인가, 보안팀이 열어줄 수 있는 일인가 | [11.3](#113-해결-수단과-각각의-대가) |
| 다음에 같은 증상을 만나면 어떤 순서로 좁히나 | [12장](#12-다시-만났을-때의-진단-절차) |
| 직접 실습해 보고 싶다 | [13장](#13-직접-해보기--미니-pki-실습) |

---

## 2. 여기서 필요한 암호학

이 문서는 서명·해시·공개키라는 단어를 계속 쓴다. 전부 깊이 알 필요는 없고 **아래 네 절이면 충분하다.** 각 절 끝의 링크가 자매 문서의 해당 설명으로 이어진다.

### 2.1 다섯 가지 도구를 한 표로

| 도구 | 풀려는 문제 | 키 | 대표 | 이 문서에서 쓰이는 곳 | 자세히 |
|---|---|---|---|---|---|
| **대칭키 암호** | 내용을 못 읽게, 빠르게 | 1개를 양쪽이 공유 | AES, ChaCha20 | TLS 연결의 실제 데이터 | [3장](cryptography-fundamentals-for-backend.md#3-대칭키-암호--빠르지만-키를-건네줄-방법이-없다) |
| **공개키 암호** | 대칭키를 건네줄 안전한 통로가 없다 | 쌍 (공개·개인) | RSA | 인증서에 담기는 공개키 | [4장](cryptography-fundamentals-for-backend.md#4-공개키-암호--잠그는-열쇠와-여는-열쇠를-분리한다) |
| **해시** | 바뀌었는지 확인 | **없음** | SHA-256 | 서명 대상을 고정 길이로 요약 | [5장](cryptography-fundamentals-for-backend.md#5-해시--되돌릴-수-없는-지문) |
| **전자서명** | 누가 만들었는지 증명 | 쌍 | RSA, ECDSA, Ed25519 | **CA 가 인증서를 보증** | [6장](cryptography-fundamentals-for-backend.md#6-전자서명--이-문서에서-가장-중요한-primitive) |
| **키 교환** | 비밀을 보내지 않고 공유 | 접속마다 새 임시 쌍 | ECDHE | TLS 세션키 합의 | [8장](cryptography-fundamentals-for-backend.md#8-키-교환--비밀을-보내지-않고-비밀을-공유하는-법) |

이 표에서 세 가지만 가져가면 된다.

- **해시에는 키가 없다.** 그래서 "단방향 키"라는 말은 부정확하다. 되돌릴 수 없는 건 "어려워서"가 아니라 정보를 버렸기 때문이다.
- **공개키 암호는 대칭키를 대체하려고 나온 게 아니라 건네주려고 나왔다.** 실제 데이터는 전부 대칭키로 암호화된다 — 같은 장비에서 재 보면 AES 가 RSA 보다 약 3,000배 빠르다.
- **인증서에 등장하는 건 암호화가 아니라 서명이다.** 이게 다음 절이다.

### 2.2 전자서명 — 이 문서의 뼈대

인증서·CA·신뢰 체인이 전부 이 하나 위에 서 있다. 핵심은 **만들 수 있는 사람과 검증할 수 있는 사람이 다르다**는 비대칭이다.

| | 만들 수 있는 사람 | 검증할 수 있는 사람 |
|---|---|---|
| 해시 | 누구나 | 누구나 |
| **전자서명** | **개인키 주인만** | **누구나** (공개키로) |

```
서명 생성 :  데이터 + 개인키              →  서명값
서명 검증 :  데이터 + 서명값 + 공개키     →  참 / 거짓
```

실제로는 원본 전체가 아니라 **원본의 해시에** 서명한다. 공개키 연산이 느리고 한 번에 처리할 수 있는 크기에 상한이 있기 때문이다.

전자서명이 보장하는 것은 **인증 + 무결성 + 부인방지**다. **기밀성은 주지 않는다** — 서명을 붙여도 데이터는 평문 그대로다. 그래서 인증서는 **누구나 읽을 수 있는 공개 파일인데도 안전하다.** 서명은 내용을 숨기려고 붙은 게 아니라 "이 내용을 CA 가 보증한다"를 붙인 것이다.

> ⚠️ **"개인키로 암호화하고 공개키로 복호화한다"는 부정확한 표현이다.**
>
> RSA 에서만 통하는 수학적 비유다. ECDSA·Ed25519 에는 **암호화 연산 자체가 존재하지 않는다** — `openssl pkeyutl -encrypt` 에 EC 나 Ed25519 키를 주면 `operation not supported for this keytype` 으로 거부되는데, 같은 키로 서명은 정상 동작한다. 서명은 암호화의 변형이 아니라 **독립된 연산**이다. 정확한 방향은 이렇다.
>
> ```
> Public key encryption  →  Private key decryption     (기밀성)
> Private key signing    →  Public key verification    (인증·무결성)
> ```
>
> 실측 근거와 RSA 에서만 그 비유가 그럴듯해 보이는 이유: [암호학 기초 6.4](cryptography-fundamentals-for-backend.md#64-개인키로-암호화한다가-왜-부정확한가) · [부록 A.1](cryptography-fundamentals-for-backend.md#a1-rsa-에서-서명과-암호화가-왜-닮아-보이나)

### 2.3 인증과 기밀성은 별개다

암호학이 지키려는 목적은 여럿이고, **서로를 포함하지 않는다.** 이번 사건을 그 기준으로 보면 이렇다.

| 목적 | 지키는 것 | 이번 사건에서 |
|---|---|---|
| 기밀성 | 제3자가 내용을 못 읽는다 | **온전했다** — 암호화는 정상 |
| 무결성 | 오는 도중 바뀌지 않는다 | **온전했다** |
| **인증** | **상대가 주장하는 그 서버가 맞다** | **깨졌다** — 상대가 검사 장비였다 |

**완벽히 암호화된 연결이 엉뚱한 상대와 맺어질 수 있다.** 암호화가 됐다는 사실은 누구와 통신 중인지에 대해 아무것도 말해주지 않는다. 그래서 인증은 암호화와 **별도의 장치**가 필요하고, 그게 인증서다. ([암호학 기초 1장](cryptography-fundamentals-for-backend.md#1-암호학은-무엇을-지키려고-있나--네-가지-목적))

### 2.4 인증서는 TLS 흐름의 어디에 들어가나

TLS 연결이 맺어지는 흐름을 인증서 중심으로 줄이면 이렇다.

```
   Client                                                   Server
     |                                                         |
     |  <──── ② Certificate ─────────────────────────────────  |
     |         공개키 + 도메인 + CA 의 서명                     |
     |                                                         |
  ┌──┴────────────────────────────────────────┐                |
  │ ③ 인증서 검증         ← 이 문서 3~9장     │                |
  │    CA 서명 검증 · 체인 · 도메인 · 유효기간 │                |
  └──┬────────────────────────────────────────┘                |
     |                                                         |
     |  <──── ④ CertificateVerify ───────────────────────────  |
     |         그 인증서의 개인키를 실제로 갖고 있다는 서명     |
     |                                                         |
     |  <════ ⑤ ECDHE 키 교환 → 양쪽이 같은 세션키 도출 ═════> |
     |                                                         |
     |  <════ ⑥ AES-GCM / ChaCha20 으로 실제 데이터 ═════════> |
```

각 단계에서 어떤 primitive 가 무슨 일을 하는지는 [암호학 기초 10장](cryptography-fundamentals-for-backend.md#10-여기까지를-tls-한-장으로)에 전체 그림과 역할 표로 정리돼 있다.

여기서 중요한 한 가지만 짚는다. **④·⑤·⑥이 전부 수학적으로 정상이어도 사칭은 막히지 않는다.** 중간자가 자기 키 쌍으로 똑같이 수행하면 서명도 통과하고 키 교환도 되고 암호화도 완벽하다. 이 구멍을 막는 유일한 단계가 **③** 이다.

그래서 이 문서의 질문은 하나로 모인다.

> **"이 공개키가 진짜 `api.example.com` 의 것"임을 어떻게 아는가?**

---

## 3. 인증서란 무엇인가 — 공개키에 이름표를 붙인 것

### 3.1 인증서가 푸는 문제

2.4 의 질문이 출발점이다. 공개키만으로는 주인을 알 수 없다.

**인증서(certificate)** 는 이 세 가지를 묶어 놓은 파일이다.

1. **공개키** — 서버의 공개키
2. **신원 정보** — 이 공개키가 누구 것인지 (도메인 이름 등)
3. **발급자의 서명** — "1과 2가 맞는 짝이다"라는 제3자의 도장

3번이 2.2 의 전자서명이다. 인증서는 **공개키에 이름표를 붙이고 제3자가 그 이름표를 보증한 것**이다.

### 3.2 인증서 안에 실제로 들어 있는 것

아래는 이 노트를 위해 직접 만든 데모 인증서다.

```
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 11:ba:43:a5:...
        Signature Algorithm: sha256WithRSAEncryption
        Issuer:  CN=Demo Intermediate CA, O=Study Lab   ← 누가 서명해 줬나
        Validity
            Not Before: Sep 16 09:28:03 2026 GMT        ← 유효기간 시작
            Not After : Sep 16 09:28:03 2027 GMT        ← 유효기간 끝
        Subject: CN=localhost, O=Study Lab              ← 누구의 것인가
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus: 00:a1:0a:9e:6b:...             ← 공개키 본체
        X509v3 extensions:
            X509v3 Subject Alternative Name:
                DNS:localhost                           ← 실제로 검사되는 도메인
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment     ← 이 키로 할 수 있는 일
            X509v3 Extended Key Usage:
                TLS Web Server Authentication           ← 용도 제한
            X509v3 Basic Constraints: critical
                CA:FALSE                                ← 이건 CA 가 아니다
```

`Modulus` 는 RSA 공개키의 `n` 이다 — 두 소수의 곱이고, 이걸 쪼갤 수 없다는 게 RSA 안전성의 근거다. ([암호학 기초 4.2](cryptography-fundamentals-for-backend.md#42-작은-숫자로-보는-rsa))

`Key Usage` 에 `Digital Signature` 와 `Key Encipherment` 가 **따로** 있다. 같은 키 쌍이라도 서명과 암호화는 다른 연산이라 용도를 분리해서 못 박는 필드다. ([암호학 기초 6.6](cryptography-fundamentals-for-backend.md#66-암호화와-서명은-다른-primitive-다))

**`Signature Algorithm: sha256WithRSAEncryption` 은 이름이 오해를 부른다.** 뒤에 `Encryption` 이 붙어 있지만 **이것은 서명 알고리즘 식별자이지 암호화가 아니다.** RSA 표준이 처음 만들어질 때 붙은 역사적 이름이 그대로 남은 것뿐이다. 정확히 읽으면 **"SHA-256 으로 해시하고, RSA 서명 방식(PKCS#1 v1.5)으로 서명했다"** 는 뜻이다. 2.2 의 경고와 같은 이야기다.

(참고로 **인증서에 박힌 서명**은 위처럼 PKCS#1 v1.5 인데, **핸드셰이크의 ④ 서명**은 실측하면 `rsa_pss_rsae_sha256` — RSA-PSS 로 나온다. 같은 RSA 키로 서로 다른 서명 방식을 쓴다.)

`Issuer`(발급자)와 `Subject`(주체)가 **다르다**는 점을 기억해 두자. 4장에서 이게 사슬이 된다.

이 형식의 이름이 **X.509** 다. "인증서 파일 표준 규격"이라고 생각하면 된다.

### 3.3 그런데 인증서만으로는 아직 아무것도 증명되지 않는다

인증서는 **공개된 파일**이다. 아무나 만들 수 있다. "나는 `api.example.com` 이다"라고 적힌 인증서를 만드는 데 1분이면 충분하다 — 자기 개인키로 자기가 서명하면 그만이다. 이걸 **자기서명 인증서(self-signed certificate)** 라고 한다.

그래서 서버가 인증서를 내미는 행위 자체는 **여전히 아무것도 보장하지 않는다.** 결정적인 건 **누가 서명했느냐**다.

## 4. CA 와 서명 체인 — 보증인을 세운다

### 4.1 CA 가 하는 일

**CA(Certificate Authority, 인증기관)** 는 "이 공개키가 진짜 이 도메인의 것이 맞다"를 확인하고 **자기 개인키로 서명해 주는** 기관이다. 확인 절차(도메인 소유 증명 등)를 거친 뒤 도장을 찍어 준다.

이제 클라이언트는 이렇게 판단할 수 있다.

> 이 인증서에는 내가 아는 CA 의 서명이 붙어 있다 → 그 CA 가 확인했다는 뜻이다 → 믿는다.

2.2 의 비대칭이 여기서 쓰인다. **CA 만 서명을 만들 수 있고, 누구나 검증할 수 있다.**

### 4.2 그럼 CA 는 누가 보증하나 — 루트는 자기가 자기를 서명한다

당연한 다음 질문이다. CA 인증서는 상위 CA 가 서명한다. 그런데 무한히 올라갈 수는 없다.

그래서 맨 위 **루트 CA(root CA)** 는 **자기가 자기를 서명한다.** `Subject` 와 `Issuer` 가 같다.

```
루트 CA          Subject = Issuer = "Demo Root CA"      ← 자기서명
  └─ 중간 CA     Issuer = "Demo Root CA"
       └─ 서버   Issuer = "Demo Intermediate CA"
```

**여기서 수학적 검증은 의미를 잃는다.** 루트의 서명을 검증해 봐야 "자기 서명이 자기 공개키로 검증된다"는 동어반복일 뿐이고, 위조범도 똑같이 만들 수 있다.

그래서 **어딘가에서는 수학이 아니라 "믿기로 한다"는 선언이 필요하다.** 그게 5장이다.

### 4.3 직접 만들어 보기

말로만 보면 안 와닿으니 3단 사슬을 직접 만들어 본다. (전체 스크립트는 [13장](#13-직접-해보기--미니-pki-실습)에.)

```
root.crt     subject=CN=Demo Root CA          issuer=CN=Demo Root CA           ← 자기서명
inter.crt    subject=CN=Demo Intermediate CA  issuer=CN=Demo Root CA
server.crt   subject=CN=localhost             issuer=CN=Demo Intermediate CA
```

`subject` 를 따라가면 위의 `issuer` 가 나온다. 이게 **체인(chain)** 이다.

## 5. 신뢰 저장소 — 검증이 끝나는 지점

### 5.1 그냥 믿기로 한다는 선언이 필요하다

체인을 따라 올라가다 보면 반드시 자기서명 루트에 닿는다. 거기서 검증을 끝내려면 **"이 루트는 믿기로 한다"** 는 사전 선언이 있어야 한다.

그 선언을 모아 둔 목록이 **신뢰 저장소(trust store)** 이고, 거기 담긴 인증서를 **신뢰 앵커(trust anchor)** 라고 부른다. 앵커 = 닻. 사슬이 고정되는 지점이라는 뜻이다.

### 5.2 여권 심사로 비유하면

| 개념 | 비유 |
|---|---|
| 서버 인증서 | 여권 |
| CA | 여권 발급 기관 |
| CA 의 서명 | 여권에 찍힌 발급국 인장 |
| 신뢰 저장소 | 우리나라가 인정하는 **발급 기관 목록** |
| 체인 검증 | 인장을 보고 발급 기관을 따라 올라가는 일 |

아무리 정교하게 만들어진 여권이라도, **우리가 인정하지 않는 기관이 발급했으면 입국 거부다.** 여권 자체의 품질 문제가 아니다. 이게 이번 사건의 전부다 — 인증서에는 아무 하자가 없었고, 발급 기관이 내 목록에 없었을 뿐이다.

### 5.3 검증 알고리즘을 단계로 보면

표준은 RFC 5280 의 certification path validation 이다. 실제 동작은 이렇다.

```
1. 서버가 보낸 인증서들을 받는다
2. 서버 인증서(leaf)의 Issuer 를 본다
3. 그 Issuer 를 Subject 로 갖는 인증서를 찾는다
     - 서버가 같이 보내준 것들 중에서, 또는
     - 내 신뢰 저장소에서
4. 찾은 인증서의 공개키로 아래 인증서의 서명을 검증한다   ← 2.2 의 서명 검증
5. 방금 찾은 인증서가 내 신뢰 저장소에 있나?
     - 있다  → 성공. 여기서 끝.
     - 없다  → 2번으로 돌아가 한 칸 더 올라간다
6. 더 올라갈 데가 없는데 저장소에 못 닿았다 → 실패
```

6번이 바로 `PKIX path building failed` 다. "**경로를 끝까지 못 만들었다**"는 뜻이지, 누가 막았다는 뜻이 아니다.

### 5.4 왜 서버는 루트를 보내주지 않나

자연스러운 의문이다. 서버가 루트까지 다 보내주면 편할 텐데?

**보내주면 검증이 무의미해지기 때문이다.** 위조범도 자기가 만든 루트를 같이 보내면 그만이다. 그러면 모든 인증서가 "자기가 데려온 보증인"으로 통과된다.

그래서 **신뢰의 출발점은 반드시 클라이언트가 미리, 다른 경로로(out-of-band) 갖고 있어야 한다.** OS 를 설치할 때, JDK 를 내려받을 때 같이 들어온다.

실제로 서버가 뭘 보내는지 보면 확인된다.

```
$ openssl s_client -connect localhost:8443 -servername localhost -showcerts </dev/null \
    | grep -E "^ *[0-9]+ s:|^ *i:"

 0 s:CN=localhost, O=Study Lab               ← 서버 인증서
   i:CN=Demo Intermediate CA, O=Study Lab
 1 s:CN=Demo Intermediate CA, O=Study Lab    ← 중간 CA (여기까지만 보내준다)
   i:CN=Demo Root CA, O=Study Lab            ← 최종 보증인인데 체인에 없다
```

`Demo Root CA` 는 **목록에 없다.** 이름만 언급될 뿐 실물은 안 온다. 클라이언트가 자기 저장소에서 찾아야 한다.

이번 사건에서 실패한 지점이 정확히 이 한 칸이었다 — 검사 장비의 루트가 macOS 키체인에는 있었고 JDK `cacerts` 에는 없었다.

## 6. keystore 와 truststore 는 정반대다

### 6.1 담는 것도, 쓰는 쪽도 반대다

둘 다 "인증서를 담는 파일"이라 이름이 헷갈리는데, 역할이 반대다.

| | 담는 것 | 개인키 | 쓰는 쪽 | 하는 일 |
|---|---|---|---|---|
| **keystore** | 내 인증서 + **내 개인키** | 있다 | 주로 서버 | 내가 나를 증명 (서명) |
| **truststore** | 남의 루트 CA 인증서 | **없다** | 주로 클라이언트 | 상대를 검증 |

**개인키가 있느냐 없느냐**가 결정적 차이다. 공개키만 든 파일은 유출돼도 되고, 개인키가 든 파일은 유출되면 그 신원으로 서명할 수 있게 된다. ([암호학 기초 4.4](cryptography-fundamentals-for-backend.md#44-개인키를-지켜야-하는-이유))

### 6.2 cacerts 비밀번호가 공개돼 있어도 괜찮은 이유

JDK 의 `<JDK>/lib/security/cacerts` 는 이름 그대로 **CA certs**, 즉 truststore 다. 기본 비밀번호는 `changeit` 으로 전 세계에 공개돼 있다.

괜찮은 이유는 **안에 개인키가 하나도 없기 때문**이다. 담겨 있는 건 전부 공개된 루트 CA 인증서다. 훔쳐 가도 얻는 게 없다.

다만 비밀번호가 **읽기**를 막지 못해도 **쓰기**는 여전히 문제다. 누가 여기에 자기 루트를 몰래 넣으면 그 JVM 은 그 사람이 발급한 모든 인증서를 믿게 된다. 그래서 이 파일은 비밀번호가 아니라 **파일 권한**으로 지킨다.

확인해 보면:

```
$ keytool -list -keystore "$(/usr/libexec/java_home)/lib/security/cacerts" \
    -storepass changeit | grep -c trustedCertEntry
146
```

146개의 루트 CA 가 기본 탑재돼 있다. **이 숫자는 JDK 배포판·버전마다 다르다** — 같은 장비에 깔린 JDK 끼리도 다르니 "몇 개다"라고 외울 값이 아니라 필요할 때 세어 볼 명령이다.

그리고 `trustedCertEntry` 라는 항목 종류 자체가 "개인키 없이 신뢰만 하는 항목"이라는 표시다. keystore 였다면 `PrivateKeyEntry` 가 나온다.

### 6.3 그럼 클라이언트가 keystore 를 쓰는 경우는 없나

있다. **mTLS(mutual TLS, 상호 인증)** 다.

보통의 HTTPS 는 **서버만** 자기를 증명한다(2.4 의 ②·④). 클라이언트는 익명이다. 그런데 서버가 "너도 신분증을 내놔"라고 요구하는 구성이 있다 — 금융 API, 내부 서비스 간 통신 등. 이때는 ②·④가 **양방향으로** 일어난다.

그러려면 클라이언트도 자기 인증서와 개인키가 담긴 **keystore** 를 갖고 있어야 한다. Java 에서는 `javax.net.ssl.keyStore` 로 지정한다. 이름이 비슷한 `trustStore` 와 헷갈리기 쉬운데, **`keyStore` = 내가 내밀 것, `trustStore` = 내가 검사할 기준**이다.

## 7. 왜 런타임마다 신뢰 목록을 따로 들고 있나

### 7.1 이식성 때문이다

이게 이번 사건의 구조적 원인이다.

JVM 은 macOS·Linux·Windows 어디서든 **똑같이** 동작해야 한다. 그런데 OS 마다 신뢰 저장소의 위치도, 형식도, 접근 API 도 전부 다르다. macOS 는 Keychain, Windows 는 인증서 저장소, Linux 는 배포판마다 제각각이다.

그래서 JVM 은 **자기 목록을 번들해서 들고 다닌다.** 어느 OS 에 갖다 놔도 같은 결과가 나오게. Python 의 `certifi`, Node 의 내장 목록도 정확히 같은 이유로 존재한다.

### 7.2 신뢰 저장소 지도

| 신뢰 저장소 | 이걸 쓰는 도구 |
|---|---|
| macOS 시스템 키체인 | 브라우저, Apple 이 번들한 git·curl, Apple LibreSSL |
| `<JDK>/lib/security/cacerts` | 모든 JVM — Gradle·Maven, Spring Boot 앱, 테스트 JVM |
| OpenSSL 번들 `cert.pem` 또는 `certifi` | Python, 클라우드 CLI 도구 |
| Node 내장 목록 | Node.js, npm |

**JDK 마다 따로다.** 장비에 JDK 가 세 개 깔려 있으면 신뢰 저장소도 세 개다. 하나에 넣어도 나머지 둘은 모른다.

자기 Python 이 어딜 보는지는 이렇게 확인한다.

```
$ python3 -c "import ssl; print(ssl.get_default_verify_paths())"
DefaultVerifyPaths(cafile='/opt/homebrew/etc/openssl@3/cert.pem',
                   capath='/opt/homebrew/etc/openssl@3/certs',
                   openssl_cafile_env='SSL_CERT_FILE', ...)
```

`openssl_cafile_env` 가 **이 값을 덮어쓸 환경변수 이름**이라는 것까지 알려 준다.

### 7.3 부작용 — OS 에 배포해도 런타임은 안 본다

여기서 이번 사건의 핵심 부작용이 나온다.

> 회사가 OS 신뢰 저장소에 CA 를 아무리 잘 배포해도, **JDK·Python·Node 는 그쪽을 쳐다보지도 않는다.**

"OS 단에 한 번 넣어 두면 인증서 관리만 하면 되지 않나"라는 기대가 자연스럽게 깨지는 지점이다. 답은 **런타임이 OS 저장소를 보도록 명시적으로 지시했을 때만 그렇다**이고, 그 지시 방법은 런타임마다 다르며 [11.3](#113-해결-수단과-각각의-대가)에서 다룬다.

## 8. 검증은 체인 확인만이 아니다

### 8.1 네 가지 축

체인이 신뢰 앵커에 닿았다고 끝이 아니다. 클라이언트는 이것들을 더 본다.

| 축 | 무엇을 보나 | 틀리면 |
|---|---|---|
| **체인** | 신뢰 앵커까지 이어지나 | `PKIX path building failed` |
| **호스트명** | SAN 이 접속한 주소와 맞나 | `No subject alternative names matching ...` |
| **유효기간** | `notBefore` ~ `notAfter` 안인가 | `certificate expired` |
| **폐기 여부** | CRL/OCSP 상 취소됐나 | `certificate revoked` |
| **용도** | 서버 인증용으로 발급됐나 | `unsupported certificate purpose` |

**오류를 만나면 어느 축에서 걸렸는지 먼저 구분해야 한다.** 메시지가 다르므로 구분은 어렵지 않은데, 습관적으로 전부 "인증서 문제"로 뭉뚱그리면 엉뚱한 곳을 고치게 된다.

### 8.2 호스트명 검사는 완전히 별개 축이다

실측으로 확인해 보면 확실하다. 같은 서버, 같은 신뢰 저장소인데 접속 주소만 바꿨다.

```
# SAN 은 DNS:localhost 로 발급돼 있다

$ java Probe.java https://localhost:8443/      (신뢰 저장소에 루트 있음)
OK   HTTP 200

$ java Probe.java https://127.0.0.1:8443/      (같은 서버, 같은 저장소)
FAIL No subject alternative names matching IP address 127.0.0.1 found
```

체인은 완벽히 유효한데 **이름이 안 맞아서** 거부됐다. 메시지에 `PKIX` 가 없다는 게 신호다. 이 경우엔 신뢰 저장소를 아무리 손봐도 해결되지 않는다.

### 8.3 SAN 과 CN — 왜 CN 은 이제 안 쓰나

옛날에는 `Subject` 의 `CN`(Common Name)에 도메인을 적었다. 그런데 `CN` 은 원래 "사람이 읽는 이름" 필드라 형식 제약이 없었고, 도메인을 여러 개 넣을 수도 없었다.

그래서 **SAN(Subject Alternative Name)** 확장이 도입됐고, 지금은 이쪽이 정본이다. 요즘 클라이언트는 **`CN` 을 아예 보지 않는다** — 위 실측에서 `CN=localhost` 인데도 IP 로 접속하니 SAN 기준으로 거부한 게 그 증거다.

인증서를 만들 때 SAN 을 빠뜨리면 `CN` 을 아무리 맞춰도 최신 클라이언트에서 전부 실패한다. 자체 인증서를 만들 때 가장 흔한 실수다.

### 8.4 유효기간과 폐기

**유효기간**은 단순하다. 인증서 안에 `notBefore`/`notAfter` 가 박혀 있고 현재 시각과 비교한다. 여기서 종종 진짜 원인이 **클라이언트 시계**인 경우가 있다 — 컨테이너 시계가 어긋나면 멀쩡한 인증서가 만료로 보인다.

**폐기(revocation)** 는 더 까다롭다. 유효기간이 남았는데 개인키가 유출돼 무효화해야 하는 경우다. 두 가지 방식이 있다.

- **CRL(Certificate Revocation List)** — CA 가 폐기된 인증서 목록을 파일로 배포. 목록이 커지고 갱신이 느리다.
- **OCSP(Online Certificate Status Protocol)** — 클라이언트가 CA 에 "이거 살아 있나?"를 실시간으로 물어본다. 매 접속마다 외부 호출이 생기고, **CA 가 누가 어디 접속하는지 알게 되는** 프라이버시 문제가 있다.
- **OCSP stapling** — 서버가 미리 CA 에게 받아 둔 "살아 있음" 응답을 **핸드셰이크에 같이 실어 보낸다.** 클라이언트가 CA 에 따로 안 물어봐도 되므로 지금 주류다.

참고로 **폐기 검사는 기본적으로 꺼져 있거나 실패를 무시하는 설정이 많다.** Java 도 기본값은 검사하지 않는다. 그래서 "폐기됐는데 통과하더라"는 상황이 실제로 생긴다.

## 9. 오류 메시지 읽는 법

### 9.1 런타임별 대조표

아래는 전부 **같은 상황**(서버 체인은 정상, 클라이언트에 루트 없음)에서 실제로 찍힌 메시지다.

| 런타임 | 메시지 |
|---|---|
| Java | `PKIX path building failed: SunCertPathBuilderException: unable to find valid certification path to requested target` |
| Node | `UNABLE_TO_GET_ISSUER_CERT_LOCALLY` |
| Python | `[SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: unable to get local issuer certificate` |

표현만 다를 뿐 **전부 같은 말**이다 — "발급자를 내 목록에서 못 찾았다".

`PKIX` 는 **Public Key Infrastructure using X.509** 의 약자로, 5.3 의 검증 알고리즘을 규정한 표준 이름이다. "path building failed" = **경로를 끝까지 만들지 못했다.**

### 9.2 Node 만 상황에 따라 메시지가 둘인 이유

실습 중에 걸린 것인데, Node 는 같은 "루트가 없다" 상황에서 두 가지 코드를 낸다.

| 체인 구조 | Node 오류 코드 |
|---|---|
| 루트 → **중간** → 서버 (중간을 보내줌) | `UNABLE_TO_GET_ISSUER_CERT_LOCALLY` |
| 루트 → 서버 (중간 없음) | `UNABLE_TO_VERIFY_LEAF_SIGNATURE` |

차이는 **어디까지 올라가다 막혔나**다. 중간 CA 가 있으면 한 칸 올라간 뒤 "그 위를 못 찾겠다"가 되고, 없으면 "받은 인증서 자체의 서명을 검증할 수가 없다"가 된다.

원인은 같으므로 **둘 다 신뢰 저장소 문제로 읽으면 된다.** 코드가 달라서 다른 문제라고 착각하기 쉬운 지점이라 적어 둔다.

### 9.3 가장 중요한 구분 — 이건 차단이 아니다

이 문서에서 가장 실무적으로 중요한 한 줄이다.

> `PKIX path building failed` 는 **누가 막은 게 아니라 내 클라이언트가 거부한 것이다.**

네트워크는 열려 있었고, TCP 연결도 됐고, 서버가 인증서도 보냈다. 그걸 받아 든 내 프로그램이 "발급자를 모르겠다"며 스스로 연결을 끊은 것이다. 거부의 **주체가 나**다.

그럼 정말로 정책에 막혔을 땐 어떻게 보이나? **증상이 완전히 다르다.**

| | 인증서 검증 실패 | 정책 차단 |
|---|---|---|
| TLS 핸드셰이크 | **실패** | 성공 |
| 받는 것 | 없음 (연결 끊김) | **HTTP 응답** — 차단 안내 페이지, 403 등 |
| 거부 주체 | 내 클라이언트 | 중간 장비 |

이 구분이 실무에서 왜 중요하냐면, 보안팀에 "막힌 것 같다"고 말하면 **"우리는 안 막았는데요"** 로 되돌아오기 때문이다. 정확히는 "차단이 아니라 인증서 검증에 실패하고 있고, 거부하는 건 우리 쪽 런타임"이라고 말해야 대화가 이어진다.

한 가지 더 — **인증서 실패가 앞단을 가리면 그 뒤에 정책 차단이 있어도 구분되지 않는다.** 핸드셰이크에서 이미 끊기므로 차단 페이지를 받아 볼 기회가 없다. 그래서 "인증서 문제를 먼저 해결한 뒤에야 정책 차단 여부를 판정할 수 있다."

## 10. TLS 가로채기가 작동하는 원리

### 10.1 특별한 기술이 아니다

**TLS 인터셉션 / SSL 검사(SSL inspection)** 라는 이름이 거창한데, 실체는 2.4 에서 말한 바로 그 중간자다.

```
[클라이언트] ──TLS 연결 ①──> [검사 장비] ──TLS 연결 ②──> [실제 서버]
```

연결이 **둘로 쪼개진다.** 검사 장비가

1. 클라이언트에게는 **목적지 도메인 이름으로 자기가 즉석에서 발급한 인증서**를 내민다
2. 뒤에서는 자기가 진짜 서버와 따로 정상 TLS 연결을 맺는다
3. 가운데서 평문을 보고 검사한 뒤 다시 암호화해 넘긴다

2.4 의 흐름이 **양쪽에서 각각 온전히 수행된다.** 키 교환도, 서명도 정상이다. 암호화도 완벽하다. 단 하나 남는 관문이 ③ 인증서 검증인데, 이게 통과하려면 **그 장비의 루트 CA 가 내 신뢰 저장소에 있어야** 한다. 그래서 회사는 직원 PC 에 자기 루트를 미리 배포한다.

즉 **"회사가 자기 루트 CA 를 클라이언트 신뢰 목록에 하나 넣은 것"** 이 기술의 전부다. 5장의 구조를 그대로 이용한 것이다.

### 10.2 바뀌는 것은 내가 받는 신분증이지 내가 보내는 트래픽이 아니다

여기서 방향을 반대로 이해하기 쉽다. "내가 나가는 트래픽에 회사 CA 서명이 덮여서 나가는 건가?" — **아니다.**

서명이 붙는 것은 **서버의 신분증**이다. 내가 보내는 데이터에 뭘 덧붙이는 게 아니라, **내가 받아 보는 상대의 신분증이 진짜 서버 것에서 검사 장비 것으로 바꿔치기되는 것**이다.

다시 말해 이 구조가 노리는 건 "내 트래픽을 증명하는 것"이 아니라 **"내가 상대를 검증하는 과정을 통과하는 것"** 이다. 그래야 중간에 끼어들 수 있으니까.

### 10.3 그래서 신뢰 목록에 CA 를 넣는 것은 가벼운 일이 아니다

이 구조를 이해하고 나면 자연스럽게 따라오는 결론이다.

> 신뢰 목록에 루트 CA 를 추가한다 = **그 CA 가 발급한 모든 도메인의 인증서를 무조건 믿겠다**
> = 그 CA 에게 **아무 사이트나 사칭할 수 있는 권한**을 주는 것

은행이든 뭐든 예외 없다. 그래서 "인터넷에서 받은 `.crt` 파일을 신뢰 저장소에 넣어라"는 안내는 원칙적으로 위험하고, **회사가 공식 배포 경로로 제공한 CA 인지**가 중요하다.

동시에 이건 **왜 회사 보안팀이 이 CA 를 관리하는지**에 대한 답이기도 하다. 그 개인키가 유출되면 회사 전 직원의 HTTPS 가 통째로 열린다.

## 11. 이번 사건 전체 매핑

### 11.1 증상과 원인

| 관찰 | 의미 |
|---|---|
| 브라우저는 정상 | macOS 키체인에는 검사 장비 루트가 있다 |
| Apple 계열 git·curl 정상 | 같은 키체인을 본다 |
| JDK·Python·Node 전부 실패 | 각자 신뢰 저장소를 보는데 거기엔 없다 |
| 오류가 전부 `PKIX`/`CERTIFICATE_VERIFY_FAILED` 계열 | 체인 축 실패. 호스트명·기간 문제 아님 |
| 여러 도메인에서 동일 | 특정 사이트 문제가 아니라 **모든 HTTPS** 가 가로채진다 |

원인은 7.3 그대로다 — 회사가 OS 에만 배포했고 런타임들은 OS 를 보지 않는다.

### 11.2 왜 이제야 드러났나 — 캐시가 가리고 있었다

이 사건에서 가장 배울 만한 부분이다.

빌드는 **되고 있었다.** 의존성이 전부 로컬 캐시에 있어서 네트워크를 탈 일이 없었기 때문이다. 즉 **"빌드가 성공한다"가 "네트워크가 정상이다"를 전혀 보장하지 않았다.**

먼저 터진 건 캐시가 없는 경로였다 — 앱이 기동하면서 클라우드 시크릿 저장소를 호출하는 단계. 그건 매번 실제 HTTPS 를 타야 하니까.

그래서 진단 절차에 **"캐시에 없는 의존성을 일부러 하나 받아 본다"** 가 들어간다. 이걸 안 하면 영향 범위를 한참 작게 잡는다.

### 11.3 해결 수단과 각각의 대가

| 대상 | 방법 | 장점 | 대가 |
|---|---|---|---|
| Java | `keytool -importcert` 로 `cacerts` 에 추가 | 그 JDK 의 모든 JVM 에 자동 적용 | **JDK 마다, 업그레이드마다 재작업**. 원본 파일 수정 |
| **Java (권장)** | `-Djavax.net.ssl.trustStoreType=KeychainStore` | **파일 무수정, JDK 교체에 안 깨짐, 인증서 회전 시 OS 만 갱신하면 됨** | JVM 옵션이라 실행 지점마다 필요 |
| Node | `NODE_EXTRA_CA_CERTS=<pem>` | 기존 목록에 **추가**된다 | — |
| Python / CLI 도구 | `SSL_CERT_FILE` · `AWS_CA_BUNDLE` | — | 기존 목록을 **대체**한다. 반드시 기존 번들과 **합친 파일**을 줘야 함 |
| 조직 차원 | TLS 검사 예외(bypass) 요청 | 개발자가 아무것도 안 해도 됨 | 보안팀 정책 결정이 필요 |

**`NODE_EXTRA_CA_CERTS` 는 추가, `SSL_CERT_FILE` 은 대체**라는 차이가 실무에서 제일 많이 사고를 낸다. 후자에 회사 루트만 달랑 넣으면 그 순간부터 **일반 공인 CA 사이트가 전부 실패**한다.

**"이건 내가 할 일인가, 보안팀이 할 일인가"** 에 대한 답은 이 표가 말해 준다. 런타임 신뢰 저장소는 **각 개발자 장비의 개발 환경 구성**이므로 기본적으로 내 몫이다. 보안팀에 물을 수 있는 것은 다른 층위다 — **공식 CA 파일을 어디서 받나**, **개발 도메인에 대한 검사 예외가 가능한가**, **개발자가 런타임 저장소에 등록하는 것이 정책상 허용되는가**, **진짜 정책 차단은 어떤 모습으로 보이나**.

### 11.4 Java 만 놓고 보면 한 줄로 끝난다

Java 계열은 `KeychainStore` 방식이 확실히 낫다.

```bash
export JAVA_TOOL_OPTIONS="-Djavax.net.ssl.trustStoreType=KeychainStore"
```

`JAVA_TOOL_OPTIONS` 는 **모든 JVM 이 기동할 때 읽는 환경변수**라, 여기 넣으면 직접 띄우는 JVM 뿐 아니라 **빌드 도구가 포크한 데몬·테스트 JVM·앱 프로세스까지 전부 상속**한다. 파일을 하나도 안 고치고, JDK 를 갈아 끼워도 안 깨지고, 회사가 인증서를 교체하면 OS 키체인만 갱신되면 된다.

지원 여부는 이렇게 확인된다.

```
$ keytool -list -storetype KeychainStore -keystore NONE -storepass ""
Keystore type: KEYCHAINSTORE
Keystore provider: Apple
```

macOS 전용이다 (Apple 프로바이더). Windows 에는 같은 역할로 `WINDOWS-ROOT` 가 있다.

**단, GUI 로 실행한 IDE 는 셸 프로파일을 읽지 않는다.** 터미널에서만 되고 IDE 에서는 계속 실패하는 상황이 생기므로 IDE 설정에도 따로 넣어야 한다.

다른 런타임 참고 사항:

- Node 에는 OS 저장소를 쓰는 `--use-system-ca` 플래그가 있지만, 시험한 환경(v25.8.1)에서는 플래그를 인식하면서도 키체인 인증서를 집어내지 못했다. 버전·플랫폼 의존이 크므로 **쓰기 전에 실제로 확인할 것.**
- Python 표준 라이브러리에는 OS 저장소를 쓰는 기본 경로가 없다. 합친 번들 파일을 주는 방식이 현실적이다.

## 12. 다시 만났을 때의 진단 절차

이런 증상을 또 만났을 때 순서대로 좁히면 된다.

**1. 어느 계층에서 실패했는지 먼저 가른다**

TLS 핸드셰이크 실패(`PKIX`·`CERTIFICATE_VERIFY_FAILED`)인가, 아니면 **HTTP 응답이 오는가**(차단 페이지·403). 전자는 인증서, 후자는 정책이다. 이걸 안 가르면 엉뚱한 팀에 엉뚱한 요청을 하게 된다.

**2. 어느 검증 축에서 걸렸는지 가른다**

8.1 의 표로 메시지를 대조한다. `PKIX` 면 체인, `subject alternative names` 면 호스트명, `expired` 면 시각. **축이 다르면 조치가 완전히 다르다.**

**3. 가로채기 여부를 확인한다**

발급자가 공인 CA 가 아니면 중간자가 있다는 뜻이다.

```bash
openssl s_client -connect <host>:443 -servername <host> </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer
```

**4. 어느 신뢰 저장소가 문제인지 특정한다**

같은 대상에 런타임별로 붙여 본다. **브라우저는 되는데 JVM 만 안 되면 답은 이미 나온 것**이다.

```bash
keytool -list -keystore "$(/usr/libexec/java_home)/lib/security/cacerts" \
  -storepass changeit | grep -ic <발급자키워드>
```

`0` 이 나오면 그 JDK 에는 없다는 뜻. 장비에 JDK 가 여러 개면 **전부** 확인한다.

**5. 원본을 건드리지 말고 사본으로 가설을 검증한다**

`cacerts` 를 복사해 루트를 넣고 `-Djavax.net.ssl.trustStore=<사본>` 으로 띄워 본다. 통하면 원인 확정이고, 안 통하면 다른 축이다. **원본을 고친 뒤 아니었음을 알게 되는 상황을 피한다.**

**6. 캐시에 가려진 영향 범위를 확인한다**

빌드가 된다고 안심하지 않는다. **캐시에 없는 의존성을 하나 받아 보면** 진짜 상태가 드러난다.

## 13. 직접 해보기 — 미니 PKI 실습

이 노트의 실측 결과를 재현하는 스크립트다. 외부 네트워크가 전혀 필요 없고, 시스템 신뢰 저장소를 건드리지 않는다. (해시·서명·HMAC 같은 primitive 단위 실습은 [암호학 기초 부록 A.2](cryptography-fundamentals-for-backend.md#a2-실습-명령-모음)에.)

```bash
mkdir pki-demo && cd pki-demo
OS=openssl   # macOS 기본 openssl 은 LibreSSL 이라 동작이 다르다. 부록 A.3 참고

# 1) 루트 CA — 자기서명. CA 에 필요한 확장을 명시한다
$OS req -x509 -newkey rsa:2048 -nodes -keyout root.key -out root.crt -days 3650 \
  -subj "/CN=Demo Root CA/O=Study Lab" -extensions v3 \
  -config <(printf "[req]\ndistinguished_name=dn\n[dn]\n[v3]\nbasicConstraints=critical,CA:TRUE\nkeyUsage=critical,keyCertSign,cRLSign\nsubjectKeyIdentifier=hash\n")

# 2) 중간 CA — 루트가 서명
cat > ca.cnf <<'EOF'
basicConstraints=critical,CA:TRUE
keyUsage=critical,keyCertSign,cRLSign
subjectKeyIdentifier=hash
EOF
$OS req -newkey rsa:2048 -nodes -keyout inter.key -out inter.csr -subj "/CN=Demo Intermediate CA/O=Study Lab"
$OS x509 -req -in inter.csr -CA root.crt -CAkey root.key -CAcreateserial -out inter.crt -days 1825 -extfile ca.cnf

# 3) 서버 인증서 — 중간 CA 가 서명. SAN 을 꼭 넣는다 (8.3 참고)
cat > srv.cnf <<'EOF'
basicConstraints=critical,CA:FALSE
keyUsage=critical,digitalSignature,keyEncipherment
extendedKeyUsage=serverAuth
subjectAltName=DNS:localhost
EOF
$OS req -newkey rsa:2048 -nodes -keyout server.key -out server.csr -subj "/CN=localhost/O=Study Lab"
$OS x509 -req -in server.csr -CA inter.crt -CAkey inter.key -CAcreateserial -out server.crt -days 365 -extfile srv.cnf

# 4) 서버 기동 — 루트는 일부러 안 보낸다 (실제 서버와 동일)
$OS s_server -cert server.crt -key server.key -cert_chain inter.crt -accept 8443 -www -quiet &
```

이제 관찰한다.

```bash
# 서버가 실제로 보내주는 체인 — 루트가 없다
$OS s_client -connect localhost:8443 -servername localhost -showcerts </dev/null \
  | grep -E "^ *[0-9]+ s:|^ *i:"

# 협상된 암호 스위트 · 키 교환 그룹 · 핸드셰이크 서명 방식 (2.4 의 ④·⑤·⑥)
$OS s_client -connect localhost:8443 -servername localhost -CAfile root.crt </dev/null 2>/dev/null \
  | grep -iE "group|protocol|cipher is|signature type"

# 신뢰 앵커 없이 검증 → 실패
$OS verify server.crt

# 신뢰 앵커를 주고 검증 → OK
$OS verify -CAfile root.crt -untrusted inter.crt server.crt
```

런타임별 확인. 각각 **기본 상태에서 실패 → 루트를 주면 성공** 이 나오면 개념이 몸에 붙은 것이다.

```bash
# Java — 루트만 담은 truststore 를 만들어 지정
keytool -importcert -noprompt -alias demo-root -file root.crt \
  -keystore ts.p12 -storetype PKCS12 -storepass changeit
java Probe.java                                                   # PKIX path building failed
java -Djavax.net.ssl.trustStore=ts.p12 \
     -Djavax.net.ssl.trustStorePassword=changeit Probe.java       # OK 200

# Node
node probe.mjs                                                    # UNABLE_TO_GET_ISSUER_CERT_LOCALLY
NODE_EXTRA_CA_CERTS=root.crt node probe.mjs                       # OK 200

# Python
python3 probe.py                                                  # CERTIFICATE_VERIFY_FAILED
SSL_CERT_FILE=root.crt python3 probe.py                           # OK 200
```

`Probe.java`:

```java
import java.net.URI;
import javax.net.ssl.HttpsURLConnection;

public class Probe {
    public static void main(String[] a) throws Exception {
        var c = (HttpsURLConnection) URI.create(
            a.length > 0 ? a[0] : "https://localhost:8443/").toURL().openConnection();
        c.setConnectTimeout(3000);
        c.setReadTimeout(3000);
        try {
            System.out.println("OK   HTTP " + c.getResponseCode());
        } catch (Exception e) {
            Throwable r = e;
            while (r.getCause() != null) r = r.getCause();
            System.out.println("FAIL " + e.getMessage());
            System.out.println("root " + r.getClass().getSimpleName() + ": " + r.getMessage());
        }
    }
}
```

호스트명 축을 따로 보고 싶으면 마지막에 이걸 더 해 본다 — 체인은 유효한데 이름이 안 맞는 경우다.

```bash
java -Djavax.net.ssl.trustStore=ts.p12 -Djavax.net.ssl.trustStorePassword=changeit \
     Probe.java https://127.0.0.1:8443/
# FAIL No subject alternative names matching IP address 127.0.0.1 found
```

## 14. 기억할 문장

- 암호화가 됐다는 사실은 **누구와 통신 중인지에 대해 아무것도 말해주지 않는다.** 인증은 별도 장치가 필요하고 그게 인증서다.
- 인증서에 쓰이는 건 암호화가 아니라 **서명**이다. **개인키로 서명 → 공개키로 검증.**
- 인증서는 **공개된 파일**이다. 내미는 행위 자체는 아무것도 증명하지 않는다. **누가 서명했느냐**가 전부다.
- 루트 CA 는 자기가 자기를 서명한다 — **거기서 수학은 끝나고 "믿기로 한다"는 선언이 시작된다.**
- 신뢰 저장소는 **검증의 끝점**이다. 체인을 따라 올라가다 여기 닿으면 성공, 못 닿으면 실패.
- **루트는 서버가 보내주지 않는다.** 보내주면 위조범도 보낼 수 있어 검증이 무의미해지니까.
- **keystore 는 내가 내밀 것, truststore 는 내가 검사할 기준.** 개인키가 있냐 없냐로 갈린다.
- 런타임은 OS 신뢰 저장소를 보지 않는다 — **이식성을 위해 자기 목록을 들고 다닌다.**
- 신뢰 목록에 CA 를 넣는 것 = **그 CA 에게 모든 도메인을 사칭할 권한을 주는 것.**
- `PKIX path building failed` 는 **차단이 아니라 내 클라이언트의 거부다.**
- **빌드가 성공한다고 네트워크가 정상인 건 아니다** — 캐시가 가리고 있을 수 있다.

---

## 부록 A

### A.1 openssl · keytool 명령 사전

**보기**

```bash
# 인증서 전체 내용
openssl x509 -in cert.crt -noout -text

# 핵심만
openssl x509 -in cert.crt -noout -subject -issuer -dates

# 서버가 보내는 체인 전체
openssl s_client -connect <host>:443 -servername <host> -showcerts </dev/null

# 자기서명인지 확인 (subject 와 issuer 가 같으면 루트)
openssl x509 -in cert.crt -noout -subject -issuer
```

**검증**

```bash
# 신뢰 앵커를 지정해 검증
openssl verify -CAfile root.crt -untrusted inter.crt server.crt
```

**Java 신뢰 저장소**

```bash
# 목록 보기
keytool -list -keystore <store> -storepass changeit

# 특정 발급자가 들어 있는지 세기
keytool -list -keystore <store> -storepass changeit | grep -ic <키워드>

# 추가
keytool -importcert -alias <별칭> -file root.crt -keystore <store> -storepass changeit

# 제거
keytool -delete -alias <별칭> -keystore <store> -storepass changeit

# 현재 JDK 의 cacerts 경로 (macOS)
echo "$(/usr/libexec/java_home)/lib/security/cacerts"
```

**형식 변환** — `.pem`(텍스트, `-----BEGIN CERTIFICATE-----`)과 `.der`(바이너리)는 같은 내용의 다른 인코딩이다.

```bash
openssl x509 -in cert.der -inform der -out cert.pem -outform pem
```

### A.2 신뢰 저장소를 건드릴 때의 안전 수칙

1. **원본을 고치기 전에 사본으로 검증한다.** `cacerts` 를 복사해 넣고 `-Djavax.net.ssl.trustStore=<사본>` 으로 확인한 뒤에 결정한다.
2. **덮어쓰는 방식인지 추가하는 방식인지 확인한다.** `SSL_CERT_FILE` 류는 대체다 — 기존 번들과 합쳐야 한다.
   ```bash
   cat /opt/homebrew/etc/openssl@3/cert.pem company-root.crt > combined-ca.pem
   ```
3. **출처를 확인한다.** 10.3 에서 본 대로 루트 하나를 넣는 것은 큰 권한을 주는 일이다.
4. **JDK 는 업그레이드하면 `cacerts` 가 새 파일로 바뀐다.** 직접 수정 방식을 택했다면 업그레이드 때마다 다시 넣어야 한다는 걸 기록으로 남겨 둔다.

### A.3 조사 중에 걸린 함정들

실제로 잘못된 결론을 낼 뻔했던 지점들이다. 같은 함정을 다시 밟지 않으려고 적어 둔다.

**macOS 기본 `openssl` 은 `-CAfile` 을 무시할 수 있다.** `/usr/bin/openssl` 은 OpenSSL 이 아니라 **Apple LibreSSL** 이고, 시스템 키체인을 함께 참조한다. 그래서 `-CAfile` 로 특정 번들만 준 검증이 `Verify return code: 0 (ok)` 로 나와도 **그게 그 번들 덕분이라는 보장이 없다.** 신뢰 저장소 실험에는 별도 설치한 OpenSSL 을 쓰거나, 아예 다른 런타임(JVM 등)으로 교차 확인해야 한다.

```bash
openssl version   # LibreSSL 이면 주의
```

**CA 인증서에 `keyUsage` 확장이 없으면 별도 오류가 난다.** 데모 CA 를 확장 없이 만들었더니 Python 에서 이런 게 나왔다.

```
CERTIFICATE_VERIFY_FAILED: CA cert does not include key usage extension
```

체인 문제로 착각하기 딱 좋은데, 실제로는 **CA 인증서 자체가 규격 미달**이라는 뜻이다. 자체 CA 를 만들 때는 `basicConstraints=CA:TRUE` 와 `keyUsage=keyCertSign,cRLSign` 을 반드시 넣는다.

**"연결이 된다"와 "검증이 된다"는 다르다.** 어떤 도구는 검증 실패를 경고만 하고 진행한다(`curl -k`, 브라우저의 "계속하기"). 진단할 때는 **검증을 실제로 수행하는 경로**로 확인해야 한다.

**같은 장비에 JDK 가 여러 개면 신뢰 저장소도 여러 개다.** `/usr/libexec/java_home` 이 가리키는 것 하나만 확인하면 놓친다. IDE 가 쓰는 JDK 는 또 다를 수 있으니 IDE 설정에서 실제 경로를 확인한다.
