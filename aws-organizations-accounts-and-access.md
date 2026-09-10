# 인증용 임시 AWS 계정을 만들다 배우는 AWS 계정 · 조직 · 로그인 구조

> 개별 AWS 서비스(ECS Fargate, ALB, RDS 등)는 써봤지만 **계정 · 조직 · 사용자 · 권한의 관계**가 헷갈리는 상태에서,
> "운영 서비스는 그대로 두고 인증용 임시 계정을 따로 만들었다가 나중에 정리한다"는 실제 과제를 따라가며 정리한 문서.
> 회사 사례에서 출발했지만 이름은 모두 중립 예시로 바꿨다(아래 대응표).
> 작성일: 2026-09-10 · AWS 공식 문서 확인일: 2026-09-10

---

## 0. 이번 사례 한 문단 요약

회사에는 이미 **운영 서비스용 AWS 계정**이 있고, 나는 회사 이메일로 그 계정에 들어가서 ECS·ALB·RDS를 만지고 있다. 그런데 그 이메일은 **그 계정의 루트 이메일이 아니다.** 이제 소프트웨어 품질인증(예: GS 인증)을 받기 위해 **인증 심사용 AWS 계정을 따로** 하나 만들려고 한다. 기존 회사 **Organization** 안에 **OU**를 새로 만들고, 그 안에 **멤버 계정**을 만들어 쓰다가, 인증이 끝나면 그 계정과 OU만 정리하고 기존 운영 계정과 내 접근 권한은 그대로 유지하고 싶다. 계정 만들 때 요구하는 "소유자 이메일"에 내 업무 이메일을 그대로 넣어도 되는지, 나중에 정리할 때 무엇을 지워야 하고 무엇은 건드리면 안 되는지가 문제다.

이 문단에 모르는 단어가 하나라도 있으면 이 문서가 도움이 된다. 아래 순서대로 읽으면 위 문단이 전부 이해된다.

### 이 문서에서 쓰는 이름

실제 사례의 이름을 아래처럼 바꿨다. 읽을 때 머릿속으로 되돌려 대입하면 된다.

- `PROD` — 이미 운영 중인 서비스가 올라가 있는 AWS 계정
- `CERT` — 이번에 새로 만들 인증 심사용 AWS 계정 (끝나면 정리 대상)
- `Cert` OU — `CERT` 계정을 담기 위해 새로 만드는 OU
- `me@company.example` — 내 업무 이메일(직원 신원으로 쓰는 주소)
- `aws-root+cert@company.example` — 회사가 관리하는 AWS 계정 소유자 전용 주소(권장안에서 등장)

---

## 1. 내가 물어본 것들 → 어디를 보면 되나

