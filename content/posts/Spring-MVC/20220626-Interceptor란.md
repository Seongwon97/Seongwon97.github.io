---
title: "[Spring] Interceptor란 무엇일까?"
date: 2022-06-27
tags: ["SpringFramework", "Interceptor"]
draft: false
---

스프링에서 공통 기능을 제거하는 대표적인 방법으로는 Filter, Interceptor, AOP가 있다. 이전 포스트인 [Filter란 무엇인가?](https://seongwon.dev/Spring-MVC/20220625-Filter%EB%9E%80/) 에 이어서 이번 포스트에서는 공통 기능 제거 방법 중 Interceptor에 대해 알아볼 것이다.

# 1. Intetceptor란?

Interceptor는 filter처럼 경로를 지정하여, 해당 경로에 매칭되는 request에 대하여 컨트롤러의 실행 전/후에 동작을 한다. Interceptor는 Filter와 다르게 J2EE의 표준 스펙 기능이 아닌 Spring에서 제공하는 기능이며 둘의 실행 시점이 다르다. Filter는 spring context외부인 dispatcher servlet으로 request가 가기 전에 web context내에서 실행되지만 interceptor은 스프링이 제공하는 기능이라 Spring Context내부에서 dispatcher servlet이 요청을 받은 이후에 실행이 된다.

![](image/20220626_Interceptor란/interceptor.png)

Interceptor도 Filter Chain처럼 여러개의 Interceptor가 등록되어 있으면 순차적으로 동작하게 된다. Dispatcher Servlet에서부터 Interceptor의 실행 과정은 다음과 같다. Dispatcher Servlet은 HandlerMapping을 통해 요청에 맞는 컨트롤러를 찾도록 해주면 결과로 HandlerExecutionChain을 반환해준다. 이때 HandlerExecutionChain 안에 1개 이상의 등록되어있는 Interceptor가 있다면 순차적으로 Interceptor를 거치게되며 그 후에 HandlerAdapter가 handler(controller)를 실행한다. 만약 Interceptor가 없으면 컨트롤러가 바로 실행된다.

> Dispatcher Servlet, HandlerMapping등의 내용들이 생소하다면 [다음 게시글](https://seongwon.dev/ETC/20211207_MVC%ED%8C%A8%ED%84%B4%EC%9D%B4%EB%9E%80/)을 참조하길 바란다.

# 2. Interceptor의 메서드

스프링은 Interceptor를 구현하는 방법으로 `HandlerInterceptor` **인터페이스를 구현**하는 방법과 `HandlerInterceptorAdapter` **클래스를 상속** 받는 방법을 제공하였다.

하지만 현재 스프링은 `HandlerInterceptorAdapter` 클래스를 deprecated로 지정하며 이제는 `HandlerInterceptor` 인터페이스를 구현하는 방법만을 사용하여야 한다.

![](image/20220626_Interceptor란/deprecatedHandlerInterceptorAdapter.png)

`HandlerInterceptor` 인터페이스는 `preHandle()`, `postHandle()`, `afterCompletion()` 메서드를 default 메서드로 갖고 있으며 우리는 Interceptor의 사용 목적에 따라 필요한 메서드를 재정의해주면 된다.

```java
package org.springframework.web.servlet;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.springframework.lang.Nullable;
import org.springframework.web.method.HandlerMethod;

public interface HandlerInterceptor {

    default boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
        throws Exception {

        return true;
    }

    default void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler,
        @Nullable ModelAndView modelAndView) throws Exception {
    }

    default void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler,
        @Nullable Exception ex) throws Exception {
    }
}
```

- `preHandle()`: 컨트롤러 호출 전에 실행되는 메서드로 요청 정보를 가공하거나 추가하는 전처리 작업을 위해 사용한다. 반환 타입이 boolean형인 것을 확인할 수 있는데, true를 반환하게 된다면 다음 Interceptor를 실행하게 되고, false면 작업을 중단하고 이후의 Interceptor와 컨트롤러를 실행하지 않는다.
  - 매개변수로 있는 handler객체는 Dispatcher Sevlet의 HandlerMapping이 찾아준 컨트롤러의 메서드를 참조할 수 있는 HandlerMethod객체이다.
- `postHandle()`: 컨트롤러 실행 후 View가 생성되기 이전에 실행되는 메서드로 후처리 작업에 사용을 한다. 보통은 ModelAndView 타입의 정보가 제공되어 이를 후처리하기 위해 사용하였는데, 최근에는 RestAPI 들을 만들며 자주 사용되지 않는다.
- `afterCompletion()` : View 처리를 포함한 모든 작업이 완료된 후에 실행된다.

![](image/20220626_Interceptor란/interceptorMethod.png)

# 3. Interceptor 사용하기

앞서 말한바와 같이 Interceptor를 구현하려면 먼저 `HandlerInterceptor`를 구현한 클래스를 작성해야합니다.

> 사용 예시를 작성하기 위해 [우아한테크코스-장바구니 미션 코드](https://github.com/Seongwon97/woowacourse-jwp-shopping-cart)의 일부를 수정하여 작성해봤습니다.

## 3.1. Interceptor 만들기

아래의 코드는 웹 서비스에서 로그인이 된 이후에 가능한 서비스들에 대해 토큰 값이 유효한지 확인을 해주는 인터셉터 코드입니다.

```java
@Slf4j
public class TokenInterceptor implements HandlerInterceptor {

    private final AuthService authService;

    public TokenInterceptor(AuthService authService) {
        this.authService = authService;
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        log.info("======= TokenInterceptor 실행 =======");
        if (authService.isValidToken(request)) {
            return true;
        }
        return false;
    }
}
```

코드를 보면 해당 인터셉터는 `HandlerInterceptor`인터페이스를 구현하였으며 preHandle메서드를 재정의하며 컨트롤러로 가기 전에 요청의 token 값이 유효한지 검사를 하고 있는 것을 알 수 있다.

> 테스트를 위해 `@Slf4j`를 통해 로그를 남겼다.

## 3.2. Interceptor 등록하기

이제 Interceptor가 동작하도록 등록해보겠다. Interceptor를 등록하기 위해서는 `WebMvcConfigurer` 클래스를 상속받은 설정 클래스를 만든 후 `addInterceptors()`메서드를 재정의하며 등록하여야 한다.

```java
@Configuration
public class AuthenticationPrincipalConfig implements WebMvcConfigurer {

    private final AuthService authService;

    public AuthenticationPrincipalConfig(AuthService authService) {
        this.authService = authService;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new TokenInterceptor(authService))
                .addPathPatterns("/api/members/me/**");
    }
}
```

Interceptor의 등록은 `addInterceptors()`메서드의 매개변수로 있는 InterceptorRegistry에 `addInterceptor()`메서드를 통해 추가를 해주면 된다.

Interceptor등록 및 설정에 대한 메서드들은 `InterceptorRegistration`클래스에서 제공을 하고 있다. 해당 클래스에서 제공하는 메서드들은 다음과 같다.

### 3.2.1. Interceptor 추가 설정 메서드

- `addPathPatterns()`: 인터셉터가 실행될 Url패턴을 설정한다.
- `excludePathPatterns()`: 실행을 제외하고 싶은 Url 패턴을 설정한다.
- `pathMatcher()`: 인터셉터가 실행되거나 제외할 Url 패턴을 설정한 PathMatcher를 인자로 주어 실행될 Url 패턴을 설정한다.
- `order()`: 인터셉터의 실행 순서를 정할 수 있다.

## 3.3. 실행 결과 확인

인터셉터를 추가하며 설정한 Url패턴에 해당하는 경로로 요청을 보냈더니 다음과 같이 인터셉터가 정상적으로 실행되는 것을 확인할 수 있었다.

![](image/20220626_Interceptor란/practiceResult.png)

# Reference

- [[Spring] 필터(Filter) vs 인터셉터(Interceptor) 차이 및 용도 - MangKyu's Diary:티스토리](https://mangkyu.tistory.com/173)
- [[Spring] 스프링 인터셉터(Interceptor)란 ?](https://popo015.tistory.com/115)
- [Spring ArgumentResolver와 Interceptor](https://tecoble.techcourse.co.kr/post/2021-05-24-spring-interceptor/)
