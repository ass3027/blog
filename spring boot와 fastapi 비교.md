업무에서 주로 사용하던 Spring Boot를 벗어나, 개인 프로젝트를 하며 FastAPI를 많이 써보았다. 요새 많이 사용되는 두 BE 프레임워크를 비교하여, 각각에 대한, 그리고 프레임워크에 대해 좀 더 이해하기 위해 글을 작성한다. 이 글에선 1, 2번만 작성하고 다음 글에서 나머지 사항들을 정리하겠다.

## 순서
1. **등장 배경**
2. **지향점**
3. 생명주기와 동시성 처리
4. DI와 제어권
5. 데이터 검증과 직렬화


## 1. 등장 배경

### Spring
Spring Boot는 Spring의 속성도 포함하기에 Spring/Spring Boot를 나눠서 설명하겠다.
초기 J2EE/EJB 기반 개발의 복잡함을 해결하기 위해 등장했다.

#### 문제점
- 비즈니스 로직과 실행 환경의 높은 결합성
  > 비즈니스 로직을 작성할 때에도 EJB에서 정해진 인터페이스와 생명주기 규약 등을 따라야 했기에 기술에 비즈니스 로직이 강하게 결합되었다.

- 낮은 테스트 독립성
  > 본래 테스트를 할 때 빠르고 가볍게 실행하며 개발의 병목이 되면 안 되지만, 단순한 업무 로직을 검증하려 해도 EJB 컨테이너 등 관련 환경을 준비해야 했고, 테스트가 힘들었다.

- 반복된 데이터 접근 코드
  > JDBC를 사용할 때 프로젝트마다 연결/자원 관리, 예외 처리 등의 코드를 매번 작성해야 했고 업무 코드와 섞이기 쉬웠다.

#### 해결책
- POJO(Plain Old Java Object) 중심 개발: 특정 인터페이스와 상속 구조의 의존을 감축
- IoC와 DI: 객체 생성과 의존성 연결을 개발자가 아닌 컨테이너에서 관리, DI를 통해 테스트 의존성을 쉽게 대체 가능
- 트랜잭션 추상화: 트랜잭션과 같은 공통 처리를 업무 로직에서 분리. 추상화를 통해 여러 트랜잭션 라이브러리를 공통 처리. 예: `PlatformTransactionManager`
- AOP 기반 선언적 트랜잭션(Declarative Transaction): 복잡했던 트랜잭션 설정을 AOP + `@Transactional`을 통해 비즈니스 로직과 트랜잭션 코드를 분리
- 데이터 접근 추상화: `JdbcTemplate` 등으로 반복되는 자원 관리와 예외 처리를 해결

추상화와 POJO, IoC, DI가 Spring에서 대표적인 특징이자 개선점이다.

### Spring Boot
기존 Spring에 있던 불편 사항인 많은 XML 설정, WAS 별도 관리, 의존성 관리들을 해결하기 위해 등장했다

#### 문제점
- 반복적인 인프라 설정

    웹 요청 처리, `DataSource`, `TransactionManager` 등 필수 구성 요소들을 직접 설정해야 했고, 프로젝트마다 반복해서 설정해야 했다. 이를 위해 XML 설정 대신 Java 설정이 나왔지만 여전히 불편하였다.

- WAS 의존 배포

    빌드 결과물인 WAR 파일을 별도 WAS(주로 Tomcat)에서 실행하였기에 별도 Tomcat 관리(설정, 옵션 값 튜닝, 업데이트) 등을 하여야 했기에 프로젝트 관리 비용이 컸다

- 의존성 관리 난이도

    Spring, Hibernate, Jackson 등 많은 라이브러리의 버전 호환성을 개발자가 직접 맞춰야 했고, 실수하면 바로 `ClassNotFoundException`이 발생해 라이브러리에 취약점이 있다 해도 잘 업데이트를 하지 않았다.

