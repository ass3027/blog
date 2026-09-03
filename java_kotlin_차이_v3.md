# Java vs Kotlin

“자바와 코틀린의 차이가 뭐냐? 코드 스타일 빼고”라는 질문을 최근에 들었다. 나는 코틀린을 “자바보다 훨씬 편한 코드 스타일을 제공하고, 연산자 오버로딩처럼 라이브러리로 구현할 법한 기능도 언어 차원에서 지원한다” 정도로만 단순하게 인식하고 있었기에 제대로 답하지 못했다.

<br/>

그래서 이 글은 주제별로 나누고, 각 주제에서 문법과 컴파일 결과를 함께 본다. 기준은 Kotlin/JVM과 Java 16+다.

<br/>

| 주제 | Java | Kotlin |
| --- | --- | --- |
| **null 처리** | 타입에 null 가능 여부 없음<br>`if`, `Optional`, 애너테이션으로 관리 | `T` / `T?`로 nullability 구분<br>Java 경계에서 계약 검사 코드 생성 |
| **데이터 클래스** | 일반 클래스 또는 Lombok<br>record(Java 16+) | `data class`<br>`copy()`, `componentN()` 자동 생성 |
| **재할당 제어** | `final`로 재할당 금지 | `val` / `var`를 선언에서 명시 |
| **기능 확장** | 정적 유틸리티 메서드<br>래퍼·Decorator 패턴 | 확장 함수 문법<br>수신 객체를 첫 인자로 받는 정적 메서드로 컴파일 |
| **함수 사용** | 람다 + 함수형 인터페이스<br>`invokedynamic` 기반 | 함수 타입 `(T) -> R`<br>`inline`으로 람다 인라이닝 가능 |
| **비동기** | `ExecutorService`, `CompletableFuture`<br>JDK 21+ virtual thread | `suspend` 함수<br>컴파일러가 상태 머신으로 변환 |
| **런타임 의존성** | JDK/JRE 표준 라이브러리 중심 | Kotlin 표준 라이브러리 필요<br>coroutine은 별도 의존성 |

## 1. null 계약

Kotlin의 가장 체감되는 장점은 null 가능 여부를 타입에 표현한다는 점이다. `String`에는 `null`을 대입할 수 없고, `String?`만 `null`을 가질 수 있다. nullable 값의 멤버에 바로 접근하면 컴파일 오류가 발생하므로, 호출하는 쪽에서 null 처리 방식을 선택해야 한다.

```kotlin
val name: String? = getName()

val length = name?.length ?: 0
```

주로 사용하는 연산자는 안전 호출 `?.`와 Elvis 연산자 `?:`다. `!!`는 “null이 아님을 내가 보장한다”는 단언이며, 값이 실제로 null이면 NPE를 발생시키므로 주의해서 사용해야 한다.

<br/>

Java에서 null 참조로 메서드를 호출하거나 필드에 접근하면 JVM이 `NullPointerException`을 던진다. Java의 타입만으로는 해당 참조가 null일 수 있는지를 구분하지 못하기 때문에, 팀 규칙·애너테이션·정적 분석 도구·`Optional` 등을 함께 사용해 관리한다.

<br/>

여기서 중요한 건, Kotlin의 검사가 **컴파일 시점에서 끝나지 않는다**는 점이다. Kotlin 컴파일러는 Java와의 상호 운용 과정에서 non-null 매개변수 계약을 지키기 위해 `Intrinsics.checkNotNullParameter()` 같은 검사 호출을 바이트코드에 생성할 수 있다. Java 쪽에서 넘어오는 값은 컴파일러도 통제할 수 없으니, Null이 발생하는 지점을 최대한 빠르게 잡으려는 것이다.(항상 그런건 아님)

<br/>

정리하면 Kotlin null safety의 핵심은 null 계약을 타입에 드러내고, nullable 값의 사용 지점을 **컴파일 시점**에 확인하는 데 있다. `!!`, 초기화 문제, Java에서 넘어온 platform type처럼 타입 시스템 밖에서 들어오는 값은 더욱 주의해야 한다.

