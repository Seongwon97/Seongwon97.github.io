---
title: "[Spring] ArgumentResolver란 무엇일까?"
date: 2022-06-29
tags: ["SpringFramework", "ArgumentResolver"]
draft: false
---

# 1. ArgumentResolver란?

ArgumentResolver는 어떠한 요청이 들어왔을 때 요청 객체로부터 원하는 객체를 만들어내는 일을 한다. 사실 우리도 지금껏 알게 모르게 ArgumentResolver를 사용해왔다. 우리가 요청의 Query Paramter, body, header 등에서 데이터를 추출하기 위해 사용하였던 아래의 스프링 어노테이션들은 모두 ArgumentResolver로 동작을 하고 있었다.

- `@RequestParam`: 쿼리 파라미터 값 바인딩
- `@ModelAttribute`: 쿼리 파라미터 및 폼 데이터 바인딩
- `@RequestBody`: 바디값 바인딩
- `@CookieValue`: 쿠키값 바인딩
- `@RequestHeader`: 헤더값 바인딩

이러한 ArgumentResolver는 AOP, Interceptor, Filter와 같이 Spring에서 중복 코드를 제거해주는 장점을 갖고 있다. (해당 장점은 ArgumentResolver의 용도 부분을 읽으면 이해가 될 것이다.)

# 2. ArgumentResolver의 용도

JWT를 통해 인증/인가 기능을 구현한 애플리케이션을 개발한다고 생각해보자. JWT를 사용한다면 요청이 들어왔을 때 애플리케이션은 해당 토큰이 애플리케이션에서 발급된 유효한 토큰인지 검사하는 작업과 토큰의 값을 파싱하여 유저 객체를 만들어내는 작업을 진행할 것이다. 만약 ArgumentResolver를 사용하지 않으면 해당 작업은 JWT 토큰과 함께 요청을 받는 모든 컨트롤러에 구현해야할 것이다. 이러한 경우 중복코드가 발생하고 컨트롤러의 책임이 증가하게 된다. ArgumentResolver를 사용한다면 이러한 문제점을 해결할 수 있다.

# 3. ArgumentResolver까지 Spring MVC 동작 과정

1. 사용자가 Http Request를 보낸다.
2. 등록된 Filter Chain이 실행되는데, 사용자가 보낸 Request Url에 매칭되는 Filter가 있으면 순차적으로 실행된다.
3. Dispatcher Servlet이 요청을 받으면 Handler Mapping을 통해 요청을 처리할 수 있는 Handler(컨트롤러)와 요청에 매핑되는 Interceptor들을 찾아 HandlerMethodExecutionChain을 만들어 반환한다.
4. HandlerMethodExecutionChain에 있는 Interceptor들이 순차적으로 실행된다.
5. <font color="FF3366">HandlerAdapter가 Argument Resolver를 호출하여 Request에서 필요한 데이터를 추출하여 객체로 반환한다.</font>
   - <font color="FF3366">이때 ArgumetResolver에서는 HTTPMessageConverters를 호출하여 요청 값과 생성 객체 값들의 타입을 확인하여 알맞은 객체를 생성하고 반환해준다.</font>
6. Handler(컨트롤러) 메서드를 호출하여 실행한다.
7. …

# 4. ArgumentResolver 사용하기

ArgumentResolver를 만들기 위해서는 `HandlerMethodArgumentResolver` 인터페이스를 구현하여야 한다.

## 4.1. ArguementResolver의 메서드

```java
public interface HandlerMethodArgumentResolver { 
    
    boolean supportsParameter(MethodParameter parameter);
    
    @Nullable 
    Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, 
                            NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception;

}
```

- `supportsParameter`는 요청받은 메서드의 인자에 원하는 어노테이션이 붙어있는지 확인하고 원하는 어노테이션을 포함하고 있으면 true를 반환한다.
- `resolveArgument`는 `supportsParameter`에서 true를 받은 경우, 즉, 특정 어노테이션이 붙어있는 어느 메서드가 있는 경우 parameter가 원하는 형태로 정보를 바인딩하여 반환하는 메서드이다

## 4.2. 적용하기

📌 우리는 ArgumentResolver를 구현하고 실행되기를 원하는 Parameter앞에 어노테이션을 붙여주면 ArgumentResolver가 적용된다.

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface AuthenticationPrincipal {
}
```

```java
public class AuthenticationPrincipalArgumentResolver implements HandlerMethodArgumentResolver {

