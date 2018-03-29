복잡한 비즈니스 트랜잭션 관리하기
======================================

모든 명령을 단일 ACID 트랜잭션으로 완전히 처리할 수는 없습니다. 가장 대표적인 예로 이체를 들 수 있습니다. 보통 하나의 계좌에서 다른 계좌로 돈을 이체하기 위해서는 원자 적이고 일관된 트랜잭션 처리가 필요하다고 보지만, 실은 그렇지 않습니다. 오히려 그렇게 하는 것은 불가능합니다. 은행 A의 한 계좌에서 은행 B의 계좌로 돈을 이체할 경우, 은행 A가 은행 B의 데이터베이스의 잠금(lock)을 얻을 수 있을까요? 이체가 진행 중이라면, 은행 B가 이체된 금액을 아직 입금하지 않았는데, 은행 A가 이체 금액을 계좌에서 차감할 수 있을까요? 실제로 은행 간 이체는 "진행 중" 입니다. 한편, 은행 B에서 이체 금액을 입금하는 동안 문제가 발생하여 뭔가 잘못되었다면, 은행 A의 고객에게 이체 금액을 되돌려 주어야 합니다. 따라서 궁극적으로 다른 형태의 일관성이 필요합니다.

ACID 트랜잭션은 불필요하거나 혹은 몇몇 경우 ACID 트랜잭션을 통한 처리는 불가능하지만, 다른 형태의 트랜잭션 관리는 여전히 필요합니다. 일반적으로 이런 트랜잭션들은 BASE(**B**asic, **A**vailability, **S**oft state, **E**ventual consistency) 트랜잭션이라고 합니다. ACID와는 반대로, BASE 트랜잭션들은 쉽게 되돌릴(롤백:rollback) 수 없습니다. 롤백하기 위해선, 트랜잭션 일부로 이미 발생한 것을 되돌릴 작업을 통해 상쇄시켜야 합니다. 은행 간 이체 예제에서, 은행 B에서 이체 금액의 입금이 실패했을 때 은행 A로 해당 금액을 다시 되돌려 주게 됩니다.

CQRS에선, Saga들을 통해 BASE 트랜잭션들을 처리할 수 있습니다. Saga들은 이벤트들에 응답하고 명령을 보내며 외부 애플리케이션을 호출할 수 있습니다. 도메인 주도 설계(Domain Driven Design)에 따르면, 여러 바운디드 컨텍스트(bounded context)간 조정 메커니즘으로 Saga들을 사용하는 것은 흔한 일입니다.

사가(Saga)
====

Saga는 비즈니스 트랜잭션을 관리할 수 있는 특별한 유형의 이벤트 리스너입니다. 일반적인 트랜잭션들은 거의 즉시 처리되어 완료되지만, 일부 트랜잭션들은 며칠 혹은 몇 주에 걸쳐 진행될 수 있습니다. Axon에서 제공하는 Saga의 개별 인스턴스를 통해 단일 비즈니스 트랜잭션을 처리할 수 있습니다. 다시 말해, Saga는 트랜잭션을 관리하기 위한 속성을 유지하여, 트랜잭션의 진행 혹은 이미 발생한 작업에 대한 롤백(rollback) 작업을 처리할 수 있습니다. 일반적으로, 그리고 보통의 이벤트 리스너와는 반대로, Saga는 이벤트를 기반으로 한 시작 시점과 종료 시점을 가집니다. 보통 Saga의 시작 시점은 분명하나, Saga의 종료 시점은 여러 개가 존재할 수 있습니다.

Axon에서 Saga 클래스는 하나 혹은 그 이상의 `@SagaEventHandler` 에노테이션으로 정의할 수 있습니다. 보통의 이벤트 처리자와는 달리, 특정 시점에 다수의 Saga 인스턴스가 존재할 수 있습니다. Saga 인스턴스들은 정의된 Saga 타입에 대한 이벤트들의 처리를 위한 단일 프로세서(추적 혹은 구독)에 의해 관리됩니다.  

생애 주기
----------

