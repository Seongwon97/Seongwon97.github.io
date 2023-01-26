---
title: "리액티브 프로그래밍과 리액티브 스트림"
date: 2023-01-20
tags: ["SpringFramework", "Reactive Programming", "Publisher", "Subscriber", "Subscription", "Processor"]
draft: false
---

# 리액티브 프로그래밍이란?

애플리케이션은 명령형(imperative)와 리액티브(reactive, 반응형)의 형로 작성될 수 있다.

### 명령형 코드

- 순차적으로 실행된다.
- 이전 작업이 완전히 끝난 이후에 다음 작업이 한 번에 하나씩 실행된다.
- 데이터는 모아서 처리되고 이전 작업이 데이터 처리를 완전히 끝내야 다음 작업을 진행할 수 있다.

### 리액티브

- 작업들을 병렬로 실행할 수 있다.
- 각 작업은 부분 집합의 데이터를 처리할 수 있다.
- 처리가 끝난 데이터를 다음 작업에 넘겨주고 다른 부분 집합의 데이터로 작업을 계속할 수 있다.

명령형 코드의 경우 DB로부터 데이터를 읽어오거나 외부 API를 호출하여 데이터를 읽어오는 등의 작업을 할 때, Thread는 다른 작업을 하지 못하고 대기 상태로 가게 되어 낭비가 발생하게 된다. 낭비를 줄이기 위해 다중 Thread를 통한 동시성 프로그래밍을 할 수 있으나, 너무 많은 Thread가 생기게 되면 동시성을 관리하기도 어려워진다. 이를 위해 만들어진 것이 리액티브 프로그래밍이다. 리액티브 프로그래밍은 함수적이면서 선언적이라 순차적으로 수행되는 작업 단계를 나타낸 것이 아니라 데이터가 흘러가는 파이프라인이나 스트림을 포함한다. 그리고 데이터 전체를 사용할 수 있을 때까지 기다리지 않고 사용 가능한 데이터가 있을 때마다 처리되므로 입력되는 데이터가 무한할 수 있고 동시에 여러 작업을 수행하여 더 큰 확장성을 얻게 해준다.

# 리액티브 스트림

리액티브 스트림은 차단되지 않은 백 프레셔(backpressure)를 갖는 비동기 스트림 처리의 표준을 제공하는 것이 목적이다.

> **백 프레셔란?**
>
>
> 백 프레셔는 데이터를 소비하는 Consumer가 처리할 수 있는 만큼으로 전달 데이터를 제한함으로써 지나치게 빠른 데이터 소스로부터 데이터 전달의 폭주를 피할 수 있는 수단이다.
>

리액티브 스트림 패키지(`package org.reactivestreams;`)는 4개의 인터페이스인 Publisher(발행자), Subscriber(구독자), Subscription(구독), Processor(프로세서)를 제공하고 있다. 전반적인 플로우는 Publisher로부터 시작해서 0개 이상의 Processor를 통해 데이터를 끌어온 다음 최종 결과를 Subscriber에게 전달하는 식으로 동작하게 된다.

### Publisher

Publisher는 `subscribe()`메서드를 제공하며 하나의 Subscriber당 하나의 발행 데이터를 생성하도록 동작한다.

```java
public interface Publisher<T> {
    public void subscribe(Subscriber<? super T> s);
}
```

### Subscriber

Subscriber는 구독이 신청되면 Publisher로부터 Subscription을 통해 이벤트를 수신받을 수 있다.

```java
public interface Subscriber<T> {
    public void onSubscribe(Subscription s);

    public void onNext(T t);

    public void onError(Throwable t);

    public void onComplete();
}
```

- `onSubscribe()`: 해당 메서드는 Publisher로부터 Subscription객체를 받으며 호출된다. 그러면 Subscriber는 Subscription객체를 통해 구독을 관리할 수 있게 된다.
- `onNext();`: Subscriber의 데이터 요청이 완료되면 해당 메서드를 통해 Publisher로부터 스트림을 통해 데이터가 전달된다.
- `onError();`: `onNext()`를 통해 데이터가 전달되는 중 에러가 발생하면 해당 메서드가 호출된다.
- `onComplete();`: Publisher에서 전송할 데이터가 없고 더 이상의 데이터를 생성하지 않는다면 Publisher가 해당 메서드를 호출하여 작업이 끝났다고 Subscriber에게 알려준다.

### Subscription

```java
public interface Subscription {
    public void request(long n);

    public void cancel();
}
```

