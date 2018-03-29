스프링 부트 자동 설정
=============================

Axon은 스프링 부트의 자동 설정을 지원하며, 이를 통해 Axon의 인프라-스트럭쳐 컴포넌트들을 아주 쉽게 설정할 수 있습니다. `axon-spring-boot-starter`를 의존성 항목에 추가하면, Axon은 커맨드 버스, 이벤트 버스와 같은 기본 인프라-스트럭쳐 컴포넌트들을 자동으로 설정할 뿐만 아니라, Aggregate와 Saga들을 실행하고 저장하는데 사용되는 컴포넌트들 또한 자동으로 설정합니다.

스프링 애플리케이션 컨텍스트 내에 존재하는 다른 컴포넌트에 따라, 스프링 애플리케이션 컨텍스트 내에 명시적으로 존재 하지 않는 컴포넌트들을 Axon은 정의하게 됩니다. 즉, 기본으로 설정되는 컴포넌트와는 다른 컴포넌트를 사용하길 원하는 상황에 해당 컴포넌트만 설정하면 됩니다.

이벤트 버스와 이벤트 스토어 설정
---------------------------------------
JPA가 사용 가능한 경우, JPA 기반의 이벤트 저장 엔진을 사용하는 이벤트 저장소가 기본으로 사용됩니다. 이를 통해 명시적인 설정 없이도 이벤트 소싱을 사용하는 Aggregate의 저장을 할 수 있습니다.

JPA를 사용할 수 없다면, Axon은 `SimpleEventBus`를 기본으로 사용합니다. 다시 말해서 개별 Aggregate에 대한 이벤트 소싱 레퍼지토리가 없다는 것을 명시하거나 스프링 설정에 `EventStorageEngine`을 설정해야 합니다.

JPA가 클래스 패스에 존재하지만 다른 이벤트 저장 엔진을 설정하려면, 이벤트 소싱을 하기 위해 `EventStorageEngine` 타입의 빈(bean)을 정의하거나 이벤트 소싱이 필요 없다면 `EventBus`를 정의하면 됩니다.

커맨드 버스 설정
-------------------------
애플리케이션 컨텍스트에 명시적으로 정의된 `CommandBus`의 구현체가 없다면, `SimpleCommandBus`가 기본으로 사용됩니다. `PlatformTransactionManager`가 애플리케이션 컨텍스트에 설정되어 사용 가능하다면, `CommandBus`는 설정된 `PlatformTransactionManager`를 사용합니다.

`CommandBus`로 정의된 빈(bean)이 `DistributedCommandBus`만 있는 경우라도, Axon은 분산 커맨드 버스에 대한 로컬 세그먼트로 커맨드 버스 구현체를 사용하도록 설정합니다. 설정된 커맨드 버스 구현체 빈은 "localSegment" 한정자(Qualifier)로 지정됩니다. `DistributedCommandBus`를 `@Primary` 에노테이션을 사용하여 정의하여 의존성 주입에 대한 높은 우선순위를 부여하는 것을 권장합니다.

쿼리(질의) 버스 설정
-----------------------
애플리케이션 컨텍스트에 명시적으로 정의된 `QueryBus` 구현체가 없는 경우, `SimpleQueryBus`가 기본으로 사용됩니다. `QueryBus`는 트랜잭션들을 관리하는 `TransactionManager`를 사용합니다.

트랜잭션 관리자 설정
---------------------------------
애플리케이션 컨텍스트에 명시적으로 정의된 `TransactionManager`의 구현체가 없는 경우, Axon은 스프링 `PlatformTransactionManager` 빈을 찾아서 `TransactionManager`로 포함시켜 `PlatformTransactionManager`를 통해 실제 트랜잭션을 관리하도록 합니다. 스프링 빈을 사용할 수 없는 경우에는 `NoOpTransactionManager`가 사용됩니다.

직렬화 객체 설정
------------------------
기본적으로, Axon은 객체들을 직렬화하기 위해 XStream 기반의 직렬화 객체를 사용합니다. 그렇지만 `Serializer` 타입의 빈을 애플리케이션 컨텍스트에 정의하여 기본으로 사용되는 직렬화 객체를 바꿀 수 있습니다.

