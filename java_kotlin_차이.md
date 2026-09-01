'자바와 코틀린의 차이가 뭐냐? 코드 스타일 빼고' 라는 질문을 최근에 들었다. 나는 코틀린에 대해서 '자바보다 훨씬 더 편한 코드 스타일을 제공하고 연산자 오버라이딩 등 라이브러리가 필요한 것들도 기본으로 제공해준다' 라고 단순하게만 인식하고 있었기에 제대로 된 답변을 하지 못했다. 해서 이 글을 통해서 대체 이 둘의 차이가 뭐냐!(코드적/내부적) 뭐가 다르고 뭐가 더 좋은점인지 정리하려한다. 

# Java vs Kotlin 

## 1. 코드 관련 부분

| 구분 | Java | Kotlin |
| --- | --- | --- |
| **Null Safety** | 타입 시스템에 Nullable 개념이 없음. 런타임 NPE(NullPointerException) 방지를 위해 수동으로 if (obj != null) 구문을 작성하거나 Optional을 사용해야 함. | 타입 자체에 Nullable(?)과 Non-Nullable을 구분함. 컴파일러 단계에서 Null 처리 구문 작성을 강제하여 빌드 오류를 발생시킴. |
| **코드 양 (Boilerplate)** | Getter/Setter, Equals 등 반복 코드가 많음 | `data class`, 스마트 캐스트 등으로 코드량이 대폭 감소 |
| **기능확장** | 기존 클래스에 기능을 추가하려면 해당 클래스를 매개변수로 받는 별도의 정적(Static) 유틸리티 클래스를 생성해야 함.  | 확장 함수(Extension Function) 문법을 제공하여 기존 클래스의 내장 메서드처럼 호출하는 구문을 작성할 수 있음. |
| **함수형 프로그래밍** | Java 8 이후 람다 지원, 객체 중심 구조 | 1급 시민 함수 지원, 고차 함수 및 확장 함수 지원 |
| **가변성 제어** | `final` 키워드를 명시해야 불변 변수 생성 | `val`(불변)과 `var`(가변)으로 명확히 구분 |

&nbsp;
>위의 표는 gemini의 답변으로 부터 나온것이다. 시민 함수는 또 뭐냐?   
일급 함수 == First-Class Function == 일급 시민 함수 == 함수가 First-Class Citizen를 만족
일급 시민 == First-Class Citizen == 아래의 세가지 조건 만족
>- 할당: 변수나 데이터 구조안에 담을 수 있다.
>- 전달: 함수의 매개변수로 전달할 수 있다.
>- 반환: 함수의 반환값으로 사용할 수 있다.
<br>

1. Null Safety   
    내가 실제 kotlin을 실습하면서 많이 느낀 점이다. java처럼 조용히 null exception을 시키지 않고 컴파일 시점에서 잡을 수 있으며
    ?나 !등의 키워드를 통해 처리할 수 있다. 예상치 못한 버그를 예방할 수 있기에 매우 좋다고 생각! 다른 언어인 ts, dart도 null safety를 지원한다.
    <br>

2. 코드 양(data class, smart cast)   
   java에서는 lombok을 통해서 자동으로 getter/setter를 생성하지만, kotlin에서는 `data class`를 통해서 더 간편하게 사용할 수 있다. 하지만 java 17(16) 부터는 `data class`는 java record와 필드 불변성, copy()를 제외하곤 많이 겹치고, smart cast 또한 java에서도 유사하게 사용 가능하기에 언어의 차이로 보기는 힘들어 보인다. 
   ```kotlin
   // kotlin
   data class User(val name: String, val age: Int)
   ```

   ```kotlin
    if(obj is String)
     obj.toUpperCase()
   ```
  
   ```java
   // java
   if(obj instanceof String str) 
     str.toUpperCase()
   ```
  &nbsp;

<br/>
<br/>

3. 기능확장
    `java`에서 `StringUtils`를 활용한 문자열 처리와 같은 기능을 `kotlin`에서는 확장 함수로 간편하게 구현할 수 있다.
    개인적으로 Spring/Java project에서 StringUtils.class를 잘 활용했는데 확장 함수를 통해 더 만족스로운 결과를 얻을 수 있을거 같다!

    예시)
    ```Kotlin
    // Kotlin 작성 코드
    fun String.lastChar(): Char = this[this.length - 1]
    
    val result = "Hello".lastChar()
    ```

4. 함수형 프로그래밍
    `java`에서 함수형 프로그래밍을 하려면 lambda와 stream api를 활용하여 구현 할 수 있지만, `kotlin`은 일급함수를 지원해 함수형 프로그래밍을 쉽게 할 수 있다.
    개인적인 생각으로 함수형 프로그래밍을 적절히 잘 사용하는게 가독성과 생산성에서 크게 도움된다고 생각한다. 실제로 `java`에서 `stream api`를 굉장히 좋아한다.
    <br/>

5. 가변성 제어
    java에서 불변을 `final`로 표현했던 것과 다르게 `val`(불변)과 `var`(가변)으로 명확히 구분하여 불변 변수를 선언할 수 있다.
    기존 java를 사용할땐 `final`을 생략하는 경우도 많았기에, 반드시 가변성을 나타내야 하는 kotlin 표현법이 더 좋다고 생각된다
    <br/>