- `request();` : Subscriber가 Publisher에게 데이터를 요청하기 위해 사용되는 메서드이다. Publisher에게 parameter로 입력하는 n개의 데이터를 요청한다는 의미를 갖는다.

  > 매개변수를 통해 요청하는 데이터 수를 전달하며 Subscriber가 Publisher로부터 받는 데이터의 폭주를 막는다. 즉, `requst();` 메서드를 통해 백 프레셔를 갖게 된다.
>
- `cancel();`: Subscriber가 Publisher로부터 더 이상의 데이터를 수신하지 않고 구독(subscribe)을 취소하는 요청을 할 때 사용된다. 하지만 요청을 한다고 바로 구독이 취소되는 것이 아니라 이전에 요청한 데이터들은 계속 전송되고 구독 취소가 될 수 있다.

### Processor

Subscriber와 Publisher인터페이스를 결합하여 Subscriber의 역할로 데이터를 수신하고 처리하며, 그 후에 역할을 바꾸어 Publisher의 역할로 처리 결과를 자신의 Subscribr들에게 발행하는 방식으로 동작한다.

```java
public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {
}
```

> **자바 스트림과의 비교**
>
>
> 자바와 리액티브 스트림은 데이터로 작업하기 위한 api를 제공한다는 유사성이 있다. 하지만 자바의 스트림은 대개 동기화되어 있고 한정된 데이터로 작업을 수행하는 반면에 리액티브 스트림은 어떤 크기의 데이터셋이건 비동기 처리를 지원한다. 그리고 백프레셔를 통해 데이터 폭주를 막으며 실시간으로 데이터를 처리한하는 특징도 있다.
>

# 코드로 동작 살펴보기

### Publisher

```java
import java.util.Arrays;

import org.reactivestreams.Publisher;
import org.reactivestreams.Subscriber;

public class MyPublisher implements Publisher<Integer> {

    Iterable<Integer> data = Arrays.asList(1, 2, 3, 4, 5, 6);

    @Override
    public void subscribe(Subscriber<? super Integer> s) {
        System.out.println("Publisher.subscribe() 실행");
        System.out.println("Publisher.subscribe() -> Subscription 객체 생성 완료");
        MySubscription subscription = new MySubscription(s, data);
        System.out.println("Publisher.subscribe() -> 생성한 Subscription를 인자로 사용하며 Subscriber의 onSubscribe()호출");
        s.onSubscribe(subscription);
        System.out.println("Publisher.subscribe() 종료");

    }
}
```

### Subscriber

```java
import org.reactivestreams.Subscriber;
import org.reactivestreams.Subscription;

public class MySubscriber implements Subscriber<Integer> {

    private Subscription subscription;
    private int bufferSize = 2;

    @Override
    public void onSubscribe(Subscription s) {
        System.out.println("Subscriber.onSubscribe() 실행");
        this.subscription = s;
        System.out.println("Subscriber.onSubscribe() -> 요청할 데이터 수를 인자로 전달하며 Subscription의 request()메서드 호출");
        subscription.request(bufferSize); // 백 프레셔 -> 소비자가 한번에 처리할 수 있는 개수를 요청
        System.out.println("Subscriber.onSubscribe() 종료");
    }

    @Override
    public void onNext(Integer integer) {
        System.out.println("Subscriber.onNext() 실행");
        System.out.println("onNext(): " + integer);

        bufferSize--;
        if (bufferSize == 0) {
            bufferSize = 2;
            subscription.request(bufferSize); // 데이터가 모두 소모되는 것을 확인하기 위해 추가한 코드.
        }
    }

    @Override
    public void onError(Throwable t) {
        System.out.println("구독 중 에러");
    }

    @Override
    public void onComplete() {
        System.out.println("==== 구독 완료 ====");
    }
}
```

### Subscription

```java
import java.util.Iterator;

import org.reactivestreams.Subscriber;
import org.reactivestreams.Subscription;

// 구독 정보(구독자, 어떤 데이터를 구독할지에 대한 정보를 갖고 있어야 한다.)
public class MySubscription implements Subscription {

    private final Subscriber subscriber;
    private final Iterator<Integer> data;

    public MySubscription(Subscriber subscriber, Iterable<Integer> data) {
        this.subscriber = subscriber;
        this.data = data.iterator();
    }

    @Override
    public void request(long n) {
        System.out.println("\nSubscription.request() 실행");
        while (n > 0) {
            if (data.hasNext()) {
                System.out.println("Subscription.request() -> 요청받은 데이터 수만큼 반환하지 않고, 현재 데이터가 존재하면 subscriber.onNext()호출");
                subscriber.onNext(data.next());
            } else {
                System.out.println("Subscription.request() -> 요청받은 데이터 수만큼 반환하지 않고, 현재 데이터가 존재하지 않으면 subscriber.onComplete()호출");
                subscriber.onComplete();
                break;
            }
            n--;
        }
    }

    @Override
    public void cancel() {

    }
}
```

