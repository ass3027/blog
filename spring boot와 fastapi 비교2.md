업무에서 주로 사용하던 spring boot를 벗어나, 개인 프로젝트를 하며 fastapi를 많이 써보았다. 요새 많이 사용되는 두 BE framework를 비교하여, 각각에 대한, 그리고 framework에 대해 좀 더 이해하기 위해 글을 작성한다.

## 순서
1. 등장 배경
2. 지향점
3. **생명주기와 동시성 처리**
4. DI와 제어권
5. 데이터 검증과 직렬화




## 3. 동시성 모델과 요청 생명주기

두 웹 프레임워크의 가장 많은 차이가 여기이다.

### Spring boot : Thread-per-reqeust, 서블릿 컨테이너

- Embedded Tomcat(내장 서블릿 컨테이너)
    어플리케이션 실행 시 내장 톰캣이 실행되며, 톰캣은 요청을 처리하기 위해 Worker Thead Pool을 유지한다

- 동시성 처리 방식과 Blocking I/O
    1. 요청을 들어오면, OS 소켓을 거쳐 톰켓의 Acceptor 스레드가 요청을 감지
    2. Thread Pool에서 1개의 Thread를 할당하여 요청을 처음부터 끝까지(DispatcherServlet-> Controller -> Service -> Repository)
    3. DB쿼리나 외부API 같은 I/O 발생 시 Blocking 처리

- 요청 생명주기(Request Lifecycle)
    `Servlet Container(Tomcat)` -> `Filter chain` -> `Dispatcher Servlet` -> `Handler Interceptor` -> `Controller` -> --> -->

### FastAPI: ASGI 이벤트 루프와 Asynchronous I/O

- Uvicorn과 이벤트 루프(Event Loop)
    FastAPI 자체는 HTTP서버가 아니며, ASGI 서버인 `Uvicorn` 위에서 실행된다. `Uvicorn`은 내부적으로 `uvloop`(libuv 기반의 고성능 이벤트 루프)를 사용해 싱글스레드(또는 코어별 프로세스) 기반의 이벤트 루프를 구동한다.

- 동시성 처리 방식과 Non-blocking I/O
    1. 요청이 드렁오면 이벤트 루프가 이를 감지하고 큐에 등록
    2. `async def`로 정의된 엔드포인트 내에서 DB쿼리나 외부API 같은 I/O 발생 시 Non-blocking 처리
    3. `def`로 정의된 동기 함수는 외부 스레드 풀로 격리하여 실행

- 요청 생명주기(Request Lifecycle)
    `Uvicron(ASGI Server)` -> `Starlette Routing & Middleware` -> `FastAPI Dependency Injection(Depends)` `Endpoint Handler(async def / def)` -> 응답 반환

| 비교 항목   | Spring Boot (MVC / Tomcat) | FastAPI (Uvicorn / Starlette)|
| --- | --- | --- |
|핵심 런타임 | Embedded Tomcat (Servlet Container)| Uvicorn (ASGI Server) + Starlette동시성 모델Thread-per-request (요청당 스레드 할당)Event Loop (싱글 스레드 기반 비동기 멀티플렉싱)|
|자원 소모 특성| 동시 접속자 수 증가 시 스레드 풀 및 메모리 소모 큼 | I/O 바운드 작업에서 적은 자원으로 고효율 처리|
|I/O 처리 방식| 기본적으로 블로킹(Blocking) I/O 중심 | 논블로킹(Non-blocking) 및 async/await 네이티브 지원|
|주의 사항 | 스레드 풀 고갈(Thread Pool Exhaustion) 관리 필요|이벤트 루프 블로킹(CPU-bound 작업 시 전체 지연) 주의|