## 2. 내부 동작 및 바이트코드 (컴파일 후 JVM 실행 기준)

| 비교 항목 | Java | Kotlin |
| --- | --- | --- |
| **Null 검사 런타임 동작** | 참조 변수가 Null일 때 객체에 접근하려는 순간 JVM 수준에서 명령을 처리하지 못하고 예외를 발생시킴. | 컴파일러가 바이트코드 생성 시 메서드 진입점에 `Intrinsics.checkNotNullParameter()`와 같은 방어적 정적 검사 코드를 묵시적으로 삽입함. 이로 인해 미세한 런타임 오버헤드가 발생함. |
| **확장 함수의 실제 구조** | 정적 유틸리티 메서드로 실행됨 | 코드상으로는 내장 메서드처럼 보이나, 컴파일 시 수신 객체(Receiver)를 첫 번째 매개변수로 넘겨받는 자바의 `public static` 메서드로 변환됨. 오버라이딩(동적 바인딩)이 불가능함. |
| **고차 함수와 메모리 제어** | 람다식 호출 시 런타임에 익명 객체를 생성하거나 `invokedynamic` 명령을 통해 힙 메모리를 할당함. | `inline` 키워드가 적용된 고차 함수는 컴파일 시점에 함수 본문과 람다의 바이트코드를 호출부 위치에 직접 삽입(인라이닝)함. 객체 생성으로 인한 런타임 메모리 할당 자체가 발생하지 않음. |
| **런타임 의존성** | 표준 JRE 클래스 라이브러리만으로 독립적인 실행이 가능함. | 코틀린 전용 컬렉션 및 헬퍼 함수를 지원하기 위해 컴파일된 결과물 실행 시 `kotlin-stdlib.jar` (런타임 라이브러리)가 반드시 JVM에 함께 로드되어야 함. |
| **최상위(Top-level) 함수** | 모든 메서드와 변수는 반드시 클래스 블록 내부에 존재해야 함. | 클래스 없이 파일 최상단에 선언된 함수는 컴파일러가 해당 파일명을 기반으로 임의의 정적 클래스(예: `FileNameKt.class`)를 생성하고 그 내부에 `static` 메서드로 컴파일함. |

&nbsp;
<br/>

1. Null 검사 런타임 동작   
    java의 경우 Null Exception을 처리하기 아주 까다롭다. 예외 처리를 놓쳐 Runtime 시점에서야 알게되는 경우도 많고 Null 예외 처리로 인해 코드가 더러워지기도 한다. 하지만 kotlin의 경우 `null`을 명시적으로 처리하는 `?` 연산자와 `let`, `run` 등의 함수를 통해 Compile 시점에 Null Safety를 보장할 수 있다.
    <br/>

2. 확장 함수의 실제 구조
    확장함수를 통해 가독성 향상 및 개발 생산성을 증가 시키면 서도, 바이트 코드 변환을 통해 정적 메소드 호출만 수행하기에 성능 저하가 없다(힙 메모리 할당 X, GC 부하 X). 

    예시)
    ```Kotlin
    // Kotlin 작성 코드
    fun String.lastChar(): Char = this[this.length - 1]
    
    val result = "Hello".lastChar()
    ```

    ```Java
    Java
    // 실제로 생성되는 Java 바이트코드 형태 (디컴파일 결과)
    public final class StringUtilsKt {
        public static final char lastChar(String $this$lastChar) {
            return $this$lastChar.charAt($this$lastChar.length() - 1);
        }
    }
    // 호출부 변환 형태
    char result = StringUtilsKt.lastChar("Hello");
    ```
    &nbsp;
    
3. 고차 함수와 메모리 제어
    Java에서 고차 함수 호출 시 컴파일러가 lambda식을 `Function0`, `Function1` 등의 함수 인터페이스를 구현하는 익명 클래스(Annonymous Class)로 컴파일 한다. 런타임에 해당 무명 클래스 인스턴스가 생성되기에 힙메모리 공간을 차지 -> GC 부하로 이어지게 된다.
    반면 Kotlin에선 inline 키워드를 통해 컴파일러가 바이트코드 생성 시 해당 람다식을 함수 본문에 inline(통째로 복사)하기에 객체 생성 없음 -> 힙메모리 차지 X -> GC 부하 X 로 이어진다.

4. 비동기 처리
    java에서 비동기를 처리할 경우 `Thread`를 보통 사용해서 처리했고, `Thread` 생성관리와 결과 처리를 위해 `Executor`, `CompletableFuture` 등을 사용했다. 하지만 kotlin에서는 `corrutine`이란 개념을 통해 비동기를 처리하고 이게 kotlin의 대표적인 장점으로 알려져 있다. `corrutine`의 경우 다뤄야 할 내용이 많기에 별도의 글에서 더 다뤄볼 예정이다.

# 마무리
왠만하면 하나의 글로 정리하고 싶었지만, 설명을 위해선 하위 개념이 나오고 또 그 하위 개념을 설명하고~~~~~ 끝이 없다. 하지만 이걸 안하면 이해도 설명도 안된다. 견뎌라.   
다음 글을 통해서 정리를 마무리 하고 링크를 걸러 오겠다. `corrutine` 설명에서 다시 만나요~