기본 직렬화 객체는 대부분 보기 어려운 XML 기반 형식을 제공하는지 하지만, 사실상 모든 항목을 직렬화 및 역 직렬화할 수 있으므로 기본 설정으로 사용하기에는 매우 좋습니다. 하지만 오랜 시간 동안 저장할 필요가 있고 애플리케이션의 경계를 넘어 공유될 수도 있는 이벤트들에 대해서, 형식을 제정의 할 필요가 있습니다.

`eventSerializer` 한정자(qualifier)를 해당 객체에 사용하여, 이벤트를 직렬화하는데 사용되는 별도의 직렬화 객체를 정의할 수 있습니다. Axon은 `eventSerializer` 한정자가 사용된 빈 객체를 이벤트 직렬화 객체로 간주하게 됩니다. 만약 정의된 빈 객체가 없다면, 기본 직렬화 객체가 다른 모든 객체를 직렬화 하는 데 사용이 될 것입니다.

예제 코드는 `eventSerializer` 한정자를 사용하여 직렬화 객체를 정의하는 코드입니다.:
```java
@Qualifier("eventSerializer")
@Bean
public Serializer eventSerializer() {
    return new JacksonSerializer();
}
```

기본 직렬화 객체를 재정의하는 것과 이벤트 직렬화 객체를 정의하는 것 모두 사용하는 경우, 스프링에 기본 직렬화 객체가 어떤 것인지 알려 주어야 합니다. 아래의 코드를 통해 그 방법을 알 수 있습니다.
```java

@Primary // 우선 순위가 높은 빈으로 설정하여, 특정 한정자가 요청되지 않으면 기본으로사용되도록 합니다.
@Bean
public Serializer serializer() {
    return new MyCustomSerializer();
}

@Qualifier("eventSerializer")
@Bean
public Serializer eventSerializer() {
    return new JacksonSerializer();
}
```

Aggregate 설정
-----------------------
`org.axonframework.spring.stereotype` 패키지에 포함된 `@Aggregate` 에노테이션을 통해 이 에노테이션이 사용된 타입을 Aggregate로 만드는데 필요한 컴포넌트들을 자동으로 설정할 수 있습니다. Aggregate루트 객체에만 `@Aggregate` 에노테이션을 사용해야 하는 것에 유의하세요.

Axon은 자동으로 `@CommandHandler` 에노테이션이 사용된 모든 메서드들을 커맨드 버스에 등록할 것이고 레퍼지토리가 없다면 레퍼지토리도 설정할 것입니다.

기본 레퍼지토리 대신 다른 레퍼지토리를 사용하려면, 애플리케이션 컨텍스트에 다른 레퍼지토리를 정의해야 합니다. `@Aggregate` 에노테이션의 `repository` 속성에 사용하자고 하는 레퍼지토리의 이름을 정의할 수도 있습니다. 단 필수 사항은 아닙니다. `repository` 속성을 정의하지 않으면, Axon은 aggregate의 이름에 `Repository`라는 접미사를 사용하여 레퍼지토리 이름으로 사용할 것입니다. (단 첫 글자는 소문자로 시작합니다) `MyAggregate`라는 타입의 클래스의 경우, 기본 레퍼지토리 이름은 `myAggregateRepository`가 됩니다. 만약 해당 이름을 가지는 빈을 찾지 못하면, Axon은 `EventSourcingRepository`를 정의할 것입니다. (단 사용 가능한 `EventStore`가 없는 경우, 실패하게 됩니다.)

Saga 설정
------------------
사가(Saga)들을 작동 시키기 위한 인프라 컴포넌트들의 설정을 하기 위해선 `org.AxonFramework.spring.stereotype` 패키지에 있는 `@Saga` 에노테이션을 사용해야 합니다. Axon은 `SagaManager`와 `SagaRepository`를 설정하게 됩니다. Saga 레퍼지토리는 실제 Saga들을 저장하는 저장소로 사용할 `SagaStore`를 애플리케이션 컨텍스트내의 사용 가능한 `SagaStore`를 사용하게 됩니다. 만약 JPA를 사용하는 경우에는 `JPASagaStore`를 기본으로 사용합니다.