#### 해결책
- Auto-configuration을 통해 많은 필요한 Bean들을 자동으로 설정해 반복된 인프라 설정 제거
- Starter POMs를 통해 `spring-boot-starter-web`만 추가해도 필요한 라이브러리를 가져오고 (예: `spring-boot-starter-data-jpa`),
- `BOM`을 통해 검증된 버전의 라이브러리들을 관리
- Embedded Tomcat을 통해 별도의 WAS 없이 executable JAR로 단독 실행 가능


Spring Boot는 기존 Java 프레임워크의 "과도한 설정과 관리, 개발 난이도"를 표준화된 자동 설정과 강력한 규격으로 통제한다


### FastAPI
FastAPI는 기존 Python 프레임워크와 플러그인을 조합해 API를 개발하는 과정에서 겪은 불편함을 개선하기 위해 등장했다. 데이터 검증, 변환, 직렬화, API 문서화에 필요한 기능을 통합하고, Python 타입 힌트를 활용해 반복 작업을 줄이는 데 초점을 맞췄다.

#### 문제점
1. API 기능을 조합하는 과정의 복잡함과 중복

    API를 만들 때에는 요청 데이터의 검증과 변환, 응답 데이터의 직렬화, API 문서화 등이 필요하다. 기존에도 Flask와 여러 플러그인을 연결해 구현할 수 있었지만, 필요한 도구를 선택하고 조합해야 했다. 데이터 구조와 문서를 별도로 정의하는 방식에서는 같은 정보를 반복해서 작성하거나, 코드를 수정한 뒤 문서를 갱신하지 않아 서로 달라지는 문제가 생길 수 있었다.

2. 스키마 선언과 개발 도구 지원의 한계

    도구 고유의 스키마 선언 방식이나 문자열 안에 작성한 문서 정보는 IDE가 Python 타입 정보로 활용하기 어려웠다. API의 데이터 구조를 정의하면서도 코드에서 자동완성과 타입 분석의 도움을 충분히 받기 어려운 경우가 있었다.

#### 해결책

1. Python 타입 힌트와 Pydantic 모델을 통한 선언 공유

    Python 타입 힌트와 Pydantic 모델로 정의한 데이터 구조를 검증과 변환, 직렬화, 문서 생성에 활용한다. 같은 선언을 여러 기능에서 공유해 중복을 줄이고, IDE도 타입 정보를 활용할 수 있게 한다. 타입 힌트 자체가 실행 중 타입을 강제하는 것은 아니며, 실제 데이터 검증과 변환은 FastAPI가 Pydantic을 활용해 수행한다.

2. API 개발 기능과 표준 기반 문서화의 통합

    Starlette의 웹 처리 기능과 Pydantic의 데이터 처리 기능을 통합하고, 경로와 매개변수, 모델 선언 등을 바탕으로 OpenAPI 스키마와 대화형 API 문서를 자동 생성한다.

FastAPI는 API 개발에 필요한 기능을 일관된 방식으로 연결하고, 타입 선언을 여러 기능에 재사용해 반복 작업을 줄이려 했다.

## 2. 지향점

### Spring Boot

Opinionated Framework
> 내가 이해한 바로는, 프레임워크가 특정 아키텍처와 설정 방식을 유도하는 것을 말한다. 이를 통해 개발자들은 너무 높은 자유도로 인해 고통받지 않을 수 있다.

- Convention over Configuration
    애플리케이션 세부 설정을 일일이 하는 대신, Spring Boot의 Auto-configuration이 classpath의 라이브러리를 감지하여 설정을 자동으로 한다. 또한 자동 설정으로 하기에 협업 시 아키텍처 일관성 또한 유도한다.

- 프레임워크 주도
    개발자가 아닌 프레임워크의 `Spring IoC Container`가 객체 생명주기, 애플리케이션 흐름을 제어하고 관리한다.

