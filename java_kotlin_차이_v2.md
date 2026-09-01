# Java vs Kotlin

“자바와 코틀린의 차이가 뭐냐? 코드 스타일 빼고”라는 질문을 최근에 들었다. 나는 코틀린을 “자바보다 훨씬 편한 코드 스타일을 제공하고, 연산자 오버로딩처럼 라이브러리로 구현할 법한 기능도 언어 차원에서 지원한다” 정도로만 단순하게 인식하고 있었기에 제대로 답하지 못했다.

해서 이 글을 통해 대체 이 둘의 차이가 뭐냐! 코드와 내부 동작은 무엇이 다르고, 각각 어떤 장단점이 있는지 정리하려 한다. 기준은 Kotlin/JVM과 현대 Java다.

## 1. 코드에서 바로 느껴지는 차이

| 구분 | Java | Kotlin |
| --- | --- | --- |
| **Null 처리** | 타입에 null 가능 여부 없음<br>`if`, `Optional`, 애너테이션으로 관리 | `T` / `T?`로 nullability 구분<br>nullable 값의 직접 접근 차단 |
| **반복 코드** | 일반 클래스 또는 Lombok<br>record(Java 16+)로 간결화 | `data class`<br>`copy()`, `componentN()` 자동 생성 |
| **기능 확장** | 정적 유틸리티 메서드<br>래퍼·Decorator 패턴 | 확장 함수 문법<br>기존 멤버처럼 호출 |
| **함수 사용** | 람다 + 함수형 인터페이스<br>Stream API | 함수 타입 `(T) -> R`<br>고차 함수·DSL |
| **재할당 제어** | `final`로 재할당 금지 | `val` / `var`를 선언에서 명시 |

### 1. Null safety

Kotlin의 가장 체감되는 장점은 null 가능 여부를 타입에 표현한다는 점이다. `String`에는 `null`을 대입할 수 없고, `String?`만 `null`을 가질 수 있다. nullable 값의 멤버에 바로 접근하면 컴파일 오류가 발생하므로, 호출하는 쪽에서 null 처리 방식을 선택해야 한다.

```kotlin
val name: String? = getName()

val length = name?.length ?: 0
```

주로 사용하는 연산자는 안전 호출 `?.`와 Elvis 연산자 `?:`다. `!!`는 “null이 아님을 내가 보장한다”는 단언이며, 값이 실제로 null이면 NPE를 발생시키므로 null safety를 위한 일반적인 해결책으로 보면 안 된다.

Kotlin null safety의 핵심은 null 계약을 타입에 드러내고, nullable 값의 사용 지점을 컴파일 시점에 확인하는 데 있다. `!!`, 초기화 문제, Java에서 넘어온 platform type처럼 타입 시스템 밖에서 들어오는 값은 별도로 주의해야 한다.

### 2. 코드 양: data class와 record

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

### 3. 확장 함수

Java에서는 보통 `StringUtils` 같은 유틸리티 클래스에 정적 메서드를 둔다. Kotlin은 이를 수신 객체의 멤버처럼 호출하는 문법을 제공한다.

```kotlin
fun String.lastChar(): Char = this[length - 1]

val result = "Hello".lastChar()
```

확장 함수는 기존 타입을 바꾸지 않고도 호출 문맥에 맞는 API를 만든다. 호출 대상은 선언된 수신 타입을 기준으로 컴파일 시점에 결정되며, 같은 시그니처의 멤버 함수가 있으면 멤버 함수가 우선한다.

### 4. 함수형 프로그래밍

Java와 Kotlin 모두 객체 지향과 함수형 스타일을 함께 지원하는 다중 패러다임 언어다. Java에서는 함수형 인터페이스와 람다, Stream API를 조합한다.

