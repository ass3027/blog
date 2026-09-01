# Java vs Kotlin

“자바와 코틀린의 차이가 뭐냐? 코드 스타일 빼고”라는 질문을 최근에 들었다. 나는 코틀린을 “자바보다 훨씬 편한 코드 스타일을 제공하고, 연산자 오버로딩처럼 라이브러리로 구현할 법한 기능도 언어 차원에서 지원한다” 정도로만 단순하게 인식하고 있었기에 제대로 답하지 못했다.

그런데 정리하다 보니 “코드 스타일 빼고”라는 단서 자체가 애매하다는 걸 알게 됐다. 스타일처럼 보이는 것 중 상당수는 취향이 아니라 타입 시스템의 결과이고, 반대로 문법은 비슷해 보여도 컴파일 결과가 전혀 다른 것들이 있다.

그래서 이 글은 주제별로 나누고, 각 주제에서 문법과 컴파일 결과를 함께 본다. 기준은 Kotlin/JVM과 현대 Java다.

| 주제 | Java | Kotlin/JVM |
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

주로 사용하는 연산자는 안전 호출 `?.`와 Elvis 연산자 `?:`다. `!!`는 “null이 아님을 내가 보장한다”는 단언이며, 값이 실제로 null이면 NPE를 발생시키므로 null safety를 위한 일반적인 해결책으로 보면 안 된다.

Java에서 null 참조로 메서드를 호출하거나 필드에 접근하면 JVM이 `NullPointerException`을 던진다. Java의 타입만으로는 해당 참조가 null일 수 있는지를 구분하지 않기 때문에, 팀 규칙·애너테이션·정적 분석 도구·`Optional` 등을 함께 사용해 관리한다.

여기서 중요한 건, Kotlin의 검사가 **컴파일 시점에서 끝나지 않는다**는 점이다. Kotlin 컴파일러는 Java와의 상호 운용 과정에서 non-null 매개변수 계약을 지키기 위해 `Intrinsics.checkNotNullParameter()` 같은 검사 호출을 바이트코드에 생성할 수 있다. Java 쪽에서 넘어오는 값은 컴파일러가 통제할 수 없으니, 계약이 깨지는 지점을 최대한 이르게 잡으려는 것이다.

다만 정확한 바이트코드 모양은 Kotlin 컴파일러 버전과 선언 위치에 따라 달라질 수 있으므로, 이를 모든 non-null 값에 항상 삽입되는 검사라고 일반화해서는 안 된다.

정리하면 Kotlin null safety의 핵심은 null 계약을 타입에 드러내고, nullable 값의 사용 지점을 컴파일 시점에 확인하는 데 있다. `!!`, 초기화 문제, Java에서 넘어온 platform type처럼 타입 시스템 밖에서 들어오는 값은 별도로 주의해야 한다.

## 2. 데이터 클래스: data class와 record

Kotlin의 `data class`는 데이터를 담는 클래스를 짧게 만들 때 유용하다.

```kotlin
data class User(val name: String, val age: Int)
```

Java도 Java 16부터 record를 제공한다.

```java
public record User(String name, int age) {}
```

둘 다 접근자와 값 기반 `equals`, `hashCode`, `toString`을 자동으로 제공한다. 따라서 단순 DTO나 값 객체에서는 과거보다 차이가 작아졌다. 다만 Kotlin `data class`는 `copy()`와 구조 분해에 쓰이는 `componentN()`을 제공하고, 생성자 프로퍼티를 `var`로 선언할 수도 있다. 반대로 Java record는 구성 요소 필드가 `final`이고 클래스도 상속할 수 없도록 설계된 데이터 운반용 타입이다.

스마트 캐스트도 차이가 줄어든 영역이다. Kotlin은 타입 검사 뒤에 타입을 자동으로 좁힌다.

```kotlin
if (obj is String) {
    println(obj.uppercase())
}
```

Java도 패턴 매칭 `instanceof`로 비슷한 코드를 쓸 수 있다.

```java
if (obj instanceof String str) {
    System.out.println(str.toUpperCase());
}
```

이 절은 “예전에는 차이였지만 지금은 아닌” 영역에 가깝다. Kotlin의 장점을 이야기할 때 관성적으로 끌려 나오지만, 현대 Java와 비교하면 남는 차이는 `copy()`와 구조 분해 정도다.

## 3. val과 var

`val`은 참조를 다시 대입할 수 없고, `var`는 다시 대입할 수 있다. Java의 `final`도 같은 수준에서 재할당을 막는다.

```kotlin
val users = mutableListOf("A")
users.add("B") // 가능: 참조는 고정이지만 객체 내부는 가변
```

