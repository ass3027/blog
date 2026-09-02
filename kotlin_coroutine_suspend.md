# Kotlin coroutine과 suspend

Kotlin coroutine은 `suspend`와 구조화된 동시성을 중심으로 한 비동기 작업 모델이다. Java virtual thread와 목적이 일부 겹치지만, coroutine은 컴파일러 변환을 기반으로 하고 virtual thread는 JVM이 제공하는 가벼운 스레드라는 차이가 있다.

## coroutine은 어디에 있나

coroutine을 “코틀린이 제공하는 비동기 라이브러리”라고만 보면 절반만 맞다. coroutine은 세 계층으로 나뉘어 있고, 각 계층이 서로 다른 곳에 있다.

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

이 함수는 순차 코드처럼 보이지만, 컴파일 후에는 “지금 몇 번째 단계인가”를 들고 다니며 호출될 때마다 해당 지점부터 재개하는 구조가 된다. 확장 함수가 정적 메서드로 바뀌는 것, `inline`이 본문을 호출 지점에 삽입하는 것과 같은 성격이지만, 변환의 규모가 더 크다.

`kotlinx-coroutines`가 그 위에 얹는 것은 스케줄링·취소·구조화다. 정리하면 “어떻게 멈추고 재개하나”는 언어가, “누가 언제 실행하고 취소하나”는 라이브러리가 담당한다.

## Java virtual thread와 비교

Java에서는 `ExecutorService`, `CompletableFuture`, reactive 라이브러리 등으로 비동기 작업을 구성해 왔고, JDK 21부터는 virtual thread가 중요한 선택지다. 다만 virtual thread는 같은 문제를 **JVM 런타임 차원**에서 푸는 접근이라, 컴파일러 변환으로 푸는 coroutine과는 해법의 층위가 다르다. 어느 쪽이든 결국 dispatcher와 스레드 같은 실행 기반은 필요하다.

| 항목 | Kotlin coroutine | Java virtual thread |
| --- | --- | --- |
| 본질 | `suspend` 기반 비동기 작업 모델 | 가벼운 Java `Thread` |
| 대기 시 | coroutine을 중단하고, 실행하던 스레드를 다른 작업에 사용할 수 있음 | 동기식 블로킹 코드처럼 작성하되, JVM이 지원하는 대기에서는 carrier thread를 비움 |
| 코드 형태 | 호출 경로에 `suspend` 전파 필요 | 기존 스레드 기반 동기 코드와 유사 |
| 취소·구조화 | `CoroutineScope`와 `Job`이 중심 | 별도 API와 설계가 필요 |