Kotlin은 `(T) -> R` 같은 함수 타입이 언어에 포함되어 있고, 확장 함수·고차 함수와 함께 사용할 수 있어 컬렉션 처리나 DSL을 비교적 간결하게 작성할 수 있다. 따라서 “Java는 함수형 프로그래밍을 못 하고 Kotlin만 할 수 있다”가 아니라, Kotlin이 함수형 스타일을 더 적은 의식으로 표현하게 해 준다고 이해하는 편이 정확하다.

### 5. val과 var

`val`은 참조를 다시 대입할 수 없고, `var`는 다시 대입할 수 있다. Java의 `final`도 같은 수준에서 재할당을 막는다.

```kotlin
val users = mutableListOf("A")
users.add("B") // 가능: 참조는 고정이지만 객체 내부는 가변
```

`val`과 `final`은 참조의 재할당을 막는다. 객체 내부의 변경 가능성은 컬렉션 타입과 클래스 설계로 별도로 결정한다. Kotlin은 모든 변수에 `val` 또는 `var`를 명시하므로, 재할당 의도를 코드에서 더 쉽게 읽을 수 있다.

## 2. JVM에서의 차이

Kotlin/JVM 코드는 Java 바이트코드로 컴파일되어 JVM에서 실행된다. 그래서 두 언어는 상호 운용성이 높지만, Kotlin 컴파일러가 생성하는 코드에는 Kotlin의 언어 기능을 지원하기 위한 구조가 추가된다.

| 비교 항목 | Java | Kotlin/JVM |
| --- | --- | --- |
| **null 계약** | 타입 수준 null 구분 없음<br>잘못된 접근은 런타임 NPE | nullable 타입으로 컴파일 타임 검사<br>Java 경계에서는 계약 검사 코드 생성 가능 |
| **확장 함수** | 유틸리티 클래스의 정적 메서드 호출 | 수신 객체를 첫 인자로 받는 정적 메서드로 컴파일<br>정적 디스패치 |
| **고차 함수** | 함수형 인터페이스와 `invokedynamic` 기반<br>캡처·JIT에 따라 할당 차이 | 함수 객체·간접 호출 비용 가능<br>`inline`으로 람다 인라이닝 가능 |
| **런타임 의존성** | JDK/JRE 표준 라이브러리 중심 | Kotlin 표준 라이브러리 사용<br>실행 JAR 포함 또는 classpath 배치 |
| **최상위 함수** | 클래스·인터페이스 등 타입 내부 선언 | 파일 최상위 선언 지원<br>`파일명Kt` 정적 메서드로 생성 |

### 1. null 검사와 런타임

Java에서 null 참조로 메서드를 호출하거나 필드에 접근하면 JVM이 `NullPointerException`을 던진다. Java의 타입만으로는 해당 참조가 null일 수 있는지를 구분하지 않기 때문에, 팀 규칙·애너테이션·정적 분석 도구·`Optional` 등을 함께 사용해 관리한다.

Kotlin은 nullable 타입에 대한 안전 호출을 강제한다. 또한 Kotlin 컴파일러는 Java와의 상호 운용 과정에서 non-null 매개변수 계약을 지키기 위해 `Intrinsics.checkNotNullParameter()` 같은 검사 호출을 생성할 수 있다. 정확한 바이트코드 모양은 Kotlin 컴파일러 버전과 선언 위치에 따라 달라질 수 있으므로, 이를 모든 non-null 값에 항상 삽입되는 검사라고 일반화해서는 안 된다.

### 2. 확장 함수의 실제 구조

앞의 `lastChar()`가 파일 최상위에 선언되었다면 JVM에서는 대략 다음과 같은 정적 메서드 형태가 된다.

```java
public final class StringUtilsKt {
    public static final char lastChar(String receiver) {
        return receiver.charAt(receiver.length() - 1);
    }
}

char result = StringUtilsKt.lastChar("Hello");
```