| 질문 | 섹션 |
|---|---|
| Organization / Root / OU / 계정은 뭐가 뭘 담는 건가 | [2장](#2-aws-조직의-전체-구조--무엇이-무엇을-담고-있나) |
| 'AWS 계정'과 '내가 로그인하는 사용자'는 뭐가 다른가 | [3장](#3-계정과-사람은-서로-다른-축이다) |
| 관리 계정 vs 멤버 계정, 루트 사용자 vs 관리자는 왜 다른 구분인가 | [3.2](#32-두-가지-구분은-서로-다른-질문에-답한다) |
| 모든 계정에 OU가 필요한가 | [2.3](#23-ou는-필수가-아니다) |
| 루트 사용자 · IAM 사용자 · IAM 역할 · Identity Center는 각각 뭔가 | [4장](#4-로그인하는-방법은-네-가지뿐이다) |
| 같은 이메일인데 직원 신원과 루트 사용자가 어떻게 구분되나 | [5장](#5-같은-이메일인데-왜-다른-신원인가) |
| '계정을 조직에 초대'와 '사람을 초대'는 어떻게 다른가 | [5.3](#53-계정-초대와-사용자-초대는-완전히-다른-일이다) |
| 계정 안의 관리자와 조직을 관리하는 권한은 어떻게 다른가 | [6장](#6-어디를-관리하는-권한인가--계정-안과-조직-위) |
| SCP는 권한을 주는 건가 막는 건가 | [6.3](#63-scp는-권한을-주지-않는다--상한만-정한다) |
| 인증 계정을 만들고 쓰고 정리하는 전체 흐름 | [7장](#7-인증용-계정을-만들고-정리하기까지) |
| 리소스 삭제 / 계정 이동 / OU 삭제 / 조직에서 제거 / 계정 폐쇄 / 조직 삭제의 차이 | [7.3](#73-지우는-방법은-여섯-가지고-서로-다른-것을-지운다) |
| 루트 이메일은 나중에 재사용할 수 있나 | [7.4](#74-루트-이메일은-폐쇄-전에-바꿔야-한다) |
| 기존 운영 계정 접근이 유지되는 근거는 무엇인가 | [7.6](#76-기존-운영-계정prod-접근이-유지되는-전제) |
| 실제 정리 순서와 대기 기간은 | [7.5](#75-실제-정리-순서와-대기-기간) |
| 회사 규모에 따라 계정·OU를 어떻게 나누나 | [8장](#8-조직-규모에-따라-어떻게-나누나) |
| 우리 사례에서 가장 단순한 구성은 | [9장](#9-이번-사례에-적용하면) |
| Control Tower는 언제 도입하나 | [부록 A.1](#a1-control-tower--언제-고려하나) |
| 멤버 계정의 루트를 아예 없앨 수 있나 | [부록 A.2](#a2-루트-접근-중앙-관리--멤버-계정의-루트를-없애는-기능) |

---

## 2. AWS 조직의 전체 구조 — 무엇이 무엇을 담고 있나

### 2.1 먼저 용어 네 개

- **AWS 계정(AWS account)** — 리소스와 요금이 담기는 **그릇**이다. 12자리 계정 번호가 있고, EC2·RDS·S3 같은 리소스는 모두 "어떤 계정 안에" 존재한다. 사람이 아니다.
- **AWS Organizations(조직)** — 여러 AWS 계정을 **한 덩어리로 묶어서** 결제와 정책을 중앙에서 다루는 서비스. 보통 회사에 하나 있다.
- **Root(조직의 루트)** — 조직 트리의 **뿌리 컨테이너**. 조직을 만들면 AWS가 하나 만들어 준다("There is one root in the organization"). **계정이 아니고, 뒤에 나올 '루트 사용자'와도 완전히 다른 것이다.** 이름이 겹쳐서 헷갈리는 최대 원인이다.
- **OU(organizational unit, 조직 단위)** — Root 밑에 만드는 **폴더**. 계정을 묶어 정책을 한꺼번에 적용하려고 쓴다. OU 안에 OU를 중첩할 수 있다.

### 2.2 하나의 구조도

```
회사 Organization                      ← 조직. 회사에 보통 하나
└── Root                               ← 조직 트리의 뿌리. 조직당 하나. 계정이 아니다
    │
    ├── 관리 계정 (management account)  ← 조직을 만든 AWS 계정. 전체 요금을 낸다
    │      └ (가정) 여기서 IAM Identity Center를 운영해 직원 로그인을 뿌린다
    │
    ├── [OU] Workloads
    │     ├── 멤버 계정: PROD           ← 운영 서비스 (ECS Fargate / ALB / RDS)
    │     └── 멤버 계정: STAGING
    │
    ├── [OU] Cert                       ← 이번에 새로 만드는 OU
    │     └── 멤버 계정: CERT           ← 인증 심사용. 끝나면 정리 대상
    │
    └── 멤버 계정: SANDBOX              ← OU에 넣지 않고 Root 바로 아래 둬도 된다
```

**가정 표시**: 위에서 회사가 실제로 Organizations를 쓰고 있는지, 관리 계정이 어디인지, Identity Center를 쓰는지, `PROD`가 어느 OU에 있는지는 **확인되지 않은 가정**이다. 확인 방법은 [9.3](#93-먼저-확인해야-할-세-가지)에 있다.

역할은 두 가지만 기억하면 된다.

- **관리 계정(management account)** — 조직을 만든 계정. 조직에 계정을 만들거나 초대하고, OU를 만들고, 정책을 붙인다. **조직 전체 요금을 이 계정이 지불한다.** 조직당 하나다.
- **멤버 계정(member account)** — 그 조직에 속한 나머지 계정 전부. `PROD`, `CERT`가 여기 해당한다. 자기 안의 리소스는 자유롭게 만들지만, 조직 구조 자체는 못 바꾼다.

### 2.3 OU는 필수가 아니다

**아니다. 계정은 OU 없이 Root 바로 아래에 있어도 된다.** 공식 문서의 표현은 이렇다. "If an account isn't in an OU, it's subject to only the policies that are attached directly to the root and any policies that are attached directly to the account." ([Moving accounts to an OU](https://docs.aws.amazon.com/organizations/latest/userguide/move_account_to_ou.html), 확인 2026-09-10)

즉 OU는 **정책을 여러 계정에 한꺼번에 걸기 위한 도구**다. 계정 하나에만 다른 규칙을 적용할 거라면 OU를 만들 이유가 별로 없다. 참고로 조직 안에서 새로 만드는 멤버 계정은 **항상 Root에 먼저 생기고**, 그 다음에 OU로 옮기는 순서다("Member accounts in an organization can only be created in the root of an organization").

---

## 3. 계정과 사람은 서로 다른 축이다

여기가 이 문서의 핵심이다.

### 3.1 계정은 그릇, 사람은 그릇에 손을 넣는 주체

```
[AWS 계정]  = 리소스와 청구서가 담기는 그릇 (12자리 번호)
     ▲
     │ 이 안에서 무언가를 하려면 "주체(principal)"로 인증돼야 한다
     │
[주체 = 로그인하는 쪽]
  ├ 루트 사용자 (root user)          : 계정을 만들 때 등록한 이메일이 곧 로그인 ID. 계정당 정확히 1개
  ├ IAM 사용자 (IAM user)            : 계정 안에 만드는 로그인. 계정마다 따로 존재
  ├ IAM 역할 (IAM role)              : 잠시 빌려 쓰는 권한 꾸러미. 사람·AWS 서비스·다른 계정이 맡는다
  └ Identity Center 사용자(직원 신원) : 조직 차원의 사람 1명 = 1개.
                                       계정에 들어갈 때 그 계정의 IAM 역할로 변신해 들어간다
```

"내가 `PROD` 계정에 들어간다"는 말은 **`PROD`라는 그릇에 대해 어떤 주체로 인증됐다**는 뜻이다. 그릇 자체가 나는 아니다. 그래서 **계정에 이메일이 하나 붙어 있다는 사실과, 그 계정에 사람이 몇 명 들어오는지는 아무 관계가 없다.**

### 3.2 두 가지 구분은 서로 다른 질문에 답한다

헷갈리는 두 쌍이 있다. 이건 층이 다르다.

- **관리 계정 vs 멤버 계정** → 질문: *"조직 트리에서 이 계정의 위치가 어디인가?"*
  조직 전체를 소유·결제하는 계정이 관리 계정, 그 밑에 묶인 계정이 멤버 계정이다. **계정 단위의 구분**이다.
- **루트 사용자 vs 관리자 권한을 가진 사용자** → 질문: *"한 계정 안에서 이 로그인 주체가 어떤 종류인가?"*
  루트 사용자는 계정과 함께 태어난 **계정 소유자 신원**이고, 관리자 권한 사용자는 나중에 만들어 `AdministratorAccess` 같은 권한을 붙인 **일반 주체**다. **주체 단위의 구분**이다.

그래서 네 가지 조합이 모두 존재한다. 멤버 계정에도 루트 사용자가 있고(→ [부록 A.2](#a2-루트-접근-중앙-관리--멤버-계정의-루트를-없애는-기능)), 관리 계정에도 평범한 관리자 사용자가 있다. **"루트 사용자니까 조직을 관리한다"는 성립하지 않는다** — 조직을 관리하는 건 *관리 계정의* 주체이고, 멤버 계정의 루트 사용자는 자기 계정 안에서만 강할 뿐이다.

왜 실전에서 중요한가: 이번 사례에서 `CERT` 계정의 루트 이메일에 내 주소를 넣는다고 해서 내가 **조직**에 대해 뭔가 할 수 있게 되는 게 전혀 아니다. 반대로 지금 내가 `PROD`에서 관리자처럼 일하고 있다고 해서 내가 `PROD`의 루트 사용자인 것도 아니다.

---

## 4. 로그인하는 방법은 네 가지뿐이다

필요한 만큼만 정리한다.

### 4.1 루트 이메일과 루트 사용자

AWS 계정을 만들 때 반드시 이메일 주소 하나를 등록한다. 이 주소가 그 계정의 **루트 이메일**이고, 동시에 **루트 사용자의 로그인 ID**다. 공식 문서 표현: "This email address cannot already be associated with another AWS account because it becomes the user name credential for the root user of the account." ([Creating a member account](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_create.html), 확인 2026-09-10)

여기서 두 가지가 따라온다.

1. **한 이메일 주소는 AWS 계정 하나의 루트 이메일만 될 수 있다.** 이미 어딘가의 루트 이메일이면 새 계정 생성이 거부된다.
2. 루트 사용자는 그 계정에서 **권한 정책으로 제한할 수 없는** 특별한 신원이다. 그래서 AWS는 일상 작업에 쓰지 말라고 권고한다.

한 가지 중요한 사실이 있는데, **조직 안에서 새로 만든 멤버 계정은 기본적으로 루트 자격증명이 아예 없다.** "When you create new member account in your organization, the account has no root user credentials by default. Member accounts can't sign in to their root user or perform password recovery for their root user unless account recovery is enabled." ([Accessing member accounts](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_access.html), 확인 2026-09-10)

즉 멤버 계정의 루트 이메일은 실전에서 **"이 계정의 소유자·연락처가 누구인가"를 나타내는 식별자**에 가깝다. 그걸로 매일 로그인하는 게 아니다. 실제 로그인은 아래 세 가지로 한다.

### 4.2 IAM 사용자 — 계정 안에 만드는 로그인

**IAM(Identity and Access Management)** 은 계정 안의 권한을 다루는 서비스다. **IAM 사용자**는 그 계정 안에 만든 로그인 계정으로, 이름·비밀번호·액세스 키를 가진다. 문제는 **계정마다 따로 만들어야** 한다는 것이다. 계정이 3개면 사람 1명에게 IAM 사용자 3개를 만들고 비밀번호도 3개 관리해야 한다.

### 4.3 IAM 역할 — 잠시 빌려 쓰는 권한 꾸러미

**IAM 역할**은 비밀번호가 없고, "누가 맡을 수 있는지(신뢰 정책)"와 "맡으면 무엇을 할 수 있는지(권한 정책)"만 정의된 꾸러미다. 사람도, AWS 서비스(예: ECS 태스크)도, **다른 계정의 주체**도 맡을 수 있다. 맡는 순간 임시 자격증명을 받아 제한된 시간 동안 그 권한으로 일한다.

이 역할이 계정 사이를 잇는 접착제다. 실제로 조직 안에서 멤버 계정을 만들면 AWS가 그 안에 **`OrganizationAccountAccessRole`** 이라는 역할을 자동으로 만들어 준다. 관리 계정의 사용자가 이 역할을 맡아 새 계정을 관리자로 다루게 하는 통로다. (초대로 들어온 계정에는 자동 생성되지 않고, 필요하면 직접 만든다.)

### 4.4 IAM Identity Center와 Permission Set — 사람은 하나, 계정은 여러 개

**IAM Identity Center**(옛 AWS SSO)는 **조직 차원에서 사람 신원을 한 번만 만들고**, 그 사람이 어느 계정에 어떤 권한으로 들어갈지를 중앙에서 배정하는 서비스다. 사람은 회사 자격증명으로 **AWS 액세스 포털**에 로그인하고, 거기서 들어갈 계정을 고른다.

**Permission Set(권한 세트)** 은 그때 쓰는 **권한 템플릿**이다. 공식 설명: "When you assign a permission set, IAM Identity Center creates corresponding IAM Identity Center-controlled IAM roles in each account, and attaches the policies specified in the permission set to those roles." ([Permission sets](https://docs.aws.amazon.com/singlesignon/latest/userguide/permissionsetsconcept.html), 확인 2026-09-10)

풀어 쓰면 이렇다.

```
[Identity Center]  사람: me@company.example  (신원 1개)
       │  "이 사람에게 PROD 계정 + AdministratorAccess 권한 세트를 배정"
       ▼
[PROD 계정]  Identity Center가 관리하는 IAM 역할이 자동 생성됨
       │
       ▼  포털에서 클릭 → 그 역할을 맡아서 콘솔 진입
    (로그인은 조직에서, 권한은 계정 안 역할로)
```

즉 **Identity Center 사용자는 IAM 사용자가 아니라, "여러 계정의 IAM 역할을 골라 맡을 수 있는 사람 신원"** 이다. 계정을 새로 만들 때 사람마다 로그인을 새로 만들 필요가 없다는 게 핵심 이점이다.

---

## 5. 같은 이메일인데 왜 다른 신원인가

### 5.1 이메일은 신원이 아니라 라벨이다

`me@company.example` 이라는 **같은 문자열**이 서로 다른 두 자리에 쓰일 수 있다.

```
(A) 직원 신원 (Identity Center 사용자 또는 IAM 사용자)
    - 사는 곳: 조직의 Identity Center (또는 특정 계정의 IAM)
    - 이메일의 역할: 로그인 아이디 또는 표시용 속성
    - 할 수 있는 일: 배정받은 계정 + 배정받은 권한 세트 범위
    - 이번 사례에서: 내가 PROD에 들어갈 때 쓰는 신원

(B) CERT 계정의 루트 사용자
    - 사는 곳: CERT 계정 하나
    - 이메일의 역할: 그 계정의 소유자 주소 = 루트 사용자 로그인 ID
    - 할 수 있는 일: CERT 계정 안에서 제한 없음. 조직에 대해서는 아무 권한 없음
    - 실제로는 자격증명이 없어서 로그인 자체가 막혀 있다 (4.1 참고)
```

AWS는 이 둘을 **같은 것으로 취급하지 않는다.** 사는 시스템이 다르고, 인증 경로가 다르고, 권한 범위가 다르다. 이메일이 같아도 "한 신원"이 되지 않는다. 사람이 헷갈릴 뿐이다 — 그리고 **바로 그 헷갈림이 사고를 만든다**([9.2](#92-루트-이메일은-업무-이메일보다-관리-전용-주소가-낫다)).

### 5.2 그럼 `PROD`는 어떻게 되어 있나

사례에서 "내 이메일은 `PROD`의 루트 이메일이 아니다"라고 했다. 그렇다면 내가 `PROD`에 들어가는 신원은 (A)다 — Identity Center 사용자이거나 `PROD` 안의 IAM 사용자다. 어느 쪽인지는 확인이 필요하고, 확인 방법은 [9.3](#93-먼저-확인해야-할-세-가지)에 있다. **어느 쪽이든 `CERT`를 만들거나 지우는 일이 내 `PROD` 접근을 건드리지 않는다.** 그 신원은 `CERT` 계정에 들어 있지 않기 때문이다.

### 5.3 계정 초대와 사용자 초대는 완전히 다른 일이다

이름이 둘 다 "초대"라서 섞이는데, 대상이 다르다.

- **계정을 조직에 초대(account invitation)** — 이미 존재하는 **AWS 계정**을 조직의 멤버로 편입시키는 것. 초대를 받은 쪽 **계정의 관리자/소유자**가 수락해야 하고, 초대는 **15일 안에** 응답하지 않으면 만료된다. 수락한 순간부터 **그 계정의 요금은 관리 계정이 낸다**. 계정은 **한 조직에만** 속할 수 있다. ([Managing account invitations](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_invites.html), 확인 2026-09-10)
- **사람을 초대(직원 사용자 추가)** — Identity Center에 **사람 신원**을 만들고 계정·권한 세트를 배정하는 것. 계정의 소속을 바꾸지 않고, 요금 주체도 바꾸지 않는다.

이번 사례에서 `CERT`는 **조직 안에서 새로 만드는** 계정이므로 계정 초대 절차가 아예 필요 없다. 내가 `CERT`에서 일하려면 필요한 건 **사람 쪽 배정**이다.

---

## 6. 어디를 관리하는 권한인가 — 계정 안과 조직 위

### 6.1 계정 안의 관리자

`CERT` 계정 안에서 `AdministratorAccess` 를 가진 주체는 그 계정 안에서 사실상 무엇이든 한다. ECS 클러스터를 만들고, RDS를 띄우고, IAM 사용자를 만든다. **하지만 그 권한은 `CERT`의 경계에서 끝난다.** `PROD`의 리소스도, 조직 구조도 건드릴 수 없다.

### 6.2 조직을 관리하는 권한

OU를 만들고, 계정을 만들고, 계정을 OU 사이로 옮기고, SCP를 붙이는 일은 **`organizations:*` 계열 권한을 가진 관리 계정의 주체**만 할 수 있다. 문서마다 "Sign in to the AWS Organizations console... in the organization's management account"라고 못 박혀 있고, 필요한 권한도 `organizations:CreateAccount`, `organizations:MoveAccount`, `organizations:DeleteOrganizationalUnit` 처럼 명시돼 있다.

그래서 이번 작업의 권한 요구는 두 층으로 갈린다.

```
[관리 계정 쪽 권한]  Cert OU 만들기 · CERT 계정 만들기 · OU로 이동 · 나중에 정리
[CERT 계정 쪽 권한]  ECS/ALB/RDS 배포하고 인증 심사 대응
```

**둘 다 필요하다면 그건 두 개의 별개 승인 요청이다.** 내가 `CERT` 안에서 관리자여도 조직 작업은 못 하고, 조직 작업을 해줄 사람이 `CERT` 안에서 뭘 만들어줄 필요는 없다. 회사에 조직 관리자가 따로 있다면, **"Cert OU와 CERT 계정을 만들어 주고, 내 신원에 CERT 관리자 권한을 배정해 달라"** 가 정확한 요청 형태다.

### 6.3 SCP는 권한을 주지 않는다 — 상한만 정한다

**SCP(service control policy, 서비스 제어 정책)** 는 Root·OU·계정에 붙이는 조직 정책이다. 헷갈리기 쉬운데, 문서가 단호하다.

> "SCPs do not grant permissions to the IAM users and IAM roles in your organization. No permissions are granted by an SCP. An SCP defines a permission guardrail, or sets limits, on the actions that the IAM users and IAM roles in your organization can perform."
> — [Service control policies](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html) (확인 2026-09-10)

정리하면 이렇다.

- SCP는 **권한의 상한선**이다. 실제 권한은 IAM 정책이 준다. 최종 권한은 **SCP가 허용한 것 ∩ IAM이 허용한 것**이다.
- SCP는 **멤버 계정의 루트 사용자에게도 적용된다.** 대신 **관리 계정의 주체에게는 적용되지 않는다.**
- SCP를 켜면 모든 대상에 기본 정책 `FullAWSAccess` 가 붙어 있다. 이걸 대체 정책 없이 떼면 **멤버 계정의 모든 AWS 동작이 실패한다.**
- 서비스 연결 역할(service-linked role)에는 적용되지 않는다.

왜 실전에서 중요한가: "OU를 만들었는데 권한이 안 생긴다"는 착각이 여기서 나온다. **OU와 SCP는 막는 도구**고, 사람이 들어가려면 [4.4](#44-iam-identity-center와-permission-set--사람은-하나-계정은-여러-개)의 배정이 별도로 있어야 한다.

---

## 7. 인증용 계정을 만들고 정리하기까지

### 7.1 전체 흐름

```
1) [관리 계정] Cert OU 생성
2) [관리 계정] CERT 멤버 계정 생성        → Root에 생성됨 + OrganizationAccountAccessRole 자동 생성
3) [관리 계정] CERT를 Cert OU로 이동      → 그 순간부터 Cert OU의 정책이 적용됨
4) [관리 계정] 내 신원에 CERT 접근 배정   → Identity Center 권한 세트 배정(권장)
                                            또는 OrganizationAccountAccessRole 맡기
5) [CERT 계정]  ECS/ALB/RDS 배포, 인증 심사 대응
6) 인증 종료 후 정리                       → 7.5의 순서를 따른다
```

3단계가 따로 있는 이유는 [2.3](#23-ou는-필수가-아니다)에서 본 대로 **새 멤버 계정은 항상 Root에 먼저 생기기 때문**이다.

### 7.2 정리 작업을 헷갈리면 사고가 난다

"정리한다"는 말 하나에 서로 다른 여섯 가지 작업이 섞여 있다. 각각 **바꾸는 대상이 다르고, 되돌릴 수 있는 정도가 다르다.**

### 7.3 지우는 방법은 여섯 가지고 서로 다른 것을 지운다

| 작업 | 계정은? | 리소스는? | 로그인은? | 비용은? |
|---|---|---|---|---|
| **리소스 삭제** (RDS/ECS 등) | 그대로 남음 | 지운 것만 사라짐 | 그대로 | 지운 리소스 과금만 멈춤 |
| **OU 간 계정 이동** | 그대로. 소속 폴더만 바뀜 | 영향 없음 | 그대로 | 그대로(관리 계정이 계속 결제) |
| **OU 삭제** | 영향 없음(비어 있어야 삭제 가능) | 영향 없음 | 그대로 | 변화 없음 |
| **조직에서 계정 제거** | 남지만 **독립 계정**이 됨 | 영향 없음 | 그대로 | **그 계정이 자기 요금을 직접 냄**. SCP 제약도 사라짐 |
| **AWS 계정 폐쇄** | 90일간 `CLOSED`, 이후 영구 폐쇄 | 폐쇄 직후엔 남아 있고, **영구 폐쇄 시 삭제**(CloudTrail 추적 제외) | 서비스 사용 불가. 폐쇄 기간엔 과거 청구서 열람 등만 가능 | 폐쇄 전 사용분은 **다음 달 청구서로 온다**. RI·Savings Plans는 만료까지 계속 청구 |
| **Organization 삭제** | 관리 계정이 **독립 계정**으로 되돌아감(폐쇄 아님) | 영향 없음 | 조직 기반 서비스(Identity Center 등)가 동작 불가 | 통합 결제가 끝남. 조직 단위 비용 데이터·리포트가 삭제됨 |

근거: [Removing a member account](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_remove.html), [Deleting an OU](https://docs.aws.amazon.com/organizations/latest/userguide/delete-ou.html), [Close an AWS account](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-closing.html), [Deleting an organization](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_org_delete.html) — 모두 확인 2026-09-10.

이 표에서 반드시 붙잡아야 할 세 가지.

1. **`Organization` 삭제는 이번 작업에 필요 없고, 하면 안 된다.** 조직을 삭제하려면 **모든 멤버 계정을 먼저 제거**해야 하고(즉 `PROD`까지 떼어내야 한다), Identity Center가 동작을 멈추며, 조직 단위 비용 데이터가 삭제되고, 복구가 불가능하다. `CERT` 하나를 정리하는 일과는 완전히 다른 층의 작업이다.
2. **"조직에서 제거"와 "계정 폐쇄"는 다르다.** 제거는 계정을 살려서 독립시키는 것(요금 책임이 그 계정으로 넘어간다), 폐쇄는 계정을 없애는 것이다.
3. **OU는 비어 있어야 삭제된다.** "You must first move all accounts out of the OU and any child OUs, and then you can delete the child OUs."

### 7.4 루트 이메일은 폐쇄 전에 바꿔야 한다

여기가 이번 사례에서 가장 중요한 제약이다. 계정을 폐쇄하면 **그때 등록돼 있던 이메일 주소는 다른 AWS 계정의 기본 이메일로 다시 쓸 수 없다.**

> "You can't use the same email address that was registered to your AWS account at the time of its closure as the primary email of another AWS account. If you want to use the same email address for a different AWS account, we recommend updating it before closure."
> — [Close an AWS account](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-closing.html) (확인 2026-09-10)

계정 별칭(account alias)도 같은 성질이라, 재사용하려면 폐쇄 전에 지워야 한다. 계정 번호는 **영구 폐쇄 후 절대 재사용되지 않는다**("its AWS account ID can never be reused").

그래서 폐쇄 전에 루트 이메일을 바꿔야 한다면 방법은 두 가지다.

- **관리 계정에서 중앙 변경** — all features 모드이고 Account Management 서비스에 대한 신뢰 액세스(trusted access)를 켜둔 조직이라면, 관리 계정(또는 위임 관리자 계정)의 주체가 Organizations 콘솔이나 CLI(`account start-primary-email-update` → `account accept-primary-email-update`)로 멤버 계정의 루트 이메일을 바꿀 수 있다. **새 주소로 일회용 코드(OTP)가 가고, 24시간 안에 입력해야 한다.** 이 방식은 원래 루트 자격증명이 없어도 된다.
- **해당 계정의 루트 사용자로 로그인해 변경** — 콘솔의 **Account** 페이지에서 바꾼다. 새 주소로 인증 코드가 온다. **이 절차는 IAM 사용자·역할로는 불가능하다.**

근거: [Update the root user email address](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-update-root-user-email.html), [Updating the root user email address for a member account](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_update_primary_email.html) — 확인 2026-09-10. 계정 변경 사항은 전파에 최대 4시간 걸릴 수 있다.

### 7.5 실제 정리 순서와 대기 기간

폐쇄 직후와 영구 폐쇄 후는 다르다.

- **폐쇄 직후(post-closure period, 90일)** — 계정은 Organizations 콘솔에 `CLOSED` 라벨로 **계속 보인다**. 조직의 계정 수 할당량도 계속 차지한다. AWS Support에 연락하고 미납금을 정산하면 **되돌릴 수 있다**(재개하면 남아 있던 서비스 요금이 다시 시작된다).
- **영구 폐쇄(90일 경과)** — 되돌릴 수 없다. AWS가 **콘텐츠와 리소스를 삭제한다**(CloudTrail 추적은 예외). 계정 번호는 영구히 재사용 불가. 조직에서 계정이 사라지는 데는 **영구 폐쇄 후 며칠 더** 걸릴 수 있다.

또 하나. **폐쇄하기 전에 조직에서 제거하면 90일 동안 할당량을 차지하는 문제를 피할 수 있다**고 문서가 권한다("To avoid having the account count against the quota, see Remove a member account from your organization before closing it"). 단 **조직에서 제거하려면 조건이 있다.**

- **조직 안에서 만든 계정은 생성 후 최소 4일이 지나야** 제거할 수 있다(초대로 들어온 계정은 이 대기 없음).
- 제거하려면 그 계정이 **독립 계정으로 동작할 정보**를 갖춰야 한다 — Support 플랜 선택, 연락처 정보 입력·검증, **유효한 결제 수단** 등록. 조직 안에서 만든 계정은 이 정보가 자동으로 채워지지 않는다.
- 제거되는 순간부터 **그 계정 소유자가 새로 발생하는 요금을 책임진다.**
- 폐쇄 관련 상한: 롤링 30일 동안 멤버 계정의 **20% 또는 250개 중 더 큰 값**(최대 1,000개)까지만 폐쇄할 수 있다.

그래서 실무 순서는 두 갈래다.

```
[갈래 1 — 결제 수단 등록이 부담스러울 때 (가장 단순)]
 1. CERT 안의 리소스 정리(백업 먼저) → 과금 중단
 2. 필요하면 루트 이메일을 폐쇄 전용 주소로 변경 (7.4)
 3. 관리 계정에서 CERT 폐쇄
 4. CERT를 Root로 이동해 Cert OU를 비운 뒤 OU 삭제
    (OU는 비어 있어야 삭제된다. CLOSED 계정은 최대 90일 트리에 남는다)
 5. 90일 후 영구 폐쇄 — 이때 AWS가 남은 리소스를 삭제한다
 → Organization은 건드리지 않는다. PROD와 내 로그인은 그대로.

[갈래 2 — 조직 할당량을 아끼거나 계정을 남길 때]
 1. 리소스 정리 + 루트 이메일 정리
 2. CERT에 Support 플랜·연락처·결제 수단 채우기 (생성 후 4일 경과 필요)
 3. 조직에서 CERT 제거 → 독립 계정이 됨 (이후 요금은 그 계정 책임)
 4. Cert OU 삭제
 5. 독립 계정 상태에서 루트 사용자로 로그인해 폐쇄 (원한다면)
```

두 갈래 모두 남는 청소가 하나 있다. **조직에서 계정을 제거해도 관리 계정 접근용으로 만들어진 IAM 역할(`OrganizationAccountAccessRole` 등)은 자동으로 삭제되지 않는다.** 접근을 확실히 끊으려면 직접 지워야 한다.

### 7.6 기존 운영 계정(PROD) 접근이 유지되는 전제

`CERT`를 만들고 지우는 동안 `PROD` 접근이 유지되는 근거는 이렇다. **다만 아래 전제가 깨지면 결론도 달라진다.**

- 내 로그인 신원이 **`CERT` 계정 안에 없다.** Identity Center 사용자든 `PROD`의 IAM 사용자든, `CERT` 폐쇄는 그 신원을 건드리지 않는다.
- **Organization 자체를 삭제하지 않는다.** 조직을 삭제하면 Identity Center가 동작을 멈추므로 SSO 로그인이 깨진다. 이번 작업에 조직 삭제는 포함되지 않는다.
- **Root나 상위 OU에 SCP를 새로 붙이지 않는다.** `Cert` OU에만 정책을 붙이면 `PROD`에 영향이 없다. Root에 붙이면 조직 전체에 걸린다 — AWS도 "Root에 SCP를 바로 붙이지 말고 OU를 만들어 소수 계정부터 시험하라"고 권고한다.
- 관리 계정과 `PROD` 계정을 폐쇄하지 않는다.

---

## 8. 조직 규모에 따라 어떻게 나누나

### 8.1 판단 기준은 인원수가 아니다

먼저 AWS의 공식 권장 사항과, 설명을 위한 구성 예시를 구분해 두자.

**AWS 공식 권장 사항** ([Recommended OUs and accounts](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/recommended-ous-and-accounts.html), 확인 2026-09-10)은 기초 OU로 **Security OU**(보안 정책·거버넌스 계정)와 **Infrastructure OU**(공용 네트워크·인프라 계정), 워크로드용 **Workloads OU**(운영/비운영 환경 모두 포함), 실험용 **Sandbox OU** 등을 든다. 다만 백서가 직접 이렇게 말한다.

> "Depending on your requirements, you might not need to establish all the recommended OUs. As you adopt AWS and learn more about your needs, you can expand the overall set of OUs."

즉 **처음부터 다 만들 필요는 없다.** 아래 세 가지는 이해를 돕기 위한 **예시 구성**이고, AWS가 규정한 등급이 아니다. 판단 기준으로는 인원수보다 다음을 쓴다: **서비스 개수 / 팀별 책임 경계 / 보안·규제 요구 / 운영 여력(계정을 늘리면 늘어나는 관리 부담을 감당할 사람이 있나)**.

### 8.2 예시 1 — 개인 또는 소규모 팀

계정 1~2개. 조직을 만들었다면 관리 계정 + 워크로드 계정 하나 정도이고, **OU는 굳이 만들지 않아도 된다.** 환경은 태그와 네이밍, 또는 별도 VPC로 구분한다.

- 왜: 계정을 나누면 VPC·IAM·CI 자격증명·비용 태깅을 계정 수만큼 반복해야 한다. 인원이 적으면 이 반복이 곧 사고 원인이 된다.
- 언제 쪼개나: **운영 리소스를 실수로 지울 위험이 실제로 감지될 때** 운영/비운영 계정을 먼저 분리한다. 계정 경계는 IAM 정책보다 훨씬 실수에 강한 격리다.

### 8.3 예시 2 — 여러 개발팀이 있는 성장기 회사

관리 계정 + `Workloads` OU(`prod`, `staging` 또는 서비스별 운영/비운영 계정) + `Sandbox` OU 정도.

- 왜: 팀별로 책임이 갈리기 시작하면 "누가 무엇을 지울 수 있는가"를 IAM 정책으로만 표현하기 어려워진다. 계정을 나누면 폭발 반경(blast radius)과 비용이 자연스럽게 갈린다.
- 이때부터 **Identity Center가 사실상 필수다.** 계정마다 IAM 사용자를 만들면 사람 수 × 계정 수만큼 자격증명이 늘어난다.
- 관리 부담: SCP 설계, 계정 생성 표준화, 네트워크 공유(Transit Gateway 등), 계정별 비용 가시성이 새 업무로 생긴다.

### 8.4 예시 3 — 보안·감사 요구가 높은 대규모 조직

위 구조에 **Security OU**(로그 보관 계정, 보안 도구 계정)와 **Infrastructure OU**(공용 네트워크)를 더한다.

- **로그·보안 계정을 언제 분리하나**: 감사 로그를 **워크로드 계정 관리자가 지울 수 없어야 한다**는 요구가 생기는 순간이다. 같은 계정에 로그를 두면 그 계정 관리자가 지울 수 있으니 요구를 만족할 수 없다.
- **임시 인증 환경(이번 `CERT` 같은 것)을 분리하는 이유**: 수명이 정해져 있고 정리 시점에 통째로 없애야 하며, 심사자에게 보여줄 범위를 좁혀야 하기 때문이다. 백서에도 이런 용도의 절차용 OU 개념(`Transitional`, `Exceptions` 등)이 있다.
- 관리 부담: 이 단계에서는 사람이 손으로 유지하기 어려워진다 → [부록 A.1](#a1-control-tower--언제-고려하나).

---

## 9. 이번 사례에 적용하면

### 9.1 가장 단순한 구성

현재 정보에서 권하는 구성은 이렇다.

```
회사 Organization (그대로 유지 — 삭제 대상 아님)
└── Root
    ├── 관리 계정            (그대로)
    ├── (기존) PROD 계정      (그대로. 내 접근도 그대로)
    └── [OU] Cert            ← 새로 만듦
          └── CERT 계정      ← 새로 만듦. 인증 끝나면 정리
```

- **계정은 하나만 새로 만든다.** 인증 심사 환경을 운영과 섞지 않는 것만으로 목적은 달성된다.
- **OU를 하나 만드는 것은 정당하다.** 계정 하나뿐이라 기술적으로는 필수가 아니지만([2.3](#23-ou는-필수가-아니다)), 인증용 계정에만 다른 규칙(예: 리전 제한, 특정 서비스 금지)을 걸거나 나중에 통째로 정리할 단위를 눈에 보이게 하는 값이 있다. 정책을 하나도 붙일 계획이 없다면 OU 없이 Root 아래 둬도 된다.
- **접근은 Identity Center 권한 세트 배정으로 받는다.** 회사가 Identity Center를 쓰고 있다면 이게 가장 깔끔하다. 아니라면 관리 계정 사용자가 `OrganizationAccountAccessRole` 을 맡아 들어가거나, `CERT` 안에 IAM 사용자를 만든다.
- **Root에는 아무 정책도 새로 붙이지 않는다.** `PROD`에 영향이 가지 않게 하는 가장 확실한 방법이다.

### 9.2 루트 이메일은 업무 이메일보다 관리 전용 주소가 낫다

검토 중이던 안은 `CERT`의 루트 이메일로 내 업무 이메일을 넣는 것이었다. **권하지 않는다.** 이유는 취향이 아니라 문서화된 제약이다.

1. **폐쇄하면 그 주소를 다시 못 쓴다.** [7.4](#74-루트-이메일은-폐쇄-전에-바꿔야-한다)의 규정 그대로, 폐쇄 시점에 등록돼 있던 주소는 이후 다른 AWS 계정의 기본 이메일이 될 수 없다. 내 상용 업무 이메일을 **일회용 계정에 태워버리는** 셈이다.
2. **한 주소는 한 계정의 루트 이메일만 될 수 있다.** 나중에 다른 AWS 계정을 만들 일이 생겼을 때 그 주소가 막혀 있다.
3. **회사 자산의 소유자 주소가 개인 퇴사와 묶인다.** 담당자가 바뀌면 소유자 주소도 바꿔야 한다.
4. **혼동 자체가 위험하다.** [5.1](#51-이메일은-신원이-아니라-라벨이다)에서 본 두 신원이 같은 문자열을 쓰면, "이 계정은 내 계정"이라는 착각으로 이어지기 쉽다.

권하는 형태는 **회사가 관리하는 AWS 계정 소유자 전용 주소**다. 예: `aws-root+cert@company.example` 같은 그룹/별칭 주소로, 여러 사람이 수신할 수 있고 담당자가 바뀌어도 유지되는 것. 인증이 끝나 폐쇄할 때도 잃는 게 없다.

이미 업무 이메일로 만들어 버렸다면 되돌릴 방법이 있다. **폐쇄하기 전에** [7.4](#74-루트-이메일은-폐쇄-전에-바꿔야-한다)의 중앙 변경으로 폐쇄 전용 주소(예: `aws-closed-cert-2026@company.example`)로 바꿔두면 업무 이메일을 회수할 수 있다. **폐쇄 후에는 불가능하다.**

### 9.3 먼저 확인해야 할 세 가지

이 문서의 여러 결론이 아래 확인에 달려 있다. 전부 읽기만 하는 확인이다.

1. **내 로그인 방식이 무엇인가** — 로그인 URL이 `<something>.awsapps.com/start` 형태의 AWS 액세스 포털이면 Identity Center 사용자다. 12자리 계정 번호와 사용자 이름으로 콘솔에 직접 로그인하면 IAM 사용자다. `aws sts get-caller-identity` 결과의 ARN이 `arn:aws:sts::<계정>:assumed-role/AWSReservedSSO_...` 이면 Identity Center, `arn:aws:iam::<계정>:user/...` 이면 IAM 사용자다.
2. **회사가 Organizations를 쓰고 있고, 관리 계정이 어디인가** — 관리 계정 주체라면 Organizations 콘솔에서 트리가 보인다. 조직이 **all features 모드**인지도 함께 확인해야 한다. SCP와 멤버 루트 이메일 중앙 변경, Organizations 콘솔에서의 멤버 계정 폐쇄가 모두 이 모드를 요구한다.
3. **`CERT`에 쓸 이메일이 이미 다른 AWS 계정의 루트 이메일인지** — 이미 쓰이고 있으면 계정 생성이 실패한다.

### 9.4 인증기관 요구사항은 별도로 확인해야 한다

이 문서는 **AWS 구조만** 다룬다. 인증 심사가 "AWS 계정을 분리해야 한다", "특정 로그를 몇 년 보관해야 한다", "특정 리전을 써야 한다" 같은 요구를 하는지는 **여기서 추정하지 않았고, 추정해서도 안 된다.** 계정을 새로 분리하는 근거는 어디까지나 [8.4](#84-예시-3--보안감사-요구가-높은-대규모-조직)의 일반 원칙(수명이 정해진 환경, 심사 범위 축소, 통째 정리 용이)이다.

인증기관 문서에서 따로 확인할 것: 심사 대상 환경의 격리 요구 유무, 로그·증적 보관 기간과 형태, 심사원 접근 방식(계정 로그인이 필요한가 화면 시연으로 되는가), 인증 후 환경 유지 의무 기간. **여기에 따라 "인증 종료 즉시 폐쇄"가 아니라 "일정 기간 보존"이 필요할 수도 있다.**

---

## 10. 이해했는지 확인하기

### 질문 1

`CERT` 계정을 만들 때 루트 이메일로 내 업무 이메일을 넣으면, 나는 그 계정과 회사 Organization에 대해 각각 무엇을 할 수 있게 되는가?

<details>
<summary>모범 답안</summary>

**둘 다 "특별히 생기는 것이 없다"** 가 정답이다.

조직 안에서 새로 만든 멤버 계정은 **기본적으로 루트 자격증명이 없다.** 계정 복구가 허용되지 않은 상태라면 그 주소로 루트 로그인도, 비밀번호 복구도 할 수 없다. 그 이메일은 실질적으로 **계정 소유자·연락처 식별자**다.

`CERT` 안에서 실제로 작업하려면 별도의 접근 배정이 필요하다 — Identity Center 권한 세트 배정, `OrganizationAccountAccessRole` 맡기, 또는 계정 안 IAM 사용자.

Organization에 대해서는 아무 권한도 생기지 않는다. OU 생성·계정 이동·SCP 부착은 **관리 계정의 주체**만 할 수 있는 일이고, 멤버 계정의 루트 이메일과는 무관하다.

부수적으로 생기는 것은 **손해**다. 그 계정을 폐쇄하면 그 이메일은 다른 AWS 계정의 기본 이메일로 재사용할 수 없다.
</details>

### 질문 2

인증이 끝났다. "`CERT` 계정과 `Cert` OU를 정리하라"는 요청을 받았을 때, 절대 하면 안 되는 작업 하나를 고르고 왜 그런지 설명하라. 그리고 OU를 삭제하려면 그 전에 무엇이 성립해야 하는가?

<details>
<summary>모범 답안</summary>

절대 하면 안 되는 작업은 **Organization 삭제**다. 조직을 삭제하려면 `PROD`를 포함한 **모든 멤버 계정을 먼저 제거**해야 하고, 관리 계정은 독립 계정으로 되돌아가며, 조직을 필요로 하는 IAM Identity Center가 동작을 멈춰 SSO 로그인이 깨진다. 조직 정책과 조직 단위 비용 데이터도 삭제되고 복구할 수 없다. `CERT` 하나를 정리하는 것과는 전혀 다른 층의 작업이다.

**OU는 비어 있어야 삭제할 수 있다.** 계정과 하위 OU를 모두 밖으로 옮긴 뒤에야 삭제된다. 그런데 계정을 **폐쇄만** 하면 최대 90일 동안 `CLOSED` 상태로 조직 트리에 남는다. 그래서 실무적으로는 `CERT`를 Root(또는 다른 OU)로 옮겨 `Cert` OU를 비운 뒤 OU를 삭제하거나, 조직에서 계정을 제거한 뒤 OU를 삭제한다.

덧붙여, 조직에서 제거해도 관리 계정 접근용으로 만들어진 IAM 역할은 자동 삭제되지 않으므로 직접 지워야 한다.
</details>

### 질문 3

`Cert` OU에 "이 OU의 계정은 서울 리전에서만 리소스를 만들 수 있다"는 SCP를 붙였다. 그런데 `CERT` 계정에 배정된 내 권한 세트에는 `AdministratorAccess` 가 들어 있다. 나는 버지니아 리전에 RDS를 만들 수 있는가? 그리고 SCP만 붙여두면 내가 `CERT`에 들어갈 수 있는가?

<details>
<summary>모범 답안</summary>

**만들 수 없다.** SCP는 권한을 주는 정책이 아니라 **상한선**이다. 최종 권한은 "SCP가 허용한 것 ∩ IAM이 허용한 것"이므로, IAM 쪽에 `AdministratorAccess`(`*/*`)가 있어도 SCP가 막은 것은 할 수 없다. 문서 표현대로, 상위에서 차단된 권한은 계정 관리자가 `AdministratorAccess` 를 붙여줘도 쓸 수 없다. 멤버 계정의 **루트 사용자에게도** SCP는 적용된다(관리 계정의 주체는 예외).

**SCP만으로는 들어갈 수 없다.** SCP는 아무 권한도 부여하지 않으므로, 사람이 계정에 들어가려면 IAM 쪽 권한이 별도로 있어야 한다 — Identity Center 권한 세트 배정, 역할 맡기, 또는 IAM 사용자. "IAM 정책이 없는 사용자는 SCP가 전부 허용해도 접근이 없다"가 문서의 명시적 서술이다.
</details>

---

## 부록 A — 선택 학습

### A.1 Control Tower — 언제 고려하나

**AWS Control Tower** 는 Organizations·Service Catalog·IAM Identity Center를 묶어 **다계정 환경(랜딩 존)을 처방된 기본값대로 세팅·통제**해 주는 서비스다. 공식 설명 중 도입 시점에 대한 문장이 가장 실용적이다.

> "If you are hosting more than a handful of accounts, it's beneficial to have an orchestration layer that facilitates account deployment and account governance."
> — [What Is AWS Control Tower?](https://docs.aws.amazon.com/controltower/latest/userguide/what-is-control-tower.html) (확인 2026-09-10)

핵심 기능은 **랜딩 존**(권장 OU·계정 구조), **컨트롤/가드레일**(예방·탐지·사전 점검 규칙), **Account Factory**(표준 템플릿으로 계정 발급), **드리프트 감지**(설정이 기준에서 벗어나는 것 탐지)다.

고려할 시점의 신호: 계정이 손으로 관리하기 어려울 만큼 늘어난다 / 계정 생성 절차를 매번 사람이 반복한다 / 감사 대응으로 "규칙이 실제로 적용되고 있음"을 증명해야 한다.

이번 사례에는 **필요 없다.** 계정 하나를 임시로 만들었다 없애는 일이라 오케스트레이션 계층을 새로 도입할 이유가 없다. 다만 **회사가 이미 Control Tower를 쓰고 있다면 주의할 점이 생긴다** — 계정은 Organizations가 아니라 Control Tower의 Account Factory로 만드는 게 권장이고(그렇지 않으면 Control Tower에 등록되지 않는다), **폐쇄하려면 먼저 그 계정을 Control Tower 관리에서 해제(unmanage)해야 한다.**

### A.2 루트 접근 중앙 관리 — 멤버 계정의 루트를 없애는 기능

IAM에는 조직의 멤버 계정 루트 자격증명을 **중앙에서 관리·삭제**하는 기능이 있다. 켜면 멤버 계정의 루트 비밀번호·액세스 키·서명 인증서를 제거하고 MFA를 해제할 수 있으며, 그 계정들은 **루트로 로그인할 수도, 비밀번호 복구를 할 수도 없다.** 앞서 본 대로 **조직에서 새로 만드는 계정은 처음부터 루트 자격증명이 없다.**

꼭 루트가 필요한 작업(예: 잘못 잠긴 S3 버킷 정책 풀기)이 생기면, 관리 계정 쪽에서 **작업 범위가 한정된 단기 루트 세션**을 발급하거나 해당 계정의 비밀번호 복구를 일시적으로 허용하는 방식으로 처리한다.

이번 사례와의 연결: `CERT` 루트 이메일을 무엇으로 하든 **일상 작업에는 루트를 쓰지 않는다**는 게 기본 전제다. 루트 이메일은 소유·복구·폐쇄 같은 계정 수명 관리용 식별자로 보는 게 맞다. 근거: [Accessing member accounts](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_access.html), [Centralize root access for member accounts](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_root-enable-root-access.html) (확인 2026-09-10).

### A.3 헷갈리는 이름 세 쌍

- **조직의 Root** ≠ **루트 사용자** — 앞은 OU 트리의 뿌리 컨테이너, 뒤는 계정 하나의 최상위 로그인 주체다.
- **관리 계정(management account)** ≠ **관리자 권한(AdministratorAccess)** — 앞은 조직 트리 상의 위치, 뒤는 계정 안 권한의 크기다([3.2](#32-두-가지-구분은-서로-다른-질문에-답한다)).
- **계정 초대** ≠ **사용자 초대** — 앞은 AWS 계정을 조직 멤버로 편입, 뒤는 사람에게 접근을 배정([5.3](#53-계정-초대와-사용자-초대는-완전히-다른-일이다)).

### A.4 근거 링크 목록 (전부 확인일 2026-09-10)

- [AWS Organizations — Managing OUs](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_ous.html) / [Deleting an OU](https://docs.aws.amazon.com/organizations/latest/userguide/delete-ou.html) / [Moving accounts to an OU](https://docs.aws.amazon.com/organizations/latest/userguide/move_account_to_ou.html)
- [Creating a member account](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_create.html) / [Accessing member accounts](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_access.html) / [Managing account invitations](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_invites.html)
- [Removing a member account](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_remove.html) / [Closing a member account](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_close.html) / [Deleting an organization](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_org_delete.html)
- [Service control policies (SCPs)](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html)
- [Close an AWS account](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-closing.html) — 90일 폐쇄 후 기간, 이메일·별칭 재사용 제한, 폐쇄 상한
- [Update the root user email address](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-update-root-user-email.html) / [Updating the root user email address for a member account](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_update_primary_email.html)
- [IAM Identity Center — permission sets](https://docs.aws.amazon.com/singlesignon/latest/userguide/permissionsetsconcept.html)
- [Organizing Your AWS Environment — Recommended OUs and accounts](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/recommended-ous-and-accounts.html)
- [What Is AWS Control Tower?](https://docs.aws.amazon.com/controltower/latest/userguide/what-is-control-tower.html)
- [Centralize root access for member accounts](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_root-enable-root-access.html)