Saga들을 저장하기 위해 다른 `SagaStore`를 사용하려면, `@Saga` 에노테이션의 `sagaStore` 속성에 사용된 속성값을 이름으로 가지는 `SagaStore` 타입의 빈을 제공해야 합니다.

Saga들이 필요로 하는 자원들을 애플리케이션 컨텍스트로부터 주입할 수 있습니다. 그런데 이것은 해당 필요 자원들을 주입하기 위해 스프링 기반의 의존성 주입이 사용된다는 것을 의미하는 것은 아닙니다. `@Autowired`와 `@javax.inject.Inject` 에노테이션들은 의존성 자원을 구별하는데 사용될 수 있지만, Axon이 이런 에노테이션들이 사용된 항목들과 메서드들을 찾아 필요 자원들은 주입합니다. 하지만 생성자 주입은 아직 지원되지 않습니다.

Saga들의 구성(설정)을 조정하기 위해, 사용자 정의 SagaConfiguration 빈을 정의할 수 있습니다. Axon은 에노테이션이 사용된 Saga 클래스들에 대한 설정을 특정 이름을 가지는 `SagaConfiguration` 타입의 빈을 확인하여 찾아냅니다. `MySaga`라는 Saga 클래스에 대해, Axon은 `mySagaConfiguration`을 찾으며, 만약 해당 빈이 없다면, 사용 가능한 컴포넌트들을 가지고 설정 객체를 생성합니다.

만약 에노테이션이 사용된 Saga에 대한 `SagaConfiguration` 인스턴스가 존재한다면, 해당 설정 인스턴스를 사용하여 해당 타입의 Saga를 위한 컴포넌트들을 검색하고 등록합니다. 그런데 SagaConfiguration 빈이 위에 언급된 것과 같은 이름을 가지지 않는다면, 해당 Saga는 두 번 등록되어 중복된 이벤트를 수신할 수 있습니다. 이를 방지하기 위해, 아래와 같이 `@Saga` 에노테이션을 사용하는 `SagaConfiguration`의 이름을 지정할 수 있습니다.

```java
@Saga(configurationBean = "mySagaConfigBean")
public class MySaga {
    // 메서드들을 선언합니다.
}

// 아래의 코드는 spring configuration안세 작성합니다.
@Bean
public SagaConfiguration<MySaga> mySagaConfigBean() {
    // SagaConfiguration 인스턴스를 생성하고 반환합니다.
}
```

이벤트 처리 설정
----------------------------
기본적으로, `@EventHandler` 에노테이션이 사용된 메서드를 포함하는 모든 싱글 톤 스코프로 등록된 스프링 빈 컴포넌트들은 이벤트 버스로 게시된 이벤트 메시지들을 수신하는 이벤트 프로세서에 구독 등록을 하게 됩니다.

애플리케이션 컨텍스트에 등록되어 사용 가능한 `EventHandlingConfiguration` 빈은 이벤트 처리자들에 대한 설정을 조정할 수 있는 메서드들을 가지고 있습니다. 이벤트 처리자와 이벤트 프로세서들에 대한 상세 설정 내용은 [설정 API](../part1/configuration-api.md)를 참고하시면 됩니다.

이벤트 처리 설정을 수정하려면, 변경하고자 하는 설정 내용을 포함하는 메서드를 정의하고 `@Autowired` 에노테이션을 해당 메서드에 설정하면 됩니다.

```java
@Autowired
public void configure(EventHandlingConfiguration config) {
    config.usingTrackingProcessors(); // 모든 프로세서들은 tracking mode를 기본으로 사용합니다.
}
```

`application.properties`를 통해서도 이벤트 처리자의 특정 설정을 변경하여 사용할 수 있습니다.

```properties
axon.eventhandling.processors.name.mode=tracking
axon.eventhandling.processors.name.source=eventBus
```

프로세서 이름에 마침표(`.`)들이 들어가 있다면, 맵 표기법을 사용하세요.
```properties
axon.eventhandling.processors["name"].mode=tracking
axon.eventhandling.processors["name"].source=eventBus
```

혹은 application.yml 파일을 사용할 수 있습니다.
```yaml
axon:
    eventhandling:
        processors:
            name:
                mode: tracking
                source: eventBus
```

