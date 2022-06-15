---
layout: post
title: 스프링 - @RequestParam, @RequestBody, @ModelAndAttribute 차이 비교
date: '2022-06-15 22:59:10 +0900'
description: '@RequestParam, @RequestBody, @ModelAndAttribute 차이가 무엇일까?'
categories: [SpringFramework, 개념 정리]
tags: [SpringFramework]
---

@RequestParam, @RequestBody, @ModelAndAttribute는 모두 request를 보내며 어떠한 정보(데이터)를 담아 보내기 위해 사용된다.

# @RequestParam

- 공식문서

  For access to the Servlet request parameters, including multipart files. Parameter values are converted to the declared method argument type.

    ```java
    @Controller
    @RequestMapping("/pets")
    public class EditPetForm {

        // ...

        @GetMapping
        public String setupForm(@RequestParam("petId") int petId, Model model) {
            Pet pet = this.clinic.loadPet(petId);
            model.addAttribute("pet", pet);
            return "petForm";
        }

        // ...
    }
    ```

- 1개의 HTTP 요청 파라미터를 받기 위해 사용한다.
- required의 default값이 true이기에 반드시 값이 전송되어야 한다.
  - 해당 값이 없다면 400 error가 발생한다.
  - 필수가 아니도록 설정하려면 required값을 false로 설정하면 된다.
- default값도 설정 가능하다.
- argument의 타입을 array나 list로 선언하면 동일한 매개변수 이름에 대한 여러 값들이 저장된다.
- @RequestParam 이 붙은 매개변수에 특정한 이름이 없이 Map<String, String>, MultiValueMap<String, String> 으로 선언이 된다면, 넘어오는 값들로 해당 map이 채워진다.

- Note that use of `@RequestParam` is optional (for example, to set its attributes). By default, any argument that is a simple value type (as determined by [BeanUtils#isSimpleProperty](https://docs.spring.io/spring-framework/docs/5.3.19/javadoc-api/org/springframework/beans/BeanUtils.html#isSimpleProperty-java.lang.Class-)) and is not resolved by any other argument resolver, is treated as if it were annotated with `@RequestParam`

  → 기본적으로 [BeanUtils#isSimpleProperty](https://docs.spring.io/spring-framework/docs/5.3.19/javadoc-api/org/springframework/beans/BeanUtils.html#isSimpleProperty-java.lang.Class-) 에게 결정되는 단순한 값 유형들은 다른 argument resolver(즉, 다른 api 메서드)에게 해석되지 않으면 해당 값 유형은 `@RequestParam`이 붙은 것처럼 다뤄지게 된다.


## @RequestBody

- 공식 문서

  You can use the `@RequestBody` annotation to have the request body read and deserialized into an `Object` through an HttpMessageConverter.

    ```java
    @PostMapping("/accounts")
    public void handle(@RequestBody Account account) {
        // ...
    }
    ```

  You can use the [Message Converters](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-config-message-converters) option of the [MVC Config](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-config) to configure or customize message conversion.

- request의 body에 있는 Json(application/json)형태의 데이터를 Java 객체로 변환시켜준다. (값을 주입이 아닌 변환!!)
  - Body의 값이 필요하기에 Body가 없는 HTTP Get메서드의 경우 RequestBody를 사용하면 에러가 발생한다.
- Body의 데이터는 Spring에서 관리하는 MessageConverter들 중 하나인 `HttpMessageConverter`를 통해 Java 객체로 변환된다.
  - message converter의 옵션은 customize할 수도 있다.
- **Json데이터를 객체로 변환하는 과정에서는 기본 생성자를 사용하여 객체를 생성한다.**
  - 기본 생성자가 없다면 에러가 발생한다.
  - 데이터를 변환할 때는 reflection을 사용해 값을 할당하여 dto에는 값들을 주입하는 생성자나 setter가 필요없다.
  - Json 데이터를 변환할 때는 Jackson라이브러리가 내부적으로 Getter나 Setter, @JsonInclude 등을 통해 필드에 있는 변수들의 이름을 찾고, Reflection을 이용해 값을 할당한다.

## @ModelAttrubute

- For access to an existing attribute in the model (instantiated if not present) with data binding and validation applied.
- request body에 있는 multipart/form-data 형태의 내용과 HTTP 파라미터의 값들을 생성자나 setter를 통해 1대1로 주입해준다. (변환이 아닌 주입!!!!)
  - 📌 RequestBody와 다르게 **데이터 주입을 위한 생성자와 Setter가 필요하다.**
  - 값을 주입해주는 생성자나 setter함수가 없다면 매핑을 시키지 못하고 null값을 갖게 된다.
  - 인수 이름이 일치하는지에 대한 판단은 JavaBeans `@ConstructorProperties` 나 런타임에 유지되는 매개변수 이름을 통해 결정된다.
  - request body와 http parameter의 값들을 1:1로 바인딩 시켜서 아래와 같이 request body에 일부 데이터를 넣고, parameter로 다른 값을 넣어줘도 값이 올바르게 주입된다.

- 매핑시키는 파라미터의 타입이 객체의 타입과 일치하는지와 같은 다양한 검증이 추가적으로 진행된다.
  - ex) 게시물의 번호를 저장하는 int형 index 변수에 "1번" 이라는 String형을 넣으려고 한다면, BindException이 발생하게 된다.
  - 검증 - 먼저 해당 필드를 인자로 받는 생성자가 있는지 검사 후, 생성자가 있으면 해당 생성자를 이용하고 없다면 setter를 이용해 값을 할당해준다.
- 특정 parameter만을 받아올 수도 있다.