### Application.main()

```java
public class Application {

    public static void main(String[] args) {
        MyPublisher publisher = new MyPublisher();
        MySubscriber subscriber = new MySubscriber();

        publisher.subscribe(subscriber);
    }
}
```

Publisher, Subscriber, Subscription인터페이스가 어떻게 동작하는지 더 편한 이해를 위해 구현한 코드를 작성해보았다. 위의 코드들을 작성하고 실행을 해본 결과 아래와 같은 결과가 나온다. 위에서 인터페이스를 살펴보았듯이 Publisher의 `subscribe()`메서드는 Subscriber 구현체를 인자로 받아 실행되면 인자로 받은 Subscriber만을 위한 구독자료(Subscription)을 만들어 제공하게 된다.

Subscriber는 본인이 처리할 수 있는 양만큼의 데이터 수를 Subscription의 `request()`메서드의 인자로 전달하여 데이터 처리양을 조절할 수 있으며 `request()`메서드는 Subscriber의 `onNext()`메서드를 통해 데이터를 전달하게 된다. 또한 데이터를 모두 전달하였으면 `onComplete()`메서드를 호출하도록 동작하게 된다.

코드에서 다루지 않았지만 구독 중에 구독을 취소하고 싶다면 Subscription의 `cancel()`메서드가 호출될 것이고 에러가 발생한다면 Subscriber의 `onError()`메서드가 동작할 것이다.

### 실행 결과
```
Publisher.subscribe() 실행
Publisher.subscribe() -> Subscription 객체 생성 완료
Publisher.subscribe() -> 생성한 Subscription를 인자로 사용하며 Subscriber의 onSubscribe()호출
Subscriber.onSubscribe() 실행
Subscriber.onSubscribe() -> 요청할 데이터 수를 인자로 전달하며 Subscription의 request()메서드 호출

Subscription.request() 실행
Subscription.request() -> 요청받은 데이터 수만큼 반환하지 않고, 현재 데이터가 존재하면 subscriber.onNext()호출
Subscriber.onNext() 실행
onNext(): 1
Subscription.request() -> 요청받은 데이터 수만큼 반환하지 않고, 현재 데이터가 존재하면 subscriber.onNext()호출
Subscriber.onNext() 실행
onNext(): 2

Subscription.request() 실행
Subscription.request() -> 요청받은 데이터 수만큼 반환하지 않고, 현재 데이터가 존재하면 subscriber.onNext()호출
Subscriber.onNext() 실행
onNext(): 3
Subscription.request() -> 요청받은 데이터 수만큼 반환하지 않고, 현재 데이터가 존재하면 subscriber.onNext()호출
Subscriber.onNext() 실행
onNext(): 4

Subscription.request() 실행
Subscription.request() -> 요청받은 데이터 수만큼 반환하지 않고, 현재 데이터가 존재하면 subscriber.onNext()호출
Subscriber.onNext() 실행
onNext(): 5
Subscription.request() -> 요청받은 데이터 수만큼 반환하지 않고, 현재 데이터가 존재하면 subscriber.onNext()호출
Subscriber.onNext() 실행
onNext(): 6

Subscription.request() 실행
Subscription.request() -> 요청받은 데이터 수만큼 반환하지 않고, 현재 데이터가 존재하지 않으면 subscriber.onComplete()호출
==== 구독 완료 ====
Subscriber.onSubscribe() 종료
Publisher.subscribe() 종료
```

# 📚 Reference
- [스프링 인 액션](https://search.shopping.naver.com/book/catalog/32441616013?cat_id=50010920&frm=PBOKPRO&query=%EC%8A%A4%ED%94%84%EB%A7%81+%EC%9D%B8+%EC%95%A1%EC%85%98&NaPm=ct%3Dldd6rd20%7Cci%3Df076961ffc3dab854d41bc1cdeea73bfa8c0a8f1%7Ctr%3Dboknx%7Csn%3D95694%7Chk%3Df155b60e8553a8baef3d19bce28770da9fec5bfa)
- [Reactive Streams](https://www.reactive-streams.org/)
- [메타코딩 - Springboot-WebFlux 5강 - reactive streams](https://www.youtube.com/watch?v=6TiUCm3K_IE&list=PL93mKxaRDidFH5gRwkDX5pQxtp0iv3guf&index=6)
