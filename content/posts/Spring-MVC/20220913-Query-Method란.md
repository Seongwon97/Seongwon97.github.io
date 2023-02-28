---
title: "[JPA] Spring Data JPA의 Query Method"
date: 2022-09-13
tags: ["SpringFramework", "JPA", "Spring Data JPA", "Query Method"]
draft: false
---
# 1. Spring Data JPA에서 쿼리를 만드는 방법

[이전 게시글](https://seongwon.dev/Spring-MVC/20220911-Spring-Data-JPA%EB%9E%80/)에서 Spring Data JPA에 대해 알아보며 해당 모듈은 Data Access Layer의 구현을 최대한 개선하는 것을 목표로 만들어졌다고 하였다. 또한 Data JPA에서 제공하는 `JpaRepository` 에서는 CURD기능들과 페이징, 정렬 기능등의 DB와의 소통을 쉽게 할 수 있는 메서드를 제공한다고 하였다. 하지만 앞서 살펴봤던 메서드들은 모두 `findAll`, `findById`와 같은 기본적인 기능이었다. 실제 애플리케이션을 개발하면 DB내용의 전체 조회, ID를 통한 조회 외에도 상황에 맞는 조건에 따라 많은 조회 메서드가 필요할 것이다.

기본 제공 메서드 외에 추가로 필요한 쿼리문 메서드들은 개발자가 직접 만들어서 사용할 수 있다. Repository에서 `JpaRepository`에서 제공하는 기본적인 메서드 외에도 레포지토리에 존재하는 메서드로부터 쿼리를 파생시키는 두 가지 방법이 있다.

- 메서드 이름을 통해 쿼리를 만든다.
- `@Query` 어노테이션을 통해 수동으로 쿼리를 정의한다.

2번 방법은 앞서 학습한 [JPQL](https://seongwon.dev/Spring/20220829-JPA-JPQL/)을 통한 쿼리 생성 방법이다. 이번 포스트와는 거리가 좀 있는 내용이라 간단하게만 살펴보겠다.

```java
String query = "select u.name from User u where u.name = :name";
List<Member> result = em.createQuery(query, Member.class)
                        .setParameter("name", "Rex")
                        .getResultList();
```

순수 JPA를 사용한다면 JPQL을 작성할 때는 위와 같이 코드를 작성해야 했을 것이다. 하지만 Spring Data JPA를 사용한다면 위의 코드가 아래와 같이 간단하게 변경될 수 있다.

```java
@Query("select u.name from User u where u.name = :name")
List<Member> result findByUsername(String username);
```

JPQL을 통한 쿼리 생성이 아닌 또 다른 방법은 메서드 이름을 통해 쿼리를 만드는 방법이 있다. 해당 방법이 바로 이번 포스트에서 살펴볼 **Query Method**이다.

Query Method는 Spring Data JPA에서 메서드 이름을 통해 쿼리를 쉽게 생성할 수 있는 기능이다. 이번 포스트에서는 해당 기능의 기본적인 몇가지 문법에 대해 알아보고자 한다.

# 2. Query Method사용하기

## 2.1. 준비하기

[이전 게시글](https://seongwon.dev/Spring/20220911-Spring-Data-JPA%EB%9E%80/)에서 언급하였듯이 Spring Data JPA는 `JpaRepository`를 상속하는 것만으로도 Spring이 알아서 관리를 해주고 `CrudRepository`에서 제공하는 기본 메서드들을 사용할 수 있다. Query method 또한`JpaRepository`를 상속하는 것만으로도 Jpa의 method들을 사용할 수 있다.

```java
public interface UserRepository extends JpaRepository<User, Long> {}
```

## 2.2. Query Method 기초 알아가기

```java
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String userId;
    private String name;
    private int age;
    
    @Embedded
    private Address address;
}
```

위와 같은 엔티티가 있을 때, DB에서 이름이 `렉스`인 유저를 찾는 메서드를 만든다 생각해보자. 순수 JPA를 사용한다면 앞서 살펴보았던 코드와 같이 쿼리를 작성 후 `createQuery()`를 통해 결과를 얻어와야 했다.

```java
String query = "select u.name from User u where u.name = :name";
List<Member> result = em.createQuery(query, Member.class)
                        .setParameter("name", "Rex")
                        .getResultList();
```

하지만 이러한 쿼리를 Spring Data JPA에서는 아래와 같이 메서드만 작성해주면 스스로 메서드 이름을 분석 후 JPQL을 생성하고 실행해주어 코드를 간결하고 편하게 작성할 수 있다.

```java
interface UserRepository extends Repository<User, Long> {
	Optional<User> findByName(String name);
}
```

메서드 이름 작성은 2.3에서 다루는 쿼리 메서드의 문법들을 보고 작성하면 된다. 제공하는 문법들의 경우 이름이 매우 직관적이어서 사용하기도 편리하다.

### 2.2.1 Query Method Parsing방법과 필드 설정에서 주의할 점

Spring Data는 query method를 **subject**와 **predicate**으로 나누어 파싱을 한다. 먼저 메서드 이름의 시작 부분인 `find~By`, `exist~By` 부분을 **subject**이라고 한다. 그리고 그 뒤의 부분을 **predicate**이라고 칭한다.

predicate에 해당하는 Property 표현은 기본적으로 entity가 직접 관리하는 속성이어야 한다. 하지만 Entity에는 Embedded 타입과 같이 depth를 더 들어가야지 확인 가능한 필드도 존재할 것이다. 이러한 필드의 경우 Camel Case를 통해 경로를 나타내며 메서드명을 작성하면 된다. 예를 들어 Person객체는 Address 객체를 갖고 Address객체가 ZipCode 필드를 갖는다고 하였을 때, ZipCode를 조회 조건으로 사용하고 싶다면 `findByAddressZipCode(ZipCode zipCode);` 와 같이 Camem case로 메서드의 경로와 조건의 기준이 되는 필드명을 표현하면 된다.

이와 같이 작성을 한다면 Data JPA는 쿼리문을 작성할 때 먼저 Person객체로부터 AddressZipCode라는 필드를 찾는다. 해당 필드가 없다는 것을 확인한 후에는 오른쪽을 기준으로 한 단계씩 Camel Case의 단어를 쪼개서 필드를 찾는다. `findByAddressZipCode(ZipCode zipCode);` 를 기준으로 생각하면 먼저 AddressZip 과 Code로 쪼개어 AddressZip객체를 Person객체로부터 찾게 된다. 이후에도 없는 것을 확인하면 단어를 쪼개는 위치를 왼쪽으로 이동하여 Address, ZipCode로 쪼개고 Address객체를 Person객체로부터 찾아내고 Address객체 내부에 ZipCode가 있는지 확인을 하며 쿼리문을 작성하게 된다.
`findByAddressZipCode(ZipCode zipCode);` 와 같이 개발자도 헷갈릴 수 있는 애매한 이름은 `_` 를 통해 수동으로 순회 지점을 표현해줄 수도 있다. 앞의 메서드의 표현을 변경해보면 `findByAddress_ZipCode(ZipCode zipCode);` 와 같이 변하여 Person의 Address필드의 ZipCode필드라는 것을 더 직관적으로 알 수 있다.

## 2.3. Query Method의 여러 문법들

### 2.3.1. Select Query

- 아래의 접두사를 가진 메서드들은 모두 select query의 일을 하는 메서드가 된다.
- 이름은 `find(get,ready,,,,) By`가 핵심이며 두 문자 뒤에 구분할 수 있는 변수명을 넣어 구분하며 만들어주면 된다. 그러면 자동으로 Select Qeury Method가 만들어진다.
- `findBy`, `getBy`, `readBy`, `queryBy`, `searchBy`, `streamBy`, `find (Entity 명) By` 는 모두 Select의 일을 하는 키워드이며 이 중에서 자신이 가독성이 높다고 생각되는 것을 이용하면 된다.

```java
    User findByEmail(String email);
    User getByEmail(String email);
    User readByEmail(String email);
    User queryByEmail(String email);
    User searchByEmail(String email);
    User streamByEmail(String email);
    User findUserByEmail(String email);
```

> Return type은 `List`, `Set`, `Object`등의 여러 타입으로 할 수 있다. 반환 타입을 지정하면 JPA가 데이터를 읽어오고 return type에 맞춰서 데이터를 반환해준다. 단, 데이터가 여러개인데 User와 같이 단일 객체로 return하면 오류가 발생한다.
>

### 2.3.2. First, Top

- `First`, `Top`은 둘 다 데이터에서 출력할 데이터의 수를 정해서 리턴하게 하는 키워드이다.
- 기본적으로 키워드 뒤에 숫자를 붙이지 않으면 가장 상위 데이터 하나가 리턴되며(Default = 1) 키워드 뒤에 출력할 데이터의 수를 붙이면 해당 수만큼 리턴한다.

```java
    List<User> findFirst1ByName(String name);	// 상위 1개의 데이터 return
    List<User> findTop2ByName(String name);   // 상위 2개의 데이터 return
    List<User> findLast1ByName(String name); // Last filter는 없다
```

### 2.3.3. And, Or

Query문에서 and, or을 사용하고 싶은 경우 메서드안에 키워드를 넣어준다

```java
    List<User> findByNameAndEmail(String name, String email);
    List<User> findByNameOrEmail(String name, String email);
```

### 2.3.4. After, Before, GreaterThan, LessThan, Between

- **After, Before, GreaterThan, LessThan**은 날짜와 값 비교를 해주는 키워드이다.
- After, GreaterThan과 Before, LessThan은 서로 같은 기능을 하는 필터지만 가독성을 위해 After, Before은 날짜에만 사용하는 것을 추천한다.
- 또한 조건을 걸다보면 초과, 미만 뿐만 아니라 이상, 이하의 조건도 필요할 것이다. 그럴 경우에는 **GreaterThanEqual**과 같이 **Equal**을 붙여주면 된다.
- ~이상 ~이하의 의미를 갖는 **Between** 키워드도 존재한다. Between은 parameter를 2개를 받으며 Between은 위의 After, Before, GreaterThan, LessThan과 다르게 해당 값들도 포함한다.(이상, 이하를 의미한다.)

```java
    List<User> findByCreatedAtAfter (LocalDateTime lastDay);
    // CreatedAt이 lastDay이후인 데이터들 return (yesterDay미포함)
    List<User> findByIdAfter(Long id);
    // input id보다 큰 id를 가진 데이터들을 return (id 미포함)

    List<User> findByCreatedAtGreaterThan (LocalDateTime yesterday);
    // CreatedAt이 lastDay이후인 데이터들 return (yesterDay미포함)
    List<User> findByCreatedAtGreaterThanEqual (LocalDateTime yesterday);
    // CreatedAt이 lastDay이후인 데이터들 return (yesterDay포함)

    List<User> findByCreatedAtBetween(LocalDateTime yesterday, LocalDateTime tomorrow);
    // CreatedAt이 lastDay와 tomorrow사이 값인 데이터들 return (yesterDay, tomorrow포함)
    List<User> findByIdBetween(Long id1, Long id2);
    // id가 id1이상, id2이하인 데이터들 return

```

> After, Before, GreaterThan, LessThan은 초과 미만을 의미하며 Between은 이상, 이하를 의미하는 것을 헷갈리면 안된다!!!
>

### 2.3.5. is(Not)Empty, is(Not)Null

- isNull은 해당 값에 Null값이 있는지 체크하는 키워드도이다.
- NotEmpty는 String과 같은 문자열이 비어있는지 체크가 아닌 Collection type의 변수가 not empty(비어있는지)를 체크한다.

```java
    List<User> findByIdIsNotNull();  // Id값에 Null값이 없는지?
    List<User> findByAddressIsNotEmpty();
```

### 2.3.6. In

- In절은 query문에서 in절이기 때문에 parameter로 iterater type인 list가 들어가게 된다. generic type이 들어가야하는 <>에는 검색하는 column의 data type을 넣는다.
- 일반적으로 in절은 다른 query의 결과값을 다시 query에 넣어야 할 때 사용한다!
- in절을 사용할 때는 과부하 걸리는 것을 예방하기 위해서 다른 query문의 결과로 얼마나 많은 데이터들이 나올지 사전에 검토를 하고 사용하는게 좋다.

```java
 List<User> findByNameIn(List<String> name);
```

### 2.3.7. StartingWith/EndingWith/Contains

- 해당 keyword들은 문자열에 사용하며 해당 문자열로 시작하는지, 끝나는지, 포함하는지를 필터링할 때 사용한다.
- `contains("rti")`와 `like("%rti%")`는 같은 것이다

```java
    List<User> findByNameStartingWith(String name);
    List<User> findByNameEndingWith(String name);
    List<User> findByNameContains(String name);
    List<User> findByNameLike(String name);
```

### 2.3.8. Is, Equals

- Is는 해당 값을 가진 data를 찾는 키워드로 Is, Equals, 또는 아무런 키워드를 넣지 않으면 Is로 보게된다.
- 아래 3개의 method는 모두 parameter의 name과 동등한 이름의 데이터들을 출력하는 method이다.

```java
    Set<User> findUserByNameIs(String name);
    Set<User> findUserByName(String name);
    Set<User> findUserByNameEquals(String name);
```

### 2.3.9. Sorting

- Sorting은 조건에 따라 데이터의 정렬을 해주는 키워드이다.
- Desc/ Asc로 정렬한다.
- 여러개의 조건으로 find하는 경우는 And를 사용하였으나 정렬 조건으로 여러개의 값을 사용하는 경우는 And를 사용하지 않고 조건을 이어서 붙인다.
- Sort 객체를 필드로 줘서 정렬시킬 수도 있다.

```java
    List<User> findTop1ByNameOrderByIdDesc(String name);
    // Id로 내림차순으로 정렬 후 입력 name과 같은 것의 맨 위의 있는 값을 뽑아온다.

    List<User> findFirst2ByNameOrderByIdDescEmailAsc(String name);
    // 여러개의 조건으로 find하는 경우는 And를 사용하였으나 정렬 조건으로 여러개의 값을 사용하는 경우는 And를 사용하지 않고 조건을 이어서 붙인다.

    List<User> findFirstByName(String name, Sort sort);
```

## 3. 그 외의 쿼리 메서드 필터 조건들

| Keyword | Sample | JPQL snippet |
| --- | --- | --- |
| Distinct | findDistinctByLastnameAndFirstname | select distinct …​ where x.lastname = ?1 and x.firstname = ?2 |
| And | findByLastnameAndFirstname | … where x.lastname = ?1 and x.firstname = ?2 |
| Or | findByLastnameOrFirstname | … where x.lastname = ?1 or x.firstname = ?2 |
| Is, Equals, (or no keyword) | findByFirstname,findByFirstnameIs,findByFirstnameEquals | … where x.firstname = ?1 |
| Between, IsBetween | findByStartDateBetween | … where x.startDate between ?1 and ?2 |
| LessThan, IsLessThan | findByAgeLessThan | … where x.age < ?1 |
| LessThanEqual, IsLessThanEqual | findByAgeLessThanEqual | … where x.age <= ?1 |
| GreaterThan, IsGreaterThan | findByAgeGreaterThan | … where x.age > ?1 |
| GreaterThanEqual, IsGreaterThanEqual | findByAgeGreaterThanEqual | … where x.age >= ?1 |
| After, IsAfter | findByStartDateAfter | … where x.startDate > ?1 |
| Before, IsBefore | findByStartDateBefore | … where x.startDate < ?1 |
| IsNull, Null | findByAge(Is)Null | … where x.age is null |
| IsNotNull, NotNull | findByAge(Is)NotNull | … where x.age not null |
| Like, IsLike | findByFirstnameLike | … where x.firstname like ?1 |
| NotLike, IsNotLike | findByFirstnameNotLike | … where x.firstname not like ?1 |
| StartingWith, IsStartingWith,StartsWith | findByFirstnameStartingWith | … where x.firstname like ?1 (parameter bound with appended %) |
| EndingWith, IsEndingWith, EndsWith | findByFirstnameEndingWith | … where x.firstname like ?1 (parameter bound with prepended %) |
| Containing, IsContaining, Contains | findByFirstnameContaining | … where x.firstname like ?1 (parameter bound wrapped in %) |
| OrderBy | findByAgeOrderByLastnameDesc | … where x.age = ?1 order by x.lastname desc |
| Not, IsNot | findByLastnameNot | … where x.lastname <> ?1 |
| In, IsIn | findByAgeIn(Collection<Age> ages) | … where x.age in ?1 |
| NotIn, IsNotIn | findByAgeNotIn(Collection<Age> ages) | … where x.age not in ?1 |
| True, IsTrue | findByActiveTrue() | … where x.active = true |
| False, IsFalse | findByActiveFalse() | … where x.active = false |
| IgnoreCase, IgnoringCase | findByFirstnameIgnoreCase | … where UPPER(x.firstname) = UPPER(?1) |

더 많은 쿼리 조건들은 Reference에 위치한 공식 문서를 참고하길 바란다.

# 📚 Reference
- [Spring Data JPA - Reference Documentation](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.query-methods.details)
- [Spring Data JPA - Reference Documentation](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repository-query-keywords)
- [Spring Data JPA - Reference Documentation](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods)
- [Derived Query Methods in Spring Data JPA Repositories](https://www.baeldung.com/spring-data-derived-queries)
- [실전! 스프링 데이터 JPA - 인프런 | 강의](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%EB%8D%B0%EC%9D%B4%ED%84%B0-JPA-%EC%8B%A4%EC%A0%84/dashboard)
- [한 번에 끝내는 Java/Spring 웹 개발 마스터 초격차 패키지 Online. | 패스트캠퍼스](https://fastcampus.co.kr/dev_online_javaend)
- [[Spring Data JPA] Query Methods - 메소드 이름으로 쿼리 생성](https://ppomelo.tistory.com/155)