따라서 확장 함수의 가치는 기존 타입을 수정하지 않고도, 도메인 문맥에 맞는 호출 문법을 제공한다는 데 있다. 일반적인 최상위 확장 함수 호출은 정적 메서드 호출로 바뀌며, 성능이 중요한 지점에서는 실제 병목을 측정해 판단하면 된다.

### 3. 고차 함수와 inline

현대 Java의 람다는 주로 `invokedynamic`과 함수형 인터페이스로 구현된다. 캡처하지 않는 람다는 재사용될 수 있고, 캡처하는 람다는 객체를 할당할 수 있으며, JIT 컴파일러가 일부 비용을 제거하기도 한다.

Kotlin에서도 고차 함수에 전달하는 람다는 함수 객체와 간접 호출 비용을 만들 수 있다. `inline` 함수는 함수 본문과 inlinable 람다를 호출 지점에 삽입해 이 비용을 줄일 수 있다.

```kotlin
inline fun <T> measure(block: () -> T): T {
    return block()
}
```

`inline`은 작은 고차 함수가 반복적으로 호출되는 지점에서 람다 객체와 간접 호출 비용을 줄이는 도구다. 대신 호출 지점의 코드 크기는 커지고, `noinline`으로 표시된 람다는 인라인되지 않는다. 따라서 비용이 확인된 작은 함수에 적용하는 것이 좋다.

### 4. 런타임과 최상위 함수

Kotlin 표준 라이브러리는 컬렉션 확장 함수, 범위 함수, null 처리 보조 기능 등 Kotlin 코드의 기반 기능을 제공한다. 배포할 때는 Gradle이나 Maven이 런타임 의존성으로 관리하거나, 실행 가능한 JAR에 함께 포함한다.

Kotlin 컴파일러는 최상위 함수와 프로퍼티를 JVM 클래스에 담는다. 예를 들어 `Example.kt`의 최상위 함수는 기본적으로 `ExampleKt`의 정적 메서드가 된다. `@file:JvmName`으로 Java에서 보이는 클래스 이름을 바꿀 수도 있다.

### 5. 비동기 처리와 coroutine

Java에서는 `ExecutorService`, `CompletableFuture`, reactive 라이브러리 등을 사용해 비동기 작업을 구성해 왔다. JDK 21부터는 virtual thread도 중요한 선택지다.

Kotlin의 coroutine은 적은 수의 스레드 위에서 많은 작업을 중단·재개할 수 있도록 작성 모델을 제공한다. `suspend` 함수로 비동기 흐름을 순차 코드처럼 표현할 수 있다는 점이 큰 장점이다. 다만 coroutine 자체는 Kotlin 표준 라이브러리 기능이 아니라 일반적으로 `kotlinx-coroutines` 라이브러리를 추가해 사용하며, 내부적으로는 결국 dispatcher와 스레드 같은 실행 기반이 필요하다.

## 마무리

웬만하면 하나의 글로 정리하고 싶었지만, 설명하려면 하위 개념이 나오고 또 그 하위 개념을 설명해야 해서 끝이 없다. 하지만 이걸 안 하면 이해도, 설명도 안 된다. 견뎌라.

이번 글에서는 Kotlin/JVM과 현대 Java의 큰 차이를 훑어 봤다. null safety, 확장 함수, `inline`, coroutine처럼 더 파고들 주제는 따로 정리해서 이 글에도 링크를 걸어 오겠다. 그때 다시 만나요~

## 참고 자료

- [Kotlin 공식 문서: Null safety](https://kotlinlang.org/docs/null-safety.html)
- [Kotlin 공식 문서: Extensions](https://kotlinlang.org/docs/extensions.html)
- [Kotlin 공식 문서: Inline functions](https://kotlinlang.org/docs/inline-functions.html)
- [Kotlin 공식 문서: Calling Kotlin from Java](https://kotlinlang.org/docs/java-to-kotlin-interop.html)
- [Java 공식 문서: Record Classes](https://docs.oracle.com/en/java/javase/21/language/records.html)
