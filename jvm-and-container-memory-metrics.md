# 배치 메모리 알람을 만들며 배우는 힙·컨테이너 메모리 지표 읽는 법

> 배치 잡이 메모리 한도에 가까워지면 죽기 전에 알 수 있도록, 실행 중 메모리를 로그로 남기고 그 로그로 알람을 거는 작업을 하면서 나온 개념을 정리한 학습 문서.
> 계기: 회사 업무(배치 자원 사용 로그 추가). 구현 계획과 코드는 회사 레포에 있음. 여기서는 일반 개념과 최소 예제만 다룬다.
> 작성일: 2026-11-05
> 함께 읽기: 힙·네이티브 메모리·cgroup 기초는 [JVM 메모리와 컨테이너 한도](jvm-memory-and-container-limits.md)에 이미 있다. 이 문서는 그 위에 "어떻게 재나"를 얹는다. 로그를 언제 남기게 했는지(Spring 이벤트)는 [Spring Boot 기동 순서와 이벤트 리스너](spring-boot-startup-lifecycle-and-events.md)에 있다.

---

## 0. 이번 작업 한 문단 요약

컨테이너에서 도는 배치 잡에 **10초마다 메모리 표본 로그**를 남기고, 잡이 끝나면 **요약 로그 1줄**을 남겼다. 요약에는 두 비율이 들어간다. 하나는 **GC 직후 힙 사용량의 최댓값 ÷ 힙 최대치**, 다른 하나는 **페이지 캐시를 뺀 컨테이너 메모리 사용량의 최댓값 ÷ 태스크 메모리 한도**다. 컨테이너 쪽 값은 **ECS 태스크 메타데이터 엔드포인트 v4**의 `/task/stats`와 `/task`에서 읽는다. 로그는 JSON 구조화 로그라서, CloudWatch **지표 필터**가 이 두 숫자를 지표로 바꾸고 알람이 임계값을 넘는 잡을 알려준다.

이 문단에 모르는 단어가 있으면 이 문서가 도움이 된다.

---

## 핵심 요약

