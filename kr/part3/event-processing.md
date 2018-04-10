이벤트 게시와 처리하기
=============================

애플리케이션에서 발생한 이벤트들은 데이터베이스의 데이터 갱신, 검색 엔진 혹은 이벤트 핸들러들과 같은 이벤트가 있어야 하는 다른 자원들에 전달되어야 합니다. 이벤트 버스는 이벤트가 있어야 하는 모든 컴포넌트들에 이벤트를 전달하는 것을 책임집니다. 이벤트를 받는 쪽에선, 이벤트 프로세서들이 적합한 이벤트 처리자를 호출하는 것을 포함한 이벤트를 처리를 담당합니다.

이벤트 게시하기
-----------------

대다수의 경우, Aggregate들은 이벤트를 적용하면서 또 다른 이벤트를 발생시킵니다. 그러나, 이따금, 이벤트 버스로 직접 이벤트(다른 구성 요소 내에서 가능)를 게시할 필요가 있습니다. 이벤트를 게시하기 위해선, 해당 이벤트를 설명하는 페이로드를 `EventMessage`에 추가해야 합니다. `GenericEventMessage.asEventMessage(Object)` 메서드를 통해, 페이로드를 `EventMessage`에 추가합니다. `EventMessage`에 추가할 객체가 `EventMessage` 타입인 경우, 추가할 객체를 반환합니다.

