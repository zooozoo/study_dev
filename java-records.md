# 괄호 한 줄이 클래스가 되는 이유 — Java record 문법 배우기

Java 코드에서 `record Point(int x, int y) { ... }`처럼 생긴 선언을 읽는 법을 정리한다. 필드 선언도 생성자도 안 보이는데 `new Point(3, 4)`가 되고 `p.x()`가 되는 이유, 그리고 언제 record를 쓰고 언제 못 쓰는지를 다룬다. 핵심은 하나다. **record 이름 뒤 괄호에 적은 목록이 곧 필드 목록이고, 컴파일러가 그 목록으로 생성자·조회 메서드·`equals`·`hashCode`·`toString`을 만들어 준다.**

- 작성일: 2026-11-11
- 검증 환경: OpenJDK 21.0.5. 본문의 동작은 [부록 A](#부록-a-실험-코드와-실제-출력) 실험으로 직접 확인했다.

## 계기 — 한 줄 요약

외부 API 응답을 해석하는 코드에서 아래 같은 줄을 만났다. 응답 JSON의 바깥 칸(요청 ID·상태 코드 등)을 꺼내 작은 객체 하나에 담는 코드다. 일반화한 모양은 이렇다.

```java
private record Envelope(String requestId, Integer status, JsonNode body) { }

// 다른 메서드 안에서
return new Envelope(text(root.get("requestId")), intValue(root.get("status")), root.path("body"));
```

`Envelope` 안에 필드 선언이 하나도 없는데 `new Envelope(...)`로 값을 넣고, 다른 곳에서 `env.requestId`로 꺼내 쓴다. "값이 어느 칸에 들어가는지 어디에 적혀 있나?"가 궁금했다면 이 문서가 도움이 된다.

## 핵심 요약

1. record 헤더(이름 뒤 괄호)의 각 항목을 **컴포넌트(component)**라 부르며, 각 컴포넌트가 `private final` 필드 하나가 된다. → [1장](#1-괄호-안의-목록이-곧-필드-목록이다)
2. 컴파일러가 컴포넌트 순서 그대로 인자를 받는 **정식 생성자(canonical constructor)**를 만든다. `new`에 넘긴 n번째 값이 n번째 컴포넌트에 들어간다. → [2장](#2-new에-넘긴-값은-순서대로-칸에-들어간다)
3. 값을 꺼내는 메서드는 `getX()`가 아니라 컴포넌트 이름 그대로인 `x()`다. → [3장](#3-꺼낼-때는-getx가-아니라-x다)
4. `equals`·`hashCode`·`toString`도 자동으로 생기며, `equals`는 모든 컴포넌트 값이 같으면 참이다. → [4장](#4-값이-같으면-같은-객체로-본다--equals와-tostring)
5. 값 검증·정규화는 괄호 없는 **compact 생성자**에 쓴다. 필드 대입은 컴파일러가 마지막에 해 준다. → [5장](#5-검증은-compact-생성자에-쓴다)
6. record는 `final`이고 필드는 다시 대입할 수 없으며 인스턴스 필드를 더 둘 수 없다. 다만 담긴 객체 내부까지 불변으로 만들지는 않는다. → [6장](#6-record가-못-하는-것--제약과-얕은-불변)
7. 바깥 클래스 안에 둔 record는 자동으로 `static`이고, 바깥 클래스 코드는 `private` 필드에 바로 접근할 수 있다. → [7장](#7-클래스-안에-둔-record--중첩-record와-private-접근)

## 면접 질문 체크리스트

각 질문을 소리 내어 설명해 보고, 막히면 연결된 본문에서 근거를 확인한다.

- [ ] record 헤더에 적은 항목은 컴파일 후 무엇이 되는가? → [1장](#1-괄호-안의-목록이-곧-필드-목록이다)
- [ ] `new Point(3, 4)`에서 3이 `x`에 들어간다는 것은 무엇으로 정해지는가? → [2장](#2-new에-넘긴-값은-순서대로-칸에-들어간다)
- [ ] record의 `equals`는 무엇을 비교하는가? `==`와는 어떻게 다른가? → [4장](#4-값이-같으면-같은-객체로-본다--equals와-tostring)
- [ ] compact 생성자와 정식 생성자를 직접 쓰는 것은 무엇이 다른가? → [5장](#5-검증은-compact-생성자에-쓴다)
- [ ] record가 불변이라고 할 때 그 범위는 어디까지인가? → [6장](#6-record가-못-하는-것--제약과-얕은-불변)
- [ ] JPA 엔티티를 record로 만들기 어려운 이유는? → [8장](#8-언제-record를-쓰나)

---

## 이 공부를 시작하게 한 질문들

| 질문 | 답이 있는 곳 |
|---|---|
| 필드 선언이 없는데 값은 어디에 저장되나? | [1장](#1-괄호-안의-목록이-곧-필드-목록이다) |
| `new Envelope(a, b, c)`의 세 번째 값은 어느 칸에 들어가나? | [2장](#2-new에-넘긴-값은-순서대로-칸에-들어간다) |
| `env.requestId()`와 `env.requestId`는 둘 다 되나? | [3장](#3-꺼낼-때는-getx가-아니라-x다), [7장](#7-클래스-안에-둔-record--중첩-record와-private-접근) |
| record 본문 `{ }` 안에는 무엇을 쓸 수 있나? | [5장](#5-검증은-compact-생성자에-쓴다), [6장](#6-record가-못-하는-것--제약과-얕은-불변) |

---

## 1. 괄호 안의 목록이 곧 필드 목록이다

record는 "값 몇 개를 묶어 나르는 객체"를 짧게 쓰기 위한 클래스 문법이다. JEP 395(JDK Enhancement Proposal — Java 기능 제안서)로 Java 16에서 정식 기능이 됐다.

```java
record Point(int x, int y) { }
```

- `Point` 뒤 괄호 `(int x, int y)`를 **record 헤더**, 그 안의 `int x`와 `int y` 하나하나를 **컴포넌트**라 부른다.
- 컴포넌트 하나가 `private final` 필드 하나가 된다. 본문 `{ }`에 필드를 따로 쓰지 않는다.

컴파일된 클래스를 `javap -p`(클래스 파일의 멤버 목록을 보여주는 JDK 도구)로 열어 보면 컴파일러가 만든 것이 그대로 보인다([부록 A](#부록-a-실험-코드와-실제-출력) 실측).

```text
final class Point extends java.lang.Record {
  private final int x;
  private final int y;
  Point(int, int);
  public final java.lang.String toString();
  public final int hashCode();
  public final boolean equals(java.lang.Object);
  public int x();
  public int y();
}
```

**왜 중요한가:** record를 처음 보면 "필드가 어디 있지?"에서 막힌다. 답은 언제나 헤더다. 칸 이름·타입·순서를 알고 싶으면 헤더 한 줄만 읽으면 된다.

## 2. new에 넘긴 값은 순서대로 칸에 들어간다

컴파일러는 컴포넌트와 같은 순서·타입으로 인자를 받는 **정식 생성자**를 만든다. 풀어 쓰면 아래와 같다.

```java
Point(int x, int y) {
    this.x = x;
    this.y = y;
}
```

그래서 대응은 **순서**로 정해진다. 이름으로 대응하지 않는다.

| `new Envelope(...)`의 인자 | 들어가는 컴포넌트 |
|---|---|
| 1번째 `text(root.get("requestId"))` | `String requestId` |
| 2번째 `intValue(root.get("status"))` | `Integer status` |
| 3번째 `root.path("body")` | `JsonNode body` |

인자 자리의 식(`text(...)`, `intValue(...)`)은 생성자가 불리기 전에 먼저 계산되고, 그 결과값이 해당 칸에 들어간다.

**왜 중요한가:** 타입이 같은 컴포넌트가 이웃해 있으면(예: `String from, String to`) 순서를 바꿔 넘겨도 컴파일 오류가 나지 않는다. 컴포넌트가 많아지면 이런 실수가 쉬워지므로 생성 지점을 읽을 때 헤더 순서와 맞춰 보는 습관이 필요하다.

## 3. 꺼낼 때는 getX()가 아니라 x()다

각 컴포넌트마다 같은 이름의 **접근자(accessor) 메서드**가 생긴다. `getX()`가 아니라 `x()`다.

```java
Point p = new Point(3, 4);
p.x();   // 3
p.y();   // 4
```

- 접근자는 `public`이다(위 `javap` 출력의 `public int x();`).
- 필드는 `private`이라 바깥에서 `p.x`로 직접 읽을 수는 없다. 예외는 [7장](#7-클래스-안에-둔-record--중첩-record와-private-접근)에서 다룬다.

**왜 중요한가:** JavaBeans 관례(`getX()`)를 기대하는 라이브러리·도구가 record의 `x()`를 못 알아보는 경우가 있다. record를 직렬화·바인딩에 쓰기 전에 그 라이브러리가 record를 지원하는지 확인해야 한다.

## 4. 값이 같으면 같은 객체로 본다 — equals와 toString

record는 `equals`·`hashCode`·`toString`도 만든다.

```java
Point a = new Point(3, 4);
Point b = new Point(3, 4);

a.equals(b);                      // true  — 모든 컴포넌트 값이 같다
a == b;                           // false — 서로 다른 객체다
a.hashCode() == b.hashCode();     // true
a.toString();                     // "Point[x=3, y=4]"
```

- `equals`는 모든 컴포넌트를 비교한다. 이런 객체를 **값 객체(value object)**라 부른다. 담긴 값이 정체성이다.
- `==`는 여전히 "같은 객체인가"를 본다. record라고 바뀌지 않는다.
- `toString`은 `이름[컴포넌트=값, ...]` 모양이다.

**왜 중요한가:** 일반 클래스는 `equals`를 직접 쓰지 않으면 `==`와 같게 동작한다. 그래서 `HashMap` 키나 `Set` 원소로 쓸 때 실수가 잦다. record는 이 실수가 없다. 반대로 `toString`이 모든 값을 찍으므로, 비밀번호·개인정보 같은 값을 record에 담아 로그에 찍으면 그대로 노출된다. 그런 record는 `toString`을 직접 다시 정의해 값을 가린다.

## 5. 검증은 compact 생성자에 쓴다

생성 시점에 값을 검사하거나 다듬고 싶으면 **compact 생성자**를 쓴다. 괄호와 매개변수 목록을 생략한 생성자다.

```java
record Point(int x, int y) {
    Point {                                     // 괄호 없음
        if (x < 0) throw new IllegalArgumentException("x < 0");
    }
}
```

- 본문이 끝난 뒤 컴파일러가 `this.x = x; this.y = y;`를 자동으로 붙인다. 그래서 대입을 직접 쓰지 않는다.
- 본문 안에서 매개변수에 새 값을 넣으면 그 값이 필드에 들어간다. 정규화에 쓴다.

```java
record Tags(List<String> values) {
    Tags { values = List.copyOf(values); }      // 바깥 리스트를 복사해 둔다
}
```

정식 생성자를 매개변수까지 다 적어 직접 쓸 수도 있다. 그때는 `this.x = x;` 대입을 모두 직접 써야 한다. 다른 모양의 생성자를 추가하는 것도 된다. 다만 결국 `this(...)`로 정식 생성자를 불러야 한다.

```java
record Point(int x, int y) {
    Point(int x) { this(x, 0); }                // 추가 생성자는 정식 생성자로 위임
}
```

본문에는 생성자 말고도 일반 메서드와 `static` 메서드·`static` 필드를 둘 수 있다.

**왜 중요한가:** 검증을 생성자 한 곳에 두면 "잘못된 값을 가진 record"가 아예 만들어지지 않는다. 호출하는 쪽마다 검사를 반복하지 않아도 된다.

## 6. record가 못 하는 것 — 제약과 얕은 불변

JDK 21에서 직접 컴파일해 확인한 제약이다([부록 A](#부록-a-실험-코드와-실제-출력)).

| 하려는 것 | 결과 |
|---|---|
| 본문에 인스턴스 필드 추가 (`private int extra;`) | 컴파일 오류 `field declaration must be static` |
| 메서드에서 필드 재대입 (`this.x = 1;`) | 컴파일 오류 `cannot assign a value to final variable x` |
| record를 상속 (`class B extends Base`) | 컴파일 오류 `cannot inherit from final Base` |
| 다른 클래스를 상속 (`record R(...) extends X`) | 불가. record는 이미 `java.lang.Record`를 상속한다 |
| 인터페이스 구현 (`record R(...) implements Comparable<R>`) | 가능 |

**얕은 불변(shallow immutability):** record는 필드를 다시 대입하지 못하게 막을 뿐이다. 필드가 가리키는 객체의 내부까지 막지는 않는다. 예를 들어 `record Tags(List<String> values)`에 바깥에서 만든 `ArrayList`를 그대로 넣으면, 바깥에서 그 리스트에 원소를 추가할 때 record 안의 값도 바뀐다. 그래서 [5장](#5-검증은-compact-생성자에-쓴다)처럼 `List.copyOf`로 복사해 두는 패턴이 쓰인다. 실험에서는 복사 후 바깥 리스트에 `"b"`를 넣어도 record 값은 `[a]`로 남았다.

**왜 중요한가:** "record = 불변"으로만 기억하면 리스트·배열·`JsonNode` 같은 변경 가능한 객체를 담을 때 버그가 생긴다. 불변이 보장되는 범위는 "어느 객체를 가리키는가"까지다.

## 7. 클래스 안에 둔 record — 중첩 record와 private 접근

record는 다른 클래스 안에 둘 수 있다. 이를 **중첩 record(nested record)**라 한다.

```java
public class Parser {
    private record Envelope(String requestId, Integer status) { }

    String describe(Envelope env) {
        return env.requestId + "/" + env.status();   // 필드 직접 접근도, 접근자도 된다
    }
}
```

- 중첩 record는 자동으로 `static`이다. 바깥 클래스의 인스턴스를 붙잡지 않는다. 실험에서 `Modifier.isStatic`이 `true`였다.
- Java에서는 같은 최상위 클래스 안의 코드끼리 서로의 `private` 멤버에 접근할 수 있다. 그래서 바깥 클래스 `Parser`의 메서드는 `env.requestId`처럼 `private` 필드를 바로 읽을 수 있다.
- record 자체를 `private`으로 두면 바깥 클래스 밖에서는 이 타입을 쓸 수 없다. 메서드 사이에서 값을 묶어 넘기는 내부 전용 상자로 쓰기 좋다.

**왜 중요한가:** 계기 코드의 `env.requestId`가 이 경우다. 바깥에서 쓰는 record였다면 `env.requestId()`로 접근해야 한다.

## 8. 언제 record를 쓰나

| 잘 맞는 곳 | 이유 |
|---|---|
| 요청·응답 DTO(Data Transfer Object — 계층 사이에 값을 나르는 객체) | 값만 담고 동작이 거의 없다 |
| 메서드 안의 중간 결과 묶음(계기 코드의 `Envelope`) | 여러 값을 한 번에 넘기고 돌려받는다 |
| `Map` 키, `Set` 원소 | `equals`·`hashCode`가 값 기준으로 이미 맞다 |
| 조회 결과 투영(projection — 쿼리 결과 중 일부 칸만 담는 객체) | 읽기 전용 값 묶음이다 |

| 맞지 않는 곳 | 이유 |
|---|---|
| JPA 엔티티 | JPA 구현체는 보통 인자 없는 생성자와 프록시(상속으로 만든 대리 객체)를 쓰는데, record는 `final`이고 필드를 바꿀 수 없다 |
| 상태가 바뀌어야 하는 객체 | 필드 재대입이 불가능하다. 바뀐 값은 새 record로 만들어야 한다 |
| 상속 계층이 필요한 타입 | record는 다른 클래스를 상속할 수도, 상속될 수도 없다 |

Java 21에서는 **record 패턴**으로 값을 바로 꺼낼 수도 있다. 실험에서 `o instanceof Point(int x, int y)`가 `x=3 y=4`를 꺼냈다.

```java
if (o instanceof Point(int x, int y)) {
    System.out.println(x + y);
}
```

---

## 부록 A. 실험 코드와 실제 출력

OpenJDK 21.0.5에서 `javac Demo.java && java Demo && javap -p 'Demo$Point.class'`로 실행했다.

```java
import java.util.*;

public class Demo {
    private record Point(int x, int y) {
        Point {
            if (x < 0) throw new IllegalArgumentException("x < 0");
        }
        Point(int x) { this(x, 0); }
        static Point origin() { return new Point(0, 0); }
        int sum() { return x + y; }
    }

    record Tags(List<String> values) {
        Tags { values = List.copyOf(values); }
    }

    public static void main(String[] a) {
        Point p = new Point(3, 4);
        System.out.println(p.x() + " " + p.x + " " + p.sum());
        System.out.println(p);
        System.out.println(p.equals(new Point(3, 4)) + " " + (p == new Point(3, 4))
                + " " + (p.hashCode() == new Point(3, 4).hashCode()));
        System.out.println(new Point(5));
        try { new Point(-1, 0); } catch (IllegalArgumentException e) { System.out.println("rejected: " + e.getMessage()); }
        List<String> src = new ArrayList<>(List.of("a"));
        Tags t = new Tags(src);
        src.add("b");
        System.out.println(t.values());
        System.out.println(Point.class.isRecord() + " "
                + java.lang.reflect.Modifier.isFinal(Point.class.getModifiers()) + " "
                + java.lang.reflect.Modifier.isStatic(Point.class.getModifiers()) + " "
                + Point.class.getSuperclass().getName());
        Object o = p;
        if (o instanceof Point(int x, int y)) System.out.println("pattern x=" + x + " y=" + y);
    }
}
```

출력:

```text
3 3 7
Point[x=3, y=4]
true false true
Point[x=5, y=0]
rejected: x < 0
[a]
true true true java.lang.Record
pattern x=3 y=4
Compiled from "Demo.java"
final class Demo$Point extends java.lang.Record {
  private final int x;
  private final int y;
  Demo$Point(int, int);
  Demo$Point(int);
  static Demo$Point origin();
  int sum();
  public final java.lang.String toString();
  public final int hashCode();
  public final boolean equals(java.lang.Object);
  public int x();
  public int y();
}
```

제약 확인용 파일과 컴파일 오류(파일별로 `javac` 실행):

```text
record Bad1(int x) { private int extra; }
→ error: field declaration must be static

record Bad2(int x) { void set() { this.x = 1; } }
→ error: cannot assign a value to final variable x

record Base(int x) {}
class Bad3 extends Base { Bad3() { super(1); } }
→ error: cannot inherit from final Base
```