`val`과 `final`은 참조의 재할당을 막는다. 객체 내부의 변경 가능성은 컬렉션 타입과 클래스 설계로 별도로 결정한다.

차이는 기능이 아니라 **기본값**에 있다. Java에서 `final`은 붙이지 않으면 그만인 선택지지만, Kotlin은 모든 변수에 `val` 또는 `var` 중 하나를 반드시 쓰게 한다. 재할당 의도를 생략할 수 없게 만든 것이고, 이건 스타일이 아니라 문법 강제다.

## 4. 확장 함수

Java에서는 보통 `StringUtils` 같은 유틸리티 클래스에 정적 메서드를 둔다. Kotlin은 이를 수신 객체의 멤버처럼 호출하는 문법을 제공한다.

```kotlin
fun String.lastChar(): Char = this[length - 1]

val result = "Hello".lastChar()
```

그런데 이게 JVM에서 정말 `String`에 메서드를 추가하는 건 아니다. 위 함수가 `StringUtils.kt` 최상위에 선언되었다면, 대략 다음과 같은 정적 메서드가 된다.

```java
public final class StringUtilsKt {
    public static final char lastChar(String receiver) {
        return receiver.charAt(receiver.length() - 1);
    }
}

char result = StringUtilsKt.lastChar("Hello");
```

즉 Java의 유틸리티 클래스와 **바이트코드 수준에서는 같은 것**이고, 다른 건 호출 문법뿐이다. 그래서 호출 대상은 선언된 수신 타입을 기준으로 컴파일 시점에 결정되고(정적 디스패치), 같은 시그니처의 멤버 함수가 있으면 멤버 함수가 우선한다. 확장 함수는 오버라이드되지 않는다.

확장 함수의 가치는 기존 타입을 수정하지 않고도 도메인 문맥에 맞는 호출 문법을 제공한다는 데 있다. 성능이 중요한 지점에서는 실제 병목을 측정해 판단하면 된다.

## 5. 고차 함수와 inline

Java와 Kotlin 모두 객체 지향과 함수형 스타일을 함께 지원하는 다중 패러다임 언어다. Java에서는 함수형 인터페이스와 람다, Stream API를 조합하고, 현대 Java의 람다는 주로 `invokedynamic`으로 구현된다. 캡처하지 않는 람다는 재사용될 수 있고, 캡처하는 람다는 객체를 할당할 수 있으며, JIT 컴파일러가 일부 비용을 제거하기도 한다.

Kotlin은 `(T) -> R` 같은 함수 타입이 언어에 포함되어 있고, 확장 함수·고차 함수와 함께 사용할 수 있어 컬렉션 처리나 DSL을 비교적 간결하게 작성할 수 있다. 따라서 “Java는 함수형 프로그래밍을 못 하고 Kotlin만 할 수 있다”가 아니라, Kotlin이 함수형 스타일을 더 적은 의식으로 표현하게 해 준다고 이해하는 편이 정확하다.

Kotlin에서도 고차 함수에 전달하는 람다는 함수 객체와 간접 호출 비용을 만들 수 있다. `inline` 함수는 함수 본문과 inlinable 람다를 호출 지점에 삽입해 이 비용을 줄일 수 있다.

```kotlin
inline fun <T> measure(block: () -> T): T {
    return block()
}
```

`inline`은 작은 고차 함수가 반복적으로 호출되는 지점에서 람다 객체와 간접 호출 비용을 줄이는 도구다. 대신 호출 지점의 코드 크기는 커지고, `noinline`으로 표시된 람다는 인라인되지 않는다. 따라서 비용이 확인된 작은 함수에 적용하는 것이 좋다.

## 6. coroutine과 suspend

여기가 Java에 대응물이 없는 영역이다. 그리고 coroutine을 “코틀린이 제공하는 비동기 라이브러리” 정도로 알고 있었다면 절반만 맞다. coroutine은 세 층으로 나뉘어 있고, 각 층이 서로 다른 곳에 산다.

| 층 | 실체 | 어디 있나 |
| --- | --- | --- |
| **언어/컴파일러** | `suspend` 키워드, CPS 변환, 상태 머신 생성 | Kotlin 컴파일러 자체 |
| **표준 라이브러리** | `Continuation`, `suspendCoroutine`, `CoroutineContext` | `kotlin-stdlib`의 `kotlin.coroutines` |
| **동시성 프레임워크** | `launch`, `async`, `Job`, `Dispatchers`, `Flow` | `kotlinx-coroutines` (별도 의존성) |

