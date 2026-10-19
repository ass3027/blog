업무에서 주로 사용하던 spring boot를 벗어나, 개인 프로젝트를 하며 fastapi를 많이 써보았다. 요새 많이 사용되는 두 BE framework를 비교하여, 각각에 대한, 그리고 framework에 대해 좀 더 이해하기 위해 글을 작성한다.

## 순서
1. 등장 배경
2. 지향점
3. 생명주기와 동시성 처리
4. DI와 제어권
5. 데이터 검증과 직렬화


## 1. 등장 배경

### Spring boot
기존 spring에 있던 불편 사항인 많은 xml 설정, WAS 별도 관리, 의존성 관리들을 해결하기 위해 등장했다

1. 너무 많은 xml 설정

    server.xml, web.xml 등 여러가지 xml상에서 설정들을 통해 bean을 생성하였기 때문에 xml 파일의 크기도 장난아니도 장난아니고, 소스와 구분된 위치에 있었기에 관리와 설정이 힘들다.

2. WAS 의존 배포

    빌드 결과물인 WAR파일을 별도 WAS(주로 Tomcat)에서 실행하였기에 별도 Tomcat 관리(설정, 옵션 값 튜닝, 업데이트)등을 하여야 했기에 프로젝트 관리 비용이 컷다

3. 의존성 관리 난이도

    Spring Hibernate, Jackson 등 많은 라이브러리의 버전 호환성을 개발자가 직접 맞춰야했고, 실수하면 바로 `ClassNotFoundException`이 발생해 라이브러리가 취약점이 있다 해도 잘 업데이트를 하지 않았다.

해결안
1. AutoConfiguration을 통해 많은 필요한 bean들을 자동으로 설정해 xml 설정 제거
2. Start POMs르 통해 `spring-boot-starter-web`만 추가해도 필요한, 검증된 버전의 라이브러리들을 관리 (ex spring-boot-starter-data-jpa)
3. Embedded Tomcat을 통해 별도의 WAS 없이 executable Jar로 단독 실행 가능


Spring boot는 기존 자바 프레임워크의 "과도한 설정과 관리, 개발 난이도"를 표준화된 자동 설정과 강력한 규격으로 통제한다


### FastAPI
기존 python web framework인 `Django`, `Flask`의 한계인 I/O 처리와 Microservice API를 해결하기 위해 등장했다.

1. 동기식 WSGI 구조로 인한 I/O 병목

    `Django`, `Flask`는 기존의 WSGI(Web Server Gateway Interface) 스펙으로 구축되었다. 이는 요청을 동기로 처리하기에 비동기 요청처리가 힘들다. 따라서 대규모 트래픽처리에서 처리량을 위해 자주 사용되는 비동기 처리에 불리하다
  

2. 타입 안정 없는 동적 타이핑

    Python 동적 타이핑의 특성상 IDE의 자동완성이 제대로 되지않아 에러를 런타임 시점에 알 수 있다.

해결안

1. ASGI(Asynchronous Server Gateway Interface)스펙과 `uvloop`을 이용해 `async/await` 키워드로 node.js, Go와 같이 비동기 I/O를 지원한다

2. Python의 정적 타입 힌트를 기반으로 Pydantic(데이터 검증 엔진)을 프레임워크에 포함


FastAPI는 기존의 "동기적 성능 한계와 데이터 처리의 번거로움을" ASGI와 타입 시스템을 통해 개선한다.


## 2. 지향점

### Spring boot

- "
