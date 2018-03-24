이벤트 처리
===============

이벤트 리스너는 애플리케이션 내에서 발생한 이벤트를 받아 처리하는 컴포넌트입니다. 이벤트 리스너들은 일반적으로 명령 모델에 의해 결정된 사항을 기반으로 보통 뷰 모델(View model)을 갱신하거나 변경 내용을 제삼자 서비스 등의 다른 컴포넌트로 전달하는 등의 로직을 수행합니다. 몇몇 경우에, 이벤트 처리자 스스로 수신한 이벤트를 기반으로 이벤트를 발생시키거나 추가 변경의 필요로 명령을 전송할 수 있습니다.

이벤트 처리자 정의하기
-----------------------

`@EventHandler` 에노테이션을 사용하여 하나의 객체에 다수의 이벤트 처리 메서드를 선언할 수 있습니다. 이벤트 처리자가 처리할 이벤트는 이벤트 처리 메서드의 매개변수로 선언합니다.

이벤트 처리 메서드에 선언 가능한 매개변수 타입은 아래와 같습니다.

* 첫 번째 매개변수는 항상 이벤트 메시지의 페이로드이어야 합니다. 메시지의 페이로드에 접근할 필요가 없는 경우, `@EventHandler` 에노테이션의 `payloadType`에 원하는 페이로드 타입을 명시해주어야 합니다. `payloadType`을 명시적으로 선언한 경우에는 첫 번째 매개변수는 아래의 설명된 규칙에 따라 결정이 됩니다. 메시지의 페이로드가 매개변수로 전달되어야 한다면, `@EventHandler` 에노테이션의 `payloadType`를 선언해선 안 됩니다.

* `@MetaDataValue` 에노테이션이 붙은 매개변수는 에노테이션에 명시된 키에 해당하는 메타 데이터값으로 해석되어 메서드로 전달됩니다. `@MetaDataValue` 에노테이션에 `required` 속성을 기본값인 `false`로 주게 되면, 없는 메타 데이터를 `null`로 처리하여 넘겨줍니다. 반면 `required`가 `true`이고 해당 메타 데이터가 존재하지 않으면, 해당 메서드는 호출되지 않습니다.

* `MetaData` 타입의 매개변수는 주입된 `CommandMessage`의 전체 `MetaData`를 포함하게 됩니다.

* `@Timestamp` 에노테이션이 붙고 타입이 `java.time.Instant` (혹은 java.time.temporal.Temporal) 인 매개변수는 `EventMessage`의 타임스탬프값으로 전달 됩니다. 해당 타임스탬프 값은 이벤트가 생성된 시간 값을 가집니다.

* `@SequenceNumber` 에노테이션이 붙고 타입이 `java.lang.Long` 혹은 `long` 타입의 매개변수는 `DomainEventMessage`의 `sequenceNumber` 값으로 전달이 됩니다. 해당 `sequenceNumber` 값은 이벤트가 발생한 Aggregate의 범위 내에서의 순서를 의미합니다.

