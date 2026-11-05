# 배치 모니터를 붙이며 배우는 Spring Boot 기동 순서와 이벤트 리스너

> "잡 코드 시작점에서 로깅을 부르고, 10초마다 찍고, 끝나는 지점에서 또 부르는" 모양을 예상했는데, 실제 구현은 잡 코드를 한 줄도 건드리지 않았다. 그게 어떻게 가능한지 이해하려고 Spring Boot의 기동·종료 순서와 이벤트 리스너를 정리한 학습 문서.
> 계기: 회사 업무(배치 자원 사용 로그 추가). 여기서는 일반 Spring 개념과 최소 예제만 다룬다.
> 작성일: 2026-11-05
> 함께 읽기: 이 모니터가 무엇을 재는지는 [힙·컨테이너 메모리 지표 읽는 법](jvm-and-container-memory-metrics.md). 프록시·`@Transactional`은 [Spring 트랜잭션 노트 3장](spring-transaction-boundaries-and-batch.md#3-transactional의-실체--프록시와-전파).
> 확인 버전: Spring Boot 3.5.6, Spring Framework 6.2.11, Spring Batch 5.2.3, JDK 21 — 소스 jar와 공식 문서로 대조.

---

## 0. 이번 작업 한 문단 요약

배치 앱에 메모리 모니터 클래스 하나를 추가했다. 이 클래스에는 `@Component`가 붙어 있어 Spring이 기동할 때 알아서 객체를 만든다. 클래스 안의 세 메서드에는 `@EventListener`가 붙어 있다. Spring이 **기동을 마쳤을 때**(`ApplicationStartedEvent`) 첫 메서드가 불려 10초 주기의 **데몬 스레드**를 시작한다. Spring Boot의 잡 실행기가 **잡을 끝냈을 때**(`JobExecutionEvent`) 둘째 메서드가 불려 스레드를 멈추고 요약을 남긴다. **앱이 닫힐 때**(`ContextClosedEvent`) 셋째 메서드가 스레드가 남아 있으면 멈춘다. 잡 코드는 이 모니터의 존재를 모른다.

이 문단에 모르는 단어가 있으면 이 문서가 도움이 된다.

---

## 핵심 요약

- 내가 부르는 게 아니라 **프레임워크가 정해진 시점에 불러 준다**. 이것을 제어의 역전(IoC)이라고 하고, 이벤트 리스너는 그 대표적인 모양이다. → [1장](#1-직접-부르기와-불러-달라고-등록하기)
- `@Component`가 붙은 클래스는 컴포넌트 스캔으로 발견돼 **빈**(Spring이 만들고 관리하는 객체)이 된다. → [2장](#2-빈과-컨테이너--spring이-객체를-대신-만든다)
- `SpringApplication.run()`은 환경 준비 → 컨텍스트 생성 → refresh(빈 생성) → `ApplicationStartedEvent` → 러너 실행 → `ApplicationReadyEvent` 순서로 진행한다. → [3장](#3-springapplicationrun은-어떤-순서로-진행되나)
- `@EventListener` 메서드는 **파라미터 타입**(또는 애너테이션 속성)으로 받을 이벤트를 정한다. 다만 컨텍스트가 만들어지기 전 이벤트는 빈이 받을 수 없다. → [4장](#4-이벤트와-리스너--방송과-구독)
- 리스너는 기본적으로 **이벤트를 발행한 스레드에서 동기로** 실행된다. 리스너의 예외와 지연이 발행자에게 그대로 간다. → [5장](#5-리스너는-발행한-스레드에서-동기로-돈다)
- Spring Batch 잡은 Spring Boot의 `JobLauncherApplicationRunner`(러너의 하나)가 실행한다. 잡이 끝나면 `JobExecutionEvent`를 발행한다. → [6장](#6-배치-잡은-누가-실행하나--joblauncherapplicationrunner)
- `main`이 끝나고 **비데몬 스레드가 모두 끝나면** JVM이 종료를 시작한다. 이때 shutdown hook이 컨텍스트를 닫으며 `ContextClosedEvent`가 나온다. → [7장](#7-앱은-언제-어떻게-끝나나)
- 주기 작업은 `ScheduledExecutorService`로 돌린다. 한 번이라도 예외가 새면 **다음 실행이 조용히 멈춘다**. → [8장](#8-10초마다--scheduledexecutorservice)
- 이벤트 스레드와 표본 스레드가 같은 값을 만지므로 **잠금**이 필요하다. → [9장](#9-두-스레드가-같은-값을-만질-때)

## 면접 질문 체크리스트

- [ ] IoC(제어의 역전)를 예를 들어 설명할 수 있나?
- [ ] 빈은 무엇이고, `@Component`가 붙은 클래스는 어떻게 빈이 되나?
- [ ] Spring Boot 기동 이벤트의 순서는? `ApplicationStartedEvent`와 `ApplicationReadyEvent`의 차이는?
- [ ] `ApplicationRunner`는 언제 실행되나?
- [ ] `@EventListener`와 `ApplicationListener` 인터페이스의 차이는? `ApplicationStartingEvent`를 `@EventListener`로 받을 수 있나?
- [ ] 이벤트 리스너는 동기인가 비동기인가? 리스너에서 예외가 나면 어떻게 되나?
- [ ] 데몬 스레드와 일반 스레드의 차이는? JVM은 언제 종료되나?
- [ ] SIGTERM과 SIGKILL을 받았을 때 Spring 앱의 종료 처리는 어떻게 다른가?
- [ ] `scheduleAtFixedRate`와 `scheduleWithFixedDelay`의 차이는? 작업이 예외를 던지면?

## 질문 → 섹션

| 실제로 물어본 질문 | 섹션 |
|---|---|
| 배치가 시작되면서 이 로그는 어떻게 시작돼 실행되나 | [3장](#3-springapplicationrun은-어떤-순서로-진행되나) · [10장](#10-전체-타임라인-한-장) |
| 시작점·10초 간격·끝 지점에서 호출하는 코드를 생각했는데, 지금은 어떤 방법인가 | [1장](#1-직접-부르기와-불러-달라고-등록하기) · [4장](#4-이벤트와-리스너--방송과-구독) |
| 잡이 끝난 걸 모니터가 어떻게 아나 | [6장](#6-배치-잡은-누가-실행하나--joblauncherapplicationrunner) |
| 10초 반복은 누가 돌리나, 앱 종료를 막지는 않나 | [7장](#7-앱은-언제-어떻게-끝나나) · [8장](#8-10초마다--scheduledexecutorservice) |
| 이 방식 말고 다른 방법은 없었나 | [11장](#11-다른-방법과-비교) |

---

## 1. 직접 부르기와 "불러 달라고 등록하기"

처음 떠올린 모양은 이랬다.

```kotlin
fun runJob() {
    monitor.start()        // 시작점에서 호출
    doTheWork()            // (그동안 10초마다 찍힘)
    monitor.finish()       // 끝 지점에서 호출
}
```

틀린 설계는 아니다. 다만 잡이 열 개면 열 군데를 고쳐야 하고, 새 잡을 만들 때 빠뜨리기 쉽다.

실제 구현은 반대 방향이다. 모니터가 **"이런 일이 생기면 나를 불러 달라"고 등록**해 두고, Spring이 그 일이 생긴 순간에 부른다.

```kotlin
@Component
class Monitor {
    @EventListener(ApplicationStartedEvent::class)
    fun start() { /* ... */ }

    @EventListener
    fun finish(event: JobExecutionEvent) { /* ... */ }
}
```

호출하는 쪽(Spring)과 호출받는 쪽(모니터)의 관계가 뒤집혔다고 해서 **제어의 역전**(IoC — Inversion of Control)이라고 부른다. 우리가 흐름을 제어하지 않고 프레임워크가 제어하며, 우리는 끼어들 자리만 채운다. 라디오 방송을 구독하는 것과 비슷하다. 방송국(Spring)은 "기동 완료", "잡 종료" 같은 소식을 내보내고, 관심 있는 청취자(리스너)만 듣는다.

**왜 실전에서 중요한가**: 이 모양 덕분에 횡단 관심사(모니터링·감사 로그처럼 여러 기능에 걸치는 부가 기능)를 업무 코드와 완전히 분리할 수 있다. 파일 하나를 지우면 기능이 통째로 사라진다.

---

## 2. 빈과 컨테이너 — Spring이 객체를 대신 만든다

### 2.1 빈과 애플리케이션 컨텍스트

- **빈**(bean): Spring이 만들고 수명을 관리하는 객체.
- **애플리케이션 컨텍스트**(ApplicationContext): 빈들을 담고 서로 연결해 주는 그릇. "Spring 컨테이너"라고도 부른다. 이벤트를 발행하는 것도 이 컨텍스트다.

우리 코드에는 `Monitor()`를 직접 생성하는 곳이 없다. 그런데도 객체가 존재하는 이유는 컨텍스트가 만들어 주기 때문이다.

### 2.2 컴포넌트 스캔 — 누가 빈이 되나

`@SpringBootApplication`(또는 `@ComponentScan`)이 붙은 클래스의 패키지와 그 하위 패키지를 Spring이 훑는다. 그러다 `@Component`(그리고 `@Service`·`@Repository`·`@Configuration`처럼 `@Component`를 품은 애너테이션)가 붙은 클래스를 찾으면 빈으로 등록한다. 이것이 **컴포넌트 스캔**이다.

```kotlin
@SpringBootApplication              // 이 클래스의 패키지 아래를 스캔한다
class BatchApplication

fun main(args: Array<String>) {
    runApplication<BatchApplication>(*args)
}
```

모니터 클래스를 그 패키지 아래에 두기만 하면 등록된다. 다른 설정 파일을 고칠 필요가 없다.

### 2.3 생성자 주입

빈을 만들 때 생성자 파라미터는 Spring이 채운다(다른 빈이나 `@Value`로 지정한 설정값). 생성자가 하나면 그것을 쓰고, 여러 개면 `@Autowired`가 붙은 생성자를 쓴다. 테스트용 생성자(가짜 측정기를 넣는 것)와 운영용 생성자를 나눌 때 이 규칙을 쓴다.

```kotlin
@Component
class Monitor internal constructor(
    private val jobName: String,
    private val readHeap: () -> Long,          // 테스트에서는 가짜 함수를 넣는다
) {
    @Autowired                                  // Spring은 이 생성자를 쓴다
    constructor(@Value("\${spring.batch.job.name:}") jobName: String)
        : this(jobName, ::readRealHeap)
}
```

---

## 3. `SpringApplication.run()`은 어떤 순서로 진행되나

### 3.1 소스로 보는 큰 줄기

Spring Boot 3.5.6의 `SpringApplication.run()` 핵심부(요약):

```java
listeners.starting(...);                                   // ApplicationStartingEvent
ConfigurableEnvironment environment = prepareEnvironment(...); // ApplicationEnvironmentPreparedEvent
context = createApplicationContext();
prepareContext(...);                                       // ContextInitialized / Prepared 이벤트
refreshContext(context);                                   // ★ 빈 생성. 끝에 ContextRefreshedEvent
afterRefresh(context, applicationArguments);
listeners.started(context, ...);                           // ★ ApplicationStartedEvent
callRunners(context, applicationArguments);                // ★ ApplicationRunner·CommandLineRunner 실행
...
listeners.ready(context, ...);                             // ApplicationReadyEvent
```

### 3.2 이벤트 순서표

Spring Boot 공식 문서(Application Events and Listeners)의 순서를 표로 옮긴다.

| 순서 | 이벤트 | 언제 | 빈이 `@EventListener`로 받을 수 있나 |
|---|---|---|---|
| 1 | `ApplicationStartingEvent` | 실행 맨 처음 | 아니오 (컨텍스트가 아직 없다) |
| 2 | `ApplicationEnvironmentPreparedEvent` | 환경(설정값)이 준비됨, 컨텍스트 생성 전 | 아니오 |
| 3 | `ApplicationContextInitializedEvent` | 컨텍스트 준비, 빈 정의 로드 전 | 아니오 |
| 4 | `ApplicationPreparedEvent` | refresh 직전 | 아니오 |
| - | `ContextRefreshedEvent` | refresh 끝 (모든 싱글톤 빈 생성 완료) | 예 |
| 5 | `ApplicationStartedEvent` | refresh 뒤, **러너 실행 전** | 예 |
| 6 | `AvailabilityChangeEvent` (Liveness CORRECT) | 살아 있음 | 예 |
| 7 | `ApplicationReadyEvent` | **러너 실행 후** | 예 |
| 8 | `AvailabilityChangeEvent` (Readiness ACCEPTING_TRAFFIC) | 트래픽 받을 준비 | 예 |
| 9 | `ApplicationFailedEvent` | 기동 중 예외 | 경우에 따라 |

공식 문서는 컨텍스트 생성 전에 나오는 이벤트를 `@Bean`으로는 받을 수 없다고 적는다. 받으려면 `SpringApplication.addListeners(...)`로 넣거나 `META-INF/spring.factories`에 등록해야 한다.

### 3.3 왜 `ApplicationStartedEvent`를 골랐나

배치 앱에서는 **러너가 곧 잡 실행**이다([6장](#6-배치-잡은-누가-실행하나--joblauncherapplicationrunner)). 그래서 이벤트 선택에 따라 결과가 갈린다.

- `ApplicationStartedEvent`(러너 **전**): 잡이 돌기 직전에 표본을 시작한다. 이것을 골랐다.
- `ApplicationReadyEvent`(러너 **후**): 잡이 **이미 끝난 뒤**에 온다. 배치에서는 너무 늦다. 웹 서버에서는 "요청 받을 준비 완료"를 뜻하는 자연스러운 시점이라 이 차이를 놓치기 쉽다.

부팅 자체(refresh 구간)는 이 방식으로 재지 못한다는 한계도 여기서 나온다. 모니터 빈이 그 구간 안에서 만들어지기 때문이다.

---

## 4. 이벤트와 리스너 — 방송과 구독

### 4.1 구성 요소

| 역할 | 무엇 |
|---|---|
| 이벤트 | 일어난 일을 담은 객체. `ApplicationEvent`를 상속하거나 아무 객체나 될 수 있다 |
| 발행자 | `ApplicationEventPublisher.publishEvent(event)`. 애플리케이션 컨텍스트가 이 역할을 한다 |
| 멀티캐스터 | 발행된 이벤트를 받을 리스너들을 찾아 하나씩 호출한다 (`SimpleApplicationEventMulticaster`) |
| 리스너 | `ApplicationListener<E>` 구현체, 또는 `@EventListener` 메서드 |

### 4.2 받을 이벤트는 어떻게 정해지나

```kotlin
@EventListener                                   // 파라미터 타입 = 받을 이벤트
fun onJobEnd(event: JobExecutionEvent) { ... }

@EventListener(ApplicationStartedEvent::class)   // 파라미터가 없으면 애너테이션에 적는다
fun onStarted() { ... }
```

`@EventListener`는 `condition`(SpEL 조건식)으로 걸러 받을 수도 있고, `@Order`로 리스너끼리의 순서를 정할 수도 있다.

### 4.3 `@EventListener`는 어떻게 동작하나

`EventListenerMethodProcessor`가 모든 싱글톤 빈이 만들어진 직후(`afterSingletonsInstantiated`) 빈들을 훑는다. 그리고 `@EventListener` 메서드를 찾아 리스너 어댑터로 감싸 등록한다. 그래서 리스너는 refresh가 끝나갈 즈음부터 이벤트를 받기 시작한다. 3.2절 표에서 앞쪽 이벤트를 못 받는 이유가 이것이다.

### 4.4 같은 이벤트를 듣는 다른 리스너 — 실제 예

Spring Boot 안에도 `JobExecutionEvent`를 듣는 리스너가 이미 있다. `JobExecutionExitCodeGenerator`는 `ApplicationListener<JobExecutionEvent>`를 구현해, 잡 결과를 모아 두었다가 프로세스 종료 코드를 정하는 데 쓴다. 우리 모니터는 같은 방송을 듣는 두 번째 청취자일 뿐이다.

---

## 5. 리스너는 발행한 스레드에서 동기로 돈다

### 5.1 소스로 보는 실체

Spring Framework 6.2.11 `SimpleApplicationEventMulticaster`(요약):

```java
Executor executor = getTaskExecutor();      // 기본값 null
for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
    if (executor != null && listener.supportsAsyncExecution()) {
        executor.execute(() -> invokeListener(listener, event));
    } else {
        invokeListener(listener, event);     // ★ 기본: 발행자 스레드에서 바로 호출
    }
}

protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
    ErrorHandler errorHandler = getErrorHandler();   // 기본값 null
    if (errorHandler != null) { try { doInvoke(...) } catch (Throwable err) { errorHandler.handleError(err); } }
    else { doInvokeListener(listener, event); }      // ★ 예외가 발행자에게 그대로 전파된다
}
```

### 5.2 그래서 생기는 두 가지 결과

1. **리스너가 오래 걸리면 발행자도 멈춘다.** 공식 문서도 리스너가 기본적으로 같은 스레드에서 돌므로 오래 걸리는 작업을 하지 말라고 권한다. 우리 리스너는 스레드를 **시작만** 하고 곧바로 돌아온다. 실제 반복 작업은 별도 스레드가 한다.
2. **리스너의 예외가 발행자를 망칠 수 있다.** `JobExecutionEvent`는 잡 실행기가 발행한다. 모니터가 예외를 던지면 그 예외가 잡 실행기 쪽으로 올라간다. 그래서 모니터의 리스너 메서드는 **모든 예외를 안에서 잡고** 경고만 남긴다.

```kotlin
private inline fun runSafely(block: () -> Unit) {
    try { block() }
    catch (e: Exception) {
        if (e is InterruptedException) Thread.currentThread().interrupt()   // 중단 신호 복구
        warnOnce(e)
    }
}
```

비동기로 돌리고 싶으면 `@Async`(`@EnableAsync` 필요)를 붙일 수 있다. 다만 그러면 예외가 발행자에게 전파되지 않고, 리스너가 언제 끝나는지 발행자가 모른다. "잡 종료 직후 마지막 표본"처럼 순서가 중요하면 동기가 맞다.

---

## 6. 배치 잡은 누가 실행하나 — `JobLauncherApplicationRunner`

### 6.1 러너

`ApplicationRunner`·`CommandLineRunner`는 "기동이 끝나면 한 번 실행할 코드"를 담는 인터페이스다. 3.1절의 `callRunners()`가 빈으로 등록된 러너들을 찾아 실행한다.

### 6.2 Spring Boot의 배치 자동설정

Spring Boot 3.5.6 `BatchAutoConfiguration`은 `spring.batch.job.enabled`가 `true`(값이 없으면 `true`)일 때 `JobLauncherApplicationRunner` 빈을 등록한다. 이 러너가 하는 일은 다음과 같다.

1. `spring.batch.job.name`으로 지정한 잡을 찾는다. 잡이 여러 개인데 이름이 없으면 기동 실패다.
2. `jobLauncher.run(job, parameters)`로 잡을 실행한다.
3. 실행이 끝나면 `publishEvent(new JobExecutionEvent(execution))`로 결과를 방송한다.

```java
// JobLauncherApplicationRunner.execute (요약)
JobExecution execution = this.jobLauncher.run(job, parameters);
if (this.publisher != null) {
    this.publisher.publishEvent(new JobExecutionEvent(execution));
}
```

`JobExecutionEvent`에 담긴 `JobExecution`에는 잡 이름, 상태(COMPLETED·FAILED 등), 시작·종료 시각, 스텝별 읽기·쓰기 건수가 들어 있다. 모니터의 요약 로그가 이 정보를 그대로 쓴다.

### 6.3 잡은 어느 스레드에서 도나

Spring Batch 5.2.3 `TaskExecutorJobLauncher`는 실행기를 따로 지정하지 않으면 `SyncTaskExecutor`를 쓴다(로그: "No TaskExecutor has been set, defaulting to synchronous executor."). 즉 잡은 **러너를 부른 스레드(보통 main)에서 동기로** 돈다. `jobLauncher.run()`은 잡이 끝나야 반환한다. 그래서 "run 반환 → 이벤트 발행"이 곧 "잡 종료 → 이벤트 발행"이 된다.

### 6.4 이벤트가 오지 않는 경우

`jobLauncher.run()`이 예외를 던지면 그 아래 `publishEvent` 줄에 닿지 못한다. 잡 파라미터가 잘못됐거나 이미 완료된 인스턴스를 다시 돌리려 할 때가 그렇다. 잡 안에서 스텝이 실패하는 경우는 다르다. 그때는 예외가 아니라 FAILED 상태의 `JobExecution`이 반환되므로 이벤트는 정상적으로 온다. 이벤트가 안 오는 경로에서도 스레드는 멈춰야 하므로 [7장](#7-앱은-언제-어떻게-끝나나)의 종료 이벤트를 안전망으로 쓴다.

---

## 7. 앱은 언제, 어떻게 끝나나

### 7.1 데몬 스레드와 JVM 종료

JDK `Thread` 문서: 스레드는 **데몬**(daemon)과 **비데몬**으로 나뉜다. JVM은 `System.exit()`가 불리거나 **비데몬 스레드가 모두 끝나면** 종료를 시작한다. 데몬 스레드는 남아 있어도 종료를 막지 않는다.

배치 앱의 `main`은 `runApplication(...)`이 돌아오면 끝난다. 이때 비데몬 스레드가 하나라도 살아 있으면 JVM은 영영 안 끝난다. 잡은 다 끝났는데 컨테이너는 계속 떠 있는 상태가 된다. 그래서 모니터의 표본 스레드는 **데몬**으로 만든다.

```kotlin
Executors.newSingleThreadScheduledExecutor { runnable ->
    Thread(runnable, "resource-monitor").apply { isDaemon = true }   // 이름도 붙여 둔다(스레드 덤프에서 찾기 쉽게)
}
```

### 7.2 shutdown hook과 `ContextClosedEvent`

JVM이 종료를 시작하면 등록된 **shutdown hook**(종료 직전에 실행할 스레드)이 돈다. Spring Boot는 기본으로 hook을 등록하고, 그 hook이 컨텍스트를 닫는다. Spring Framework 6.2.11 `AbstractApplicationContext.doClose()`의 순서는 이렇다.

1. `publishEvent(new ContextClosedEvent(this))` — **빈들이 아직 살아 있을 때** 방송한다.
2. `lifecycleProcessor.onClose()` — `SmartLifecycle` 빈 정지.
3. `destroyBeans()` — `@PreDestroy`·`DisposableBean` 등 빈 파괴.

모니터는 1번에서 스레드를 멈춘다. 잡 종료 이벤트에서 이미 멈췄다면 아무것도 안 한다(두 번 불려도 안전하게 만든다).

### 7.3 신호별로 무엇이 실행되나

| 종료 원인 | shutdown hook | `ContextClosedEvent` | 모니터에 남는 것 |
|---|---|---|---|
| `main` 끝, 비데몬 스레드 없음 | 실행 | 나온다 | 표본 + 요약 |
| SIGTERM (오케스트레이터의 정지 요청) | 실행 | 나온다 | 그때까지의 표본(잡 도중이면 요약은 없음) |
| SIGKILL (커널 OOM, 강제 종료) | **실행 안 됨** | 안 나온다 | 그때까지의 표본만 |

이것이 [메모리 노트 9장](jvm-and-container-memory-metrics.md#9-왜-요약만이-아니라-주기-표본도-남기나)에서 주기 표본이 필요한 이유다.

---

## 8. 10초마다 — `ScheduledExecutorService`

### 8.1 두 가지 반복 방식

| 메서드 | 다음 실행 시점 |
|---|---|
| `scheduleAtFixedRate(task, initialDelay, period, unit)` | **시작 시각** 기준으로 period마다 (0, 10, 20초…) |
| `scheduleWithFixedDelay(task, initialDelay, delay, unit)` | 이전 실행이 **끝난 뒤** delay만큼 쉬고 |

표본은 시각이 일정한 편이 읽기 좋아서 `scheduleAtFixedRate(…, 0, 10초)`를 썼다. initialDelay 0이라 시작 즉시 1번 찍힌다. JDK 문서에 따르면 실행이 period보다 오래 걸려도 **다음 실행이 늦게 시작할 뿐 겹쳐 돌지는 않는다**.

### 8.2 함정 — 예외 한 번에 반복이 멈춘다

JDK 21 `ScheduledExecutorService` Javadoc: 작업 실행이 예외를 던지면 "Subsequent executions are suppressed"(이후 실행이 억제된다). 로그도 안 남는다. 예외는 반환된 `Future` 안에 들어가는데, 아무도 그 `Future`를 들여다보지 않는다. 메타데이터 호출이 한 번 실패했다고 표본이 영영 멈추는 셈이다.

그래서 반복 작업 본문을 통째로 감싼다.

```kotlin
executor.scheduleAtFixedRate({ runSafely { sample() } }, 0, 10_000, TimeUnit.MILLISECONDS)
```

### 8.3 멈추기

```kotlin
executor.shutdown()                                   // 새 실행 예약 중단
executor.awaitTermination(2_000, TimeUnit.MILLISECONDS) // 진행 중인 한 번이 끝나길 잠깐 기다림
```

`shutdown()`만 부르면 이미 돌고 있는 표본과 "마지막 표본"이 동시에 돌 수 있다. 짧게 기다려 순서를 정리한다. 상한을 두는 이유는 종료 경로를 붙잡지 않기 위해서다.

---

## 9. 두 스레드가 같은 값을 만질 때

이 모니터에는 스레드가 둘 관여한다.

- **표본 스레드**: 10초마다 최댓값·표본 수를 갱신한다.
- **이벤트 스레드**(main): 잡 종료 때 그 값을 읽어 요약을 쓰고, 실행기 참조를 비운다.

잠금 없이 두면 한쪽이 쓰는 도중 다른 쪽이 읽어 값이 어긋날 수 있다. 또 한 스레드가 쓴 값이 다른 스레드에 안 보일 수도 있다(메모리 가시성). 그래서 공유 상태를 읽고 쓰는 부분은 하나의 잠금(`synchronized(lock)`)으로 묶는다.

원칙 두 가지:

1. **느린 일은 잠금 밖에서 한다.** HTTP 호출·로그 출력처럼 오래 걸릴 수 있는 일을 잠금 안에서 하면 다른 스레드가 그만큼 기다린다. 값을 읽어 지역 변수로 복사한 뒤 잠금을 풀고 작업한다.
2. **기다리기도 잠금 밖에서 한다.** `awaitTermination`을 잠금 안에서 부르면, 끝나길 기다리는 표본 작업이 그 잠금을 기다리며 서로 막힌다(교착). 실행기 참조만 잠금 안에서 꺼내고, 기다리는 건 밖에서 한다.

```kotlin
private fun stopSampling() {
    val running = synchronized(lock) { executor.also { executor = null } } ?: return  // 꺼내기만
    running.shutdown()
    running.awaitTermination(2_000, TimeUnit.MILLISECONDS)                           // 기다리기는 밖에서
}
```

---

## 10. 전체 타임라인 한 장

```
[main 스레드]                                         [resource-monitor 데몬 스레드]
java -jar app.jar
 └ main() → SpringApplication.run()
    ├ 환경 준비 · 컨텍스트 생성
    ├ refresh: 컴포넌트 스캔 → 빈 생성 (Monitor 포함)
    │          @EventListener 등록
    ├ ApplicationStartedEvent ──▶ Monitor.start()
    │                               └ 데몬 스레드 시작 ──────▶ 표본 #1 (0초)
    ├ callRunners
    │  └ JobLauncherApplicationRunner                         표본 #2 (10초)
    │     └ jobLauncher.run(job)  ← 잡이 여기서 동기로 돈다    표본 #3 (20초)
    │     └ JobExecutionEvent ──▶ Monitor.finish()
    │                               ├ 스레드 정지 ───────────▶ (종료)
    │                               ├ 마지막 표본
    │                               └ 요약 로그 1줄
    ├ ApplicationReadyEvent
    └ main() 반환
비데몬 스레드 없음 → JVM 종료 시작 → shutdown hook
 └ context.close() → ContextClosedEvent ──▶ Monitor.onClosed() (이미 멈췄으면 아무것도 안 함)
```

---

## 11. 다른 방법과 비교

| 방법 | 어떻게 | 장점 | 대가 |
|---|---|---|---|
| 잡 코드에서 직접 호출 | 시작·끝에 `monitor.start()`/`finish()` | 흐름이 눈에 보인다 | 잡마다 넣어야 한다. 빠뜨리기 쉽다. 업무 코드가 섞인다 |
| `JobExecutionListener` | 잡 정의에 `.listener(monitor)` 등록 | Spring Batch 표준. `beforeJob`/`afterJob` 제공 | 잡 정의마다 등록해야 한다 |
| `@Scheduled` | `@EnableScheduling` + 메서드에 `@Scheduled(fixedRate=…)` | 코드가 짧다 | 앱 전역 스케줄러 설정에 묶인다. 잡 종료 시점과 맞춰 멈추기가 어렵다 |
| 앱 이벤트 + 자체 스케줄러 (이번 선택) | `@EventListener` 3개 + `ScheduledExecutorService` | 잡 코드 무수정, 새 잡 자동 적용, 시작·정지 시점을 직접 제어 | 부팅 구간은 못 잰다. 스레드·잠금을 직접 다뤄야 한다 |

"1 실행 = 1 잡"인 배치(컨테이너가 잡 하나 돌고 끝나는 구조)라서 앱 수명 이벤트를 잡 수명처럼 쓸 수 있었다. 한 프로세스가 여러 잡을 계속 돌리는 구조라면 `JobExecutionListener` 쪽이 더 맞다.

---

# 부록

## A.1 최소 재현 예제

Spring Boot + Spring Batch 프로젝트에 이 파일 하나만 넣으면 동작을 볼 수 있다.

```kotlin
import org.slf4j.LoggerFactory
import org.springframework.boot.autoconfigure.batch.JobExecutionEvent
import org.springframework.boot.context.event.ApplicationStartedEvent
import org.springframework.context.event.ContextClosedEvent
import org.springframework.context.event.EventListener
import org.springframework.stereotype.Component
import java.util.concurrent.Executors
import java.util.concurrent.ScheduledExecutorService
import java.util.concurrent.TimeUnit

@Component
class TickMonitor {
    private val log = LoggerFactory.getLogger(javaClass)
    private val lock = Any()
    private var executor: ScheduledExecutorService? = null
    private var ticks = 0

    @EventListener(ApplicationStartedEvent::class)
    fun start() = safely {
        synchronized(lock) {
            if (executor != null) return@safely
            executor = Executors.newSingleThreadScheduledExecutor { r ->
                Thread(r, "tick-monitor").apply { isDaemon = true }
            }.also { it.scheduleAtFixedRate({ safely { tick() } }, 0, 1, TimeUnit.SECONDS) }
        }
    }

    @EventListener
    fun onJobEnd(event: JobExecutionEvent) {
        safely { stop() }
        safely { log.info("job={} status={} ticks={}", event.jobExecution.jobInstance.jobName, event.jobExecution.status, synchronized(lock) { ticks }) }
    }

    @EventListener(ContextClosedEvent::class)
    fun onClosed() = safely { stop() }

    private fun tick() {
        val n = synchronized(lock) { ++ticks }
        log.info("tick {} on {}", n, Thread.currentThread().name)
    }

    private fun stop() {
        val running = synchronized(lock) { executor.also { executor = null } } ?: return
        running.shutdown()
        running.awaitTermination(2, TimeUnit.SECONDS)
    }

    private inline fun safely(block: () -> Unit) {
        try { block() } catch (e: Exception) {
            if (e is InterruptedException) Thread.currentThread().interrupt()
            log.warn("monitor failed: {}", e.toString())
        }
    }
}
```

실험해 볼 것:

1. 잡에 `Thread.sleep(5000)`을 넣고 실행한다. tick이 `tick-monitor` 스레드에서 1초마다 찍히고, 잡 로그는 `main`에서 찍히는지 본다.
2. `isDaemon = true`를 지우고 `stop()` 호출도 지워 본다. 잡이 끝나도 프로세스가 안 끝나는 것을 확인한다.
3. `tick()` 안에서 일부러 예외를 던지고 `safely`를 빼 본다. 첫 예외 뒤 tick이 조용히 멈추는 것을 확인한다.
4. `@EventListener(ApplicationReadyEvent::class)`로 바꿔 본다. tick이 잡이 끝난 뒤에야 시작되는 것을 확인한다.

---

## 참고

- Spring Boot 3.5 Reference — SpringApplication, Application Events and Listeners: https://docs.spring.io/spring-boot/3.5/reference/features/spring-application.html
- Spring Boot 3.5.6 소스: `SpringApplication#run`, `autoconfigure.batch.BatchAutoConfiguration`, `JobLauncherApplicationRunner#execute`, `JobExecutionExitCodeGenerator`
- Spring Framework 6.2.11 소스: `SimpleApplicationEventMulticaster#multicastEvent`·`invokeListener`, `EventListenerMethodProcessor`, `AbstractApplicationContext#doClose`, `@EventListener` Javadoc(비동기·순서)
- Spring Batch 5.2.3 소스: `TaskExecutorJobLauncher#afterPropertiesSet` (기본 `SyncTaskExecutor`)
- JDK 21 Javadoc: `java.lang.Thread`(데몬 스레드와 JVM 종료), `ScheduledExecutorService#scheduleAtFixedRate`(예외 시 이후 실행 억제, 겹치지 않음)