- 메모리는 **두 군데에서** 떨어진다. 힙이 차면 JVM 안에서 죽고, 힙과 네이티브 메모리의 합이 컨테이너 한도를 넘으면 커널이 밖에서 죽인다. 그래서 계기판도 둘이 필요하다. → [1장](#1-무엇을-재려는가--두-가지-고갈에-계기판-둘)
- 힙 **순간 사용량**은 톱니 모양이라 높게 찍혀도 위험하다는 뜻이 아니다. 쓰레기가 쌓였다가 GC가 치우기 때문이다. → [2장](#2-힙-순간-사용량은-톱니다)
- **GC 직후 힙**(`MemoryPoolMXBean.getCollectionUsage()`)은 청소 뒤에도 남은 양, 즉 실제로 살아 있는 객체의 양에 가깝다. 알람은 이 값으로 건다. → [3장](#3-gc-직후-힙--청소하고도-남은-양)
- 힙 비율의 분모는 **힙 전체 최대치**다. 풀별 최대치는 정의되지 않을(-1) 수 있다. → [4장](#4-분모는-힙-최대치다)
- cgroup이 세는 컨테이너 메모리에는 **페이지 캐시**도 들어 있다. 캐시는 커널이 필요하면 회수하므로 그대로 쓰면 과장된다. → [5장](#5-컨테이너-메모리에는-페이지-캐시도-들어-있다)
- 그래서 cgroup v1은 `usage - cache`, v2는 `usage - inactive_file`로 본다. Docker CLI와 ECS agent가 쓰는 식이다. → [6장](#6-그래서-회수-가능한-캐시를-빼고-본다)
- ECS에서는 컨테이너 안에서 `ECS_CONTAINER_METADATA_URI_V4` 환경변수로 메타데이터를 읽는다. `/task/stats`는 Docker stats, `/task`의 `Limits.Memory`는 태스크 한도다. → [7장](#7-값은-어디서-읽나--ecs-태스크-메타데이터-v4)
- 비율을 만들 때 분모가 없거나 0이면 NaN·Infinity가 생긴다. 이 값이 JSON에 들어가면 지표 필터가 깨진다. → [8장](#8-비율을-만들-때의-함정)
- 요약 로그만 남기면 강제 종료된 잡은 아무것도 남지 않는다. 그래서 **주기 표본**을 함께 남긴다. → [9장](#9-왜-요약만이-아니라-주기-표본도-남기나)
- 로그 필드 이름은 알람과의 **계약**이다. 이름이 바뀌면 알람이 에러 없이 조용히 멈춘다. → [10장](#10-로그를-지표로--구조화-로그와-지표-필터)
- JDK `HttpClient`의 요청 timeout은 **응답 헤더까지만** 잰다. 본문이 멈추면 따로 막아야 한다. → [12장](#12-측정이-잡을-막으면-안-된다--http-timeout의-함정)

## 면접 질문 체크리스트

- [ ] 힙 사용량 그래프가 톱니 모양인 이유는? 순간값으로 알람을 걸면 왜 안 되나?
- [ ] `getUsage()`와 `getCollectionUsage()`는 무엇이 다른가?
- [ ] 컨테이너 메모리 사용량(cgroup `memory.current`)에 페이지 캐시가 들어가나? 그러면 왜 OOM이 안 나나?
- [ ] `docker stats`의 메모리 값은 어떻게 계산되나? cgroup v1과 v2에서 무엇이 다른가?
- [ ] 왜 `inactive_file`만 빼고 `active_file`은 빼지 않나?
- [ ] ECS 컨테이너 안에서 자기 태스크의 메모리 한도와 사용량을 어떻게 아나?
- [ ] 로그 기반 지표의 장단점은? 필드 이름을 바꾸면 무슨 일이 생기나?
- [ ] 측정 코드가 본 작업을 망치지 않게 하려면 무엇을 지켜야 하나?

## 질문 → 섹션

| 실제로 물어본 질문 | 섹션 |
|---|---|
| cgroup 메모리와 힙 메모리 지표는 각각 무엇을 보여주나 | [1장](#1-무엇을-재려는가--두-가지-고갈에-계기판-둘) · [5장](#5-컨테이너-메모리에는-페이지-캐시도-들어-있다) |
| 힙은 왜 "사용량"이 아니라 "GC 직후"로 보나 | [2장](#2-힙-순간-사용량은-톱니다) · [3장](#3-gc-직후-힙--청소하고도-남은-양) |
| 힙 알람 임계값(예: 60%)은 어떻게 정하나 | [11장](#11-임계값은-어떻게-정하나) |
| 컨테이너 사용량 계산식이 왜 v1·v2에서 다른가 | [6장](#6-그래서-회수-가능한-캐시를-빼고-본다) |
| 메타데이터 호출이 멈추면 잡도 멈추나 | [12장](#12-측정이-잡을-막으면-안-된다--http-timeout의-함정) |

---

## 1. 무엇을 재려는가 — 두 가지 고갈에 계기판 둘

[JVM 메모리 노트 6장](jvm-memory-and-container-limits.md#6-메모리가-떨어지는-두-가지-방식--이-문서의-핵심)의 결론부터 가져온다.

| 고갈 방식 | 무엇이 차나 | 누가 죽이나 | 남는 흔적 |
|---|---|---|---|
| 안에서 터진다 | 힙 | JVM (`OutOfMemoryError`) | 로그를 남길 수 있다. `ExitOnOutOfMemoryError`면 종료 코드 3 |
| 밖에서 죽는다 | 힙 + 네이티브 메모리 합계 | 리눅스 커널 (cgroup OOM) | 로그를 못 남긴다. 종료 코드 137 |

힙 상한을 컨테이너 한도보다 충분히 작게 잡아 두면([JVM 노트 8.1절](jvm-memory-and-container-limits.md#81-힙-상한을-컨테이너보다-확실히-작게-잡는다)) 두 경로가 따로 놀게 된다. 힙이 꽉 차도 컨테이너 사용률은 낮을 수 있고, 반대로 네이티브 메모리가 새면 힙은 멀쩡한데 컨테이너가 죽는다. 그래서 하나의 "메모리 사용률"로는 둘 다 못 잡는다. **힙 비율**과 **컨테이너 비율**을 따로 잰다.

**왜 실전에서 중요한가**: 컨테이너 메모리 알람만 걸어 두면 힙 OOM은 대부분 놓친다([JVM 노트 11.4절](jvm-memory-and-container-limits.md#114-자원-사용률-알람--다만-무엇을-감시하는지-알고-걸어야-한다)). 힙 알람만 걸어 두면 네이티브 누수를 놓친다.

---

## 2. 힙 "순간 사용량"은 톱니다

JVM은 객체를 만들 때마다 힙에 자리를 잡는다. 다 쓴 객체(쓰레기)는 바로 치우지 않고, 어느 정도 쌓이면 **GC**(Garbage Collector — 더 이상 참조되지 않는 객체를 찾아 메모리를 회수하는 JVM 구성 요소)가 한꺼번에 치운다. 그래서 힙 사용량을 시간 축으로 그리면 이렇게 생긴다.

```
사용량
 ▲     /|    /|    /|
 │    / |   / |   / |      ← 쌓이다가(할당) GC 때 뚝 떨어진다
 │   /  |  /  |  /  |
 │  /   | /   | /   |
 │ /    |/    |/    |      ← 바닥선 = GC 후에도 남는 양
 └──────────────────────▶ 시간
```

꼭대기는 "아직 안 치운 쓰레기"를 포함한다. 꼭대기가 최대치의 90%를 찍어도 다음 GC 뒤 30%로 내려간다면 정상이다. 진짜 위험 신호는 **바닥선이 올라가는 것**이다. 살아 있는 객체가 늘어서, 치워도 공간이 안 생긴다는 뜻이기 때문이다.

`MemoryMXBean.getHeapMemoryUsage().getUsed()`가 주는 값이 바로 이 순간 사용량(톱니의 아무 지점)이다. 10초마다 찍으면 톱니의 어디를 찍었는지는 운에 달린다.

---

## 3. GC 직후 힙 — 청소하고도 남은 양

### 3.1 메모리 풀과 두 가지 사용량

힙은 한 덩어리가 아니라 여러 **메모리 풀**(memory pool — GC가 따로 관리하는 힙의 구역)로 나뉜다. G1 GC라면 보통 `G1 Eden Space`, `G1 Survivor Space`, `G1 Old Gen` 세 풀이 보인다. 풀 이름과 개수는 GC 종류마다 다르다.

JDK의 `java.lang.management` API는 풀마다 `MemoryPoolMXBean`을 하나씩 준다. 여기에 사용량 메서드가 둘 있다.

| 메서드 | 뜻 |
|---|---|
| `getUsage()` | 지금 이 순간의 사용량 (톱니의 현재 위치) |
| `getCollectionUsage()` | 이 풀에서 **JVM이 가장 최근에 쓰레기를 회수한 직후**의 사용량. 지원하지 않으면 `null` |

JDK 21 Javadoc의 표현은 "the memory usage after the Java virtual machine most recently expended effort in recycling unused objects in this memory pool"이다. 그리고 이 메서드가 GC를 일으키지는 않는다고 명시돼 있다. 그냥 마지막 GC 때 기록해 둔 값을 읽는다.

### 3.2 GC 직후 힙 = 힙 풀들의 `getCollectionUsage()` 합

```kotlin
import java.lang.management.ManagementFactory
import java.lang.management.MemoryType

fun heapAfterGcBytes(): Long? {
    val sum = ManagementFactory.getMemoryPoolMXBeans()
        .filter { it.type == MemoryType.HEAP }        // 힙 풀만 (Metaspace 등 제외)
        .mapNotNull { it.collectionUsage?.used }      // 지원 안 하는 풀은 건너뛴다
        .sum()
    return sum.takeIf { it > 0 }                      // 아직 GC가 한 번도 없으면 0 → "값 없음"
}
```

알아둘 점:

1. **아직 GC가 없으면 0이다.** 기동 직후에는 GC가 한 번도 안 돌았을 수 있다. 0을 그대로 쓰면 "힙 사용 0%"라는 거짓말이 된다. 그래서 "값 없음"(null)으로 다룬다.
2. **풀마다 "마지막 GC" 시점이 다르다.** Eden은 young GC마다 갱신되지만, Old 영역 풀이 어느 GC에서 갱신되는지는 GC 종류에 따라 다르다(이 노트에서 GC별로 확인하지는 않았다). 그래서 합계는 "한 시점의 정확한 스냅샷"이 아니라 **바닥선의 근사치**다. 알람 목적으로는 충분하다.
3. **최댓값을 기억한다.** 10초마다 읽어서 그중 가장 큰 값을 잡 종료 요약에 남긴다. "이 잡이 가장 무거웠던 순간의 바닥선"이다.

### 3.3 GC 로그와는 무엇이 다른가

GC 로그(`-Xlog:gc*`, [JVM 노트 11.1절](jvm-memory-and-container-limits.md#111-gc-로그--켜두지-않으면-사후에-못-만든다))에도 GC 전후 크기가 다 찍힌다. 다만 GC 로그는 평문이라 지표로 뽑기 어렵다. 이번 작업은 **지표로 뽑을 숫자 하나**가 필요해서, 앱이 직접 읽어 JSON 필드로 남겼다. 둘은 경쟁 관계가 아니다. 알람은 지표로 받고, 원인 분석은 GC 로그로 한다.

---

## 4. 분모는 힙 최대치다

비율에는 분모가 필요하다. `MemoryMXBean.getHeapMemoryUsage().getMax()`는 힙 전체가 커질 수 있는 최대치, 즉 `-Xmx`나 `-XX:MaxRAMPercentage`로 정해진 값이다([JVM 노트 5.3절](jvm-memory-and-container-limits.md#53-힙-상한을-절대값이-아니라-비율로-주는-이유)).

`MemoryUsage.getMax()`는 **정의되지 않으면 -1**을 돌려준다(JDK Javadoc: "or -1 if undefined"). 풀 단위 max는 GC에 따라 -1인 경우가 있어서, 풀별 max를 더해 분모로 쓰면 틀릴 수 있다. 그래서 분자는 풀별 합, 분모는 **힙 전체 max**로 잡는다. max가 0 이하이면 비율을 만들지 않는다([8장](#8-비율을-만들-때의-함정)).

```
heapAfterGcPeakPct = (GC 직후 힙 합계의 최댓값) ÷ (힙 전체 max) × 100
```

---

## 5. 컨테이너 메모리에는 페이지 캐시도 들어 있다

### 5.1 cgroup이 세는 것

컨테이너의 메모리 한도는 cgroup이 건다([JVM 노트 부록 A.1](jvm-memory-and-container-limits.md#a1-cgroup--리눅스가-자원-한도를-거는-방법)). cgroup v2 커널 문서는 `memory.current`를 "그 cgroup과 하위 cgroup이 지금 쓰는 메모리 총량"이라고 정의한다. 그리고 여기에 세는 메모리에 **"page cache and anonymous memory"**가 포함된다고 적는다.

| 종류 | 무엇인가 | 회수 가능? |
|---|---|---|
| anonymous memory (`anon`) | 프로그램이 직접 할당한 메모리. JVM 힙·스택·네이티브 메모리가 여기에 든다 | 스왑이 없으면 회수 불가 |
| page cache (`file`) | 파일을 읽고 쓸 때 커널이 메모리에 캐싱해 둔 내용. 로그 파일·JAR 파일 등 | 대부분 회수 가능(디스크에서 다시 읽으면 된다) |

즉 컨테이너가 로그를 많이 쓰거나 큰 파일을 읽으면, 앱이 쓰는 메모리는 그대로인데 `memory.current`가 올라간다.

> [JVM 노트 부록 A.4](jvm-memory-and-container-limits.md#a4-왜-메모리-사용량rss이-힙보다-큰가)는 "RSS가 컨테이너 한도가 걸리는 대상"이라고 단순화했다. 더 정확히는 cgroup은 **RSS(프로세스의 anon 메모리 위주) + 페이지 캐시 등**을 함께 센다. 다만 캐시는 한도에 닿으면 먼저 회수되므로, 실제로 OOM을 결정하는 건 회수할 수 없는 부분이다.

### 5.2 그럼 왜 캐시 때문에 OOM이 나지 않나

한도에 닿으면 커널은 먼저 **회수(reclaim)** 한다. 캐시처럼 버려도 되는 페이지를 비우고, 그래도 부족할 때만 OOM Killer를 부른다. cgroup v2 문서도 같은 순서를 적는다. 사용량이 한도에 닿았는데 회수로도 줄일 수 없으면 그 cgroup에서 OOM killer가 호출된다.

그래서 `memory.current`가 한도의 95%여도 그중 절반이 캐시라면 위험하지 않다. 날것 그대로 알람을 걸면 **거짓 경보**가 난다.

---

## 6. 그래서 회수 가능한 캐시를 빼고 본다

### 6.1 표준 계산식

`docker stats`가 보여주는 메모리 값과 ECS agent가 계산하는 값은 같은 식을 쓴다(ECS agent `agent/stats/utils_unix.go`의 `getMemUsage`, 코드 주석에 Docker CLI와 맞춘다고 적혀 있다).

| cgroup 버전 | 계산 | 빼는 값의 뜻 |
|---|---|---|
| v1 | `usage - cache` | `memory.stat`의 `cache` = 페이지 캐시 전체 |
| v2 | `usage - inactive_file` | `memory.stat`의 `inactive_file` = 최근에 안 쓴 파일 캐시 |
| 공통 | 빼는 값이 없거나 `usage` 이상이면 `usage` 그대로 | 음수나 0이 되는 것을 막는다 |

### 6.2 왜 v2는 `inactive_file`만 빼나

커널은 회수할 순서를 정하려고 페이지를 **LRU 목록**(Least Recently Used — 최근에 덜 쓴 것부터 버리기 위한 목록)으로 관리한다. 파일 캐시는 `active_file`(최근에 쓴 것)과 `inactive_file`(한동안 안 쓴 것)로 나뉜다. 회수는 inactive부터 일어난다. 그래서 "당장 버릴 수 있는 캐시"를 보수적으로 잡으려고 inactive만 뺀다. active 캐시까지 빼면 실제 압박을 과소평가할 수 있다.

v1에는 `cache` 하나로 집계된 값이 있었고 v2 `memory.stat`에는 `cache` 키가 없다. 그래서 **`cache` 키가 있으면 v1식, 없으면 v2식**으로 판별해도 결과가 같다. ECS agent는 설정 플래그로 고르지만 판별 결과는 같다.

### 6.3 실제 응답 예 (Docker stats 형식, 일부)

```json
"memory_stats": {
  "usage": 1048576000,
  "max_usage": 1258291200,
  "stats": {
    "cache": 314572800,
    "inactive_file": 524288000,
    "anon": 600000000
  },
  "limit": 9223372036854771712
}
```

- v1이면 `cache`가 있으므로 `1048576000 - 314572800`로 계산한다.
- `max_usage`는 v1에서만 오는 값(기록상 최대치)이다. v2에서는 빠질 수 있으므로 참고값으로만 쓴다.
- `limit`이 터무니없이 큰 값(사실상 무제한)인 경우가 있다. 컨테이너 단위 한도를 안 걸고 **태스크 단위**로만 한도를 건 경우다. 그래서 한도는 이 값이 아니라 [7장](#7-값은-어디서-읽나--ecs-태스크-메타데이터-v4)의 `Limits.Memory`에서 읽는다.

---

## 7. 값은 어디서 읽나 — ECS 태스크 메타데이터 v4

### 7.1 엔드포인트

AWS Fargate(플랫폼 버전 1.4.0 이상)는 각 컨테이너에 `ECS_CONTAINER_METADATA_URI_V4` 환경변수를 넣어 준다. 이 주소는 컨테이너 안에서만 닿는 로컬 HTTP 엔드포인트다.

| 경로 | 돌려주는 것 |
|---|---|
| `${ECS_CONTAINER_METADATA_URI_V4}/task` | 태스크 메타데이터. 태스크 단위 `Limits`(CPU·메모리 한도), 컨테이너 목록 등 |
| `${ECS_CONTAINER_METADATA_URI_V4}/task/stats` | 태스크 안 **모든 컨테이너**의 Docker stats. 컨테이너 ID → stats 객체의 맵 |
| `${ECS_CONTAINER_METADATA_URI_V4}/stats` | 이 컨테이너 하나의 Docker stats |

```
GET $ECS_CONTAINER_METADATA_URI_V4/task
→ { ..., "Limits": { "CPU": 0.5, "Memory": 2048 }, "Containers": [...] }
```

### 7.2 읽을 때 알아둘 것

1. **사이드카가 있으면 합산한다.** 한도는 태스크 단위인데 사용량은 컨테이너마다 나온다. 그래서 `/task/stats`의 모든 컨테이너 사용량을 더한다.
2. **값이 비어 있을 수 있다.** 공식 문서는 Fargate에서 컨테이너가 약 1초 돌아야 stats를 돌려준다고 적는다. 방금 뜬 컨테이너나 이미 멈춘 컨테이너는 stats가 `null`이거나 `memory_stats`가 비어 있을 수 있다. 읽을 수 있는 컨테이너만 더하고, 하나도 없으면 "값 없음"으로 둔다.
3. **`Limits`는 한도를 정의하지 않으면 빠진다**(문서: "This parameter is omitted if no resource limits are defined"). 그러면 비율을 만들지 않는다.
4. **단위**: 공식 응답 문서는 `Limits`의 메모리 단위를 명시하지 않는다. 태스크 정의의 메모리 값(MiB)과 같은 숫자가 나오는 것으로 보고 MiB로 다뤘지만, 운영에서 한 번 값을 보고 확인하는 게 안전하다.
5. **로컬에는 없다.** 환경변수가 없으면 호출하지 않고 컨테이너 필드만 빼면 된다. 힙 지표는 어디서든 나온다.

---

## 8. 비율을 만들 때의 함정

| 함정 | 결과 | 대응 |
|---|---|---|
| 분모가 0 | `x / 0.0` = `Infinity`, `0 / 0.0` = `NaN` | 분모가 없거나 0 이하면 비율 필드 자체를 빼기 |
| NaN·Infinity가 JSON에 들어감 | JSON 표준에 없는 값이라 직렬화기마다 다르게 쓰거나 실패한다. 지표 필터가 숫자로 못 읽는다 | 위와 같음 |
| MiB와 바이트 혼동 | 1024×1024배 틀린다 | 내부 계산은 바이트, 한도(MiB)는 바이트로 바꿔서 나누기 |
| MB와 MiB 혼동 | 약 5% 차이 | AWS 콘솔이 "Megabytes"라고 표시해도 Container Insights 메모리 지표의 실제 단위는 MiB라고 문서가 밝힌다 |
| "값 없음"을 0으로 기록 | 0%로 기록돼 알람이 영영 안 울린다 | 필드를 빼기. 지표 필터는 필드가 없는 로그를 건너뛴다 |

```kotlin
fun percent(value: Long?, limit: Long?): Double? {
    if (value == null || limit == null || limit <= 0) return null
    return Math.round(value * 1000.0 / limit) / 10.0   // 소수 1자리
}
```

---

## 9. 왜 요약만이 아니라 주기 표본도 남기나

잡이 끝날 때 한 줄만 남기면 깔끔하다. 문제는 **메모리 때문에 죽는 잡은 끝을 맞지 못한다**는 점이다.

- 커널 OOM(137)은 SIGKILL이다. 앱은 아무 코드도 실행하지 못한다([JVM 노트 2.3절](jvm-memory-and-container-limits.md#23-137이-특별한-이유--유언을-남길-수-없다)).
- `ExitOnOutOfMemoryError`로 죽어도(종료 코드 3) 정상 종료 경로를 타지 않는다.

정작 가장 알고 싶은 실행에서 요약 줄이 없는 셈이다. 그래서 **10초마다 표본**을 한 줄씩 남긴다. 죽기 직전까지의 추이가 로그에 남아, 사후에 "어느 쪽이 차오르고 있었나"를 볼 수 있다.

간격은 비용과 해상도의 절충이다. 짧으면 로그가 많아지고, 길면 짧은 잡은 표본이 한두 개뿐이다. 그래서 시작 즉시 1번, 그리고 잡 종료 때 1번을 더 찍는다.

언제 시작하고 언제 멈추게 했는지는 [Spring 노트](spring-boot-startup-lifecycle-and-events.md)에서 다룬다.

---

## 10. 로그를 지표로 — 구조화 로그와 지표 필터

### 10.1 구조화 로그

**구조화 로그**(structured log)는 사람이 읽을 문장 대신 기계가 파싱할 수 있는 키-값(보통 JSON)으로 남기는 로그다. SLF4J 2.x의 fluent API와 Spring Boot의 구조화 로그(ECS 형식 등)를 함께 쓰면 키-값이 JSON 최상위 필드로 나온다.

```kotlin
log.atInfo()
    .addKeyValue("event", "batch_resource_summary")
    .addKeyValue("heapAfterGcPeakPct", 31.0)        // 숫자로 넣으면 JSON 숫자로 나온다
    .addKeyValue("containerMemoryPeakPct", 39.5)
    .log("batch resource summary")
```

```json
{"@timestamp":"...","log.level":"INFO","message":"batch resource summary",
 "event":"batch_resource_summary","heapAfterGcPeakPct":31.0,"containerMemoryPeakPct":39.5}
```

### 10.2 지표 필터

CloudWatch Logs의 **지표 필터**(metric filter)는 들어오는 로그 중 패턴에 맞는 줄에서 값을 꺼내 CloudWatch 지표로 만든다. JSON 로그라면 이런 식이다.

```
필터 패턴:  { $.event = "batch_resource_summary" }
지표 값:    $.heapAfterGcPeakPct
```

그 지표에 "최댓값 > 60" 같은 알람을 건다.

### 10.3 필드 이름은 계약이다

코드에서 `heapAfterGcPeakPct`를 `heapPeakPct`로 바꾸면 어떻게 될까. 로그는 잘 나오고 테스트도 통과한다. 그런데 지표 필터가 더 이상 값을 못 찾아 **지표가 끊기고, 알람은 "데이터 없음" 상태로 조용히 멈춘다.** 에러는 어디에도 안 난다.

그래서 운영과 같은 인코더로 로그를 직렬화해 보고 **필드 이름과 숫자 타입을 고정하는 테스트**를 둔다. 이 테스트가 깨지면 지표 필터도 같이 바꿔야 한다는 신호다.

### 10.4 Container Insights가 있는데 왜 따로 남기나

ECS Container Insights에도 `MemoryUtilized`가 있다. 다만 차이가 있다.

| | Container Insights | 직접 남긴 로그 |
|---|---|---|
| 단위 | 태스크 정의 패밀리·서비스·클러스터 차원으로 집계(문서: 여러 태스크의 평균) | 잡 실행 1번 단위 |
| 힙 정보 | 없다 | GC 직후 힙까지 있다 |
| 잡 이름·상태 | 없다 | 같은 줄에 있다 |

배치처럼 여러 잡이 같은 태스크 정의를 쓰면, 집계 지표에서는 한 잡의 급등이 평균에 묻힐 수 있다(추정: 문서가 말하는 평균 집계에서 미루어 본 것). 잡 단위로 보려면 직접 남기는 게 낫다.

---

## 11. 임계값은 어떻게 정하나

정답 숫자는 없다. 원칙만 정리한다.

1. **실측 분포를 먼저 본다.** 평소 잡들의 GC 직후 힙 최댓값이 최대치의 몇 % 근처인지 본다. 임계값은 평소 값보다 충분히 위, 그러나 OOM보다는 한참 아래에 둔다.
2. **힙은 GC 직후 값이라 낮게 잡아도 된다.** 바닥선이 최대치의 60%라는 것은 살아 있는 객체만으로 60%를 차지한다는 뜻이다. 순간값 60%와는 무게가 다르다.
3. **컨테이너는 힙 상한만큼은 이미 예약된 셈이다.** 힙 상한을 컨테이너의 65%로 잡았다면, 힙이 커지는 것만으로도 컨테이너 사용률이 65%를 넘는다. 컨테이너 임계값은 그보다 위에 둬야 힙이 아닌 쪽(네이티브 누수)을 잡는 의미가 생긴다.
4. **처음에는 보수적으로 시작하고 조정한다.** 알람이 너무 자주 울리면 사람들이 무시하게 된다. 몇 주 운영해 보고 올리거나 내린다.

---

## 12. 측정이 잡을 막으면 안 된다 — HTTP timeout의 함정

측정 코드의 제1원칙은 **측정 대상(잡)을 망치지 않는 것**이다.

1. **모든 예외를 삼킨다.** 메타데이터 호출 실패, JSON 파싱 실패 등은 경고 로그 1회만 남기고 "값 없음"으로 처리한다.
2. **한 번만 경고한다.** 10초마다 실패하면 경고가 수백 줄 쌓인다.
3. **기다리는 시간에 상한을 둔다.** 잡 종료 시 마지막 표본을 찍는데, 그 호출이 멈추면 잡 종료 처리도 멈춘다.

3번에 함정이 있다. JDK 11+의 `HttpClient`는 요청에 `timeout`을 줄 수 있다.

```kotlin
val request = HttpRequest.newBuilder(uri).timeout(Duration.ofSeconds(1)).GET().build()
httpClient.send(request, HttpResponse.BodyHandlers.ofString())
```

Javadoc은 "응답을 그 시간 안에 받지 못하면 `HttpTimeoutException`"이라고만 적는다. 그런데 JDK 21 구현(`jdk.internal.net.http.MultiExchange`)을 보면 **응답 헤더를 받는 시점에 타이머를 해제**한다. 즉 서버가 헤더만 보내고 본문을 보내다 멈추면, 이 timeout은 더 이상 작동하지 않고 무한히 기다린다.

본문까지 포함해 제한하려면 비동기로 보내고 결과를 기다리는 쪽에 상한을 건다.

```kotlin
val future = httpClient.sendAsync(request, HttpResponse.BodyHandlers.ofString())
val response = try {
    future.get(1_000, TimeUnit.MILLISECONDS)   // 본문까지 1초. 넘으면 TimeoutException
} finally {
    future.cancel(true)                        // 남은 요청 정리
}
```

**왜 실전에서 중요한가**: 이런 버그는 평소에는 절대 드러나지 않는다. 메타데이터 서버가 비정상인 드문 순간에, 하필 잡 종료 경로를 붙잡는다. 로컬 HTTP 서버로 "헤더만 보내고 본문을 멈추는" 상황을 만들어 테스트로 재현할 수 있다.

---

## 참고

- JDK 21 `java.lang.management.MemoryPoolMXBean#getCollectionUsage`, `MemoryUsage#getMax` Javadoc (로컬 JDK `src.zip`에서 확인)
- JDK 21 `java.net.http.HttpRequest.Builder#timeout` Javadoc, `jdk.internal.net.http.MultiExchange` 소스 (헤더 수신 후 `cancelTimer()`)
- Linux kernel — Control Group v2 문서 (`memory.current`, `memory.stat`, `memory.max`): https://docs.kernel.org/admin-guide/cgroup-v2.html
- amazon-ecs-agent `agent/stats/utils_unix.go` `getMemUsage`: https://github.com/aws/amazon-ecs-agent/blob/master/agent/stats/utils_unix.go
- Amazon ECS task metadata endpoint v4 for Fargate (경로·응답 필드): https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-metadata-endpoint-v4-fargate.html
- Amazon ECS Container Insights metrics (`MemoryUtilized` 단위가 MiB라는 주석): https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-metrics-ECS.html