단일 Saga 인스턴스는 단일 트랜잭션에 대한 처리를 담당합니다. 다시 말해, Saga 생애주기의 시작과 종료를 표시할 필요가 있습니다. Saga에서, 이벤트 처리자들에 `@SagaEventHandler` 에노테이션을 붙여야 합니다. 특정 이벤트가 트랜잭션의 시작을 나타낸다면, `@SagaEventHandler` 에노테이션을 사용한 메서드에 `@StartSaga` 에노테이션을 추가로 사용해야 합니다. `@StartSaga` 에노테이션은 새로운 Saga를 생성하고 원하는 이벤트가 발생 했을 때 해당 Saga의 이벤트 처리 메서드를 호출하도록 합니다.

 기본적으로, 새로운 Saga는 이미 생성된 Saga가 없는 경우에만 생성이 됩니다. `@StartSaga` 에노테이션의 `forceNew` 속성값을 `true`로 설정하여, 새로운 Saga 인스턴스를 생성하도록 강제할 수 있습니다.

 Saga의 종료는 두 가지 방법으로 처리할 수 있습니다. 특정 이벤트가 Saga 생애주기의 종료를 언제나 나타낸다면, 해당 이벤트를 처리하는 `@SagaEventHandler` 에노테이션을 사용한 메서드에 `@EndSaga` 에노테이션을 사용하면 됩니다. Saga의 생애주기는 이벤트 처리 메서드 호출 후에 종료됩니다. 다른 방법으로, Saga내에서 `end()` 메서드를 호출하여 Saga의 생애 주기를 종료할 수 있습니다. `end()`메서드를 사용하면 조건에 따라 Saga의 생애 주기를 종료할 수 있습니다.

이벤트 처리
--------------

Saga 내에서의 이벤트 처리는 보통의 이벤트 처리자와 거의 같습니다. 메서드와 매개변수 분석에 대해서도 같은 규칙이 적용됩니다. 큰 차이점은 수신된 이벤트를 처리하는 이벤트 처리자의 인스턴스는 하나만 존재하지만, Saga 인스턴스는 여러 개가 존재할 수도 있고 각기 다른 이벤트에 관심이 있습니다. 예를 들어, 1번 주문과 관련된 트랜잭션을 처리하는 Saga는 2번 주문에 대한 이벤트들에는 관심이 없습니다. 이는 2번 주문을 처리하는 Saga도 같습니다.

모든 이벤트를 모든 Saga 인스턴스에 전달하는 것 -자원을 낭비하는 것이 분명한- 대신, Axon은 Saga와 관련된 속성을 포함하고 있는 이벤트들만을 전달합니다. 이를 위해 `AssociationValue`들을 사용하며, `AssociationValue`는 키와 값으로 구성이 되어 있습니다. 키값은 예를 들어 "주문 아이디" 혹은 "주문"과 같은 식별자의 유형을 나타냅니다. 키와 연결된 값은 "1" 혹은 "2"와 같이 식별자 유형에 대한 값을 나타냅니다.

