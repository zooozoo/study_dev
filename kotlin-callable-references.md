# `::`로 함수를 값처럼 넘기기 — Kotlin 호출 가능 참조(callable reference) 배우기

Kotlin 코드에서 `::sessionUserType`, `String::length`, `Person::class.java`처럼 `::`가 붙은 표현이 무엇을 만드는지, 언제 람다 대신 쓰는지 정리한다. 핵심은 하나다. **`::`는 이미 있는 함수(또는 프로퍼티·생성자·클래스)를 "호출하지 않고 이름으로 가리켜서" 값으로 만든다.**

- 작성일: 2026-09-18
- 검증 환경: Kotlin 1.9.25 (JVM). 본문의 동작은 모두 [부록 A.2](#a2-실험-코드와-실제-출력) 실험으로 직접 확인했다.

## 계기 — 한 줄 요약

공통 모듈에 있는 WebSocket 인증 인터셉터(메시지가 처리되기 전에 끼어들어 검사하는 객체)가 **"로그인한 사용자를 어떤 유형으로 기록할지"를 모듈마다 다르게** 정해야 했다. 관리자용 서버는 토큰 안의 사용자 유형을, 일반 사용자용 서버는 고정값을 써야 했다. 그래서 공통 설정 클래스에 오버라이드 가능한 함수를 하나 두고, 그 함수를 인터셉터에 **함수 자체로** 넘겼다. 일반화한 모양은 이렇다.

```kotlin
abstract class BaseSocketConfig {
    protected abstract fun moduleRole(): String

    // 기본은 모듈 구분값. 필요한 모듈만 오버라이드한다.
    protected open fun sessionUserType(user: LoginUser): String = moduleRole()

    fun buildInterceptor() =
        AuthInterceptor(moduleRole(), ::sessionUserType)   // ← 이 `::`가 오늘의 주제
}

class AuthInterceptor(
    private val role: String,
    private val sessionUserType: (LoginUser) -> String,  // "LoginUser를 받아 String을 돌려주는 함수"
)
```

이 문단의 `(LoginUser) -> String`, `::sessionUserType`, "함수 자체로 넘긴다"가 낯설다면 이 문서가 도움이 된다.

## 이 공부를 시작하게 한 질문들

| 질문 | 답이 있는 곳 |
|---|---|
| `::`는 대체 무엇을 만드는가? 함수를 호출하는 것과 뭐가 다른가? | [1장](#1-함수도-값이다--함수-타입과-람다), [2장](#2-는-이미-있는-함수를-이름으로-가리킨다) |
| `::foo`, `Type::foo`, `obj::foo`, `::Type`은 각각 뭐가 다른가? | [3장](#3-의-네-가지-모양) |
| `String::length`는 왜 인자를 하나 받는 함수가 되나? | [4장](#4-수신자를-묶느냐-마느냐--unbound와-bound) |
| 클래스 안에서 `this` 없이 `::sessionUserType`만 써도 되나? 하위 클래스가 오버라이드하면 어느 쪽이 불리나? | [5장](#5-클래스-안의-member는-thismember다--계기-코드-해석) |
| `protected` 함수를 다른 클래스에 넘겨도 괜찮은가? | [5.3절](#53-protected-함수도-넘길-수-있다--권한-대신-능력을-건넨다) |
| 이름이 같은 함수가 여러 개면 어느 걸 가리키나? | [6장](#6-같은-이름이-여러-개면--기대-타입으로-고른다) |
| 기본 인자가 있는 함수를 더 적은 인자의 함수 타입에 넘길 수 있나? | [7장](#7-kotlin-14의-적응adaptation--인자-위치에서만-된다) |
| 람다로 쓸까, `::`로 쓸까? | [8장](#8-람다냐-참조냐--고르는-기준) |
| `Foo::class.java`의 `::`도 같은 것인가? | [9장](#9-class--같은-기호의-다른-쓰임) |
| Java의 `String::length`와 같은 문법인가? | [10장](#10-java-메서드-참조와-비교) |

---

## 1. 함수도 값이다 — 함수 타입과 람다

Kotlin에서는 함수를 숫자나 문자열처럼 **변수에 담고, 인자로 넘기고, 반환**할 수 있다. 이걸 가능하게 하는 것이 **함수 타입**이다.

| 함수 타입 표기 | 뜻 |
|---|---|
| `() -> Unit` | 인자 없음, 돌려주는 값 없음(`Unit`은 Java의 `void`에 해당) |
| `(Int) -> Boolean` | `Int` 하나를 받아 `Boolean`을 돌려주는 함수 |
| `(String, Int) -> Person` | `String`과 `Int`를 받아 `Person`을 돌려주는 함수 |

함수 타입의 값을 만드는 가장 흔한 방법은 **람다(lambda — 이름 없이 그 자리에서 쓰는 짧은 함수)**다.

```kotlin
val isEvenLambda: (Int) -> Boolean = { n -> n % 2 == 0 }
listOf(1, 2, 3, 4).filter(isEvenLambda)   // [2, 4]
```

`filter`는 "각 원소를 받아 남길지 말지 알려주는 함수"를 인자로 받는다. 그 함수를 넘겨야 하는 자리에 람다를 넣은 것이다.

**왜 실전에서 중요한가**: 계기 코드의 `AuthInterceptor`는 "사용자 유형을 계산하는 방법"을 생성자에서 **함수로** 받는다. 인터셉터는 계산 방법을 모르고, 넘겨받은 함수만 호출한다. 그래서 모듈마다 다른 계산 방법을 끼워 넣을 수 있다(전략 패턴을 함수 하나로 표현한 것).

## 2. `::`는 이미 있는 함수를 이름으로 가리킨다

넘기고 싶은 함수가 **이미 이름을 가진 함수로 존재**하면, 람다로 한 번 더 감쌀 필요 없이 `::이름`으로 가리키면 된다.

```kotlin
fun isEven(n: Int): Boolean = n % 2 == 0

listOf(1, 2, 3, 4).filter(::isEven)               // [2, 4]
listOf(1, 2, 3, 4).filter { n -> isEven(n) }      // 같은 결과, 람다로 감싼 버전
```

차이를 분명히 하자.

| 표현 | 의미 | 결과 |
|---|---|---|
| `isEven(3)` | **호출** — 지금 실행한다 | `false` (Boolean 값) |
| `::isEven` | **참조** — 실행하지 않고 함수 자체를 가리킨다 | `(Int) -> Boolean` 타입의 값 |

`::isEven`은 괄호가 없다. 호출하지 않았기 때문이다. 이렇게 `::`로 만든 값을 **호출 가능 참조(callable reference)**라고 부른다. 함수뿐 아니라 프로퍼티·생성자도 가리킬 수 있어서 "함수 참조"보다 넓은 이름을 쓴다.

## 3. `::`의 네 가지 모양

`::` 앞에 무엇이 오느냐로 모양이 나뉜다.

| 모양 | 예 | 만들어지는 함수 타입 | 설명 |
|---|---|---|---|
| `::최상위함수` | `::isEven` | `(Int) -> Boolean` | 클래스 밖에 선언된 함수 |
| `타입::멤버` (unbound) | `String::length`, `Person::name` | `(String) -> Int`, `(Person) -> String` | **어느 객체의 것인지 정하지 않은** 멤버. 객체가 첫 번째 인자로 들어온다([4장](#4-수신자를-묶느냐-마느냐--unbound와-bound)) |
| `객체::멤버` (bound) | `greeter::greet`, `"abc"::length` | 객체가 이미 정해졌으므로 나머지 인자만 | **특정 객체에 묶인** 멤버 |
| `::클래스이름` (생성자) | `::Person` | `(String, Int) -> Person` | 생성자를 함수처럼 가리킨다 |

프로퍼티도 같은 방식이다. `Person::age`는 `(Person) -> Int`처럼 쓸 수 있다.

```kotlin
data class Person(val name: String, val age: Int)
val people = listOf(Person("kim", 30), Person("lee", 25))

people.map(Person::name)                 // [kim, lee]
people.sortedBy(Person::age)             // lee(25), kim(30) 순

val create: (String, Int) -> Person = ::Person
create("jung", 40)                       // Person(name=jung, age=40)
```

## 4. 수신자를 묶느냐 마느냐 — unbound와 bound

**수신자(receiver)**는 멤버 함수를 부를 때 점 앞에 오는 객체다. `greeter.greet("park")`에서 수신자는 `greeter`다.

### 4.1 unbound — 수신자가 첫 번째 인자로 남는다

`Greeter::greet`처럼 **타입** 이름을 앞에 쓰면, 어느 `Greeter`인지 아직 정해지지 않았다. 그래서 수신자를 **첫 번째 인자로 받는** 함수가 된다.

```kotlin
class Greeter(private val prefix: String) {
    fun greet(name: String): String = "$prefix, $name"
}

val g: (Greeter, String) -> String = Greeter::greet
g(Greeter("Yo"), "choi")                 // "Yo, choi"
```

`String::length`가 `(String) -> Int`인 이유도 같다. 인자로 들어온 문자열이 곧 수신자다.

### 4.2 bound — 수신자가 미리 묶인다

**객체**를 앞에 쓰면 수신자가 그 객체로 고정되고, 수신자는 인자 목록에서 사라진다.

```kotlin
var greeter = Greeter("Hi")
val g = greeter::greet        // (String) -> String — 수신자는 이미 정해졌다
g("park")                     // "Hi, park"
```

### 4.3 묶이는 시점은 "참조를 만든 순간"이다

bound 참조는 만들 때의 객체를 붙잡는다. 나중에 변수가 다른 객체를 가리켜도 참조는 따라가지 않는다.

```kotlin
var greeter = Greeter("Hi")
val g = greeter::greet
greeter = Greeter("Bye")      // 변수만 바뀌었다
g("park")                     // "Hi, park"  ← 여전히 처음 객체
greeter.greet("park")         // "Bye, park"
```

**왜 실전에서 중요한가**: 참조를 필드에 저장해 두는 객체(계기 코드의 인터셉터처럼)는 **만들 때 묶인 객체를 계속 쓴다**. 설정 객체를 바꿔 끼우는 구조라면 이 점을 알고 있어야 한다.

## 5. 클래스 안의 `::member`는 `this::member`다 — 계기 코드 해석

### 5.1 `this`를 생략할 수 있다

Kotlin 공식 문서에 따르면 `this::foo`와 `::foo`는 같은 뜻이다. 즉 클래스 안에서 `::sessionUserType`이라고 쓰면 **지금 이 객체(`this`)에 묶인 bound 참조**가 된다. 그래서 수신자가 인자에서 빠지고, 타입은 `(LoginUser) -> String`이 되어 생성자 파라미터 타입과 정확히 맞는다.

```kotlin
fun buildInterceptor() =
    AuthInterceptor(moduleRole(), ::sessionUserType)
//                                 └ this::sessionUserType, 타입 (LoginUser) -> String
```

람다로 풀어 쓰면 `{ user -> this.sessionUserType(user) }`와 같다.

### 5.2 오버라이드한 쪽이 불린다(가상 디스패치)

`sessionUserType`이 `open`이고 하위 클래스가 오버라이드했다면, 참조로 넘겨도 **실제 객체의 오버라이드 버전**이 불린다. 참조가 "`Base`에 적힌 코드"가 아니라 "`this` 객체의 `sessionUserType` 멤버"를 가리키기 때문이다. 이렇게 실행 시점의 실제 객체 타입을 보고 호출할 구현을 고르는 것을 **가상 디스패치(virtual dispatch)**라고 한다.

```kotlin
abstract class Base {
    protected open fun label(p: Person): String = "base:${p.name}"
    fun makeFormatter() = Formatter(::label)
}
class Child : Base() {
    override fun label(p: Person): String = "child:${p.name}"
}
class Formatter(private val labelOf: (Person) -> String) {
    fun format(p: Person) = "[" + labelOf(p) + "]"
}

object : Base() {}.makeFormatter().format(Person("kim", 30))   // "[base:kim]"
Child().makeFormatter().format(Person("kim", 30))              // "[child:kim]"
```

계기 코드가 바로 이 구조다. 공통 클래스는 `::sessionUserType`을 넘기기만 하고, 관리자용 모듈만 `sessionUserType`을 오버라이드한다. 인터셉터는 어느 모듈에서 도는지 몰라도 올바른 값을 얻는다.

### 5.3 `protected` 함수도 넘길 수 있다 — 권한 대신 능력을 건넨다

위 예의 `label`은 `protected`다. `Formatter`는 `Base`의 하위 클래스가 아니므로 `base.label(p)`를 직접 부를 수 없다. 그런데 `Base`가 **자기 안에서** 만든 참조 `::label`을 받으면 호출할 수 있다(실험으로 확인). 접근 검사는 참조를 **만드는 곳**에서 한 번 이뤄지고, 그 뒤에는 함수 타입 값일 뿐이기 때문이다.

**왜 실전에서 중요한가**: 함수를 `public`으로 열지 않고도 필요한 협력 객체 하나에만 "이 계산을 할 수 있는 능력"을 건넬 수 있다. 공개 범위를 넓히지 않는 설계 수단이다.

## 6. 같은 이름이 여러 개면 — 기대 타입으로 고른다

오버로드(이름은 같고 파라미터가 다른 함수 여러 개)가 있으면, 컴파일러는 **그 자리에서 기대하는 함수 타입**을 보고 하나를 고른다.

```kotlin
fun parse(s: String): Int = s.toInt()
fun parse(s: String, radix: Int): Int = s.toInt(radix)

val p1: (String) -> Int = ::parse          // 첫 번째 parse
val p2: (String, Int) -> Int = ::parse     // 두 번째 parse
p1("10")        // 10
p2("ff", 16)    // 255

val p = ::parse // 컴파일 오류: Overload resolution ambiguity
```

기대 타입이 없으면 어느 것인지 정할 수 없어 컴파일 오류가 난다. 해결책은 변수 타입을 명시하거나, 함수 타입 파라미터에 바로 넘기는 것이다.

## 7. Kotlin 1.4의 적응(adaptation) — 인자 위치에서만 된다

Kotlin 1.4부터 참조의 모양이 기대 타입과 정확히 같지 않아도 컴파일러가 맞춰 준다(공식 1.4 변경사항 문서).

| 적응 종류 | 예 | 뜻 |
|---|---|---|
| 기본 인자 | `fun add(a: Int, b: Int = 100)` → `(Int) -> Int` | 빠진 인자는 기본값으로 채운다 |
| `Unit` 반환 | `list::add`(Boolean 반환) → `(Int) -> Unit` | 반환값을 버린다 |
| `vararg` | `fun foo(x: Int, vararg y: String)` → `(Int) -> Unit`, `(Int, String) -> Unit` 등 | 가변 인자 개수를 맞춘다 |
| `suspend` 변환 | 일반 함수 → `suspend () -> Unit` | 코루틴용 함수 타입에 넘긴다 |

```kotlin
fun applyTo(x: Int, f: (Int) -> Int): Int = f(x)
applyTo(1, ::add)                          // 101 — 기본값 b=100 이 채워졌다

val sink = mutableListOf<Int>()
listOf(1, 2, 3).forEach(sink::add)         // add는 Boolean을 돌려주지만 forEach는 (T) -> Unit을 기대
```

**주의 — 실험으로 확인한 함정**: 이 적응은 **함수 인자로 넘길 때만** 된다. 같은 참조를 **변수에 대입**하면 Kotlin 1.9.25에서 컴파일 오류가 난다.

```kotlin
val addOne: (Int) -> Int = ::add
// e: Type mismatch: inferred type is KFunction2<Int, Int, Int> but (Int) -> Int was expected

val unitSink: (Int) -> Unit = sink::add
// e: None of the following functions can be called with the arguments supplied
```

변수에 담아야 한다면 람다로 감싸면 된다: `val addOne: (Int) -> Int = { add(it) }`.

## 8. 람다냐 참조냐 — 고르는 기준

| 상황 | 추천 | 이유 |
|---|---|---|
| 넘길 함수가 이미 있고 인자를 그대로 전달만 한다 | `::참조` | 짧고, 함수 이름이 의도를 드러낸다 |
| 인자를 가공하거나 순서를 바꾸거나 값을 추가한다 | 람다 | 참조는 인자를 손댈 수 없다 (`{ user -> format(user, "KR") }`) |
| 여러 줄의 로직이 필요하다 | 람다 또는 새 이름 있는 함수를 만들어 참조 | 긴 람다보다 이름 있는 함수가 읽기 쉽다 |
| 오버로드가 많아 모호하다 | 기대 타입 명시 또는 람다 | [6장](#6-같은-이름이-여러-개면--기대-타입으로-고른다) |
| 기본 인자·`Unit` 적응이 필요한데 변수에 담아야 한다 | 람다 | [7장](#7-kotlin-14의-적응adaptation--인자-위치에서만-된다) |
| 하위 클래스가 바꿀 수 있는 동작을 협력 객체에 넘긴다 | `::openMember` | 가상 디스패치가 그대로 유지된다([5.2절](#52-오버라이드한-쪽이-불린다가상-디스패치)) |

## 9. `::class` — 같은 기호의 다른 쓰임

`::class`는 함수가 아니라 **클래스 자체를 나타내는 객체**를 얻는다.

| 표현 | 결과 | 흔한 용도 |
|---|---|---|
| `Person::class` | `KClass<Person>` (Kotlin의 클래스 정보 객체) | Kotlin 리플렉션(reflection — 실행 중에 클래스 구조를 들여다보는 기능) |
| `Person::class.java` | `Class<Person>` (Java의 클래스 정보 객체) | Java 라이브러리에 클래스를 넘길 때. 예: `LoggerFactory.getLogger(MyService::class.java)` |
| `person::class` | 그 객체의 실제 클래스 | 런타임 타입 확인 |

로거 선언에서 자주 보는 `Foo::class.java`의 `::`도 "호출하지 않고 가리킨다"는 같은 발상이다. 다만 가리키는 대상이 함수가 아니라 클래스다.

## 10. Java 메서드 참조와 비교

Java 8의 메서드 참조도 `::`를 쓴다. 발상은 같고 표기가 조금 다르다.

| 하려는 것 | Kotlin | Java |
|---|---|---|
| 정적/최상위 함수 | `::isEven` | `Util::isEven` (Java에는 최상위 함수가 없다) |
| 타입의 멤버(unbound) | `String::length` | `String::length` |
| 특정 객체의 멤버(bound) | `greeter::greet` | `greeter::greet` |
| 현재 객체의 멤버 | `this::foo` 또는 `::foo` | `this::foo` (`this` 생략 불가) |
| 생성자 | `::Person` | `Person::new` |
| 프로퍼티 | `Person::age` | 없음 (getter 메서드 `Person::getAge`) |
| 결과의 타입 | 함수 타입 `(Int) -> Boolean` (실제로는 `KFunction1`) | 함수형 인터페이스 (`Predicate<Integer>` 등) — 대입 대상이 정한다 |

---

## 부록

### A.1 참조의 실제 타입 — `KFunction`과 `KProperty`

`::isEven`의 정확한 타입은 `KFunction1<Int, Boolean>`이다. 공식 문서 기준으로 함수 참조는 파라미터 개수에 따라 `KFunction<out R>`의 하위 타입이 되고, 이 타입은 함수 타입 `(Int) -> Boolean`으로도 쓸 수 있다. 그래서 함수 타입 파라미터에 바로 넘길 수 있다.

프로퍼티 참조는 `KProperty`다. 값을 읽고 이름을 얻을 수 있다.

```kotlin
::isEven.name                       // "isEven"
Person::age.name                    // "age"
Person::age.get(Person("lee", 25))  // 25
```

같은 객체·같은 멤버로 만든 bound 참조는 `==` 비교에서 같다고 나왔다(`gg::greet == gg::greet` → `true`). 이 실험은 `kotlin-reflect` 의존성을 넣은 상태에서 실행했다.

### A.2 실험 코드와 실제 출력

Kotlin 1.9.25, JDK 21, `kotlin("jvm")` + `application` 플러그인으로 실행했다.

```kotlin
fun isEven(n: Int): Boolean = n % 2 == 0
data class Person(val name: String, val age: Int)

class Greeter(private val prefix: String) {
    fun greet(name: String): String = "$prefix, $name"
}

abstract class Base {
    protected open fun label(p: Person): String = "base:${p.name}"
    fun makeFormatter(): Formatter = Formatter(::label)
}
class Child : Base() {
    override fun label(p: Person): String = "child:${p.name}"
}
class Formatter(private val labelOf: (Person) -> String) {
    fun format(p: Person) = "[" + labelOf(p) + "]"
}

fun parse(s: String): Int = s.toInt()
fun parse(s: String, radix: Int): Int = s.toInt(radix)
fun add(a: Int, b: Int = 100): Int = a + b
fun applyTo(x: Int, f: (Int) -> Int): Int = f(x)

fun main() {
    val nums = listOf(1, 2, 3, 4)
    println("1 top-level: " + nums.filter(::isEven))
    val people = listOf(Person("kim", 30), Person("lee", 25))
    val lenRef: (String) -> Int = String::length
    println("2 unbound String::length: " + lenRef("hello"))
    println("2 Person::name: " + people.map(Person::name))
    println("2 sortedBy(Person::age): " + people.sortedBy(Person::age).map(Person::name))

    var greeter = Greeter("Hi")
    val g = greeter::greet
    greeter = Greeter("Bye")
    println("3 bound captured at creation: " + g("park") + " / new: " + greeter.greet("park"))
    val g2: (Greeter, String) -> String = Greeter::greet
    println("3 unbound member takes receiver first: " + g2(Greeter("Yo"), "choi"))

    println("4/5 base: " + object : Base() {}.makeFormatter().format(people[0]))
    println("4/5 child: " + Child().makeFormatter().format(people[0]))

    val ctor: (String, Int) -> Person = ::Person
    println("6 constructor ref: " + ctor("jung", 40))

    val p1: (String) -> Int = ::parse
    val p2: (String, Int) -> Int = ::parse
    println("7 overload by expected type: " + p1("10") + " " + p2("ff", 16))

    println("8 default-arg adaptation (argument position): " + applyTo(1, ::add))

    val sink = mutableListOf<Int>()
    nums.forEach(sink::add)
    println("11 Unit coercion: " + sink)

    println("9 ::class: " + Person::class.simpleName + " / java: " + Person::class.java.name)
    val gg = Greeter("x")
    println("12 equality bound refs: " + (gg::greet == gg::greet))
    println("13 KFunction name: " + ::isEven.name + ", KProperty: " + Person::age.name + "=" + Person::age.get(people[1]))
}
```

출력:

```
1 top-level: [2, 4]
2 unbound String::length: 5
2 Person::name: [kim, lee]
2 sortedBy(Person::age): [lee, kim]
3 bound captured at creation: Hi, park / new: Bye, park
3 unbound member takes receiver first: Yo, choi
4/5 base: [base:kim]
4/5 child: [child:kim]
6 constructor ref: Person(name=jung, age=40)
7 overload by expected type: 10 255
8 default-arg adaptation (argument position): 101
11 Unit coercion: [1, 2, 3, 4]
9 ::class: Person / java: Person
12 equality bound refs: true
13 KFunction name: isEven, KProperty: age=25
```

컴파일 오류로 확인한 것(각각 한 줄씩 넣어 컴파일):

| 넣은 줄 | 결과 |
|---|---|
| `val p = ::parse` | `Overload resolution ambiguity` |
| `val addOne: (Int) -> Int = ::add` | `Type mismatch: inferred type is KFunction2<Int, Int, Int> but (Int) -> Int was expected` |
| `val unitSink: (Int) -> Unit = sink::add` | `None of the following functions can be called with the arguments supplied` |

### A.3 참고 문서

- Kotlin 공식 문서 — Reflection 중 "Callable references" (함수·프로퍼티·bound·생성자 참조, `this::foo`와 `::foo`가 같다는 설명): https://kotlinlang.org/docs/reflection.html
- Kotlin 1.4 변경사항 — callable reference 개선(기본 인자·`Unit`·`vararg`·`suspend` 적응): https://kotlinlang.org/docs/whatsnew14.html