    private final AuthService authService;

    public AuthenticationPrincipalArgumentResolver(AuthService authService) {
        this.authService = authService;
    }

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(AuthenticationPrincipal.class);
    }

    @Override
    public Long resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) {
        HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
        return authService.extractIdFromRequest(request);
    }
}
```

먼저 `AuthenticationPrincipal` 어노테이션을 생성한 후, `supportsParameter()`메서드를 통해 해당 어노테이션을 사용하는지 인지하도록 설정을 한다. 그 후 `resolveArgument()`메서드를 통해 request로부터 값을 추출 후 반환하도록 설정한다.

```java
@Configuration
public class AuthenticationPrincipalConfig implements WebMvcConfigurer {

		...

    @Override
    public void addArgumentResolvers(List argumentResolvers) {
        argumentResolvers.add(createAuthenticationPrincipalArgumentResolver());
    }

    @Bean
    public AuthenticationPrincipalArgumentResolver createAuthenticationPrincipalArgumentResolver() {
        return new AuthenticationPrincipalArgumentResolver(authService);
    }
}
```

`WebMvcConfigurer`를 구현한 설정 클래스를 통해 argumentResolver를 추가한다.

```java
@RestController
@RequestMapping("/api/members")
public class MemberController {

    private final MemberService memberService;

    public MemberController(MemberService memberService) {
        this.memberService = memberService;
    }

    @GetMapping("/me")
    @ResponseStatus(HttpStatus.OK)
    public FindMemberInfoResponse findMemberInfo(@AuthenticationPrincipal long id) {
        return memberService.findMemberInfo(id);
    }
}
```

이제 앞서 만들었던 `AuthenticationPrincipal`어노테이션을 사용하면 request에서 특정 값을 추출하여 반환을 하게 된다.

# 5. Interceptor와 ArgumentResolver의 차이

ArgumentResolver는 인터셉터 이후에 동작을 하며, 어떠한 요청이 컨트롤러에 들어왔을 때, 요청에 들어온 값으로부터 원하는 객체를 반환하기위해 사용한다. 반면에 인터셉터는 실제 컨트롤러가 실행되기 전에 요청을 가로채며 특정 객체를 반환할 수 없고 오직 boolean 혹은 void 반환 타입만 존재한다. 하지만 파라미터로 받는 객체의 값을 변환하여 컨트롤러에 데이터를 전달할 수는 있다.

사실 내 개인적인 생각으로는 둘의 사용 목적 자체가 다르다고 생각된다. Interceptor에서도 물론 원하는 특정 객체를 직접 반환하는 것이 아닌 파라미터 객체 값을 변환하는 방법으로 컨트롤러에 데이터를 넘겨줄 수 있지만, 스프링에서 해당 방법을 권장한다면 ArgumentResolver 자체가 만들어지지 않았을 것이다. Interceptor에서 컨트롤러로 데이터를 가공하여 넘겨주는 것은 단순히 가능한 작업일 뿐, 권장하지는 않는 작업이라 생각한다.

결론적으로 Request로부터 특정 객체를 추출하는 작업은 ArgumentResolver에서 수행하고 Interceptor는 인증/인가, 로깅 등의 본인만이 할 수 있는 작업을 진행하는 것이 가장 좋다고 생각된다.

# Reference

- [Spring ArgumentResolver와 Interceptor](https://tecoble.techcourse.co.kr/post/2021-05-24-spring-interceptor/)
- [[Spring] Filter, Interceptor, Argument Resolver란?](https://steady-coding.tistory.com/601)
- [A Custom Data Binder in Spring MVC](https://www.baeldung.com/spring-mvc-custom-data-binder)
- [[Spring] @RequestBody에 ArgumentResolver(아규먼트 리졸버)가 동작하지 않는 이유, RequestBodyAdvice로 @RequestBody에 부가 기능 구현하기](https://mangkyu.tistory.com/250)
- [[Spring MVC] HTTP Message Converters (HTTP 메시지 컨버터)](https://maenco.tistory.com/entry/Spring-MVC-HTTP-Message-Converters-HTTP-%EB%A9%94%EC%8B%9C%EC%A7%80-%EC%BB%A8%EB%B2%84%ED%84%B0)
