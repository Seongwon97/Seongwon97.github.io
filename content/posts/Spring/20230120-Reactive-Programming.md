---
title: "리액티브 프로그래밍이란?"
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