위의 예제 코드들에서 `name` 하위의 `source` 속성은 명시된 이벤트 프로세서의 이벤트 소스로 사용되는 `SubscribableMessageSource` 혹은 `StreamableMessageSource` 의 구현체 빈의 이름에 대한 참조입니다. 애플리케이션 컨텍스트에 정의된 이벤트 버스 혹은 이벤트 저장소를 기본 이벤트 소스로 사용합니다.

쿼리(질의) 처리 설정
----------------------------
모든 싱글 톤 스프링 빈들을 스캔하여 `@QueryHandler` 에노테이션이 사용된 메서드들을 찾아냅니다. 발견된 `@QueryHandler` 에노테이션이 사용된 메서드들은 각각 쿼리 버스에 질의 처리자로 등록됩니다.

### 병렬 프로세싱 ###
Tracking 프로세서들은 다수의 스레드를 사용하여 이벤트를 병렬로 처리할 수 있습니다. 모든 스레드가 같은 모드로 실행될 필요는 없습니다.

Tracking 프로세서는 해당 인스턴스에서 사용할 스레드의 개수와 프로세서가 정의해야 하는 세그먼트의 초기 개수를 설정할 수 있습니다.

```properties
axon.eventhandling.processors.name.mode=tracking
# 정의된 모드에서 사용할 스레드의 최대 개수를 설정합니다.
axon.eventhandling.processors.name.threadCount=2
# 세그먼트의 초기 개수를 설정합니다. (예, 전체 스레드의 최대 개수를 정의합니다.)
axon.eventhandling.processors.name.initialSegmentCount=4
```

AMQP 활성화
-------------
AMQP와 연동을 하기 위해서, 반드시 `axon-amqp` 모듈이 클래스 패스에 있어야 하며 AMQP `ConnectionFactory`를 애플리케이션 컨텍스에 사용 가능한 상태로 등록되어 있어야 합니다. (예, `spring-boot-starter-amqp`를 의존성(dependency)에 추가하여 AMQP `ConnectionFactory`를 애플리케이션 컨텍스에 등록합니다)

애플리케이션내에서 생성된 이벤트들을 AMQP 채널로 전달하려면, 아래와 같은 단 한 줄의 `application.properties` 설정을 추가하기만 하면 됩니다.
```properties
axon.amqp.exchange=ExchangeName
```
위와 같이 설정하면, 게시된 이벤트들은 자동으로 설정된 이름에 해당하는 AMQP Exchange로 전달됩니다. 기본적으로, 이벤트를 전달할 때, 어떤 AMQP 트랜잭션들도 사용되지 않지만, `axon.amqp.transaction-mode` 속성값을 `trasactional`혹은 `publisher-check`으로 재정의하여 변경할 수 있습니다.

큐로부터 이벤트들을 받고 받은 이벤트들을 Axon 애플리케이션 안에서 처리하려면, `SpringAMQPMessageSource`를 설정해야 합니다.
```java
@Bean
public SpringAMQPMessageSource myQueueMessageSource(AMQPMessageConverter messageConverter) {
    return new SpringAMQPMessageSource(messageConverter) {

        @RabbitListener(queues = "myQueue")
        @Override
        public void onMessage(Message message, Channel channel) throws Exception {
            super.onMessage(message, channel);
        }
    };
}
```
위와 같이 `SpringAMQPMessageSource`를 설정한 후에, 설정한 `SpringAMQPMessageSource` 빈을 메시지 소스로 사용하도록 프로세서를 설정합니다.
```properties
axon.eventhandling.processors.name.source=myQueueMessageSource
```


분산 명령 처리
---------------------

대부분은, 설정 파일을 변경하지 않고 분산 커맨드를 구성할 수 있습니다.  

첫 번째로, 분산 커맨드 버스 모듈을 의존성(dependency)에 추가합니다. (예, JGroups 혹은 SpringCloud)

분산 커맨드 버스 모듈을 의존성(dependency)에 추가한 후, 애플리케이션 컨텍스에 아래와 같은 분산 커맨드 버스를 활성화하는 하나의 속성을 추가합니다.

```properties
axon.distributed.enabled=true
```

사용된 커넥터의 타입에 상관없이 공통으로 사용되는 설정은 아래와 같습니다.
```properties
axon.distributed.load-factor=100
```