&nbsp;

> **Platform type**
Platform type은 Java의 타입을 Kotlin에서 사용할 때 자동으로 변환되는 타입이다. Java의 `String`은 Kotlin의 `String!`으로 변환된다.

## 2. 데이터 클래스: data class와 record

Kotlin의 `data class`는 데이터를 담는 클래스를 짧게 만들 때 유용하다.

```kotlin
data class User(val name: String, var age: Int)
```

Java도 Java 16부터 record를 제공한다.

```java
public record User(String name, int age) {}
```

둘 다 접근자와 값 기반 `equals`, `hashCode`, `toString`을 자동으로 제공한다. 따라서 단순 DTO나 값 객체에서는 과거보다 차이가 작아졌다. 다만 자동 생성 범위와 불변성에 대한 제약은 다르다.

<br/>

| 비교 항목 | Kotlin `data class` | Java `record` |
| --- | --- | --- |
| 자동 생성 | `equals()`, `hashCode()`, `toString()`, `componentN()`, `copy()` | `equals()`, `hashCode()`, `toString()`, 구성 요소 접근자 |
| 구조 분해 | `componentN()`을 이용해 지원 | 언어 차원의 구조 분해 문법 없음 |
| 복사 | `copy()`로 일부 프로퍼티만 바꾼 새 객체 생성 가능 | 필요한 구성 요소를 지정해 새 객체를 직접 생성 |
| 구성 요소의 가변성 | 주 생성자 프로퍼티를 `val` 또는 `var`로 선언 가능 | 모든 구성 요소가 `final` |
| 값 비교 대상 | 주 생성자에 선언한 프로퍼티 | record header에 선언한 모든 구성 요소 |
| 상속·구현 | data class 자체는 상속할 수 없지만, 다른 클래스를 상속하거나 인터페이스를 구현할 수 있음 | 암묵적으로 `final`이며 다른 클래스는 상속할 수 없지만, 인터페이스는 구현할 수 있음 |

&nbsp;

>구조 분해는 객체의 프로퍼티를 여러 변수에 한 번에 꺼내는 문법이다. `data class`가 자동 생성하는 `componentN()` 덕분에 다음처럼 쓸 수 있다.
&nbsp;

```
val user = User("Min", 20)
val (name, age) = user
```

이는 `user.component1()`과 `user.component2()`를 각각 호출하는 것과 같다. Java record는 `user.name()`과 `user.age()` 같은 접근자를 제공하지만, 이 `componentN()` 기반 문법은 없다.

<br/>

Kotlin의 `data class`는 `var`도 허용하므로 반드시 불변 객체는 아니다. 반면 Java record는 구성 요소를 바꿀 수 없도록 제약한다. 다만 record의 구성 요소가 가리키는 객체까지 깊게 불변인 것은 아니므로, 가변 컬렉션 등을 담을 때는 별도 설계가 필요하다.

## 3. val과 var

`val`은 참조를 다시 대입할 수 없고, `var`는 다시 대입할 수 있다. Java의 `final`도 같은 수준에서 재할당을 막는다.

```kotlin
val users = mutableListOf("A")
users.add("B") // 가능: 참조는 고정이지만 객체 내부는 가변
```

하지만 Java에선 final이 선택의 영역이지만 Kotlin은 모든 변수에 `val`/`var` 선택이 필수이다. 문법에 의해 강제되기 때문에 불변 객체를 쉽게 구분할 수 있다.

## 4. 확장 함수

Java에서는 보통 `StringUtils` 같은 유틸리티 클래스에 정적 메서드를 둔다. Kotlin은 이를 수신 객체의 멤버처럼 호출하는 문법을 제공한다.

```kotlin
//StringUtils.kt
fun String.lastChar(): Char = this[length - 1]

val result = "Hello".lastChar()
```