`@SagaEventHandler` 에노테이션이 사용된 메서드의 전개 순서는 `@EventHandler` 에노테이션이 사용된 메서드의 전개 순서와 동일합니다. (참조: [이벤트 처리자 정의하기](event-handling.md#이벤트-처리자-정의하기)). 수신한 이벤트와 이벤트 처리 메서드의 매개변수들이 일치하고 이벤트 처리 메서드에 정의되어 있는 속성들이 Saga와 연결된 경우에, 해당 메서드는 호출이 됩니다.

`@SagaEventHandler` 에노테이션은 두 개의 속성을 가집니다. 가장 중요한 속성은 `associationProperty`이며, 속성값은 이벤트와 연결된 Saga를 찾기 위해 사용되는 수신 이벤트의 속성 명입니다. 연결 값의 키가 속성의 이름이며, 값은 속성의 getter 메서드를 통해 반환되는 값입니다.

예를 들어, 수신 메시지 객체가 `String getOrderId()` 메서드를 가지며, 해당 메서드는 "123"을 반환한다고 하면, 해당 이벤트를 처리하는 메서드의 `@SagaEventHandler` 에노테이션은 orderId를 `associationProperty`의 값으로 가지고 있어야 합니다. 즉, `@SagaEventHandler(associationProperty="orderId")`와 같이 에노테이션을 사용해야 합니다. 그러면, 이 이벤트는 "orderId"를 키로 가지는 `AssociationValue`와 연결된 모든 Saga에게 전달됩니다. 이는 하나 혹은 그 이상의 Saga에 전달되거나 어떤 Saga에도 전달 되지 않음을 의미합니다.

 때때로, 연결하고자 하는 속성의 이름이 사용하고자 하는 연결 이름이 아닐 때가 있습니다. 예를 들어, 구매 주문에 대한 판매 주문을 처리하기 위한 Saga를 정의해야 한다면, "buyOrderId"와 "sellOrderId"를 가지는 트랜잭션 객체를 정의할 수 있습니다. Saga를 "orderId"로 연결하고자 한다면, `@SagaEventHandler(associationProperty="sellOrderId", keyName="orderId")`와 같이 `@SagaEventHandler` 에노테이션에 다른 키 이름을 정의할 수 있습니다.

객책간 연관 관리 하기
---------------------

Saga를 통해 주문, 배송 그리고 송장 등과 같은 다수의 도메인 개념에 걸쳐 발생하는 트랜잭션을 관리할 때, Saga를 해당 도메인 객체의 인스턴스와 연결 시켜줘야 합니다. Saga와 도메인 객체의 인스턴스 간 연결은 해당 주문 그리고 배송 등과 같은 연결 유형을 식별하기 위한 **키**와 도메인 객체의 식별자를 나타내는 **값**이 필요합니다.

위와 같은 Saga와 도메인 객체는 다양한 방법으로 연결할 수 있습니다. 우선, `@StartSaga` 에노테이션이 사용된 이벤트 처리자를 호출하여 새로운 Saga가 생성되면, 생성된 Saga는 `@SagaEventHandler` 에노테이션에 명시된 속성과 자동으로 연결이 됩니다. 다른 연결은 `SafaLifecycle.associateWith(String key, String/Number value)` 메서드를 사용하여 생성할 수 있습니다. 생성한 특정 연결을 삭제하려면, `SagaLifecycle.removeAssociationWith(String key, String/Number value)` 메서드를 사용하세요.

> *참고*
>
> 사가(Saga) 내의 도메인 개념을 연관시키는 API는 식별자의 문자열 표현이 저장된 연관 값 항목에 필요하므로 의도적으로 문자열 또는 숫자를 식별 값으로 허용합니다. 데이터베이스의 문자열 열 항목은 데이터베이스 엔진 간의 비교를 보다 단순하게 만들기 때문에 간단한 String 표현을 사용하는 API의 간단한 식별자 값을 사용하는 것은 의도적으로 설계된 것입니다. 따라서 Object#toString()는 다루기 힘든 문자열 식별자를 제공할 수 있으므로 예를 들어 associateWith(String, Object) 메서드가 없는 것은 의도적입니다.

주문과 관련된 트랜잭션 처리를 위해 생성된 Saga를 생각해보세요. 주문 생성 이벤트를 처리하는 메서드에 `@StartSaga` 에노테이션이 사용되었기 때문에, 해당 Saga는 자동적으로 주문과 연결이 됩니다. 해당 Saga는 주문에 대한 송장을 생성하고 주문에 대한 배송을 준비시키도록 합니다. 배송이 완료되고 송장에 대한 지급이 완료되면, 트랜잭션과 Saga는 종료됩니다.

위에서 설명한 Saga에 대한 코드는 아래와 같습니다.

``` java
public class OrderManagementSaga {

    private boolean paid = false;
    private boolean delivered = false;
    @Inject
    private transient CommandGateway commandGateway;

    @StartSaga
    @SagaEventHandler(associationProperty = "orderId")
    public void handle(OrderCreatedEvent event) {
        // shipmentId와 invoiceId를 생성합니다.
        ShippingId shipmentId = createShipmentId();
        InvoiceId invoiceId = createInvoiceId();

        // 명령을 전송하기 전에 shipmentId와 invoiceId를 Saga와 연결합니다.
        SagaLifecycle.associateWith("shipmentId", shipmentId);
        SagaLifecycle.associateWith("invoiceId", invoiceId);
        // 명령을 전송합니다.
        commandGateway.send(new PrepareShippingCommand(...));
        commandGateway.send(new CreateInvoiceCommand(...));
    }

    @SagaEventHandler(associationProperty = "shipmentId")
    public void handle(ShippingArrivedEvent event) {
        delivered = true;
        if (paid) { SagaLifecycle.end(); }
    }

    @SagaEventHandler(associationProperty = "invoiceId")
    public void handle(InvoicePaidEvent event) {
        paid = true;
        if (delivered) { SagaLifecycle.end(); }
    }

    // ...
}
```
클라이언트들이 식별자를 생성하게 함으로써, 요청-응답 형태의 명령 없이도 Saga와 도메인 객체들을 쉽게 연결할 수 있습니다. 명령을 보내기 전에, 도메인 객체들과 이벤트를 연결합니다. 이렇게 하면, 명령 일부로 생성되는 이벤트들을 감지할 수 있습니다. 송장에 대한 지급과 배송이 완료되면 Saga를 종료합니다.

마감 시한을 지키도록 하기
--------------------------

비즈니스 행위를 통해 이벤트가 발생하였을 때, Saga를 통해 쉽게 처리할 수 있습니다. 해당 이벤트가 Saga에게 전달되기 때문입니다. 하지만 아무것도 일어나지 않았을 때 Saga를 통해 처리하려면 어떻게 해야 할까요? 이를 위해 마감 시한(deadline)을 사용합니다. 송장의 예에서, 신용 카드 지급은 수초 안에 결제가 이루어지지만, 송장의 완료 처리는 보통 여러 주에 걸쳐 이루어집니다.

Axon은 `EventScheduler`를 제공하여 이벤트를 발생시킬 수 있도록 합니다. 송장의 예에서, 송장이 30일 이내에 지급 완료되길 원한다면, Saga를 통해 `CreateInvoiceCommand`를 보낸 이후, `InvoicePaymentDeadlineExpiredEvent`를 30일이 되는 시점에 발생시킬 수 있습니다. 이벤트 스케쥴러(EventScheduler)는 특정 이벤트를 발생시키도록 일정을 생성(schedule)한 후, `ScheduleToken`을 반환합니다. 송장에 대한 지급이 이루어 지면, 반환된 `ScheduleToken`을 통해 해당 일정을 취소할 수 있습니다.

Axon은 두 개의 `EventScheduler`구현체들을 제공합니다. 하나는 순수 Java로 작성된 것이고 다른 하나는 backing scheduling 메커니즘인 Quartz2를 사용한 구현체입니다.

`EventScheduler`의 순수 자바 구현체는 이벤트 게시 일정을 세우기 위해 `ScheduledExecutorService`를 사용합니다. 이 스케쥴러는 매우 안정적인 타이밍을 제공하지만, 메모리 기반 구현을 제공합니다. 따라서 JVM이 종료되면, 모든 일정이 사라지게 됩니다. 따라서 긴 기간에 걸친 일정을 처리하기에는 적당하지 않습니다.

`SimpleEventScheduler`는 `EventBus`와 `SchedulingExecutorService`를 함께 설정해 주어야 합니다. (`ScheduledExecutorService` 생성 시 필요한 헬퍼 메서드는 [`java.util.concurrent.Executors`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executors.html)를 참고하세요.)

`QuartzEventScheduler`는 더욱더 안정적이고 엔터프라이즈에 적당한 구현체입니다. Quartz를 기반 스케쥴링 메커니즘으로 사용하면, 영속화, 클러스터링 그리고 실패(misfire) 관리와 같은 더 강력한 기능들을 제공합니다. 해당 기능들을 사용하여 이벤트 게시를 확실히 보장할 수 있습니다. 조금 늦을 수는 있지만 결국 이벤트를 게시할 수 있습니다.

`QuartzEventScheduler`는 Quartz `Scheduler`와 `EventBus`를 필요로 합니다. 추가로, 기본적으로 "AxonFramework-Events"로 설정된 Quartz job들이 계획 되어 있는 그룹의 이름을 설정할 수 있습니다.

하나 혹은 그 이상의 컴포넌트들은 계획 되어 있는 이벤트들을 수신할 수 있습니다. 이 컴포넌트들은 컴포넌트를 호출하는 스레드에 묶여 있는 트랜잭션에 의존할 수 있습니다. 계획 된 이벤트들은 `EventScheduler`를 관리하는 스레드에 의해 게시됩니다. 이런 스레드들 상의 트랜잭션을 관리하기 위해서, `TransactionManager` 혹은 작업 단위(Unit of Work)에 묶인 트랜잭션을 생성하는 `UnitOfWorkFactory`를 설정할 수 있습니다.

> **참고**
>
> 스프링을 사용한다면, `QuartzEventSchedulerFactoryBean` 혹은 `SimpleEventSchedulerFactoryBean`을 사용하여 더욱 쉽게 설정을 할 수 있습니다. `QuartzEventSchedulerFactoryBean` 그리고 `SimpleEventSchedulerFactoryBean`에 스프링의 트랜잭션 인프라의 핵심 인터페이스인 PlatformTransactionManager를 직접 설정할 수 있습니다.

필요 자원 주입하기
-------------------

Saga들은 일반적으로 이벤트에 기반을 두고 상태를 유지하는 것 그 이상의 것들을 처리합니다. Saga들은 외부의 컴포넌트들과 연동할 수 있으며 이를 위해, 해당 컴포넌트들에 접근하는 데 필요한 자원들을 사용해야 합니다. 보통 이런 자원들은 Saga 상태의 일부는 아니며 영속화의 대상도 아닙니다. 하지만 Saga가 복원된 이후, Saga 인스턴스에 이벤트가 전달되기 전에 반드시 이런 자원들을 주입해야 합니다.

Saga 인스턴스에 자원을 주입하기 위해 `ResourceInjector`가 사용되며, `ResourceInjector`는 `SagaRepository`에 의해 사용됩니다. Axon은 애플리케이션 컨텍스트로부터 자원을 받아 에노테이션이 사용된 필드와 메서드에 해당 자원을 주입하는 `SpringResourceInjector`과 등록된 자원들을 `@Inject`에노테이션이 사용된 필드와 메서드에 주입하는 `SimpleResourceInjector`를 제공합니다.

> **팁**
>
> 주입해야 할 자원들은 Saga와 함께 영속화하면 안 되기 때문에, `transient` 키워드를 해당 자원 필드에 반드시 사용해야 합니다. 이렇게 하면, 해당 필드의 내용을 저장소(repository)에 저장하기 위한 직렬화 메커니즘을 방지할 수 있습니다. 저장소는 Saga가 역 직렬화된 이후, 자동으로 필요한 자원을 다시 주입합니다.

`SimpleResourceInjector`는 주입되어야 할 자원의 묶음(collection)을 미리 정의하는 것을 허용합니다. `SimpleResourceInjector`는 Saga의 메서드(setter)와 필드를 스캔하여 `@Inject` 에노테이션이 사용된 필드와 메서드를 찾게 됩니다.

설정 API를 사용할 경우, Axon은 `ConfigurationResourceInjector`를 기본으로 사용합니다. `ConfigurationResourceInjector`는 설정 객체(Configuration)내의 모든 사용 가능한 자원을 주입할 수 있습니다. `EventBus`, `EventStore`, `CommandBus` 그리고 `CommandGateway`와 같은 컴포넌트들을 기본적으로 `ConfigurationResourceInjector`을 통해 주입할 수 있으며, 주입이 필요한 컴포넌트들은 `configurer.registerComponent()` 메서드를 사용하여 주입할 수 있도록 등록할 수 있습니다.

`SpringResourceInejctor`는 스프링의 의존성 주입 메커니즘을 사용하여 aggregate에 필요한 자원을 주입합니다. 다시 말해, 필요한 자원을 필드에 직접 주입하거나 setter 메서드를 통한 주입을 사용할 수 있습니다. 주입이 필요한 메서드 혹은 필드는 필요한 자원을 스프링이 주입할 수 있도록 `@Autowired`와 같은 에노테이션을 사용해야 합니다.

Saga를 위한 기반구조
===================

이벤트를 적절한 Saga 인스턴스에 전송해야 합니다. 이를 위해, 몇 가지 기반구조(infrastructure)를 구성하는 클래스들이 필요합니다. 가장 중요한 컴포넌트는 `SagaManager`와 `SagaRepository`입니다.

사가 매니져
------------

이벤트를 처리하는 다른 컴포넌트들처럼, 이벤트 처리는 이벤트 프로세서(processor)에 의해 처리됩니다. 하지만 Saga들은 싱글 톤(singleton) 인스턴스가 아니고 개별적인 생애 주기를 가지기 때문에, 별도의 관리가 필요합니다.

Axon은 `AnnotatedSagaManager`를 통해 Saga 인스턴스의 생애 주기를 관리 하도록 지원합니다. `AnnotatedSagaManager`은 이벤트 처리를 위한 실제 인스턴스를 호출하기 위해, 이벤트 프로세서로 전달해 주어야 합니다. (* EventProcessor의 구현체인 `SubscribingEventProcessor` 혹은 `TrackingEventProcessor`의 생성자를 보면 `EventHandlerInvoker`를 인자로 받는 것을 볼 수 있습니다. 그리고 `AnnotatedSagaManager`는 `EventHandlerInvoker`의 하위 타입입니다.) `AnnotatedSagaManager`는 관리할 Saga의 타입을 사용해 초기화되며, Saga 유형을 저장하고 검색할 수 있는 Saga 레퍼지토리도 필요로 합니다. 하나의 `AnnotatedSagaManager`는 오로지 하나의 Saga 유형을 관리할 수 있습니다.

설정 API를 사용할 경우, Axon은 인지할 수 있는 대부분 컴포넌트에 대한 기본값을 사용합니다. 하지만 `SagaStore`의 구현체를 정의하여 사용하는 것을 권장합니다. `SagaStore`는 Saga 인스턴스를 '물리적인' 저장소에 저장하는 메커니즘입니다. 기본으로 사용되는 `AnnotatedSagaRepository`는 Saga 인스턴스를 저장하고 필요에 따라 저장된 Saga 인스턴스를 반환하기 위해 `SagaStore`를 사용합니다.

```java
Configurer configurer = DefaultConfigurer.defaultConfiguration();
configurer.registerModule(
        SagaConfiguration.subscribingSagaManager(MySagaType.class)
                         // Axon은 메모리 기반 SagaStore를 기본으로 사용하지만, 다른 SagaStore를 사용하는 것을 권장합니다.
                         .configureSagaStore(c -> new JpaSagaStore(...)));

// 다른 방법으로, 모든 Saga 유형에 대해 단일 SagaStore를 등록하는것도 가능합니다.
configurer.registerComponent(SagaStore.class, c -> new JpaSagaStore(...));
```

사가 레퍼지토리와 사가 스토어
------------------------------

`SagaManager`는 Saga들을 저장하고 반환하는 역할을 하는 `SagaRepository`를 필요로 합니다. `SagaRepository`는 Saga 인스턴스들의 식별자 그리고 Saga 인스턴스와 연관된 값들을 포함한 Saga 인스턴스를 반환합니다.

그런데 `SagaRepository`에 특별히 필요한 사항들이 있습니다. Saga들에 대한 동시성 처리는 매우 깨지기 쉬우므로, 레포지토리는 식별자의 동치로 구분되는 같은 Saga 유형의 인스턴스는 반드시 JVM 상에 단 하나만 존재하도록 보장해야 합니다. 즉, 같은 식별자를 가지는 Saga 인스턴스는 반드시 하나만 존재해야 합니다. 이때 동일 여부는 식별자의 `equals`메서드로 판단합니다.

Axon은 `AnnotatedSagaRepository` 구현체를 제공합니다. `AnnotatedSagaRepository`는 Saga 인스턴스를 조회하고 동일 시점에 단일 Saga 인스턴스에 접근하도록 합니다. `AnnotatedSagaRepository`는 Saga 인스턴스의 영속화를 `SagaStore`를 통해 수행합니다.

애플리케이션에서 사용하는 저장 엔진에 따라 구현체를 선택하여 사용합니다. Axon에서 제공하는 `JdbcSagaStore`, `InMemorySagaStore`, `JpaSagaStore` 그리고 `MongoSagaStore`를 사용할 수 있습니다.

경우에 따라, Saga 인스턴스를 캐싱하여 사용하는 것이 더 좋을 수 있습니다. 이 경우, 캐싱 기능을 사용하기 위해 다른 구현체를 기반으로 `CachingSagaStore`를 사용할 수 있습니다. `CachingSagaStore`는 생성자에서 `delegate`라는 SagaStore의 구현체를 인자로 받습니다. `delegate`로 Axon에서 제공하는 `JdbcSagaStore`, `InMemorySagaStore`, `JpaSagaStore` 그리고 `MongoSagaStore`과 같은 `SagaStore`의 구현체를 사용하면 됩니다. `CachingSagaStore`는 연속 기재 (write-through) 캐시입니다. 다시 말해, 데이터 안전성을 보장하기 위해 저장 작업이 즉시 백업 저장소로 즉시 전달됩니다.

### JPA 기반의 사가 스토어(JpaSagaStore)
`JpaSagaStore`는 Saga의 상태와 연결 값들을 저장하기 위해 JPA를 사용합니다. Saga 자체에 JPA 에노테이션들을 사용할 필요는 없습니다. Axon은 `Serializer`를 사용하여 Saga들을 직렬화 합니다. (이벤트 직렬화와 비슷하게, `JavaSerializer` 혹은 `XStreamSerializer`를 사용할 수 있습니다.)

`JpaSagaStore`는 `EntityManager` 인스턴스에 대한 접근을 제공하는 `EntityManagerProvider`와 함께 설정되어야 합니다. 이런 추상화를 통해 애플리케이션에서 관리되거나 컨테이너에서 관리되는 `EntityManager`들을 사용할 수 있습니다. 선택적으로, Saga 인스턴스를 직렬화 하기 위한 시리얼라이져(serializer)를 정의하여 사용할 수 있으며, Axon에서는 `XStreamSerializer`를 기본으로 사용합니다.

### JDBC 기반의 사가 스토어(JdbcSagaStore)
`JdbcSagaStore`는 Saga의 상태와 연결 값들을 저장하기 위해 일반 JDBC를 사용합니다. `JpaSagaStore`와 비슷하게, Saga 인스턴스는 자신들이 어떻게 저장이 되는지에 대해 신경 쓸 필요는 없습니다. Saga 인스턴스들은 시리얼라이져(serializer)를 통해 직렬화됩니다.  

`JdbcSagaStore`는 `DataSource` 혹은 `ConnectionProvider`과 함께 초기화 됩니다. 반드시 필요하진 않지만, `JdbcSagaStore`를 `ConnectionProvider`와 함께 초기화 할 때, `UnitOfWorkAwareConnectionProviderWrapper`로 `ConnectionProvider`구현체를 한번 감싸서 `UnitOfWorkAwareConnectionProviderWrapper`를 `ConnectionProvider`로 사용하는 것을 권장합니다. 이렇게 하면, 현재의 작업 단위(Unit of Work)에 연결된 데이터베이스 연결(connection)이 있는지 확인하여 작업 단위 내의 모든 작업이 단일 연결로 처리되는 것을 보장할 수 있습니다.

JPA와는 달리, JdbcSagaRepository(* 3. x에서 더는 사용되지 않는 것으로 보이고 SagaSqlSchema를 대신 사용하는 것으로 보입니다.)는 Saga에 대한 정보를 저장하고 조회하기 위해 일반적인 SQL 구문을 사용합니다. 따라서 몇몇 질의 작업은 데이터베이스에 의존적인 SQL 문법을 사용해야 합니다. 또한, 특정 데이터베이스 공급 업체가 제공하는 비표준 기능을 사용하고자 하는 경우가 있습니다. 이런 경우에 대처하기 위해, 직접 `SagaSqlSchema` 구현체를 작성하여 `JdbcSagaStore`에 제공할 수 있습니다. `SagaSqlSchema`는 기반 데이터베이스위에서 레포지토리가 필요로 하는 모든 작업을 정의한 인터페이스로, 개별 작업에 사용되는 SQL 구문을 재정의하여 사용할 수 있도록 합니다. `SagaSqlSchema`의 기본 구현체는 `GenericSagaSqlSchema`이며, 그 외에 `PostgressSagaSqlSchema`, `Oracle11SagaSqlSchema` 그리고 `HsqlSagaSchema`들이 있습니다.

### MongoDB 기반의 사가 스토어(MongoSagaStore)
이름에서 알 수 있듯이, `MongoSagaStore`는 MongoDB 데이터베이스를 대상으로 Saga의 상태와 연결 값들을 저장합니다. `MongoSagaStore`는 모든 사가(Saga)를 MongoDB 데이터베이스의 하나의 컬렉션(collection)에 저장하며 사가 인스턴스 당 하나의 도큐먼트(document)가 생성됩니다.

`MongoSagaStore`는 또한, 고유한 식별자를 가지는 단일 Saga 인스턴스가 JVM 상에 존재하도록 보장합니다. 따라서 동시성 문제로 인해 상태 값을 잃어버리지 않도록 보장해 줍니다.

`MongoSagaStore`는 `MongoTemplate`과 선택적으로 `Serializer`를 사용하여 초기화됩니다. `MongoTemplate`은 Saga를 저장하는 컬렉션에 대한 참조를 제공합니다. Axon은 `MongoClient`와 데이터베이스 이름, 그리고 Saga들이 저장되는 컬렉션 이름이 있어야 하는 `DefaultMongoTemplate`을 제공합니다. 데이터베이스 이름과 컬렉션 이름은 생략할 경우, 데이터베이스 이름으로 "axonframework"가 사용되며 컬렉션 이름으로 "sagas"를 사용합니다.

캐싱
-------

Saga 스토리지(storage)로 데이터베이스를 사용할 경우, Saga 인스턴스를 저장하고 조회하는 것은 상대적으로 비용이 많이 드는 작업입니다. 특히 짧은 시간 동안 같은 Saga 인스턴스를 조회하는 상황에선, 캐시를 사용하는 것이 애플리케이션의 성능 향상에 많은 도움이 될 수 있습니다.

Axon은 `CachingSagaStore`의 구현체를 제공하며, `CachingSagaStore`는 실제 Saga 저장소 역할을 하는 다른 `SagaStore` 구현체의 래퍼(wrapper)입니다. Saga 혹은 연결 값을 로딩할 때, `CachingSagaStore`는 먼저 자신 내부의 캐시를 검색하고 캐시가 해당 객체를 가지고 있지 않으면 delegate로 전달받은 사가 스토어를 통해 필요한 객체를 반환받습니다. Saga 인스턴스를 저장할 때, delegate로 전달받은 사가 스토어가 일관된 Saga의 상태를 가질 수 있도록 해당 사가 스토어에 저장 작업을 위임합니다.

캐시를 설정하기 위해선, 간단히 `SagaStore` 구현체를 `CachingSagaStore`에 전달해주면 됩니다. `CachingSagaStore`의 생성자는 세 개의 매개변수를 받습니다. 첫 번째 매개변수는 사가 스토어이며, 두번째 매개변수는 연결 값에 대한 캐시 그리고 마지막 매개변수는 Saga에 대한 캐시입니다. 애플리케이션에 따라서 두 번째와 세 번째 인자들은 같은 캐시 객체 혹은 다른 캐시 객체를 참조할 수도 있습니다.
