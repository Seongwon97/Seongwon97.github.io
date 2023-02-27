---
title: "[Spring] @RestController와 @Controller 비교하기"
date: 2022-06-21
tags: ["SpringFramework", "Spring MVC", "Controller", "RestController"]
draft: false
---

# 1. @RestController와 @Controller

스프링은 컨트롤러 어노테이션으로 `@RestController`와 `@Controller`어노테이션을 제공하고 있다. 이 두 어노테이션은 사용자의 요청을 받아 비즈니스 로직을 매핑해주고 결과를 반환해준다는 존재의 목적은 같지만 기본 반환값이 다르다는 차이점이 있다.

먼저 `@RestController`와 `@Controller` 코드를 비교해보겠다.

![](image/20220621_Controller와_RestController/controllerCode.png)

![](image/20220621_Controller와_RestController/restControllerCode.png)

두 코드를 비교해보면 `@RestController`에 `@ResponseBody` 어노테이션이 붙었다는 것 외에 다른 점을 찾을 수 없다.

ResponseBody는 무엇일까? 전통적인 Spring MVC에서는 클라이언트가 요청을 하면 요청 값에 따른 View를 반환해줬다. 하지만 최근에는 프론트와 백엔드가 분리되며 백엔드 단에서는 View를 만들어 반환해주는 것이 아닌 요청에 따른 데이터만을 Http Response Body에 담아 반환해주는 형태를 추구하고 있다. 컨트롤러에서 간편하게 Http Response Body에 데이터를 담는 것을 지원하기 위해 스프링은 `@ResponseBody`어노테이션을 제공하고 있다.

본론으로 들어와 `@RestController`에 `@ResponseBody`의 차이를 통해 유추할 수 있는 것은 기본적으로 `@Controller`의 경우 View를 반환하며 `@RestController`는 Json타입의 ResponseBody를 반환한다는 것을 알 수 있다. (`@Controller`에서 Response Body를 반환하려면 `@ResponseBody` 어노테이션을 붙여줘야 한다. )

즉, RestController는 Controller + ResponseBody라고 봐도 된다.

# 2. 반환값에 따른 동작 방식 차이 비교 및 코드 작성 예시

## 2.1. View 반환하기

![](image/20220621_Controller와_RestController/withViewResolver.png)

앞서 말했듯이 @Controller는 기본적으로 클라이언트의 요청에 대해 View를 반환해준다. View를 반환하는 순서는 다음과 같다.

자세한 동작 순서는 [MVC 동작 방식 이해하기](https://seongwon.dev/Spring-MVC/20220621-%EC%8A%A4%ED%94%84%EB%A7%81MVC-%EB%8F%99%EC%9E%91%EB%B0%A9%EC%8B%9D/) 게시글에서 다루었기에 생략하고 컨트롤러가 반환해주는 4번 과정부터 보겠다.

1. 컨트롤러 메서드는 View의 이름을 반환한다.
2. Handler Adapter는 ReturnValueHandler를 호출하여 View이름을 통해ModelAndView를 만들어 Dispatcher Servlet에 반환해준다.
3. DispatcherServlet은 ViewResolver를 호출하여 View를 찾아 렌더링 후 반환한다.

### 2.1.1 코드 사용 예시 @Controller

```java
@Controller
@RequestMapping("/api/products")
public class ProductController {
		...
    @GetMapping("/{productId}")
    public String product(Model model, @PathVariable long productId) {
        String productName = productService.findProduct(productId).getName();
				model.addAttribute("productName", productName);
        return "product";
    }
}
```

## 2.2. ResponseBody 반환하기

![](image/20220621_Controller와_RestController/withMessageConverter.png)

ResponseBody를 통해 데이터를 반환하는 경우 View를 반환하는 과정과 다르다. 이번에도 Controller에서의 반환값 이후의 과정부터 살펴보겠다.

1. 컨트롤러는 결과 값(data)를 ResponseEntity로 감싸서 반환한다.
2. ResultValueHandler는 ResponseEntity를 받아 MessageConverter를 통해 값을 변경 후 반환해준다. → Json으로 직렬화(Serialize)

- 변환은 Http Accept 헤더와 컨트롤러가 반환한 객체의 타입에 따라 StringHttpMessageConverter, MappingJackson2HttpMessageConverter등의 알맞은 Converter가 동작하게 된다.

3. 결과 값을 사용자에게 반환한다.

### 2.2.1 코드 사용 예시 @Controller

`@Controller`에서 View가 아닌 데이터를 담은 ResponseBody를 반환하려면 아래와 같이 메서드에 `@ResponseBody`를 붇여주면 된다.

```java
@Controller
@RequestMapping("/api/products")
public class ProductController {
		...
    @GetMapping("/{productId}")
    @ResponseBody
    public ResponseEntity<ProductResponse> product(@PathVariable long productId) {
        return ResponseEntity.ok(productService.findProduct(productId));
    }
}
```

### 2.2.2 코드 사용 예시 @RestController

`@RestController`는 `@Controller`+`@ResponseBody`라고 말했듯이 메서드 위에 붙였던 `@ResponseBody` 어노테이션이 생략 가능하다. 코드로 표현하면 다음과 같다.

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {
		...
    @GetMapping("/{productId}")
    public ResponseEntity<ProductResponse> product(@PathVariable long productId) {
        return ResponseEntity.ok(productService.findProduct(productId));
    }
}
```

# 3. 어떤 어노테이션을 사용하는 것이 좋을까?

REST API만을 제공하는 프로젝트가 아니라면 View를 직접 반환하여야 하는 상황도 존재할 것이다. 이러한 경우 `@Controller`, `@RestController`중 어떤 어노테이션을 사용할지 고민될 것이다.

개발에 정답이라는 것은 없지만 나는 `@Controller`, `@RestController`가 달린 컨트롤러 클래스를 두개 생성한 후, view와 관련된 메서드는 `@Controller`가 달린 클래스에 위치시키고 다른 REST API의 메서드들은 `@RestController`에 위치시키는 방법을 추천한다.

이와 같이 View를 관리하는 컨트롤러, 데이터를 반환하는 REST API를 제공하는 컨트롤러로 분리를 하여 관리할 경우 추후 유지 보수도 쉬울뿐더러 코드의 가독성도 올라갈 것이다.

# Reference

- [[Spring] @Controller와 @RestController 차이 - MangKyu's Diary:티스토리](https://mangkyu.tistory.com/49)
- [[Spring] Message Converter](https://vprog1215.tistory.com/279?category=1033244)
- [@Controller와 @RestController의 차이점](https://dncjf64.tistory.com/288)