JVM에서 실제 `String`에 메서드를 추가하는 건 아니고, 위 함수가 `StringUtils.kt` 최상위에 선언되었다면, 대략 다음과 같은 정적 메서드가 된다.

```java
public final class StringUtilsKt {
    public static final char lastChar(String receiver) {
        return receiver.charAt(receiver.length() - 1);
    }
}

char result = StringUtilsKt.lastChar("Hello");
```

즉 Java의 util 클래스와 **바이트코드 수준에서는 같은 것**이고, 다른 건 호출 문법뿐이다. 그래서 호출 대상은 선언된 수신 타입을 기준으로 컴파일 시점에 결정되고(정적 디스패치), 같은 시그니처의 멤버 함수가 있으면 멤버 함수가 우선된다. 확장 함수는 오버라이드되지 않는다.
<br/>

확장 함수의 가치는 기존 타입을 수정하지 않고도 도메인 문맥에 맞는 호출 문법을 제공한다는 데 있다. 성능이 중요한 지점에서는 실제 병목을 측정해 판단하면 된다.
<br/>

이를 통해 `method chaining`을 더 쉽게 활용할 수 있고, 코드의 가독성을 더 더욱 높일 수 있다. `operator override`와 함께 내가 kotlin에서 좋아하는 문법 중 하나이다.

```java
String target = "sejinLee";
char whatIWant = StringUtil.getLastName(target).getCharAt(0);
```

```kotlin
val target = "sejinLee"
val whatIWant = target.getLastName().first()
```

## 5. 고차 함수와 inline

Java와 Kotlin 모두 객체 지향과 함수형 스타일을 함께 지원하는 `다중 패러다임` 언어다. Java에서는 함수형 인터페이스와 람다, Stream API를 조합하고, JVM이 실행 시점에 연결하는 방식으로 구현된다. 캡처하지 않는 람다는 함수 객체를 재사용할 수 있고, 캡처하는 람다는 값을 보관하기 위해 새 객체가 필요하다.

> **캡쳐**는 바깥 범위의 값을 사용하는 것을 의미한다. 여기서는 람다에서 외부 변수를 참조를 하는지를 말한다.
또한 캡쳐하는 람다도 JVM 최적화를 통해 특정 상황에선 재사용 가능하다.

&nbsp;
<br/>

Kotlin은 함수 타입(`(T) -> R`)이 언어에 포함되어 있고, 확장 함수·고차 함수와 함께 사용할 수 있어 컬렉션 처리나 DSL을 더 간결하게 작성할 수 있다. 따라서 **'Java도 되지만 Kotlin이 함수형 스타일을 더 잘 표현하게 해 준다'**고 이해하는 편이 정확하다.

<br/>

Kotlin에서도 고차 함수에 전달되는 람다는 함수 객체와 간접 호출 비용을 만들 수 있다. `inline` 함수는 함수 본문과 inlinable 람다를 호출 지점에 삽입해 이 비용을 줄일 수 있다.

<br/>

```kotlin
inline fun <T> measure(block: () -> T): T {
    return block()
}
```

하지만 `inline`시 코드 크기가 늘어나기에 **본문이 단순하고, 성능이 중요한 기능에서 많이 호출되는 경우**에 쓰는 것이 권장되고, `noinline`으로 표시된 람다는 인라인되지 않는다.

## 6. coroutine과 suspend

여기는 Java에 1:1로 대응하는 기능이 없는 영역이다. Kotlin coroutine은 `suspend`와 **구조화된 동시성**을 중심으로 한 비동기 작업 모델이고, Java virtual thread는 JVM이 제공하는 가벼운 스레드다.

<br/>

핵심은 `suspend`가 단순한 코드 스타일이 아니라, 컴파일러가 중단·재개 가능한 코드로 변환하는 언어 기능이라는 점이다. 스케줄링·취소·구조화 같은 실행 정책은 `kotlinx-coroutines`가 맡는다.

<br/>