이벤트 버스
---------
`EventBus`는 특정 이벤트를 구독한 (subscribe - 이벤트 처리를 위해 등록된) 이벤트 처리자에게 이벤트를 전달하기 위한 메커니즘입니다. Axon을 통해 `SimpleEventBus`와 `EmbeddedEventStore` 두 개의 이벤트 버스 구현체를 사용할 수 있습니다. 두 구현체 모두 구독(subscribing)과 추적(tracking) 프로세서를 지원합니다. ([이벤트 프로세서](#이벤트-프로세서) 참조). 반면, `EmbeddedEventStore`를 통해 이벤트를 영속화(persist)하고 나중에 이벤트들을 다시 재현할 수 있습니다. `SimpleEventBus`는 volatile 저장소를 가지고 있고, 특정 컴포넌트로 게시된 이벤트를 더는 신경 쓰지 않습니다(forget events).

설정 API를 사용한다면, `SimpleEventBus`가 기본으로 설정됩니다. `SimpleEventBus` 대신 `EmbeddedEventStore`를 설정할 수 있고, 이를 위해 실제 이벤트가 저장될 저장 엔진(Storage)의 구현체를 `EmbeddedEventStore`에 제공해 주어야 합니다.

```java
    Configurer configurer = DefaultConfigurer.defaultConfiguration();
    configurer.configureEmbeddedEventStore(c -> new InMemoryEventStorageEngine());
```

이벤트 프로세서
----------------
이벤트 처리자를 통해 이벤트를 받았을 때 처리할 비즈니스 로직을 정의할 수 있습니다. 이벤트 프로세서들은 이벤트 처리를 위한 기술적 부분을 담당하는 컴포넌트입니다. 작업 단위(Unit of Work)와 트랜잭션을 시작하며 이벤트를 처리 동안 생성된 모든 메시지에 관련 데이터가 올바르게 첨부될 수 있도록 합니다.

이벤트 프로세서들은 구독(subscribing)과 추적(tracking)과 같이 두 개의 형식으로 표현됩니다. 구독 이벤트 프로세서들은 이벤트가 발생하는 원천에 자신을 등록하고 이벤트 게시 메커니즘에 의해 관리되는 스레드로 호출이 됩니다. 반면, 추적 이벤트 프로세서는 자기 자신을 관리하는 스레드를 사용하여 이벤트 원천으로부터 이벤트를 가져옵니다.

### 처리자를 프로세서에 할당하기

모든 프로세서는 JVM 인스턴스들 상에서 프로세서를 식별하기 위한 식별 값을 가집니다. 같은 이름을 가지는 두 개의 프로세서는 같은 프로세서의 두 개의 인스턴스로 취급됩니다.

모든 이벤트 처리자들은 이벤트 처리자 클래스의 패키지 이름을 가지는 프로세서에 포함됩니다.

예를 들면 아래와 같은 클래스들이 해당합니다.

- `org.axonframework.example.eventhandling.MyHanaler`,
- `org.axonframework.example.eventhandling.MyOhterHandler`, 그리고
- `org.axonframework.example.eventhandling.module.MyHandler`

위의 클래스들에 대한 두 개의 프로세서들이 생성됩니다.

- 두개의 처리자를 포함하는 `org.axonframework.example.eventhandling`
- 단일 처리자를 포함하는 `org.axonframework.example.eventhandling.module`

설정 API를 통해 위와 다른 처리자 클래스를 프로세서에 할당하거나 특정 인스턴스를 특정 프로세서에 할당하는 등 의 전략을 설정할 수 있습니다.

### 단일 이벤트 프로세서내의 이벤트 처리자 순서 정하기

단일 이벤트 프로세서의 이벤트 처리자들의 순서를 정하려면, 이벤트 처리자가 등록되는 순서를 먼저 알아야 합니다. 그 이유는 이벤트 처리자의 호출 순서는 등록된 순서대로 호출이 되기 때문입니다 (이벤트 처리자의 등록은 [이벤트-처리자-등록하기](../part2/event-handling.html#이벤트-처리자-등록하기)를 참조하세요). 즉, 이벤트를 수신했을 때 이벤트 프로세서는 이벤트 처리자가 설정 API를 통해 등록된 순서대로 이벤트 처리자를 호출하게 됩니다.

객체 간 의존성을 주입하기 위해 스프링을 사용한다면, 이벤트 처리자의 등록 순서는 `@Order` 에노테이션에 명시하면 됩니다. `@Order` 에노테이션을 이벤트 처리자 클래스에 선언하고 순서에 해당하는 `정수 (integer)`값을 명시하면 됩니다.

다른 이벤트 프로세서에 있는 이벤트 처리자들 간의 순서를 정할 수는 없습니다.

### 이벤트 프로세서 설정하기

이벤트 프로세서는 개별 이벤트에 의해 수행되는 비즈니스 로직에 상관없이 이벤트를 처리하기 위한 기술적인 부분만을 담당합니다. 하지만, "일반적인" (싱글턴, 스테이트리스(stateless)) 이벤트 처리자들의 구성 방식은 Saga들과는 약간 다릅니다. 두 가지 유형의 처리자는 다른 관점으로 처리되기 때문입니다.

#### 이벤트 처리자 ####

Axon은 이벤트 구독(subscribing) 프로세서를 기본으로 사용합니다. 설정 API의 ```EventHandlingConfiguration```클래스를 통해 처리자를 지정하고 프로세서의 설정 방법을 변경할 수 있다.

`EventHandlingConfiguration` 클래스에는 다음과 같이 프로세서를 설정하기 위한 메서드를 정의하고 있습니다.

- `registerEventProcessorFactory` 메서드를 통해, 명시적인 팩토리 메서드가 없는 경우에 사용되는 이벤트 프로세서를 생성하는 기본 팩토리 메서드를 설정할 수 있습니다.

- `registerEventProcessor(String name, EventProcessorBuilder builder)` 메서드는 주어진 `name` 매개변수를 이름으로 가지는 프로세서를 생성하는 팩토리 메서드를 정의합니다. 사용 가능한 이벤트 처리자 빈들에 대한 프로세서 명으로 주어진 `name` 매개 변수명이 채택이 될 때만 프로세서는 생성이 됩니다.

- `registerTrackingProcessor(String name)` 메서드는 기본 설정값으로 주어진 이름을 가지는 이벤트 추적 프로세서를 정의합니다. TransactionManager와 TokenStore를 함께 설정하며, 두 객체 기본적으로 주 설정 객체를 통해 받아 올 수 있습니다.

- `registerTrackingProcessor(String name, Function processorConfiguration, Function>> sequencingPolicy)` 메서드를 통해 주어진 이름을 가지는 이벤트 추적 프로세서를 정의하고, 다중 스레드 설정을 알기 위해 주어진 `TrackingEventProcessorConfiguration` 객체를 사용합니다. `SequencePolicy`는 순차적인 이벤트 처리 설정을 정의하기 위해 사용됩니다. 상세 내용은 [병렬 프로세싱](#병렬-프로세싱)을 참고하세요.

- `usingTrackingProcessor()` 메서드를 통해 이벤트 구독 프로세서 대신 기본으로 추적 이벤트 프로세서를 설정하도록 할 수 있습니다.

#### 사가 ####

`SagaConfiguration` 클래스를 통해 Saga들을 설정할 수 있습니다. `SagaConfiguration` 클래스는 추적 프로세서 혹은 구독 프로세서 둘 중 하나의 인스턴스를 초기화하기 위한 정적 메세드들을 제공합니다.

구독 모드로 Saga를 설정하기 위해선 다음과 같이 하면 됩니다.

```java
SagaConfiguration<MySaga> sagaConfig = SagaConfiguration.subscribingSagaManager(MySaga.class);
```

Saga에서 처리될 메세지를 제공하는 기본 이벤트 버스 및 스토어를 사용하지 않을 경우, 다른 메세지 제공 객체를 설정할 수 있습니다.

```java
SagaConfiguration.subscribingSagaManager(MySaga.class, c -> /* 메세지 제공 객체를 설정합니다. */);
```

`subscribingSagaManager()` 메서드의 또 다른 형태를 통해, `EventProcessingStrategy` 객체를 매개변수로 넘길 수 있습니다. 기본적으로, Saga는 동기적으로 호출되지만, 해당 메서드를 통해 비동기적으로 Saga를 호출할 수 있습니다. 그런데, 추적 프로세서는 비동기 호출 방식을 사용합니다.

추적 프로세서를 사용하도록 Saga를 설정하기 위해선, 아래와 같이 하면 됩니다.

```java
SagaConfiguration.trackingSagaManager(MySaga.class);
```

위와 같이 작성하면, 기본 속성을 사용하게 됩니다. 즉, 단일 쓰레드로 이벤트를 처리하게 됩니다. 이를 변경하려면 아래와 같이 작성합니다.

```java
SagaConfiguration.trackingSagaManager(MySaga.class)
                 // 4개의 쓰레드를 설정합니다.
                 .configureTrackingProcessor(c -> TrackingProcessingConfiguration.forParallelProcessing(4))
```

`TrackingProcessingConfiguration` 클래스의 메서드를 통해, 몇 개의 세그먼트를 생성할지와 어떤 스레드 팩토리(ThreadFactory)를 통해 프로세서 스레드를 생성할지를 설정할 수 있습니다. 상세 내용은 [병렬 프로세싱](#병렬-프로세싱)을 참고하세요.

Saga를 위한 이벤트 처리 방법에 대한 상세 내용은 `SagaConfiguration` 클래스의 API 문서를 확인해 보세요.


### 토큰 스토어 ###

구독 이벤트 프로세서와는 달리, 추적 프로세서는 진행 상태를 저장하기 위한 토큰 저장소(token store)가 필요합니다. 추적 프로세서가 이벤트 스트림을 통해 수신하는 각각의 메시지는 토큰을 함께 가지고 있습니다. 이 토큰을 통해, 프로세서는 마지막 이벤트를 수신한 시점 이후의 시점에서 이벤트 스트림을 다시 열 수 있습니다.

설정 API는 전역 설정 인스턴스에서 필요로 하는 대부분의 다른 구성 요소와 함께 토큰 저장소를 설정합니다. 토큰 저장소를 명시적으로 정의하지 않는다면, `InMemoryTokenStore`를 기본으로 사용합니다. 단, `InMemoryTokenStore`는 *운영 환경에서 사용하기에는 적절하지 않습니다*.

다른 토큰 저장소를 설정하려면, `Configurer.registerComponent(TokenStore.class, conf -> /* 토큰 저장소를 생성 등록합니다. */)`

프로세서를 정의하는 `EventHandlingConfiguration` 혹은 `SagaConfiguration`의 추적 프로세서와 함께 사용할 토큰 저장소를 재정의하여 사용할 수 있습니다. 가능하다면, 이벤트 처리자가 뷰 모델을 갱신하는 데이터베이스와 같은 데이터베이스를 토큰 저장소로 사용하는 것을 권장합니다. 이렇게 하면, 토큰의 변경과 함께 뷰 모델의 변경 사항을 원자 적으로 저장할 수 있으며, 정확하게 한꺼번에 처리되도록 하는 것을 보장할 수 있습니다.

### 병렬 프로세싱 ###

Axon Framework 3.1버전에선, 추적 프로세서는 이벤트 스트림을 처리하기 위해 여러 개의 스레드를 사용할 수 있습니다. 동시에 여러 개의 스레드로 이벤트 스트림을 처리하기 위해서, 소위 세그먼트(segment)라는 번호로 식별되는 식별자를 요청합니다. 일반적으로 단일 스레드는 단일 세그먼트를 처리합니다.

사용될 다수의 세그먼트를 정의할 수 있습니다. 처음 프로세서가 시작되었을 때, 여러 개의 세그먼트를 초기화할 수 있습니다. 세그먼트의 개수가 동시에 이벤트들을 처리할 최대 스레드 개수를 정의하게 됩니다. 추적 프로세서로 실행되는 각각의 노드는 설정된 개수만큼의 스레드를 시작하고 이벤트들의 처리를 시작합니다.

이벤트 처리자들이 특정 순서대로 이벤트들을 처리하기를 원할 수 있습니다. 이런 경우, 해당 프로세서는 정해진 순서대로 이벤트들이 이벤트 처리자로 전송되도록 보장해야 합니다. Axon은 순차적인 이벤트 처리를 위해 `SequencingPolicy`를 사용합니다. `SequencingPolicy`는 주어진 메시지의 처리 순서 값을 반환하는 함수입니다. 두 메시지에 대한 값이 같다는 것은 해당 메시지들은 반드시 차례대로 처리되어야 하는 것을 의미합니다. 처리될 순서는 이벤트 버스가 이벤트를 게시한 순서입니다. 기본적으로, Axon 컴포넌트들은 `SequentialPerAggregatePolicy`를 사용합니다. 이 경우, 같은 aggregate 인스턴스에 의해 게시된 이벤트들은 차례대로 처리됩니다.

Saga 인스턴스는 다중 스레드에 의해 절대 호출되지 않습니다. 따라서, 순차적 처리 정책은 Saga에는 해당하지 않습니다. Axon은 각각의 Saga 인스턴스들이 이벤트 버스에 의해 게시된 순서대로 처리할 이벤트들을 수신하도록 보장합니다.

> **참고**
>
> 구독 프로세서는 자체 스레드를 관리하지 않습니다. 즉, 구독 프로세서들은 이벤트 수신 방법을 설정할 수 없습니다. 일반적으로 명령 처리 컴포넌트에서의 동시성 레벨인 aggregate별 순차적 처리에 기반을 두고 이벤트를 처리합니다.

#### 다중 노드를 활용한 처리 ####

추적 프로세서는 동일 노드 혹은 논리적으로 같은 추적 프로세서를 가지는 다른 노드에서 이벤트 처리 스레드들이 모두 실행 중인지에 대해 신경 쓰지 않습니다. 같은 이름을 가지는 두 개의 추적 프로세서가 다른 머신(machine)에서 활성화되어 있을 때, 추적 프로세서들은 논리적으로 같은 프로세서의 인스턴스로 취급됩니다. 추적 프로세서들은 이벤트 스트림의 세그먼트에 대해 서로 경쟁을 하며, 각각의 인스턴스는 다른 노드에서 처리되는 것을 방지하기 위해 세그먼트를 요청하여 할당받습니다.

 `TokenStore` 인스턴스는 JVM의 이름(보통 호스트 이름과 프로세스 ID의 조합입니다.)을 `nodeId`로 사용합니다. 하지만 다중 노드 처리를 위해 `TokenStore`를 재정의하여 다른 이름을 사용하도록 할 수 있습니다.

이벤트들을 분산하여 처리하기
-------------------

몇몇 경우엔, 메시지 브로커와 같은 외부 시스템으로 이벤트를 게시할 필요가 있습니다.

### 스프링 AMQP

Axon은 Rabbit MQ와 같은 AMQP 메시지 브러커와 이벤트를 주고 받는 기능을 제공합니다.

#### AMQP 익스체인지로 이벤트 전달하기

`SpringAMQPPublisher`는 AMQP 익스체인지로 이벤트를 전달합니다. `SpringAMQPPublisher`를 초기화하기 위해선 일반적으로 `EventBus` 혹은 `EventStore`인 `SubscribableMessageSource`가 필요합니다. 이론적으로, 이벤트 게시자가 구독할 수 있는 모든 이벤트 소스를 설정할 수 있습니다.

`SpringAMQPPublisher`를 설정하기 위해선, Spring의 Bean으로 간단히 등록하면 됩니다. 트랜잭션 지원, 게시자 인지(브로커가 지원하는 경우에 한함) 그리고 익스체인지(exchange) 이름과 같은 것들을 설정할 수 있는 setter 메서드를 제공합니다.

기본 exchange 이름은 "Axon.EventBus"입니다.

> **참고**
>
> 익스체인지(exchange)는 자동으로 생성되지 않습니다. 따라서 사용하려는 큐, 익스체인지 그리고 바인딩들을 반드시 선언해야 합니다. 보다 자세한 내용은 [스프링 문서](https://docs.spring.io/spring-amqp/reference/htmlsingle/)를 참고해 주세요.

#### AMQP 큐로부터 이벤트 읽어오기

스프링의 지원을 받아 AMQP 큐로부터 이벤트 읽어올 수 있습니다. 그런데, AMQP 큐로부터 이벤트 읽어와서 해당 이벤트가 일반적인 이벤트 메시지만 Axon을 통해 처리하려면, Axon으로 연결을 해줘야 합니다.

`SpringAMQPMessageSource`를 통해 이벤트 스토어 혹은 이벤트 버스 대신, 이벤트 프로세서가 메시지를 읽어오게 할 수 있습니다. `SpringAMQPMessageSource`는 스프링 AMQP와 프로세서가 필요로 하는`SubscribableMessageSource`사이의 어댑터 같은 역할을 해줍니다.

`SpringAMQPMessageSource`를 설정하는 가장 쉬운 방법은 `onMessage`메서드를 제정의 하는 bean 객체를 정의하고 `@RabbitListener` 에노테이션을 붙여 주는 것입니다. 아래의 예제와 같이 말이죠.

```java
@Bean
public SpringAMQPMessageSource myMessageSource(Serializer serializer) {
    return new SpringAMQPMessageSource(serializer) {
        @RabbitListener(queues = "myQueue")
        @Override
        public void onMessage(Message message, Channel channel) throws Exception {
            super.onMessage(message, channel);
        }
    };
}
```

스프링의 `@RabbitListener` 에노테이션을 사용한 메서드가 주어진 큐(예제에서는 'myQueue')의 각각의 메시지에 대해 호출이 될 수 있도록 해줍니다. 해당 메서드는 단순히 현재 구독 중인 프로세서들에 이벤트를 게시하게 해주는 `super.onMessage()` 메서드를 호출합니다.

프로세서들을 이 메시지 소스(MessageSource)에 등록하려면, 올바른 `SpringAMQPMessageSource` 인스턴스를 구독하려는 프로세서에 전달해야 합니다.

```java
// @Configuration 파일내에 작성해야 합니다.
@Autowired
public void configure(EventHandlingConfiguration ehConfig, SpringAmqpMessageSource myMessageSource) {
    ehConfig.registerSubscribingEventProcessor("myProcessor", c -> myMessageSource);
}
```

추적 프로세서는 `SpringAMQPMessageSource`와 연동할 수 없으니, 이점에 유의하세요.

비동기 방식으로 이벤트 처리하기
-----------------------------

비동기 방식으로 이벤트를 처리하기 위해서 추적 이벤트 프로세서(Tracking Event Processor)를 사용하는 것을 권장합니다. 추적 이벤트 프로세서를 사용하여 구현하게 되면, 시스템 장애가 발생한 상황(이벤트들이 영속화되어 있다고 가정합니다.)이라도 모든 이벤트의 처리를 보장할 수 있습니다.

그러나, `SubscribingProcessor`를 사용하여 이벤트를 비동기적으로 처리하는 것도 가능합니다. `SubscribingProcessor`를 사용하여 이벤트를 비동기적으로 처리하기 위해선, `SubscribingProcessor`를 반드시 `EventProcessingStrategy`와 함께 설정해줘야 합니다. `EventProcessingStrategy`는 괸리 되어야 할 이벤트 리스터들의 호출 방식을 변경하는 데 사용이 됩니다.

기본 전략(`DirectEventProocessingStrategy`)은 이벤트를 전달하는 스레드 내에서 해당 이벤트 처리자들을 호출하는 것입니다. 이렇게 하면 프로세서가 이미 존재하는 트랜잭션을 사용하도록 할 수 있습니다.

Axon에서 제공하는 다른 전략은 `AsynchronousEventProcessingStrategy`인데, 이벤트 리스너를 비동기적으로 호출하기 위해 `Executor`를 사용합니다.

비록 `AsynchronousEventProcessingStrategy`가 비동기 방식으로 실행되더라도, 여전히 몇몇 이벤트들은 확실히 차례대로 처리되어야 합니다. `SequencePolicy`를 통해 이벤트들이 차례대로 실행되어야 하는지, 병렬로 처리되어야 하는지 혹은 두 가지 방법을 결합하여 처리되어야 하는지를 정의합니다. 정책들을 통해 주어진 이벤트의 순번을 받게 됩니다. 만약 정책을 통해 두 개의 이벤트에 대해 같은 순번을 받았다면, 두 개의 이벤트는 반드시 차례대로 처리되어야 하는 것을 말합니다. 순번이 `null`인 경우, 다른 이벤트와 상관없이 해당 이벤트는 병렬로 처리될 수 있다는 것을 말합니다.

Axon은 다음과 같이 `FullConcurrencyPolicy`, `SequentialPolicy` 그리고 `SequentialPerAggregatePolicy`들과 같은 공통 정책을 제공합니다.

* `FullConcurrencyPolicy`는 특정 이벤트 처리자가 모든 이벤트를 동시에 처리할 수 있다는 것을 말하며, 이벤트 간에 특정 순서로 처리되어야 할 필요가 없음을 의미합니다.

* `SequentialPolicy`는 모든 이벤트가 차례대로 실행되도록 합니다. 특정 이벤트는 이전 이벤트의 처리가 종료되어야 처리될 수 있습니다.

* `SequentialPerAggregatePolicy`를 통해 같은 aggregate에서 발생한 도메인 이벤트들을 차례대로 처리되도록 강제합니다. 그렇지만, 다른 aggregate에서 발생한 이벤트들은 동시에 처리될 수 있습니다. 이런 처리 방식은 데이터베이스 테이블들의 aggregate의 상세를 갱신하는 이벤트 리스너를 사용할 때 적당합니다.

기본적으로 제공되는 정책 외에도, 필요에 맞게 새로운 정책을 직접 정의하여 사용할 수 있습니다. 정책을 직접 정의할 때는 `SequencingPolicy` 인터페이스를 반드시 구현해야 하며, `SequencingPolicy` 인터페이스가 포함하고 있는 하나의 메서드 `getSequenceIdentifierFor` 메서드를 구현해야 합니다. `getSequenceIdentifierFor` 메서드는 주어진 이벤트에 대한 순번을 반환하는 메서드 입니다. 위에서도 말한 것처럼, 같은 순번을 가지는 이벤트들을 반드시 차례대로 처리되어야 하며, 다른 순번을 가지는 이벤트들은 동시에 처리될 수 있습니다. 성능상의 이유로, 정책 구현 객체는 다른 이벤트에 상관없이 병렬로 처리될 수 있는 이벤트에 대해선 `null`을 반환해야 합니다. `null`을 반환하면, Axon 내부적으로 이벤트 처리에 대한 제약 사항들을 확인하지 않아도 되므로 빠르게 처리할 수 있습니다.

`AsynchronousEventProcessingStrategy`를 사용할 경우, `ErrorHandler`를 명시적으로 정의하는 것을 권장합니다. 기본 `ErrorHandler`는 발생한 예외를 상위로 전달하게 되지만, 비동기 실행환경에선, 예외를 Executor를 제외한 전달할 대상이 없습니다. 이로 인해 이벤트가 처리되지 않는 상황이 발생할 수 있습니다. 대신 에러 사항을 보고한 후 이벤트 처리를 계속할 수 있는 `ErrorHandler`를 사용하는 것을 권장합니다. `ErrorHandler`를 `EventProcessingStrategy`를 받는 `SubscribingEventProcessor`의 생성자를 통해 설정합니다.