- 장기적 안정성과 확장성
    초기 학습 곡선이 가파르고 코드가 상대적으로 복잡할 수 있지만, 복잡한 비즈니스 로직, 큰 트랜잭션을 안정적으로 다룰 수 있는 "대규모 시스템의 표준 통제"를 지향합니다


### FastAPI

Unopinionated / Minimalist Framework
> 표준을 통제하기보다, 자율성/생산성 극대화를 목표

- DRY(Don't Repeat Yourself)

    "중복 작업을 하지 마라"는 뜻인데, FastAPI는 Pydantic 모델과 Python 타입 힌트, `Depends`를 통해 데이터 스키마, 타입 정보, 재사용 가능한 의존성 주입을 통해 중복을 제거한다.

- Unopinionated / Minimalist

    프레임워크가 Spring Boot처럼 컨테이너를 강제하지 않고, 개발자가 필요한 라이브러리들을 직접 선택, 조합, 설정합니다.

- 웹 표준 기반의 고성능 비동기 I/O(ASGI & Modern Standards)
    FastAPI는 Starlette를 기반으로 ASGI 규약을 지원하며, Uvicorn 같은 ASGI 서버에서 실행할 수 있다. `async/await` 문법을 활용한 비동기 I/O를 지원한다.



Spring Boot가 거대한 IoC 컨테이너를 통해 객체의 생명주기와 전역적인 설정을 주도하는 '프레임워크 중심(Framework-driven)' 구조라면, FastAPI는 전역 컨테이너 없이 개발자가 필요한 컴포넌트를 직접 조립하는 **'개발자 중심(Developer-driven)'**의 미니멀한 구조를 취한다.

## 정리

Spring Boot는 대규모 프로젝트에서 사용되던 `EJB`를 개선/대체하기 위해 나온 Spring을 계승하기에 대규모 프로젝트를 위한 여러 기능을 제공한다. 여기에 Spring Boot에서 더욱 편의성을 강화하여 여러 자동 설정(Auto-configuration), 의존성 관리 편의(각종 Starter 라이브러리)를 제공한다.

반면 FastAPI는 Python 타입 힌트와 Pydantic 모델을 활용해 데이터 검증, 변환, 직렬화와 API 문서 생성을 연결하여 반복 작업을 줄인다. 또한 ASGI 기반으로 비동기 I/O를 지원하며, 개발자가 필요한 라이브러리를 선택하고 조합할 수 있는 자율성을 제공한다.

따라서 규모가 큰 프로젝트거나 프레임워크가 말 그대로 프레임워크 역할을 해주는 것(레이어 권장, 설정, 의존성 관리)을 원한다면 `Spring Boot`.

규모가 작고, 가벼움과 자율성, 프레임워크가 최소한의 중복 제거만을 해주길 원한다면 `FastAPI`를 쓰면 좋을 것 같다.

## 참고 자료

- [Spring Framework 개요·등장 배경·설계 철학](https://docs.spring.io/spring-framework/reference/overview.html)
- [Spring 3.0 M3: Java 기반 Bean 설정 도입](https://spring.io/blog/2009/05/06/spring-framework-3-0-m3-released/)
- [Spring의 선언적 트랜잭션과 AOP](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-decl-explained.html)
- [JdbcTemplate의 자원 관리와 예외 처리](https://docs.spring.io/spring-framework/reference/data-access/jdbc/core.html)
- [Spring Boot의 목적과 기본 설정 방향](https://docs.spring.io/spring-boot/)
- [Spring Boot 자동 설정](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html)
- [Spring Boot의 Starter·BOM·의존성 관리](https://docs.spring.io/spring-boot/reference/using/build-systems.html)
- [FastAPI의 개발 배경과 설계](https://fastapi.tiangolo.com/history-design-future/)
- [FastAPI가 기존 도구에서 얻은 아이디어와 구성 요소](https://fastapi.tiangolo.com/alternatives/)
- [FastAPI의 타입 선언·검증·문서화·의존성 주입](https://fastapi.tiangolo.com/features/)
