# `docker exec`로 만든 파일은 어디로 가나로 배우는 Docker 이미지 레이어와 Copy-on-Write

`docker exec`로 컨테이너에 들어가 파일을 하나 만들었다. 이 파일은 이미지에 들어갔나, 컨테이너를 껐다 켜면 남나, 지우면 어디까지 사라지나. 이 질문 하나를 따라가면 Docker 이미지가 왜 층(layer)으로 쌓이는지, 컨테이너가 이미지를 복사하지 않고도 어떻게 자기만의 파일을 갖는지, 그리고 그 밑에서 Linux의 OverlayFS와 Copy-on-Write가 무엇을 하는지가 한 줄로 이어진다.

작성일: 2026-09-20
기준 환경: Docker Engine 29.0 (Docker Desktop, linux/arm64, containerd 이미지 저장소 + overlayfs 스냅샷터), 기본 이미지 `ubuntu:24.04`. 문서 안의 명령 출력은 모두 이 환경에서 직접 실행해 얻었다. Linux 서버의 Docker Engine(overlay2 드라이버)과 다른 부분은 [부록 A.1](#a1-docker-desktop과-linux-docker-engine의-차이)에 따로 적었다.

---

## 0. 이 문서 전체를 한 문단으로

**Docker 이미지**는 실행 파일이 아니라 **읽기 전용 파일시스템 스냅샷(레이어)들과 실행 메타데이터(CMD, ENV 등)의 묶음**이다. `docker run`은 이 이미지를 복사하지 않고, 읽기 전용 레이어들 **위에 그 컨테이너 전용 쓰기 가능 레이어(writable layer) 하나를 얹고** 프로세스를 띄운다. 컨테이너 안에서 파일을 만들거나 고치면 전부 이 쓰기 레이어에 기록되고 이미지는 그대로다. `docker stop`은 프로세스만 끝내므로 쓰기 레이어가 남고 `docker start`하면 그대로 다시 보인다. `docker rm`은 컨테이너 객체를 지우므로 쓰기 레이어도 함께 사라진다. "안 바꾸는 동안은 공유하고, 바꿀 때만 그 부분을 복사한다"는 이 전략이 **Copy-on-Write**이고, Linux가 `fork()`에서 메모리 페이지에 쓰는 것과 같은 아이디어를 파일시스템 계층에서 쓴 것이다. 그 구현이 **OverlayFS**로, 이미지 레이어들(lowerdir)과 쓰기 레이어(upperdir)를 겹쳐 컨테이너에는 하나의 `/`(merged)로 보여 준다.

이 문단에 모르는 단어가 하나라도 있으면 이 문서가 도움이 된다.

---

## 질문 → 어디를 볼 것인가

| 질문 | 섹션 |
|---|---|
| 이미지와 컨테이너는 정확히 뭐가 다른가 | [1장](#1-image와-container--무엇이-무엇을-만드나) |
| 이미지는 왜 여러 층으로 되어 있나. Dockerfile 한 줄이 층 하나인가 | [2장](#2-image는-왜-층으로-쌓이나) · [2.3절](#23-dockerfile-instruction과-레이어의-관계--전부-층을-만들지는-않는다) |
| `app.jar`만 바꿨는데 왜 앞 단계는 다시 안 도나 | [2.4절](#24-빌드-캐시--앞-레이어를-재사용할-수-있는-이유) |
| 컨테이너를 만들면 이미지를 통째로 복사하나 | [3장](#3-container-writable-layer--복사하지-않고-한-층-얹는다) |
| `docker exec`로 만든 파일은 어디에 저장되나. `exec`는 새 컨테이너인가 | [4장](#4-docker-exec로-파일을-만들면--새-프로세스-같은-컨테이너) |
| 이미지에 있던 파일을 컨테이너에서 고치면 이미지가 바뀌나 | [5장](#5-이미지에-있던-파일을-고치면--위층이-아래층을-가린다) |
| `stop`했다 `start`하면 파일이 남나. `rm`은 뭐가 다른가 | [6장](#6-stop--start--rm--쓰기-레이어의-수명은-컨테이너-객체의-수명이다) |
| Copy-on-Write가 뭔가. 예전에 `fork()`에서 본 그것인가 | [7장](#7-copy-on-write--미리-복사하지-않고-쓸-때-복사한다) · [8장](#8-linux-메모리-cow와-docker-파일시스템-cow--같은-전략-다른-계층) |
| 메모리 CoW와 Docker CoW는 뭐가 다른가 | [9장](#9-둘의-중요한-차이--페이지-단위와-파일-단위-copy-up) |
| lowerdir · upperdir · merged · workdir이 뭔가 | [10장](#10-overlayfs--여러-디렉터리를-겹쳐-하나의-로-보여-준다) |
| 같은 이미지로 만든 컨테이너 둘은 파일을 공유하나 | [11장](#11-같은-이미지로-만든-컨테이너-둘--공유하는-것과-따로-갖는-것) |
| DB 데이터는 왜 쓰기 레이어에 두면 안 되나 | [12장](#12-volume으로-이어지는-이유) |
| 전체가 어떻게 하나로 이어지나 | [13장](#13-전체-개념-연결) |
| 직접 확인해 보려면 | [14장](#14-실습--10단계로-직접-확인하기) |
| 헷갈리는 오해 모음, 다음에 뭘 공부하나 | [15장](#15-마지막-정리) |
| Docker Desktop에서 경로가 문서와 다르다 | [부록 A.1](#a1-docker-desktop과-linux-docker-engine의-차이) |

---

## 1. Image와 Container — 무엇이 무엇을 만드나

### 1.1 흐름

```text
Dockerfile        ← "이런 파일시스템을 만들고, 이렇게 실행하라"는 레시피 (텍스트)
    ↓  docker build
Image             ← 레시피대로 만든 결과물. 파일시스템 스냅샷 + 실행 메타데이터. 실행 중이 아님
    ↓  docker run
Container         ← 이미지를 바탕으로 만든 실행 인스턴스. 프로세스가 돌고 있음
```

**이미지는 아직 프로세스가 아니다.** 디스크에 놓인 파일들과 "실행할 때는 이 명령을 써라"는 설정의 묶음이다. 클래스와 인스턴스의 관계로 생각하면 가깝다. 이미지 하나로 컨테이너를 몇 개든 만들 수 있고, 각 컨테이너는 따로 살고 따로 죽는다.

### 1.2 이미지 안에 든 것

```text
Filesystem  (읽기 전용 스냅샷)
- /bin, /usr, /etc, /lib …     ← 베이스 OS의 사용자 공간 파일들 (커널은 없다)
- application files             ← COPY로 넣은 우리 앱
- libraries

Metadata  (실행 관련 설정)
- CMD          기본 실행 명령
- ENTRYPOINT   실행 진입점
- ENV          환경변수
- WORKDIR      작업 디렉터리
- EXPOSE, USER, LABEL …
```

메타데이터는 `docker inspect`로 볼 수 있다. 이 문서의 실습 이미지에서:

```bash
docker inspect --format '{{json .Config.Cmd}} {{json .Config.Env}}' studylab:1
# ["sleep","infinity"] ["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"]
```

이미지에 **커널은 들어 있지 않다**는 점을 짚고 가자. 컨테이너는 호스트 Linux 커널을 그대로 쓰고, 이미지는 그 위에서 도는 사용자 공간 파일들만 담는다. 그래서 `ubuntu:24.04` 이미지가 110MB밖에 안 된다.

### 1.3 이 문서가 쓰는 두 정의, 그리고 단순화된 부분

> **Docker Image** = 컨테이너를 만들기 위한 **읽기 전용 파일시스템 스냅샷(레이어)들** + **실행 관련 메타데이터**의 집합

> **Container** = 이미지의 읽기 전용 레이어들 + 그 컨테이너 **전용 쓰기 가능 레이어** + **실행 중인 프로세스**

두 정의는 이 문서를 읽는 데 충분하지만, 엄밀히는 다음이 생략되어 있다.

- **컨테이너는 프로세스가 없어도 존재한다.** `docker stop`한 컨테이너는 프로세스가 0개지만 여전히 `docker ps -a`에 보이고 쓰기 레이어를 갖고 있다. 정확히는 "컨테이너 = Docker가 관리하는 **객체**(설정 + 쓰기 레이어) + (실행 중일 때) 프로세스"다. 6장의 핵심이 바로 이 구분이다.
- **"실행 중인 프로세스"에는 격리가 빠져 있다.** 컨테이너의 프로세스는 자기만의 namespace(PID, 네트워크, 마운트 등 — 커널이 자원 이름 공간을 분리하는 기능)와 cgroup(CPU·메모리 한도) 안에서 돈다. 파일시스템 격리는 그중 마운트 namespace 하나일 뿐이다. 이 문서는 파일시스템 축만 다루고, 나머지는 [15.3절](#153-다음-학습-순서)에 순서를 적어 두었다.
- **레이어 = 파일시스템 스냅샷**은 "그 시점의 전체 상태"가 아니라 **이전 레이어와의 차이(diff)** 다. 2장에서 바로 본다.
- **컨테이너에는 볼륨·바인드 마운트처럼 레이어 밖의 저장소가 붙을 수 있다.** 12장.

---

## 2. Image는 왜 층으로 쌓이나

### 2.1 문제 — 이미지가 통짜 스냅샷이라면

Spring 앱 이미지가 있다고 하자. 베이스 OS 110MB + 패키지 200MB + JDK 300MB + 우리 `app.jar` 30MB. 이미지를 하나의 덩어리로 저장하면:

- `app.jar` 한 줄 고칠 때마다 640MB를 **다시 만들고, 다시 올리고, 다시 받는다.**
- 같은 베이스 OS를 쓰는 서비스가 열 개면 디스크에 같은 110MB가 **열 벌** 있다.
- 서버에서 컨테이너를 열 개 띄우면 640MB를 열 번 복사해야 한다.

Docker는 이미지를 **변경 단위로 쪼개서 겹쳐 쌓는** 방식으로 이 셋을 한 번에 푼다.

### 2.2 레이어 — 이전 층을 고치지 않고 차이를 다음 층으로 쌓는다

이 Dockerfile을 예로 든다.

```dockerfile
FROM ubuntu:24.04

RUN apt-get update && apt-get install -y curl

COPY app.jar /app/app.jar

CMD ["java", "-jar", "/app/app.jar"]
```

만들어지는 이미지는 개념적으로 이렇다. 아래에서 위로 쌓인다.

```text
┌──────────────────────────────┐
│ app.jar                      │  ← 레이어 3: COPY 결과. "이 파일이 추가됨"만 담김
│ COPY 결과                    │
├──────────────────────────────┤
│ curl 및 package 변경         │  ← 레이어 2: RUN 결과. 설치로 바뀐 파일들의 차이만 담김
│ RUN 결과                     │
├──────────────────────────────┤
│ Ubuntu filesystem            │  ← 레이어 1: 베이스 이미지 (ubuntu:24.04 자체가 가진 레이어)
│ Base Image                   │
└──────────────────────────────┘
```

여기서 세 가지가 핵심이다.

**각 레이어는 읽기 전용이다.** Docker 공식 문서의 표현대로 "각 레이어는 파일시스템 변경의 집합(추가·삭제·수정)"이고, 만들어진 뒤에는 바뀌지 않는다. 레이어에는 내용의 해시(digest)가 붙어 있어서 내용이 같으면 같은 레이어다.

**이전 레이어를 수정하는 게 아니라, 변경 사항을 다음 레이어로 쌓는다.** `RUN apt-get install curl`은 레이어 1의 `/usr/bin`을 고치지 않는다. "레이어 1 위에서 이 명령을 돌렸더니 이 파일들이 추가·변경됐다"는 **차이분**을 레이어 2로 만든다. 이미지를 읽을 때는 위에서 아래로 겹쳐 본다. 같은 경로의 파일이 여러 층에 있으면 **위층이 이긴다.**

**레이어는 이미지 사이에서 공유된다.** `ubuntu:24.04`를 베이스로 쓰는 이미지가 열 개여도 레이어 1은 디스크에 한 번만 있다. 받을 때도 이미 있는 레이어는 건너뛴다(`docker pull`이 "Already exists"를 찍는 게 이것이다).

### 2.3 Dockerfile instruction과 레이어의 관계 — 전부 층을 만들지는 않는다

"Dockerfile 한 줄 = 레이어 하나"는 대략 맞지만 정확하지는 않다. **파일시스템을 바꾸는 명령만 파일시스템 레이어를 만든다.** Docker 문서는 "파일시스템을 수정하는 명령이 새 레이어를 만들고, 메타데이터만 바꾸는 변경은 레이어를 만들지 않는다"고 적는다.

| instruction | 하는 일 | 파일시스템 레이어 |
|---|---|---|
| `FROM` | 베이스 이미지의 레이어들을 가져옴 | 베이스의 레이어들 |
| `RUN` | 명령을 실행해 **현재 이미지 위에 새 레이어를 만든다** (Dockerfile 레퍼런스 문장 그대로) | 만든다 |
| `COPY` / `ADD` | 파일을 이미지 파일시스템에 추가 | 만든다 |
| `CMD` / `ENTRYPOINT` | 실행 명령 지정. "빌드 시점에는 아무것도 실행하지 않는다" | 만들지 않음 (메타데이터) |
| `ENV` / `WORKDIR` / `EXPOSE` / `LABEL` / `USER` | 설정·메타데이터 | 만들지 않음 |

이건 `docker history`로 직접 보인다. 이 문서의 실습 이미지(RUN 한 줄, COPY 한 줄, CMD 한 줄)를 보면:

```text
$ docker history studylab:1
IMAGE          CREATED BY                                      SIZE      COMMENT
6069eada30f3   CMD ["sleep" "infinity"]                        0B        buildkit.dockerfile.v0
<missing>      COPY app.txt /app/app.txt # buildkit            12.3kB    buildkit.dockerfile.v0
<missing>      RUN /bin/sh -c mkdir -p /etc/myapp && echo '…   16.4kB    buildkit.dockerfile.v0
<missing>      /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B
<missing>      /bin/sh -c #(nop) ADD file:ff1ce8d2ee022926e…   110MB     ← ubuntu:24.04의 실제 파일시스템
<missing>      /bin/sh -c #(nop)  LABEL org.opencontainers.…   0B
<missing>      /bin/sh -c #(nop)  ARG LAUNCHPAD_BUILD_ARCH     0B
<missing>      /bin/sh -c #(nop)  ARG RELEASE                  0B

$ docker inspect --format '{{len .RootFS.Layers}} layers' studylab:1
3 layers
```

history에는 8줄이 나오지만 **실제 파일시스템 레이어는 3개**다(ubuntu의 ADD 1개 + 우리 RUN 1개 + COPY 1개). `CMD`, `LABEL`, `ARG`는 0B이고 `RootFS.Layers`에 안 잡힌다. history의 한 줄은 "이미지 설정 이력의 한 단계"이지 "레이어 하나"가 아니다.

> `RUN apt-get update && apt-get install -y curl`을 **한 줄**로 쓰는 이유도 여기서 나온다. 둘을 두 `RUN`으로 나누면 `apt-get update`가 별도 레이어로 캐시되어, 나중에 install 줄만 바꿔도 옛 패키지 목록으로 설치를 시도한다. Docker 문서가 "항상 같은 RUN 문 안에서 update와 install을 결합하라"고 하는 이유다.

### 2.4 빌드 캐시 — 앞 레이어를 재사용할 수 있는 이유

레이어가 "이전 레이어 + 이 명령"의 결과이고 읽기 전용이라면, **입력이 같으면 결과도 같다**고 볼 수 있다. Docker는 이걸 이용해 빌드할 때 각 단계를 캐시와 비교한다. 문서의 규칙:

- 베이스 이미지부터 시작해 **명령을 순서대로 캐시와 비교**한다.
- `COPY` / `ADD`는 **복사할 파일의 내용 체크섬**(수정 시각은 제외)을 비교한다.
- `RUN`은 **명령 문자열만** 비교한다. 명령이 만들어 낸 파일은 보지 않는다.
- **한 번 캐시가 깨지면 그 뒤의 모든 명령은 캐시를 쓰지 않고 다시 실행된다.**

그래서 `app.jar`만 바뀌면:

```text
FROM ubuntu:24.04          → 캐시 HIT   (베이스 그대로)
RUN apt-get install curl   → 캐시 HIT   (명령 문자열 같음)
COPY app.jar /app/app.jar  → 캐시 MISS  (파일 체크섬 달라짐)  ← 여기부터 다시
CMD [...]                  → 다시 (뒤는 전부)
```

앞 두 레이어는 그대로 재사용되고 COPY 레이어만 새로 만든다. 실측:

```text
# 두 번째 빌드 (아무것도 안 바꿈)
#6 [2/3] RUN mkdir -p /etc/myapp && echo 'env: production' > /etc/myapp/config.yml
#6 CACHED
#7 [3/3] COPY app.txt /app/app.txt
#7 CACHED

# 세 번째 빌드 (app.txt 내용만 변경)
#6 [2/3] RUN mkdir -p /etc/myapp && echo 'env: production' > /etc/myapp/config.yml
#6 CACHED                                    ← RUN은 재사용
#7 [3/3] COPY app.txt /app/app.txt           ← COPY만 다시 실행
```

여기서 실전 규칙 하나가 나온다. **자주 바뀌는 것을 Dockerfile의 뒤쪽에 둔다.** 의존성 설치(잘 안 바뀜)를 앞에, 앱 코드(매번 바뀜)를 뒤에. 순서를 거꾸로 두면 코드 한 줄 고칠 때마다 의존성 설치가 다시 돈다.

### 2.5 기억할 것

- 오해: "이미지는 하나의 큰 파일이다" → 아니다. 읽기 전용 차이분 레이어들의 스택이고, 레이어는 이미지끼리 공유된다.
- 오해: "Dockerfile 한 줄마다 레이어 하나" → 파일시스템을 바꾸는 명령만 레이어를 만든다. `CMD`·`ENV`는 메타데이터다.
- 한 문장: **이미지 레이어는 "이전 층 위에서 이 명령을 돌린 차이"를 읽기 전용으로 굳힌 것이고, 그래서 공유되고 캐시된다.**

---

## 3. Container Writable Layer — 복사하지 않고 한 층 얹는다

### 3.1 문제

이미지 레이어가 전부 읽기 전용이면 컨테이너는 어떻게 파일을 쓰나. 로그도 남기고 임시 파일도 만들어야 하는데. 이미지를 통째로 복사해서 쓰기 가능하게 만들면 2.1절의 낭비가 컨테이너마다 반복된다.

### 3.2 해결 — 읽기 전용 스택 위에 쓰기 가능한 층 하나

`docker run`(정확히는 `docker create`)은 이미지를 복사하지 않는다. 이미지 레이어들은 그대로 두고 **그 위에 그 컨테이너만의 얇은 쓰기 가능 레이어 하나를 얹는다.** Docker 문서는 이걸 "container layer"라고도 부른다.

```text
┌──────────────────────────────┐
│ Container Writable Layer     │  ← Read / Write. 이 컨테이너 전용. 처음엔 비어 있음
├──────────────────────────────┤
│ Image Layer 3  (COPY)        │  ← Read Only
├──────────────────────────────┤
│ Image Layer 2  (RUN)         │  ← Read Only
├──────────────────────────────┤
│ Image Layer 1  (Base)        │  ← Read Only
└──────────────────────────────┘
```

컨테이너 안의 프로세스가 `/`를 보면 이 네 층이 겹쳐진 하나의 파일시스템으로 보인다. 읽기는 위에서 아래로 내려가며 처음 만나는 파일을 읽고, **쓰기는 무조건 맨 위 쓰기 레이어에** 간다.

> **Container의 파일 변경은 원본 Image를 수정하는 것이 아니라, 해당 Container의 Writable Layer에 반영된다.**

이게 이 문서에서 가장 중요한 문장이다. 4장·5장·6장은 전부 이 문장의 귀결이다.

### 3.3 왜 이게 실전에서 중요한가

- 컨테이너 100개를 띄워도 이미지는 디스크에 한 벌이고, 늘어나는 건 컨테이너별 쓰기 레이어뿐이다. 그래서 컨테이너 생성이 밀리초 단위다.
- 반대로 쓰기 레이어에 로그나 업로드 파일을 쌓으면 그 컨테이너만 커지고, 컨테이너를 지우면 같이 사라진다. 12장으로 이어진다.

---

## 4. `docker exec`로 파일을 만들면 — 새 프로세스, 같은 컨테이너

### 4.1 `docker exec`는 무엇을 하나

```bash
docker exec -it my-container bash
```

이 명령은 **새 컨테이너를 만들지 않는다.** Docker 문서: "`docker exec`는 실행 중인 컨테이너 안에서 새 명령을 실행한다." 즉,

> **이미 실행 중인 컨테이너의 namespace 안에서 새로운 프로세스를 하나 더 띄우는 것**

이다. 컨테이너 안에는 이미 PID 1(이미지의 CMD로 뜬 프로세스)이 돌고 있고, `exec`는 그 옆에 프로세스를 추가한다.

```text
Container (같은 PID namespace, 같은 마운트 namespace)

PID 1
└── java -jar app.jar        ← docker run 이 띄운 주 프로세스

docker exec bash
        ↓
PID 15
└── bash                     ← 같은 컨테이너 안에 추가된 프로세스
```

실측. `sleep infinity`로 띄운 컨테이너에 `exec`로 들어가 확인하면:

```text
$ docker exec cfg sh -c 'echo "PID1: $(tr "\0" " " </proc/1/cmdline)"; echo "this shell PID: $$"'
PID1: sleep infinity
this shell PID: 15
```

PID 1은 그대로 `sleep`이고, 내 셸은 같은 PID 공간의 15번이다. 같은 마운트 namespace를 쓰므로 **PID 1이 보는 `/`와 내 셸이 보는 `/`가 같은 파일시스템**이다. 그래서 `exec`에서 만든 파일이 앱에서도 보인다.

문서의 제약 두 가지도 여기서 이해된다. `exec`한 명령은 **PID 1이 살아 있는 동안만** 돌고, 컨테이너가 재시작되어도 다시 시작되지 않는다. 컨테이너의 수명은 PID 1의 수명이기 때문이다.

### 4.2 만든 파일은 어디로 가나

```bash
docker exec -it my-container bash
touch /hello.txt
```

3장의 규칙대로 쓰기는 맨 위 층으로 간다.

```text
Container Writable Layer
└── /hello.txt              ← 여기에 생김

Image Layers
└── 변경 없음                ← 이미지는 건드리지 않음
```

이걸 확인하는 명령이 `docker diff`다. 문서: "컨테이너가 만들어진 이후 파일시스템에서 바뀐 파일과 디렉터리를 나열한다." 기호는 `A` 추가, `D` 삭제, `C` 변경.

```text
$ docker exec test touch /hello.txt
$ docker diff test
A /hello.txt
```

`docker diff`는 **쓰기 레이어의 내용을 보여 주는 것**과 같다. 이미지 레이어와의 차이가 곧 쓰기 레이어이기 때문이다. 이미지 쪽은 `docker history`를 다시 찍어 봐도 아무 변화가 없다.

### 4.3 기억할 것

- 오해: "`docker exec`로 고치면 이미지가 바뀐다" → 아니다. 그 컨테이너의 쓰기 레이어만 바뀐다. 같은 이미지로 새 컨테이너를 만들면 없다(11장·14장에서 확인).
- 오해: "`docker exec`는 컨테이너를 하나 더 띄운다" → 아니다. 기존 컨테이너의 namespace에 프로세스를 하나 추가한다.
- 한 문장: **`exec`는 같은 방에 사람을 한 명 더 들여보내는 것이고, 그 사람이 남긴 흔적은 그 방(쓰기 레이어)에만 남는다.**

---

## 5. 이미지에 있던 파일을 고치면 — 위층이 아래층을 가린다

### 5.1 상황

4장은 **없던 파일을 만든** 경우다. 이미지에 **이미 있는 파일**을 고치면 어떻게 되나. 이미지 레이어는 읽기 전용인데.

이미지에 이 파일이 있다고 하자.

```text
/etc/myapp/config.yml
env: production
```

컨테이너에서 고친다.

```bash
vi /etc/myapp/config.yml      # env: development 로 수정
```

### 5.2 동작 — 이미지 레이어는 그대로, 쓰기 레이어에 새 버전

이미지 레이어의 파일은 바뀌지 않는다. 대신 **쓰기 레이어에 같은 경로로 수정된 버전이 생긴다.**

```text
Container Writable Layer

/etc/myapp/config.yml
env: development              ← 컨테이너가 보는 것 (위층이 우선)

──────────────────────────

Image Layer

/etc/myapp/config.yml
env: production               ← 그대로 남아 있음. 가려져서 안 보일 뿐
```

컨테이너 안에서 읽으면 위에서부터 찾으므로 `development`가 보인다. 하지만 이미지에는 여전히 `production`이 있다. 실측:

```text
$ docker exec cfg sh -c "sed -i 's/production/development/' /etc/myapp/config.yml && cat /etc/myapp/config.yml"
env: development

$ docker diff cfg
C /etc
C /etc/myapp
C /etc/myapp/config.yml       ← C = 변경. 쓰기 레이어에 이 파일의 새 버전이 있다

$ docker run --rm studylab:1 cat /etc/myapp/config.yml     # 같은 이미지로 새 컨테이너
env: production               ← 이미지는 안 바뀌었다
```

이때 뒤에서 일어나는 일이 "쓰기 레이어에 파일을 **먼저 복사한 뒤** 그 복사본을 고친다"는 **copy-up**이고, 이게 Copy-on-Write의 파일시스템 버전이다. 7~9장에서 이 동작의 정체를 본다.

### 5.3 기억할 것

- 한 문장: **이미지의 파일을 고치는 게 아니라, 쓰기 레이어에 고친 버전을 두어 원본을 가린다.**

---

## 6. `stop` · `start` · `rm` — 쓰기 레이어의 수명은 컨테이너 객체의 수명이다

### 6.1 헷갈리는 이유

"컨테이너를 껐다"는 말이 `stop`인지 `rm`인지에 따라 데이터 운명이 완전히 다르다. 둘을 구분하려면 1.3절의 "컨테이너 = 객체 + (실행 중일 때) 프로세스" 구분이 필요하다.

```text
Container 객체               ← Docker가 관리하는 기록: 설정, 이름, 네트워크 설정, 쓰기 레이어
    │
    └── (실행 중일 때) 프로세스 트리 (PID 1 + exec로 추가된 것들)
```

### 6.2 `stop` → `start`: 프로세스만 끝나고 객체는 남는다

```bash
docker run -d --name test ubuntu sleep infinity
docker exec test touch /hello.txt
docker stop test
docker start test
```

`docker stop`은 PID 1에 `SIGTERM`을 보내고, 유예 시간(기본 10초) 안에 안 끝나면 `SIGKILL`로 죽인다. **프로세스가 끝날 뿐 컨테이너 객체는 남는다.** `docker ps -a`에 `Exited`로 보이고 쓰기 레이어도 그대로다.

```text
docker stop

Process 실행      X
Container 객체    O   ← docker ps -a 에 Exited 로 보임
Writable Layer    O   ← /hello.txt 그대로
```

`docker start`는 **같은 객체를 다시 실행**한다. 새 컨테이너를 만드는 게 아니라 기존 설정과 기존 쓰기 레이어로 PID 1을 다시 띄운다.

```text
docker start

기존 Container 객체를 다시 실행
기존 Writable Layer 그대로 사용   ← /hello.txt 가 보인다
```

실측:

```text
$ docker stop test
$ docker ps -a --filter name=test --format '{{.Names}} {{.Status}}'
test Exited (137) Less than a second ago
$ docker start test
$ docker exec test ls -l /hello.txt
-rw-r--r-- 1 root root 0 Sep 20 05:59 /hello.txt     ← 남아 있다
```

> 곁가지: 종료 코드 137은 128 + 9, 즉 `SIGKILL`로 죽었다는 뜻이다. `sleep`이 PID 1로 떠 있으면 `SIGTERM`의 기본 동작이 적용되지 않아(컨테이너의 PID 1은 핸들러를 직접 등록하지 않은 시그널을 무시한다) `SIGTERM`으로 안 끝나고 결국 `SIGKILL`을 받는다. 앱 컨테이너에서 `stop`이 매번 10초 걸리고 137로 끝난다면 PID 1이 `SIGTERM`을 처리하지 않는 것이다. 종료 코드 읽는 법은 [JVM 메모리 노트 2장](jvm-memory-and-container-limits.md#2-프로세스가-죽는다는-것--종료-코드부터)에 있다.

### 6.3 `rm`: 객체를 지우면 쓰기 레이어도 함께 사라진다

```bash
docker rm test
```

`docker rm`은 **컨테이너 객체를 삭제**한다. 문서: "컨테이너가 삭제되면 쓰기 레이어도 삭제된다." 객체가 없어지니 그 객체에 딸린 쓰기 레이어(= `/hello.txt`, 고친 `config.yml`, 안에 쌓인 로그 전부)가 같이 없어진다. 실행 중이면 `-f`로 `SIGKILL` 후 삭제한다. 이미지는 남는다. 이미지는 컨테이너의 것이 아니기 때문이다.

```text
docker rm

Process 실행      X
Container 객체    X   ← docker ps -a 에서도 사라짐
Writable Layer    X   ← /hello.txt 소멸
Image             O   ← 그대로. 다른 컨테이너를 또 만들 수 있다
```

> `docker rm -v`는 컨테이너에 붙은 **익명 볼륨**까지 지운다. 이름 있는 볼륨은 `-v`를 줘도 지우지 않는다. 볼륨은 컨테이너 객체 밖에 있기 때문이다(12장).

### 6.4 한 줄로 비교

| | `docker stop` | `docker rm` |
|---|---|---|
| 프로세스 | 끝남 | 끝남 (실행 중이면 `-f` 필요) |
| 컨테이너 객체 | **남음** | 삭제 |
| 쓰기 레이어 | **남음** | 삭제 |
| 다시 살리기 | `docker start` (같은 데이터) | 불가. `docker run`으로 **새** 컨테이너 |
| 이미지 | 그대로 | 그대로 |

> **Container Writable Layer의 수명은 프로세스의 수명이 아니라 Container 객체의 수명과 연결되어 있다.**

`stop`은 프로세스를 죽이고 `rm`은 객체를 죽인다. 쓰기 레이어는 객체를 따라간다.

### 6.5 기억할 것

- 오해: "`stop`하면 안의 파일이 날아간다" → 아니다. `rm`해야 날아간다.
- 오해: "`rm`하고 `run`하면 아까 그 컨테이너가 다시 뜬다" → 아니다. 이름이 같아도 **새 객체, 빈 쓰기 레이어**다.
- 한 문장: **`stop`은 잠깐 나간 것, `rm`은 방을 뺀 것.**

---

## 7. Copy-on-Write — 미리 복사하지 않고, 쓸 때 복사한다

### 7.1 먼저 `fork()`에서 본 것

5장의 "먼저 복사한 뒤 고친다"는 처음 보는 아이디어가 아니다. Linux 프로세스에서 이미 봤다.

`fork()`는 부모 프로세스와 똑같은 자식을 만든다. 순진하게 구현하면 부모의 메모리 전체를 자식에게 복사해야 한다. 그런데 자식은 대개 `fork()` 직후 `exec()`로 다른 프로그램을 덮어쓰거나, 부모 메모리의 극히 일부만 건드린다. 전체 복사는 거의 다 버려질 일이다.

그래서 Linux는 복사하지 않는다. `fork(2)` 매뉴얼의 문장: "Linux에서 `fork()`는 copy-on-write 페이지로 구현되어 있어, 치르는 비용은 부모의 페이지 테이블을 복제하고 자식의 task 구조체를 만드는 시간과 메모리뿐이다."

```text
fork() 직후

Parent
   │
   ├──────┐
   │      │
   ▼      ▼
 Page A  Page B          ← 물리 메모리 페이지. 부모·자식이 같은 것을 가리킴
   ▲      ▲
   │      │
   ├──────┘
   │
Child

처음에는 같은 Physical Memory Page를 공유 (둘 다 읽기 전용으로 표시)
```

### 7.2 쓰는 순간에만 복사

공유 페이지는 커널이 **읽기 전용**으로 표시해 둔다. 자식이 페이지 B에 **쓰려고 하면** 하드웨어가 예외를 일으키고, 커널이 그 순간 페이지 B만 복사해 자식에게 주고 쓰기를 진행시킨다.

```text
Before Write

Parent ──┐
         ├── Page B          (공유, read-only 표시)
Child  ──┘


After Child Write

Parent ───── Original Page B      ← 부모는 원본 그대로

Child  ───── Copied Page B        ← 자식만 복사본을 갖고
                 ↓
               Modify             ← 거기에 쓴다
```

페이지 A는 아무도 안 썼으니 끝까지 공유된다. 복사는 **실제로 쓴 페이지에 대해서만, 쓴 시점에** 일어난다.

> **Copy-on-Write는 "미리 전부 복사하지 않고, 실제 Write가 발생할 때 필요한 부분만 복사한다"는 최적화 전략이다.**

핵심은 두 가지 관찰이다. (1) 복사본 대부분은 읽기만 되므로 공유해도 안전하다. (2) 쓰기는 드물고 국소적이므로 그때 그 부분만 복사하면 된다. 이 두 조건이 맞는 곳이면 어디든 같은 전략이 통한다. Docker 이미지가 정확히 그런 곳이다.

---

## 8. Linux 메모리 CoW와 Docker 파일시스템 CoW — 같은 전략, 다른 계층

### 8.1 Docker가 같은 아이디어를 쓰는 이유

이미지 파일시스템도 7.2절의 두 조건을 만족한다. 컨테이너는 이미지 파일의 대부분을 **읽기만** 하고(라이브러리, 바이너리), 쓰는 파일은 소수다(로그, 임시 파일, 설정 한두 개). 그러니 컨테이너를 만들 때 이미지를 복사할 이유가 없다.

```text
               Image
          Read-Only Layers
                 │
          읽기 전용으로 공유
          ┌──────┴──────┐
          │             │
    Container A    Container B
    Writable A     Writable B      ← 각자 쓴 것만 여기에
```

컨테이너가 이미지 파일을 바꾸지 않는 동안은 이미지 레이어를 그대로 공유한다. 바꾸는 순간 그 컨테이너의 쓰기 레이어에 바뀐 버전이 생긴다(5장). 부모 프로세스의 원본 페이지가 남듯 이미지 원본도 남는다.

### 8.2 나란히 비교

| Linux Memory CoW | Docker Filesystem CoW |
|---|---|
| 물리 메모리 **페이지**를 공유 | 이미지 **레이어**를 공유 |
| `fork()` 시 메모리 전체를 복사하지 않음 | 컨테이너 생성 시 이미지 전체를 복사하지 않음 |
| Write 시 **그 페이지**를 복사해 자식에게 | Write 시 **그 파일**을 쓰기 레이어로 복사(copy-up)해 컨테이너에게 |
| 부모 원본 페이지 유지 | 이미지 원본 레이어 유지 |
| 공유 단위를 읽기 전용으로 표시해 쓰기를 감지 | 이미지 레이어를 읽기 전용으로 두고 쓰기를 위층으로 보냄 |
| 구현: 커널 메모리 관리(페이지 테이블, 페이지 폴트) | 구현: 스토리지 드라이버(OverlayFS 등) |

### 8.3 같은 것과 다른 것

두 줄이 같은 표에 있다고 **같은 기술**은 아니다. 커널의 메모리 관리자와 파일시스템 드라이버는 코드도 계층도 완전히 다르다.

> **Copy-on-Write라는 동일한 일반적 전략을, 서로 다른 계층(메모리 관리 vs 파일시스템)에서 각자 구현한 것**

이라고 이해하는 게 정확하다. 전략이 같으니 사고 방식을 그대로 옮겨 올 수 있고("안 쓰면 공유, 쓰면 그때 복사"), 구현이 다르니 세부 동작은 다르다. 그 다른 세부가 9장이다.

---

## 9. 둘의 중요한 차이 — 페이지 단위와 파일 단위 copy-up

### 9.1 메모리 CoW의 단위는 페이지

Linux 메모리 CoW는 **페이지** 단위로 일어난다. 페이지는 커널이 메모리를 관리하는 최소 단위로, x86-64와 이 문서의 arm64 Linux 환경에서 4KB다(`getconf PAGESIZE`로 확인. 아키텍처·설정에 따라 16KB·64KB인 경우도 있다).

```text
Process Memory
      ↓
4KB Page 단위로 쪼개져 있음
      ↓
자식이 1바이트를 써도 그 페이지(4KB)만 복사. 나머지 페이지는 계속 공유
```

100MB짜리 배열을 공유하다가 한 바이트만 바꾸면 4KB만 복사된다. **복사 비용이 실제로 건드린 크기에 비례**한다.

### 9.2 OverlayFS의 단위는 파일 — copy-up

Docker가 Linux에서 기본으로 쓰는 스토리지 드라이버는 OverlayFS 기반(overlay2, 또는 containerd의 overlayfs 스냅샷터)이다. 여기서 아래층(lowerdir)에 있는 기존 파일을 고치면 **copy-up**이 일어난다.

```text
lowerdir  (이미지 레이어, 읽기 전용)

config.yml
    ↓
 copy-up          ← 파일 전체를 upperdir로 복사
    ↓
upperdir  (쓰기 레이어)

config.yml
    ↓
 modify           ← 복사본을 고친다
```

Docker 문서의 표현을 그대로 옮기면: "overlay2 드라이버는 이미지(lowerdir)에서 컨테이너(upperdir)로 파일을 복사하는 copy_up 연산을 수행한다. **모든 OverlayFS copy_up은 파일 전체를 복사한다. 파일이 크고 그중 작은 일부만 고치더라도.**" 그리고 "각 copy_up은 주어진 파일이 처음 수정될 때 한 번만 일어나고, 이후 같은 파일에 대한 쓰기는 이미 복사된 사본에 대해 이루어진다."

### 9.3 그래서 생기는 차이

```text
1GB 파일의 1바이트를 고치면

메모리 CoW   → 4KB 복사
OverlayFS    → 1GB 복사 (처음 한 번), 이후 쓰기는 사본에
```

메모리 CoW는 세밀하고, OverlayFS copy-up은 굵다. 컨테이너 안에서 큰 파일(DB 데이터 파일, 대용량 로그)을 **이미지 레이어에 두고 계속 고치면** 첫 쓰기에 큰 복사가 일어나고, 그 뒤로도 그 파일은 쓰기 레이어에 통째로 산다. 이것도 12장에서 "쓰기가 많은 데이터는 볼륨으로"라고 하는 이유 중 하나다.

> **주의 — 드라이버에 따라 다르다.** 위 설명은 OverlayFS 기준이다. Docker의 스토리지 드라이버는 여러 종류가 있고(overlay2, btrfs, zfs, 과거의 aufs·devicemapper 등) CoW의 단위와 비용은 드라이버와 밑의 파일시스템 구현에 따라 다르다. 예를 들어 블록 단위로 CoW하는 드라이버도 있다. "Docker CoW = 파일 단위"가 아니라 "**OverlayFS의 copy-up은 파일 단위**"라고 기억해야 한다.

### 9.4 기억할 것

- 오해: "메모리 CoW와 Docker CoW는 같은 구현이다" → 전략만 같다. 단위(페이지 vs 파일)와 계층(커널 MM vs 파일시스템)이 다르다.
- 한 문장: **메모리는 4KB씩, OverlayFS는 파일째로 복사한다. 그래서 큰 파일을 컨테이너 레이어에서 고치는 건 비싸다.**

---

## 10. OverlayFS — 여러 디렉터리를 겹쳐 하나의 `/`로 보여 준다

### 10.1 무엇인가

지금까지 "레이어를 겹쳐 본다"고 말로만 했다. 그 겹치기를 실제로 하는 것이 Linux 커널의 **OverlayFS(overlay filesystem)** 다. 커널 문서의 정의: 한 파일시스템을 다른 파일시스템 위에 겹쳐서(overlay), 그 결과로 보이는 파일시스템을 제공한다. 위(upper)는 보통 쓰기 가능, 아래(lower)는 읽기 전용이어도 된다.

Docker는 이미지 레이어들을 lower로, 컨테이너 쓰기 레이어를 upper로 놓고 OverlayFS를 마운트해서 컨테이너의 `/`로 준다.

### 10.2 용어 네 개와 동작 하나

```text
              Container가 보는 /
                    merged                ← 겹쳐진 결과. 컨테이너는 이것만 본다
                      │
             ┌────────┴────────┐
             │                 │
         upperdir           lowerdir      (여러 개면 콜론으로 이어 씀)
       Writable Layer      Image Layers
        Read/Write          Read Only
             │
          workdir                          ← OverlayFS 내부 작업용. upperdir와 같은 파일시스템에 있어야 함
```

| 용어 | 뜻 | Docker에서 |
|---|---|---|
| **lowerdir** | 아래층 디렉터리들. 읽기 전용으로 취급. 여러 개를 `:`로 이어 지정하고, **왼쪽이 더 위층** | 이미지 레이어들 (overlay2는 최대 128개) |
| **upperdir** | 위층 디렉터리. 모든 쓰기가 여기로 간다 | 컨테이너 쓰기 레이어 |
| **merged** | lower들과 upper를 겹쳐 하나로 보여 주는 마운트 지점 | 컨테이너의 `/` |
| **workdir** | OverlayFS가 원자적 연산(copy-up 중간 단계 등)에 쓰는 빈 작업 디렉터리. 커널 문서: upperdir와 **같은 파일시스템**에 있어야 한다 | Docker가 쓰기 레이어 옆에 만든다 |
| **copy-up** | lower의 파일을 쓰기 위해 열면 **먼저 upper로 복사**하고(메타데이터·권한·확장 속성 포함) 그 사본을 연다 | 5장·9장의 동작 |

읽기·쓰기·삭제가 어떻게 되는지 한 번에:

```text
읽기   /usr/bin/curl      → upper에 없음 → lower들을 위에서부터 찾음 → 이미지 레이어에서 읽음 (복사 없음)
쓰기   /hello.txt (신규)  → upper에 그냥 생성
수정   /etc/myapp/config.yml (lower에 있음)
                          → copy-up: lower → upper 로 파일 복사 → upper 사본을 수정
                          → merged에서는 upper 것이 보임 (lower 것은 가려짐)
삭제   /etc/motd (lower에 있음)
                          → lower는 못 지우므로 upper에 "whiteout" 표식 파일을 둠
                          → merged에서는 없는 것처럼 보임
```

**whiteout**은 "이 이름은 지워진 것으로 취급하라"는 표식이다. 읽기 전용인 아래층에서 파일을 진짜 지울 수 없으니 위층에 지웠다는 기록을 남기는 것이다. `docker diff`의 `D`가 바로 이것이다.

### 10.3 컨테이너 안에서 직접 보기

컨테이너 안에서 `/proc/mounts` 첫 줄을 보면 `/`가 overlay로 마운트되어 있고 lowerdir·upperdir·workdir이 그대로 보인다. 실측(경로의 긴 공통 접두어는 줄였다):

```text
$ docker exec test sh -c 'head -1 /proc/mounts'
overlay / overlay rw,relatime,
  lowerdir=…/snapshots/1746/fs:…/snapshots/1745/fs,
  upperdir=…/snapshots/1747/fs,
  workdir=…/snapshots/1747/work 0 0
```

- `lowerdir`에 둘: `1745`가 `ubuntu:24.04`의 파일시스템 레이어이고, 그 위의 `1746`은 Docker가 컨테이너마다 준비하는 작은 초기화 레이어다(`/etc/hosts`, `/etc/resolv.conf`, `/etc/hostname`처럼 컨테이너별로 달라야 하는 파일을 여기 둔다). 11장에서 컨테이너마다 이 번호가 다른 것을 볼 수 있다.
- `upperdir`이 이 컨테이너의 쓰기 레이어다. `touch /hello.txt`는 호스트의 이 디렉터리에 파일을 만든 것과 같다.
- `workdir`은 upperdir과 같은 스냅샷 밑에 있다. "같은 파일시스템" 조건이 이렇게 지켜진다.

컨테이너 안에서는 `/etc`, `/usr`, `/app`이 하나의 파일시스템처럼 보이지만, 실제로는 **OverlayFS가 이 디렉터리들을 겹쳐 만든 merged 뷰**다. 이미지 레이어 3개 + 쓰기 레이어 1개가 호스트 디스크에 디렉터리 4개로 따로 있고, 커널이 겹쳐 보여 준다.

> 이 경로는 Docker Desktop(containerd 스냅샷터) 기준이다. Linux 서버의 Docker Engine + overlay2에서는 `/var/lib/docker/overlay2/<id>/diff` 같은 경로가 보인다. → [부록 A.1](#a1-docker-desktop과-linux-docker-engine의-차이)

### 10.4 기억할 것

- 한 문장: **컨테이너의 `/`는 진짜 디렉터리가 아니라, 이미지 레이어들(lower)과 쓰기 레이어(upper)를 OverlayFS가 겹쳐 보여 주는 merged 뷰다.**

---

## 11. 같은 이미지로 만든 컨테이너 둘 — 공유하는 것과 따로 갖는 것

```text
                  Image
              Read-Only Layers          ← 하나. 디스크에 한 벌
                 /      \
                /        \
               ▼          ▼
        Container A    Container B
        Writable A     Writable B       ← 각자 하나씩. 서로 모름
```

컨테이너 A에서:

```bash
touch /a.txt
```

컨테이너 B에서는 `/a.txt`가 **안 보인다.** A의 `touch`는 A의 upperdir에 파일을 만들었고, B의 merged 뷰는 B의 upperdir + 공통 lowerdir로 구성되기 때문이다. B가 보는 어느 층에도 `/a.txt`가 없다.

실측:

```text
$ docker exec app-a touch /a.txt
$ docker diff app-a
A /a.txt
$ docker diff app-b
                                  ← 비어 있음
$ docker exec app-b ls /a.txt
ls: cannot access '/a.txt': No such file or directory

# 두 컨테이너의 마운트 구성
app-a  upperdir=…/snapshots/1772/fs   lowerdir=…/1771/fs:…/1745/fs
app-b  upperdir=…/snapshots/1775/fs   lowerdir=…/1774/fs:…/1745/fs
                                                          ^^^^ 이미지 레이어 1745 는 공유
       ^^^^ upperdir 은 다름                  ^^^^ 컨테이너별 초기화 레이어도 다름
```

**공유하는 것은 이미지 레이어(1745), 따로 갖는 것은 쓰기 레이어(1772 vs 1775)** 다. 그래서 같은 이미지로 컨테이너 100개를 띄워도 이미지는 한 벌이고, 각 컨테이너가 쓴 것만 100벌이다.

이게 이미지의 **불변성(immutability)** 이 실전에서 뜻하는 바다. 어떤 컨테이너가 무슨 짓을 해도 이미지는 안 바뀌므로, "같은 이미지 = 같은 시작 상태"가 보장된다. 배포에서 "이미지 태그가 같으면 어디서 띄워도 같다"고 믿을 수 있는 근거가 여기 있다.

- 한 문장: **이미지는 공유 원본, 쓰기 레이어는 각자의 사본. 컨테이너끼리 파일을 나누려면 레이어가 아니라 볼륨이 필요하다.**

---

## 12. Volume으로 이어지는 이유

지금까지의 결론을 데이터 관점에서 다시 쓰면 이렇다.

- 쓰기 레이어는 **컨테이너 객체에 종속**된다. `rm`하면 사라진다(6장).
- 쓰기 레이어는 **컨테이너 사이에 공유되지 않는다**(11장).
- 쓰기 레이어에 큰 파일을 두고 고치면 **copy-up 비용**이 든다(9장).

DB 데이터 파일, 사용자가 업로드한 파일, 지워지면 안 되는 로그는 셋 다에 걸린다. 컨테이너는 배포할 때마다 지우고 새로 만드는 게 정상인데, 그때마다 DB가 날아가면 안 된다. **컨테이너의 수명과 데이터의 수명을 분리**해야 한다.

그 수단이 **볼륨(Volume)** 이다. 호스트의 별도 저장 공간을 컨테이너 안 경로에 **마운트**해서, 그 경로에 대한 읽기·쓰기가 레이어를 거치지 않고 바로 볼륨으로 가게 한다.

```text
Container
   │  /var/lib/postgresql/data 에 대한 I/O 는
   │  OverlayFS 레이어가 아니라
   │  mount 된 볼륨으로 직접 간다
   ▼
Volume   ← 컨테이너 객체 밖에 있음. docker rm 해도 남음. 다른 컨테이너에 다시 붙일 수 있음
```

```text
Container Writable Layer
→ Container 객체에 종속. rm 하면 소멸. 컨테이너 간 공유 안 됨. copy-up 비용

Volume
→ Container lifecycle과 분리. rm 해도 유지(익명 볼륨은 rm -v 시 삭제). 여러 컨테이너에 마운트 가능. 레이어를 안 거침
```

Docker 문서의 권고도 같다. "쓰기가 많은 데이터, 컨테이너 수명을 넘어 남아야 하는 데이터, 컨테이너 간에 공유해야 하는 데이터에는 볼륨을 쓰라."

볼륨과 바인드 마운트의 종류·동작·권한 문제는 이 문서 범위 밖이고, [15.3절](#153-다음-학습-순서)의 첫 번째 주제로 남긴다. 여기서는 **"쓰기 레이어에 두면 안 되는 데이터가 있고, 그걸 위한 별도 통로가 볼륨"** 까지만 연결해 두면 된다.

---

## 13. 전체 개념 연결

### 13.1 명령 관점 — 데이터의 수명

```text
Dockerfile                  ← 레시피
    ↓  docker build
Image                       ← 읽기 전용 레이어들 + 메타데이터. 레이어는 이미지 간 공유·캐시
    ↓
Read-Only Layers            ← 파일시스템을 바꾼 instruction(RUN/COPY/ADD)마다 한 층
    ↓  docker run
Container                   ← 이미지를 복사하지 않고
    ↓
Writable Layer 추가         ← 그 위에 컨테이너 전용 쓰기 층 하나
    ↓  docker exec … touch / vi
docker exec로 파일 변경      ← 같은 컨테이너 안에 프로세스 추가
    ↓
Writable Layer에 변경 기록   ← 신규 파일은 그대로, 기존 파일은 copy-up 후 수정. 이미지는 불변
    ↓  docker stop / start
Writable Layer 유지         ← 프로세스만 끝났다 다시 뜸. 객체와 쓰기 층은 그대로
    ↓  docker rm
Writable Layer 제거         ← 객체가 사라지며 쓰기 층도 소멸. 이미지는 남음
                               (남아야 하는 데이터는 볼륨으로)
```

### 13.2 파일시스템 관점 — 한 개의 `/`가 만들어지는 법

```text
Image Layers            ← lowerdir (여러 개, 읽기 전용, 컨테이너 간 공유)
      +
Writable Layer          ← upperdir (컨테이너마다 하나) + workdir
      ↓
OverlayFS               ← 커널이 겹친다. 읽기는 위→아래 탐색, 쓰기는 upper로, 기존 파일 수정은 copy-up, 삭제는 whiteout
      ↓
merged filesystem       ← 겹친 결과
      ↓
Container에서는 하나의 / 로 보임
```

두 그림을 잇는 고리가 Copy-on-Write다. **"안 바꾸는 동안 공유하고, 바꿀 때 그 부분만 복사한다"** 는 전략이 있어서 이미지는 한 벌로 공유되고, 컨테이너는 순식간에 만들어지고, 그러면서도 각자 파일을 쓸 수 있다. 같은 전략을 Linux는 `fork()`의 메모리 페이지에, Docker는 OverlayFS의 파일에 쓴다.

---

## 14. 실습 — 10단계로 직접 확인하기

아래는 이 문서의 실측을 그대로 재현하는 순서다. Docker만 있으면 된다. 각 단계에 **이 명령으로 무엇을 확인하는지**를 적었다.

```bash
# 1. 컨테이너 생성 — 이미지에서 컨테이너 객체 + 쓰기 레이어를 만들고 PID 1(sleep)을 띄운다
docker run -d --name test ubuntu:24.04 sleep infinity

# 2. 컨테이너 내부에 파일 생성 — exec 는 같은 컨테이너에 프로세스를 추가하는 것임을 확인
docker exec test touch /hello.txt
docker exec test sh -c 'echo "PID1: $(tr "\0" " " </proc/1/cmdline)"; echo "my PID: $$"'
#   → PID1: sleep infinity / my PID: (1이 아닌 숫자)

# 3. docker diff 로 변경 확인 — 쓰기 레이어에 무엇이 생겼는지. A=추가 C=변경 D=삭제
docker diff test
#   → A /hello.txt

#    (덤) 컨테이너의 / 가 OverlayFS merged 뷰임을 확인 — lowerdir/upperdir/workdir 이 보인다
docker exec test sh -c 'head -1 /proc/mounts'

# 4. stop — 프로세스만 끝낸다. 객체는 Exited 상태로 남는다
docker stop test
docker ps -a --filter name=test --format '{{.Names}} {{.Status}}'
#   → test Exited (137) …      (137 = SIGKILL. sleep 이 PID 1 이라 SIGTERM 을 안 받아서)

# 5. start — 같은 객체를 다시 실행한다. 새 컨테이너가 아니다
docker start test

# 6. 파일이 남아 있는지 확인 — 쓰기 레이어가 stop/start 를 넘어 유지됨
docker exec test ls -l /hello.txt
#   → -rw-r--r-- 1 root root 0 … /hello.txt

# 7. 컨테이너 제거 — 객체와 쓰기 레이어가 함께 사라진다 (-f: 실행 중이라 SIGKILL 후 삭제)
docker rm -f test
docker ps -a --filter name=test
#   → 없음

# 8. 같은 이미지로 새 컨테이너 생성 — 새 객체, 새(빈) 쓰기 레이어
docker run -d --name test2 ubuntu:24.04 sleep infinity

# 9. 이전 파일이 없는지 확인 — 이미지는 안 바뀌었고, 옛 쓰기 레이어는 사라졌다
docker exec test2 ls /hello.txt
#   → ls: cannot access '/hello.txt': No such file or directory
docker diff test2
#   → (비어 있음)
docker rm -f test2

# 10. 이미지 레이어 확인 — 어떤 instruction 이 몇 바이트짜리 층을 만들었나, 실제 레이어 수는 몇인가
docker history ubuntu:24.04
#   → ADD file:… 110MB 한 줄만 크기가 있고 CMD/LABEL/ARG 는 0B
docker inspect --format '{{len .RootFS.Layers}} layers' ubuntu:24.04
#   → 1 layers
docker inspect --format '{{json .Config.Cmd}}' ubuntu:24.04
#   → ["/bin/bash"]      (이미지 메타데이터)
```

여기까지가 4장·6장·11장이다. 5장(기존 파일 수정 = copy-up)과 2장(빌드 캐시)까지 보려면 작은 이미지를 하나 만든다.

```bash
mkdir studylab && cd studylab
cat > Dockerfile <<'EOF'
FROM ubuntu:24.04
RUN mkdir -p /etc/myapp && echo 'env: production' > /etc/myapp/config.yml
COPY app.txt /app/app.txt
CMD ["sleep", "infinity"]
EOF
echo 'version 1' > app.txt

# 빌드 캐시 — 두 번째 빌드는 전부 CACHED, app.txt 만 바꾼 세 번째 빌드는 RUN 은 CACHED 이고 COPY 만 다시
docker build -t studylab:1 .
docker build --progress=plain -t studylab:1 . 2>&1 | grep -E 'CACHED|\[[0-9]/3\]'
echo 'version 2' > app.txt
docker build --progress=plain -t studylab:1 . 2>&1 | grep -E 'CACHED|\[[0-9]/3\]'

# history 8줄 vs 실제 레이어 3개 — 메타데이터 instruction 은 층을 만들지 않는다
docker history studylab:1
docker inspect --format '{{len .RootFS.Layers}} layers' studylab:1

# 기존 파일 수정 — copy-up. diff 에 C 로 잡히고, 같은 이미지의 새 컨테이너는 원본을 본다
docker run -d --name cfg studylab:1
docker exec cfg sh -c "sed -i 's/production/development/' /etc/myapp/config.yml && cat /etc/myapp/config.yml"
#   → env: development
docker diff cfg
#   → C /etc  /  C /etc/myapp  /  C /etc/myapp/config.yml
docker run --rm studylab:1 cat /etc/myapp/config.yml
#   → env: production        ← 이미지는 그대로
docker rm -f cfg

# 두 컨테이너 — 쓰기 레이어는 독립, 이미지 레이어는 공유
docker run -d --name app-a ubuntu:24.04 sleep infinity
docker run -d --name app-b ubuntu:24.04 sleep infinity
docker exec app-a touch /a.txt
docker exec app-b ls /a.txt            # → No such file
for c in app-a app-b; do docker exec $c sh -c 'head -1 /proc/mounts' | grep -o 'upperdir=[^,]*'; done
#   → upperdir 이 서로 다르다. lowerdir 의 이미지 레이어는 같다
docker rm -f app-a app-b
docker rmi studylab:1
```

---

## 15. 마지막 정리

### 15.1 핵심 5줄 요약

1. **Image**는 읽기 전용 파일시스템 레이어들과 실행 메타데이터의 묶음이지, 실행 중인 것이 아니다. 레이어는 "이전 층 위에서 한 instruction이 만든 차이"라서 이미지 간에 공유되고 빌드 캐시로 재사용된다.
2. **Container**는 그 이미지 레이어들 위에 **컨테이너 전용 쓰기 레이어 하나**를 얹고 PID 1을 띄운 것이다. 이미지는 복사되지 않는다.
3. 컨테이너 안의 모든 파일 변경(`exec`로 만든 것 포함)은 **쓰기 레이어에만** 기록되고 이미지는 불변이다. `stop`/`start`는 프로세스만 오가므로 쓰기 레이어가 남고, `rm`은 객체를 지우므로 쓰기 레이어도 사라진다.
4. 이 구조의 원리가 **Copy-on-Write**다. 안 바꾸는 동안은 공유하고, 바꿀 때 그 부분만 복사한다. Linux `fork()`는 이걸 메모리 페이지(4KB)에, Docker는 파일시스템에 쓴다. 전략은 같고 구현과 단위는 다르다.
5. 그 구현이 **OverlayFS**다. 이미지 레이어들(lowerdir)과 쓰기 레이어(upperdir)를 겹쳐 하나의 merged `/`로 보여 주고, 기존 파일 수정은 파일 단위 copy-up, 삭제는 whiteout으로 처리한다. 컨테이너보다 오래 살아야 할 데이터는 이 레이어 밖, 즉 볼륨에 둔다.

### 15.2 자주 헷갈리는 부분

| 오해 | 실제 | 어디서 |
|---|---|---|
| `docker exec`로 수정하면 이미지가 변경된다 | **X.** 그 컨테이너의 쓰기 레이어만 바뀐다. 같은 이미지로 만든 새 컨테이너에는 없다 | [4장](#4-docker-exec로-파일을-만들면--새-프로세스-같은-컨테이너) · [5장](#5-이미지에-있던-파일을-고치면--위층이-아래층을-가린다) |
| `docker stop`하면 쓰기 레이어가 사라진다 | **X.** 프로세스만 끝난다. 객체와 쓰기 레이어는 남고 `start`하면 그대로 보인다 | [6.2절](#62-stop--start-프로세스만-끝나고-객체는-남는다) |
| `docker rm`과 `docker stop`은 비슷하다 | **X.** `stop`은 프로세스를, `rm`은 컨테이너 객체(와 쓰기 레이어)를 없앤다. `rm` 뒤의 `run`은 새 객체다 | [6.4절](#64-한-줄로-비교) |
| Linux 메모리 CoW와 Docker CoW는 완전히 동일한 구현이다 | **X.** 같은 **전략**을 다른 계층에서 구현한 것. 단위도 페이지 vs 파일로 다르다 | [8.3절](#83-같은-것과-다른-것) · [9장](#9-둘의-중요한-차이--페이지-단위와-파일-단위-copy-up) |
| 컨테이너를 만들 때 이미지 파일시스템 전체를 복사한다 | **X.** 이미지 레이어는 그대로 두고 빈 쓰기 레이어 하나만 얹는다. 그래서 생성이 빠르고 100개를 띄워도 이미지는 한 벌이다 | [3장](#3-container-writable-layer--복사하지-않고-한-층-얹는다) · [11장](#11-같은-이미지로-만든-컨테이너-둘--공유하는-것과-따로-갖는-것) |
| `docker exec`는 컨테이너를 하나 더 띄운다 | **X.** 기존 컨테이너의 namespace 안에 프로세스를 추가한다. PID 1이 죽으면 같이 끝난다 | [4.1절](#41-docker-exec는-무엇을-하나) |
| Dockerfile 한 줄마다 레이어가 하나 생긴다 | **부분적으로만.** 파일시스템을 바꾸는 `RUN`/`COPY`/`ADD`만 층을 만든다. `CMD`/`ENV`/`LABEL`은 메타데이터라 `history`에 0B로 나온다 | [2.3절](#23-dockerfile-instruction과-레이어의-관계--전부-층을-만들지는-않는다) |
| 컨테이너 안의 `/`는 호스트 어딘가의 진짜 디렉터리다 | **X.** 여러 디렉터리를 OverlayFS가 겹쳐 보여 주는 merged 뷰다. 실체는 lowerdir 여러 개 + upperdir 하나 | [10장](#10-overlayfs--여러-디렉터리를-겹쳐-하나의-로-보여-준다) |

### 15.3 다음 학습 순서

이번 문서는 컨테이너의 **파일시스템 축** 하나를 끝까지 따라간 것이다. 다음은 이 축과 맞닿은 순서대로.

1. **Docker Volume / Bind Mount** — 12장에서 "쓰기 레이어 밖의 통로"라고만 한 것. 볼륨·바인드 마운트·tmpfs의 차이, 익명/이름 볼륨, 컨테이너 간 공유, 권한(UID) 문제. 이번 문서의 `rm -v` 각주가 출발점.
2. **OverlayFS 조금 더 자세히** — 10장의 다음 단계. 디렉터리 병합 규칙, whiteout과 opaque 디렉터리의 실체, 여러 lower의 순서, `redirect_dir`·`index` 같은 옵션, overlay2 드라이버가 lower를 128개까지 다루는 방법, copy-up이 성능에 미치는 영향의 실측.
3. **Container = Process라는 관점** — 4장의 "PID 1 + exec로 추가된 프로세스"를 뒤집어 보기. 컨테이너는 VM이 아니라 **격리된 프로세스 트리**이고, PID 1이 죽으면 컨테이너가 끝나며, `stop`의 SIGTERM을 PID 1이 받아야 하는 이유(6장의 137). 이 관점이 잡히면 4·5번이 자연스럽다.
4. **Linux namespace** — 4장에서 "같은 namespace 안에서"라고 한 그것. 마운트 namespace가 이 문서의 merged `/`를 컨테이너의 `/`로 만들고, PID namespace가 `sleep`을 PID 1로 만든다. 그 외 net·uts·ipc·user namespace.
5. **Linux cgroup** — namespace가 "무엇이 보이나"라면 cgroup은 "얼마나 쓸 수 있나". 메모리·CPU 한도. [JVM 메모리 노트 부록 A.1](jvm-memory-and-container-limits.md#a1-cgroup--리눅스가-자원-한도를-거는-방법)에 기초가 있다.
6. **Docker → containerd → runc → Linux Kernel 관계** — 이 문서에서 `/proc/mounts` 경로에 `containerd`와 `snapshotter`가 나온 이유. `docker` CLI는 데몬(dockerd)에 요청하고, 이미지·스냅샷 관리는 containerd가, 실제 namespace·cgroup을 만들고 프로세스를 띄우는 것은 runc가 한다. [부록 A.1](#a1-docker-desktop과-linux-docker-engine의-차이)의 차이가 이 계층 구조에서 나온다.
7. **OCI Image / Runtime Specification** — 2장의 "레이어 = 내용 해시가 붙은 차이분 tar"와 1장의 "메타데이터"가 표준으로는 어떻게 정의되어 있나(manifest, config, layer blob). 이 표준 덕에 Docker로 만든 이미지를 다른 런타임(containerd 단독, Podman, Kubernetes의 CRI 구현)이 그대로 실행한다.

---

# 부록

## A.1 Docker Desktop과 Linux Docker Engine의 차이

이 문서의 실측은 macOS의 Docker Desktop에서 했다. Docker Desktop은 Linux VM 안에서 엔진을 돌리고, 이미지 저장소로 **containerd 이미지 스토어 + overlayfs 스냅샷터**를 쓴다. Linux 서버에 직접 설치한 Docker Engine(전통적인 **overlay2 스토리지 드라이버**)과 다음이 다르다. **동작 원리(OverlayFS, copy-up, 쓰기 레이어)는 같고 경로와 표시만 다르다.**

| 항목 | Docker Desktop (이 문서 환경) | Linux Docker Engine + overlay2 |
|---|---|---|
| `docker info`의 Storage Driver | `overlayfs` (driver-type `io.containerd.snapshotter.v1`) | `overlay2` |
| 레이어 저장 경로 | `/var/lib/desktop-containerd/daemon/io.containerd.snapshotter.v1.overlayfs/snapshots/<n>/fs` | `/var/lib/docker/overlay2/<id>/diff` |
| `/proc/mounts`의 lowerdir/upperdir | 위 snapshots 경로 | `/var/lib/docker/overlay2/l/<short>` 심볼릭 링크와 `<id>/diff` |
| `docker inspect <container>`의 `GraphDriver` | **없음** (템플릿에서 참조하면 "map has no entry for key GraphDriver") | `Name: overlay2`와 `LowerDir`/`UpperDir`/`MergedDir`/`WorkDir` 경로 |
| 호스트에서 레이어 디렉터리 직접 보기 | VM 안이라 macOS에서는 바로 못 봄 | `sudo ls /var/lib/docker/overlay2/...`로 가능 |

Linux 서버에서 이 문서의 10장을 확인하려면 `docker inspect --format '{{json .GraphDriver.Data}}' <container>`가 lowerdir·upperdir·merged·workdir 경로를 그대로 보여 준다. Docker Desktop에서는 대신 컨테이너 안의 `/proc/mounts`를 보면 된다(10.3절).

## A.2 참고한 공식 문서

- Docker Docs — Storage drivers 개요(이미지와 레이어, 컨테이너 레이어, copy-on-write, copy_up), OverlayFS storage driver(lowerdir/upperdir/merged/workdir, copy_up은 파일 전체, whiteout, lower 128개)
- Docker Docs — Docker concepts: Understanding the image layers(레이어 = 파일시스템 변경 집합, 이미지 간 재사용)
- Docker Docs — Build cache, Cache invalidation(COPY/ADD는 파일 체크섬, RUN은 명령 문자열, 이후 단계 전부 무효화), Building best practices(`apt-get update && install` 결합)
- Docker Docs — Dockerfile reference(`RUN`은 새 레이어 생성, `CMD`는 빌드 시 실행 안 함), CLI reference: `container exec` / `stop`(SIGTERM → 유예 10초 → SIGKILL) / `rm`(`-f`, `-v`) / `diff`(A/D/C)
- Linux kernel documentation — Overlay Filesystem(upper/lower, workdir 조건, copy_up, whiteout, 다중 lower)
- `fork(2)` man page(Linux의 fork는 copy-on-write 페이지로 구현)
- 실측: Docker Engine 29.0.1 / Docker Desktop, linux/arm64, `ubuntu:24.04` (2026-09-20)
