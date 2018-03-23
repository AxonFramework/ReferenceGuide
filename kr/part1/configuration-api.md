설정 API
=================

Axon을 통해 비즈니스 로직과 인프라 관련 설정을 서로 영향을 주지 않도록 분리하여 설정할 수 있습니다. 메시지를 처리할 때 발생할 수 있는 트랜잭션 처리를 위한 트랜잭션 관리(management)와 같은 인프라 관련 고려 사항들이 있을 수 있습니다. 이런 고려 사항들은 Axon에서 제공하는 빌딩 블록을 통해 처리할 수 있습니다. 실제 메시지의 페이로드와 처리자(핸들러)의 내용은 Axon에 독립적인 (가능한 한 최대로) Java 클래스로 구현할 수 있습니다.

이런 인프라 관련 컴포넌트들에 대한 설정을 더욱 쉽게 만들고 각 기능 컴포넌트 간의 관계 정의를 하기 위한 구성 API를 제공합니다.

설정 구성하기
--------------------------

아래와 같이 기본 설정 객체를 받아오기는 굉장히 쉽습니다

``` java
Configuration config = DefaultConfigurer.defaultConfiguration()
                                        .buildConfiguration();
```

위와 같은 구성을 통해 메시지를 처리하는 구현체를 사용하여 메시지를 전달하는 빌딩 블록들을 사용할 수 있습니다.

위의 구성은 너무 단순하여 그다지 유용하지 못합니다. 이 구성을 유용하게 사용하기 위해선, 명령 모델들과 이벤트 처리자(핸들러)들을 이 구성에 등록해야 합니다.

명령 모델들과 이벤트 처리자(핸들러)들을 이 구성에 등록하기 위해선, `.defaultConfiguration()`을 통해 반환받은 `Configurer` 인스턴스를 사용해야 합니다.

``` java
Configurer configurer = DefaultConfigurer.defaultConfiguration();
```

Configurer 객체는 명령 모델 혹은 이벤트 처리자를 등록하기 위한 많은 메서드들을 제공합니다. 더욱 더 상세한 등록과 설정 방법은 각 구성 요소를 설명하는 장(chapter)에서 다룹니다.

컴포넌트를 등록하는 일반적인 방법은 아래와 같습니다.:

``` java
Configurer configurer = DefaultConfigurer.defaultConfiguration();
configurer.registerCommandHandler(c -> doCreateComponent());
```

위의 예제 코드의 `registerCommandHandler` 호출 부분의 람다 표현식을 주목해서 보세요. 해당 표현 식의 매개 변수인 `c`는 전체 구성을 설명하는 구성 객체입니다. 만일 직접 구현한 컴포넌트에서 기능상의 필요로 다른 컴포넌트들이 필요할 경우, 이 설정 객체를 통해 필요한 객체를 받아 볼 수 있습니다.

예를 들어, 직렬화 객체(serializer)를 필요로 하는 명령 처리자(핸들러)를 등록하기 위한 코드는 다음과 같습니다.:

``` java
configurer.registerCommandHandler(c -> new MyCommandHandler(c.serializer());
```

모든 컴포넌트가 명시적인 접근 메세드를 가지고 있는 것은 아닙니다. 다음의 예제 코드는 특정 컴포넌트를 설정 객체로부터 받아오는 내용을 보여줍니다.
``` java
configurer.registerCommandHandler(c -> new MyCommandHandler(c.getComponent(MyOtherComponent.class));
```

`configurer.registerComponent(componentType, builderFunction)` 메서드를 사용하여, 해당 컴포넌트를 반드시 Configurer에 등록해야 합니다. 매개변수로 전달되는 빌더 함수(builder function)는 `Configuration`을 입력 매개변수로 받게 됩니다.

스프링을 사용하여 설정 구성하기
---------------------------------------

스프링 프레임워크(이하 스프링)를 사용하면, 명시적으로 ```Configurer```를 사용할 필요가 없습니다. 대신, 단지 스프링 ```@Configuration``` 클래스 중 하나에 ```@EnableAxon``` 애노테이션을 붙여주면 됩니다.

Axon은 빌딩 블록의 특정 구현체들을 위치시키기 위해 스프링 애플리케이션 컨텍스트를 사용하고 특정 구현체들이 없을 경우 기본 구현 객체들을 제공합니다. 그래서 ```Configurer```를 사용해 빌딩 블록을 등록하는 대신, 스프링 애플리케이션 컨텍스트에 등록되는 ```@Bean``` 객체들로 선언하여 사용하면 됩니다.