핵심은 **중단·재개 메커니즘 자체가 라이브러리가 아니라 컴파일러 기능**이라는 점이다. `suspend fun`은 바이트코드에서 `Continuation`을 마지막 인자로 받는 함수로 바뀌고, 함수 본문은 중단 지점마다 분기하는 상태 머신으로 변환된다.

```kotlin
suspend fun load(id: Long): User {
    val user = fetchUser(id)          // 중단 가능 지점
    val profile = fetchProfile(user)  // 중단 가능 지점
    return user.with(profile)
}
```

이 함수는 순차 코드처럼 보이지만, 컴파일 후에는 “지금 몇 번째 단계인가”를 들고 다니며 호출될 때마다 해당 지점부터 재개하는 구조가 된다. 확장 함수가 정적 메서드로 바뀌는 것, `inline`이 본문을 호출 지점에 삽입하는 것과 같은 성격이되, 변환의 규모가 가장 크다.

`kotlinx-coroutines`가 그 위에 얹는 건 스케줄링·취소·구조화다. 정리하면 “어떻게 멈추고 재개하나”는 언어가, “누가 언제 실행하고 취소하나”는 라이브러리가 담당한다.

Java에서는 `ExecutorService`, `CompletableFuture`, reactive 라이브러리 등으로 비동기 작업을 구성해 왔고, JDK 21부터는 virtual thread가 중요한 선택지다. 다만 virtual thread는 같은 문제를 **JVM 런타임 차원**에서 푸는 접근이라, 컴파일러 변환으로 푸는 coroutine과는 해법의 층위가 다르다. 어느 쪽이든 결국 dispatcher와 스레드 같은 실행 기반은 필요하다.

그리고 이 절이 처음의 질문에 대한 가장 분명한 답이기도 하다. `suspend`는 코드 스타일 문제로 환원되지 않는다.

## 7. 언어 밖의 차이

문법도 컴파일 결과도 아니지만 실제로 체감되는 차이가 남아 있다.

**런타임 의존성.** Kotlin 표준 라이브러리는 컬렉션 확장 함수, 범위 함수, null 처리 보조 기능 등 Kotlin 코드의 기반 기능을 제공한다. 배포할 때는 Gradle이나 Maven이 런타임 의존성으로 관리하거나, 실행 가능한 JAR에 함께 포함한다. coroutine을 쓴다면 `kotlinx-coroutines`가 추가로 붙는다. Java는 JDK 표준 라이브러리만으로 출발한다.

**최상위 선언.** Kotlin 컴파일러는 최상위 함수와 프로퍼티를 JVM 클래스에 담는다. `Example.kt`의 최상위 함수는 기본적으로 `ExampleKt`의 정적 메서드가 된다. `@file:JvmName`으로 Java에서 보이는 클래스 이름을 바꿀 수도 있다. Java에서는 모든 선언이 타입 내부에 있어야 하므로, 이건 상호 운용 시 이름이 어떻게 보일지의 문제가 된다.

## 마무리

웬만하면 하나의 글로 정리하고 싶었지만, 설명하려면 하위 개념이 나오고 또 그 하위 개념을 설명해야 해서 끝이 없다. 하지만 이걸 안 하면 이해도, 설명도 안 된다. 견뎌라.

처음 질문으로 돌아가면, “코드 스타일 빼고”라는 단서는 사실 나눠 볼 필요가 있었다. data class나 스마트 캐스트처럼 현대 Java가 따라잡아 차이가 거의 없어진 것도 있고, null 계약이나 `val`/`var`처럼 스타일로 보이지만 실은 타입 시스템이 강제하는 것도 있으며, `suspend`처럼 애초에 스타일의 문제가 아닌 것도 있다.

null safety, 확장 함수, `inline`, coroutine처럼 더 파고들 주제는 따로 정리해서 이 글에도 링크를 걸어 오겠다. 그때 다시 만나요~

## 참고 자료

- [Kotlin 공식 문서: Null safety](https://kotlinlang.org/docs/null-safety.html)
- [Kotlin 공식 문서: Extensions](https://kotlinlang.org/docs/extensions.html)
- [Kotlin 공식 문서: Inline functions](https://kotlinlang.org/docs/inline-functions.html)
- [Kotlin 공식 문서: Coroutines](https://kotlinlang.org/docs/coroutines-overview.html)
- [Kotlin 공식 문서: Calling Kotlin from Java](https://kotlinlang.org/docs/java-to-kotlin-interop.html)
- [Java 공식 문서: Record Classes](https://docs.oracle.com/en/java/javase/21/language/records.html)
- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444)