`CommandRouter`와 `CommandBusConnector`들이 애플리케이션 컨텍스트에 존재하면, Axon은 자동으로 분산 커맨드 버스를 설정하게 됩니다. 이 경우, `axon.distributed.enabled` 속성은 명시하지 않아도 되며, 라우터와 커넥터들의 자동 설정이 활성화됩니다.

### JGroup 사용하기 ###

분산 커맨드 버스 모듈은 다른 노드들을 발견하고 서로 통신하기 위해 JGroup들을 사용합니다. 자동 설정(AutoConfiguration)은 `JGroupsConnector`를 기본 설정값들로 설정합니다. 해당 기본 설정값들은 각각의 환경에 맞게 조정해야 할 수도 있습니다.

기본으로, JGroupConnector는 GossipRouter를 로컬 호스트의 포트 12001번을 사용하도록 위치시킵니다.

JGroupConnector에 대한 설정값들은 모두 `axon.distributed.jgroups`라는 접두사를 사용합니다.

```properties
# JGroupConnector 인스턴스에 바인드 되는 주소, 기본적으로 글로벌 IP 주소를 찾습니다.
axon.distributed.jgroups.bind-addr=GLOBAL
# 로컬 인스턴스에 바인드할 포트 번호
axon.distributed.jgroups.bind-port=7800

# 개별 노드들이 연결할 JGroups Cluster의 이름
axon.distributed.jgroups.cluster-name=Axon

# JGroups 설정 내용을 담고 있는 JGroups 설정 파일
axon.distributed.jgroups.configuration-file=default_tcp_gossip.xml

# Gossip 서버들의 IP 주소와 포트 번호 (콤마(,)로 구분하여 명시합니다.)
axon.distributed.jgroups.gossip.hosts=localhost[12001]
# true값으로 설정할 경우, Gossip Server를 gossip hosts에 명시된 주소들 중 첫 번째 주소를 사용하여 내장된 Gossip Server를 시작합니다.
axon.distributed.jgroups.gossip.auto-start=false
```

JGroups 설정 파일은 커넥터의 기능들을 좀 더 세세하게 제어하기 위해 사용합니다. 더 많은 정보를 보려면, [JGroups의 참조 문서](http://www.jgroups.org/manual4/index.html)를 살펴보세요.

### 스프링 클라우드 사용하기 ###

스프링 클라우드는 디스커버리(그룹 혹은 클러스터내의 특정 노드를 찾는 것)에 대한 훌륭한 추상화를 제공합니다. Axon은 스프링 클라우드의 사용성을 보고하고 다른 커멘트 버스 노드들을 찾기 위해 이런 추상화들을 사용할 수 있습니다. 커멘트 버스 노드들과의 통신을 위해, Axon은 기본으로 스프링 HTTP를 사용합니다.

스프링 클라우드를 사용하기 위해 설정해야 할 내용은 많지 않습니다. 스프링 클라우드 자동 설정은 이미 존재하는 스프링 클라우드 디스커버리(Discovery) 클라이언트를 사용합니다. (단, 반드시 `@EnableDiscoveryClient`가 사용이 되었는지와 필요한 클라이언트가 클래스 패스에 추가되어 있어야 합니다)

그렇지만, 일부 디스커버리 클라이언트들은 인스턴스 메타데이터를 동적으로 갱신할 수 없는데, Axon은 이를 감지하면 HTTP를 사용해서 해당 노드를 질의합니다. 이는 보통 30초 간격으로 발생하는 각각의 디스커버리 허트비트(heartbeat)마다 이루어집니다.

이런 기능은 `application.properties`에 선언할 수 있는 다음의 설정을 통해 활성/비활성화할 수 있습니다.

```properties
# 사용 가능한 메터 데이터가 없는 경우, http를 사용할 것인지를 설정
axon.distributed.spring-cloud.fallback-to-http-get=true
# 로컬 데이터를 게시하고 다른 노드들의 데이터를 수신할 주소를 설정
axon.distributed.spring-cloud.fallback-url=/message-routing-information
```

더 세세한 제어를 위해선, `SpringCloudHttpBackupCommandRouter` 혹은 `SpringCloudCommandRouter`를 애플리케이션 컨텍스트에 등록해야 합니다.