> **구조화된 동시성**은 비동기 작업도 일반 함수 호출처럼, 시작한 코드 범위 안에서 끝나게 하는 규칙이다. 코드 블록의 중첩 구조와 coroutine의 생명주기 구조를 맞춘다고 생각하면 된다.

```kotlin
suspend fun loadPage(): Page = coroutineScope {
    val user = async { loadUser() }
    val orders = async { loadOrders() }

    Page(user.await(), orders.await())
}
```

- `loadPage()`는 두 작업이 끝난 뒤에만 반환한다. 한 작업이 실패하면 다른 작업도 취소되고 실패가 호출자에게 전달된다.
- `loadPage()`를 실행하던 coroutine이 취소되면 내부 작업도 함께 취소된다.

&nbsp;

구현 원리와 Java virtual thread와의 자세한 비교는 다른 글에서 다루겠다..!!

## 7. 언어 밖의 차이

- **런타임 의존성.**   
Java는 대부분 JDK 표준 라이브러리만으로 실행할 수 있다. Kotlin/JVM 애플리케이션은 Kotlin 표준 라이브러리가 필요하며, coroutine의 `launch`, `async`, `Flow` 등을 사용하면 `kotlinx-coroutines` 의존성도 추가된다.

- **최상위 선언.**   
Java에서는 함수와 상수가 반드시 클래스 안에 있어야 한다. 그래서 담을 곳이 필요 없는 유틸리티도 `StringUtils` 같은 클래스를 만들고 `static` 메서드를 채워 넣는다. Kotlin은 파일 최상위에 바로 선언할 수 있어서 이 껍데기 클래스가 필요 없다.

```kotlin
// Example.kt
const val VERSION = "1.0"

fun greet(name: String) = "Hello, $name"
```

물론 JVM에는 클래스 밖의 함수라는 개념이 없다. 그래서 컴파일러가 파일 이름을 딴 클래스를 대신 만들어 정적 메서드로 넣는다. 위 코드를 Java에서 쓰면 `ExampleKt.greet("Min")`이 되고, `@file:JvmName("Greetings")`을 붙이면 `Greetings.greet("Min")`이 된다.

<br/>

정리하면 차이는 **껍데기 클래스를 개발자가 만드느냐, 컴파일러가 만드느냐**다. 확장 함수와 같은 이야기이고, Kotlin 쪽에서는 보이지 않던 클래스 이름이 Java에서 호출할 때 다시 드러난다.

## 마무리

웬만하면 하나의 글로 정리하고 싶었지만, 설명하려면 하위 개념이 나오고 또 그 하위 개념을 설명해야 해서 끝이 없다. 하지만 이걸 안 하면 이해도, 설명도 안 된다. 견뎌라.

<br/>

처음 질문으로 돌아가면, “코드 스타일 빼고”라는 단서는 사실 나눠 볼 필요가 있었다. data class와 record처럼 현대 Java와 Kotlin의 겹치는 범위가 넓어진 것도 있고, null 계약이나 `val`/`var`처럼 스타일로 보이지만 실은 타입 시스템이 강제하는 것도 있으며, `suspend`처럼 애초에 스타일의 문제가 아닌 것도 있다.

<br/>

coroutine의 더 파고들 주제는 따로 정리해서 이 글에도 링크를 걸어 오겠다. 그때 다시 만나요~

## 참고 자료

- [Kotlin 공식 문서: Null safety](https://kotlinlang.org/docs/null-safety.html)
- [Kotlin 공식 문서: Extensions](https://kotlinlang.org/docs/extensions.html)
- [Kotlin 공식 문서: Inline functions](https://kotlinlang.org/docs/inline-functions.html)
- [Kotlin 공식 문서: Coroutines](https://kotlinlang.org/docs/coroutines-overview.html)
- [Kotlin 공식 문서: Calling Kotlin from Java](https://kotlinlang.org/docs/java-to-kotlin-interop.html)
- [Java 공식 문서: Record Classes](https://docs.oracle.com/en/java/javase/21/language/records.html)
- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444)
