---
title: "[Spring 테스트] @SpringBootTest를 이용해 통합 테스트하기"
date: 2022-06-23
tags: ["SpringFramework", "Test", "SpringBootTest"]
draft: false
---

이번 게시글은 Spring에서 제공하는 테스트 라이브러리에 대한 간단한 설명과 @SpringBootTest를 이용한 통합 테스트를 하는 방법에 대해 정리해보고자 한다.

# 1. 테스트 의존성 추가하기

스프링은 테스트를 위해 아래의 2개의 Dependency를 제공하고 있다.

- **spring-boot-test**: core기능을 갖고 있다.
- **spring-boot-starter-test**: Auto-configuration을 지원한다.

## 1.2. spring-boot-starter-test가 포함하는 모듈들

- [JUnit 5](https://junit.org/junit5/): The de-facto standard for unit testing Java applications.
- [Spring Test](https://docs.spring.io/spring-framework/docs/5.3.20/reference/html/testing.html#integration-testing) & Spring Boot Test: Utilities and integration test support for Spring Boot applications.
- [AssertJ](https://assertj.github.io/doc/): A fluent assertion library.
- [Hamcrest](https://github.com/hamcrest/JavaHamcrest): A library of matcher objects (also known as constraints or predicates).
- [Mockito](https://site.mockito.org/): A Java mocking framework.
- [JSONassert](https://github.com/skyscreamer/JSONassert): An assertion library for JSON.
- [JsonPath](https://github.com/jayway/JsonPath): XPath for JSON.

spring-boot-starter-test는 이와 같이 많은 모듈들을 제공하기에 우리는 해당 의존성만을 추가하여도 테스트 환경을 쉽게 구축할 수 있다.

```java
dependencies {
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

# 2. Spring DI로 인해 편리해진 Test

DI의 장점으로는 단위 테스트를 하기 쉽다는 점이 있다. 객체의 의존성을 외부에서 주입을 해주기 때문에 우리는 테스트를 하며 진짜 객체가 아닌 Mock객체를 이용해 테스트 할 수 가 있다.

덕분에 통합 테스트를 진행할 때 실제 서비스가 연결되어있는 인프라(운영 DB 등)에 연결을 하지 않아도 테스트를 진행할 수 있다.

# 3. @SpringBootTest

스프링 부트는 통합 테스트를 쉽게 하도록 `@SpringBootTest` 어노테이션을 제공하고 있다. `@SpringBootTest`를 사용하면 테스트들은 모든 빈들을 생성하며 Application Context를 만들어 테스트한다. 덕분에 여러 컴포넌트간의 연결을 테스트하는 통합 테스트를 진행할 수 있으나 테스트에 필요 없는 빈들까지 모두 등록을 하기때문에 테스트 자체가 무겁다는 단점이 있다. 만약 컨트롤러 테스트를 진행하고 싶은데 테스트에 필요한 빈들만을 등록하여 테스트를 하고 싶다면 `@WebMvcTest`를 사용하여 테스트 하는 방법도 존재한다.

이번 게시글에서는 먼저 `@SpringBootTest`를 사용한 테스트 방법에 대해 자세히 알아보겠다.

> 스프링 테스트 코드들을 보면 `@RunWith(SpringRunner.class)`, `@ExtendWith(SpringExtension.class)` 등이 붙은 코드들을 본 적이 있을 것이다. `@RunWith(SpringRunner.class)`는 JUnit 4버전에 필수적으로 들어가야하는 코드로 JUnit5에는 붙이지 않아도 된다. 그리고 `@ExtendWith(SpringExtension.class)`는 `@SpringBootTest`에 기본적으로 들어가있어서 `@SpringBootTest`를 사용하는 경우 붙이지 않아도 된다.
> 즉, 우리가 최신 버전의 SpringBoot버전으로 `@SpringBootTest`를 사용할 경우 두 어노테이션들은 불이지 않아도 된다.

## 3.1. webEnvironment

`@SpringBootTest`는 기본적으로 서버를 실행시키지 않고 테스트를 진행한다. 이를 설정하기 위해서는 webEnvironment옵션을 설정해야 한다.

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class IntegrationTest {}
```

- **MOCK**
  - Default값으로 WebApplicationContext를 로드하여 **가짜 웹 환경**을 만들어 테스트한다. 해당 설정으로는 **Embedded server(내장된 서블릿 컨테이너)를 실행하지 않고 테스트를 Mock Servlet을 만들어 테스트 하는 것으로 대체**됩니다.
  - 테스트를 Mock기반의 테스트를 하기 때문에 다음 어노테이션을 같이 사용할 수 있다.
    - [`@AutoConfigureMockMvc` or `@AutoConfigureWebTestClient`](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.testing.spring-boot-applications.with-mock-environment)
    - `@AutoConfigureMockMvc`를 사용하면 별다른 설정 없이 컨트롤러 테스트를 용이하게 해주는 MockMvc를 사용해 테스트 할 수 있다.
    - `@AutoConfigureMockMvc`는 Mock 테스트시 필요한 의존성을 제공해준다. `MockMvc`객체를 통해 실제 컨테이너를 실행하는 것은 아니지만 로직상으로 테스트 가능하다.
- **RANDOM_PORT**
  - `WebServerApplicationContext`를 로드하고 랜덤한 포트에 실제 웹 환경과 동일한 환경(서블릿 환경)을 열어 테스트한다.
- **DEFINED_PORT**
  - RANDOM_PORT와 동일하게 구동되나 테스트 환경이 랜덤 포트가 아닌 `application.properties`에서 사전에 정의한 포트를 사용해 테스트한다. (default port는 8080이다.)
- **NONE**
  - SpringApplication를 통해 ApplicationContext를 로드하나 mock이나 실제 환경과 같은 웹 환경을 제공하지 않는다.

> 테스트들은 `@Transactional`이면 각각의 테스트들이 끝난 후 자동으로 roll-back됩니다. 하지만 `WebEnvironment.RANDOM_PORT, DEFINED_PORT`는 실제 테스트는 별도의 스레드에서 테스트를 수행하여 roll-back되지 않는다.🤡

### 3.1.1. Detecting Test Configuration**

SpringBoot는 특정 `@ContextConfiguration(loader=...)`이 정의되지 않은 경우 SpringBootContextLoader를 기본 ContextLoader를 사용합니다. `@Configuration`이 사용되지 않고 명시적 클래스가 지정되지 않은 경우 `@SpringBootConfiguration`을 자동으로 검색합니다.

> Spring Test Framework는 테스트의 ApplicationContext를 캐싱합니다. 덕분에 동일한 Configuration을 갖게 된다면 Context를 로드하는 작업은 한번만 하게 됩니다.

## Testing with a mock environment

`@SpringBootTest`는 앞서 말했듯이 webEnvironment를 설정하지 않으면 기본적으로 Mock 서블릿 환경을 만들어 테스트를 한다.

이를 테스트 하기 위해서는 MockMvc와 WebTestClient를 사용할 수 있다.

```java
import org.junit.jupiter.api.Test;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.reactive.server.WebTestClient;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@SpringBootTest
@AutoConfigureMockMvc
class MyMockMvcTests {

    @Test
    void testWithMockMvc(@Autowired MockMvc mvc) throws Exception {
        mvc.perform(get("/")).andExpect(status().isOk()).andExpect(content().string("Hello World"));
    }

    // If Spring WebFlux is on the classpath, you can drive MVC tests with a WebTestClient
    @Test
    void testWithWebTestClient(@Autowired WebTestClient webClient) {
        webClient
                .get().uri("/")
                .exchange()
                .expectStatus().isOk()
                .expectBody(String.class).isEqualTo("Hello World");
    }

}
```

- 모든 빈들을 만들며 ApplicationContext을 생성하지 않고 필요한 빈들만을 등록하며 Web Layer의 테스트에 집중하기 위해서는 `@WebMvcTest`를 사용할 수 있다.

Mock환경에서 테스트하는 것은 실제 Servlet Container를 만들어 테스트하는 것보다 빠르다. 하지만 Spring MVC 계층에서 Mocking이 발생하기에 더욱 low level인 Servlet Container에 대한 테스트는 MockMvc로 테스트할 수가 없다.

- ex) 서블릿 컨테이너가 제공하는 error page가 랜더링 되는지 등의 테스트를 할 수 없다. 이를 테스트 하기 위해서는 실제 서버로 테스트해야한다.

## Testing with a running server

실제 Servlet Container에서 테스트를 하기 위해서는 `WebEnvironment.RANDOM_PORT, DEFINED_PORT`에서 테스트를 해야한다.

`@LocalServerPort`주석을 사용 하여 테스트에 사용된 실제 [포트를 삽입 할 수 있습니다.](https://docs.spring.io/spring-boot/docs/current/reference/html/howto.html#howto.webserver.discover-port) 편의를 위해 시작된 서버에 대한 REST 호출을 수행해야 하는 테스트 는 다음 예제와 같이 실행 중인 서버에 대한 상대 링크를 확인하고 응답 확인을 위한 전용 API와 함께 제공되는 [`WebTestClient`](https://docs.spring.io/spring-framework/docs/5.3.20/reference/html/testing.html#webtestclient-tests) 를 `@Autowire`로 불러올 수 있다.

```java
import org.junit.jupiter.api.Test;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;
import org.springframework.test.web.reactive.server.WebTestClient;

@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class MyRandomPortWebTestClientTests {

    @Test
    void exampleTest(@Autowired WebTestClient webClient) {
        webClient
            .get().uri("/")
            .exchange()
            .expectStatus().isOk()
            .expectBody(String.class).isEqualTo("Hello World");
    }

}
```

# Reference

- [Core Features 8. Testing](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.testing)