* 매개변수가 [Message](http://www.axonframework.org/apidocs/3.0/org/axonframework/messaging/Message.html) 하위 구현체인 경우, 해당 메서드는 전체 `EventMessage`를 주입 받습니다. (메시지가 매개변수에 할당 가능한 경우에만 가능합니다) 첫 번째 매개변수가 [Message](http://www.axonframework.org/apidocs/3.0/org/axonframework/messaging/Message.html) 인 경우, Java의 타입 이레이져(type erasure)가 컴파일 후에 제너릭 타입을 지우기 때문에 제너릭 타입을 명시하더라도 매개변수의 타입을 알 수 없습니다. 이런 경우, 페이로드 타입의 매개변수를 먼저 선언하고, 그다음에 [Message](http://www.axonframework.org/apidocs/3.0/org/axonframework/messaging/Message.html)의 매개변수를 선언해야 합니다.

* 스프링을 사용하고 Axon Spring Boot Starter 모듈에 대한 의존성을 선언하거나 `@EnableAxon` 에노테이션을 `@Configuration` 클래스에 사용하여 Axon Configuration이 활성화되면, 다른 모든 매개변수들은 주입 가능한 빈(bean)이 애플리케이션 컨텍스트내에 있다면 자동으로 주입이 됩니다. 이를 활용하여 `@EventHandler` 에노테이션이 붙은 메서드에 자동으로 필요한 자원을 주입할 수 있습니다.

`ParameterResolverFactory` 인터페이스를 직접 구현하고 `/META-INF/service/org.axonframework.common.annotation.ParameterResolverFactory` 파일을 생성한 후 구현체 클래스를 설정하여 `ParameterResolver`들을 추가적으로 설정할 수 있습니다. 더 상세한 내용은 [고급 사용자 지정](../part4/advanced-customizations.md)을 참고하세요.

모든 상황에서, 리스너 인스턴스마다 하나의 이벤트 처리 메서드가 호출이 됩니다. Axon은 다음과 같은 규칙을 적용하여 가장 적합한 호출 대상 메서드를 찾아냅니다.

1. 실제 인스턴스의 클래스(this.getClass()로 반환되는 클래스) 계층 구조상에서, ```@EventHandler``` 에노테이션이 붙은 메서드들을 평가합니다.

2. 모든 매개변수를 값으로 치환할 수 있는, 하나 이상의 메서드가 발견되었을 경우, 가장 구체적인 타입을 선언한 메서드를 선택합니다.

3. 현재 클래스 구조상에서 메서드를 발견할 수 없는 경우, 상위 타입의 클래스 구조를 같은 방법으로 탐색합니다.

4. 클래스 구조상의 최상위에서도 메서드를 발견할 수 없는 경우, 해당 이벤트는 무시됩니다.

```java
// EventB는 EventA를 상속한다고 가정합니다.
// EventC는 EventB를 상속한다고 가정합니다.
// SubListener의 단일 인스턴스가 등록되었다고 가정합니다.

public class TopListener {

    @EventHandler
    public void handle(EventA event) {
    }

    @EventHandler
    public void handle(EventC event) {
    }
}

public class SubListener extends TopListener {

    @EventHandler
    public void handle(EventB event) {
    }
}
```

위의 예제를 살펴보면, `SubListener`의 이벤트 처리 메서드는 `EventB`와 `EventC` 인스턴스에 대해 호출이 됩니다. 다시 말하면, `TopListener`의 이벤트 처리 메서드는 `EventC`에 대해 전혀 호출되지 않습니다. 그 이유는 3번 규칙 때문인데 현재의 클래스 레벨인 `SubListener`에서 `EventC`의 상위 타입인 `EventB`를 처리 할 수 있도록 정의 되어 있기 때문입니다. `EventA` 인스턴스는 `EventB`로 다운 캐스팅할 수 없기 때문에, `EventA` 타입의 이벤트들은 `TopListener`의 이벤트 처리 메서드로 처리됩니다.

이벤트 처리자 등록하기
-----------------------------
이벤트를 처리할 수 있는 컴포넌트들은 `EventHandlingConfiguration`을 사용하여 정의할 수 있습니다. `EventHandlingConfiguration`은 Axon의 전역 설정 객체인 `Configurer`에 모듈로 등록되는 객체입니다. 일반적으로 하나의 애플리케이션은 정의된 하나의 `EventHandlingConfiguration` 객체를 가지지만, 모듈화되어 있는 더 규모가 큰 애플리케이션은 모듈마다 `EventHandlingConfiguration` 객체를 가질 수도 있습니다.

`@EventHandler` 메서드를 가진 객체를 등록하기 위해선, `EventHandlingConfiguration`의 `registerEventHandler` 메서드를 사용합니다.

```java
// EventHandlingConfiguration 정의
EventHandlingConfiguration ehConfiguration = new EventHandlingConfiguration()
    .registerEventHandler(conf -> new MyEventHandlerClass());

//  Axon Configuration에 정의된 EventHandlingConfiguration 등록
Configurer axonConfigurer = DefaultConfigurer.defaultConfiguration()
    .registerModule(ehConfiguration);
```

스프링의 자동 설정(AutoConfiguration)을 사용하여 이벤트 처리자들을 등록하는 상세한 내용은 [이벤트 처리 설정](../part3/spring-boot-autoconfig.md#event-handling-configuration)을 참조하시면 됩니다.
