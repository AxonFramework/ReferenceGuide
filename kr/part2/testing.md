테스트
=======
CQRS의 가장 큰 이점 중 하나이면서 특히 이벤트 소싱의 이점 중 하나는 이벤트와 명령으로 테스트를 순수하게 표현할 수 있다는 점입니다. 기능적 컴포넌트, 이벤트 그리고 명령들은 도메인 전문가 혹은 비즈니스 소유자(owner)가 알기 쉽게 되어 있습니다. 테스트를 이벤트와 명령으로 명확한 기능적 의미를 표현하도록 하는 것뿐만 아니라, 구현 선택에 거의 의존하지 않도록 할 수 있습니다.

이번 장에서 설명할 기능들은 `axon-test` 모듈이 있어야 합니다. `axon-test` 모듈은 메이븐 의존성 설정을 아래와 같이 설정하여 사용할 수 있습니다. 혹은 전체 패키지를 내려받아 사용할 수 있습니다.

The fixtures described in this chapter work with any testing framework, such as JUnit and TestNG.
이번 장에서 설명할 테스트 픽스쳐들(fixture)은 JUnit과 TestNG와 같은 테스트 프레임워크와도 연동할 수 있습니다.

명령 컴포넌트 테스트
-------------------------

명령 처리 컴포넌트는 가장 큰 복잡성을 가지는 CQRS 기반 아키텍쳐의 전형적인 컴포넌트입니다. 다른 컴포넌트보다 더 복잡하다는 것은, 명령 컴포넌트를 테스트하기 위한 추가적인 필요 사항들이 있다는 것을 말합니다.

 보다 더 복잡하긴 하지만, 명령 처리자 컴포넌트의 API는 상당히 쉽습니다. 명령 처리자 컴포넌트는 처리할 명령을 받고, 이벤트를 발생시키는 기능을 수행합니다. 몇몇 경우에, 명령 실행의 일부로써 질의를 수행할 수도 있습니다. 그 외에는, 명령들과 이벤트들이 API의 유일한 부분입니다. 따라서 명령과 이벤트들로 완전한 테스트할 시나리오를 작성할 수 있습니다. 일반적으로 테스트는 다음과 같은 형태로 작성할 수 있습니다.

-   given: 주어진 이미 발생한 특정 이벤트 들이 있고
-   when: 특정 명령을 실행할 때,
-   expect: 실행된 명령을 통해 특정 이벤트들이 게시되거나 저장된다.

Axon Framework은 위에서 설명한 형태의 테스트를 작성할 수 있도록 테스트 픽스쳐를 제공합니다. `AggregateTestFixture`는 특정 기반 구조의 설정, 필요한 명령 처리자와 레퍼지토리의 조합 그리고 테스트 시나리오를 given-when-then 형태의 이벤트와 명령들로 작성할 수 있도록 해줍니다.

다음의 예제 코드를 통해 JUnit4 기반으로 테스트 픽스쳐를 사용하여 given-when-then 형태의 테스트 코드를 작성하는 방법을 볼 수 있습니다.

``` java
public class MyCommandComponentTest {
    private FixtureConfiguration<MyAggregate> fixture;

    @Before
    public void setUp() {
        fixture = new AggregateTestFixture<>(MyAggregate.class);
    }

    @Test
    public void testFirstFixture() {
        fixture.given(new MyEvent(1))
               .when(new TestCommand())
               .expectSuccessfulHandlerExecution()
               .expectEvents(new MyEvent(2));
        /*
        위 4줄의 코드가 실제 테스트 시나리오와 시나리오에 대한 기댓값을 정의한 것입니다. 첫 번째 줄의 코드는 이미 발생한 이벤트를 정의한 것이며, 이 이벤트들은 테스트상에서 aggregate의 상태를 정의합니다. 좀 더 구체적으로 말하면, aggregate가 로딩되었을 때, 이벤트 저장소가 반환하는 이벤트들을 말합니다. 두 번째 줄의 코드는 시스템상에서 처리해야 할 명령을 정의합니다.

        마지막 두 줄의 코드는 기대 행위와 기댓값을 정의한 것으로, 이 중 첫 번째 코드는 명령 처리자가 성공적으로 명령을 처리했는지를 확인하는 코드이며, 마지막 줄의 코드는 명령의 실행 결과로 발생한 이벤트를 기대한 이벤트와 맞는지를 검증하는 코드입니다.
        */
    }
}
```

given-when-then 테스트 픽스쳐는 설정, 실행 그리고 검증의 세 단계로 정의 됩니다. 개별 단계들은 각기 다른 인터페이스(interface)로 표현되며, 각각의 인터페이스는 다음과 같습니다.

- 설정: FixtureConfiguration
- 실행: TestExecutor
- 검증: ResultValidator

The static `newGivenWhenThenFixture()` method on the `Fixtures` class provides a reference to the first of these, which in turn may provide the validator, and so forth.
`Fixtures`클래스의 정적 메서드인 `newGivenWhenThenFixture()`는 `FixtureConfiguration` 인스턴스에 대한 참조를 제공하며, `FixtureConfiguration` 인스턴스를 통해 검증자와 다른 객체들을 받을 수도 있습니다. (Axon Framework의 버전 2.x에서만 `Fixtures` 클래스를 제공합니다.)

> **참고**
>
> 위의 세 단계간에 이동을 최적화 하기 위해선, 위의 예제 코드와 같이 이 메서드들이 반환하는 인터페이스를 사용하는 것이 제일 좋습니다.

설정 단계(예, 첫 번째 "given" 이전) 동안, 테스트 실행에 필요한 빌딩 블록들을 제공할 수 있습니다. 테스트를 위해 특별히 사용될 이벤트 버스, 커맨드 버스 그리고 이벤트 저장소를 픽스쳐의 일부분으로 제공합니다. `FixtureConfiguration`를 통해 이벤트 버스, 커맨드 버스 그리고 이벤트 저장소에 대한 참조를 얻을 수 있습니다. (이벤트 버스, 커맨드 버스 그리고 이벤트 저장소에 대한 getter 메서드를 제공합니다) Aggregate에 직접 등록되지 않은 명령 처리자들은 `FixtureConfiguration`의 `registerAnnotatedCommandHandler` 메서드를 통해 명시적으로 설정이 되어야 합니다. 에노테이션 기반의 명령 처리자 외에도 테스트 관련 인프라 설정 방법을 정의하는 다양한 구성 요소와 설정을 구성할 수 있습니다.

픽스쳐가 설정이 되면, 이미 발생한("given") 이벤트를 정의할 수 있습니다. 테스트 픽스쳐는 이런 이벤트들을 `DomainEventMessage`로 감싸서 처리합니다. 주어진("given") 이벤트들이 Message의 구현체라면, 메시지의 페이로드와 메타 데이터는 `DomainEventMessage`에 포함됩니다. 그렇지 않은 경우는, 주어진 이벤트는 페이로드로 사용이 됩니다. 0에서부터 시작하는 일련의 `DomainEventMessage`의 일련번호는 차례대로 증가합니다.

혹은 주어진("given") 명령어를 제공할 수도 있습니다. 이 경우, 테스트 대상이 되는 실제 명령이 실행될 때, 주어진 명령어에 의해 발생하는 이벤트들이 Aggregate의 이벤트 소스로 사용됩니다. `givenCommands(...)` 메서드를 사용하여 명령 객체들을 설정합니다.

실행 단계에서는 명령 처리자가 처리할 명령을 제공해야 합니다. 해당 명령 처리를 위해 호출되는 처리자(Aggregate 상의 처리자 혹은 외부 처리자)의 작동은 감시되고 검증 단계에 등록된 기대 사항들과 비교됩니다.

> **참고**
>
> 테스트의 실행 단계 동안, Axon을 통해 테스트 하에서의 Aggregate의 비정상적인 상태 변경을 발견할 수 있습니다. Aggregate가 "주어진(given)" 이벤트들과 저장된 이벤트들로부터 복원(sourced)된 상태에서 Aggregate의 상태를 변경시킬 명령을 실행한 후에 Aggregate의 상태를 비교하여 비정상적인 상태 변경을 발견합니다. > 만약 상태가 기대한 상태와 같지 않다면, Aggregate의 이벤트 처리자 메서드가 아닌 다른 곳에서 Aggregate의 상태가 변경된 것입니다. 정적이고 transient 한 필드들은 비교 대상에서 제외됩니다.
>
> 비정상적 상태 변경의 감지 기능은 `setReportIllegalStateChange(boolean reportIllegalStateChange)` 메서드를 통해 키거나 끌 수 있습니다.

마지막 단계인 검증 단계에선, 반환 값들과 이벤트들을 통해 명령 처리 컴포넌트의 작업을 확인할 수 있습니다.

테스트 픽스쳐를 통해 명령 처리자의 반환 값을 검증할 수 있습니다. 명시적으로 기대 되는 반환 값을 정의하거나 해당 메서드가 성공적으로 결과를 반환했는지를 확인할 수 있습니다. 또한, 명령 처리자가 던질 수 있는 예외를 기댓값으로 설정하여 확인할 수 있습니다.  

게시된 이벤트를 검증할 수 있으며, 이벤트를 확인할 방법은 두 가지 방법이 있습니다.  

첫 번째 방법은 실제 이벤트와 문자 그대로 비교되어야 하는 이벤트 인스턴스를 전달하는 것입니다. 기대 이벤트의 모든 속성을 실제 발생한 이벤트의 동일 속성들과 비교(`equals()` 메서드 사용)합니다. 만약 한 속성이라도 같지 않다면, 해당 테스트는 실패할 것이고 상세한 결과를 담은 에러 결과를 생성합니다.  

다른 검증 방법은 Hamcrest 라이브러리의 Matcher들을 사용하여 기댓값들을 표현하는 것입니다. `Matcher`는 `matches(Object)`와 `describe(Description)` 메서드를 포함하고 있는 인터페이스입니다. `matches(Object)` 메서드는 주어진 객체가 비교 대상과 일치 하는지를 검증하여 참혹은 거짓을 반환합니다. `describe(Description)` 메서드는 기대 사항을 표현할 수 있도록 해줍니다. 예를 들어, "GreaterThanTwoMatcher"에 "다른 두 이벤트보다 큰 값을 가진 이벤트"라는 설명을 추가할 수 있습니다. 설명(Description)은 테스트가 실패했을 때 실패한 원인에 대해 에러 메시지를 표현하는데 사용할 수 있습니다.  

이벤트들에 대한 `matcher`들을 생성하는 것은 장황하고 에러를 발생시킬 수 있습니다. 간단히 코드를 작성하기 위해, Axon에서 제공하는 이벤트 검증을 위한 `matcher`들을 사용하여 발생한 이벤트들을 검증할 수 있습니다.  

아래의 예를 통해 이벤트 목록에 대한 `matcher`들과 목적을 확인할 수 있습니다.

-   **List with all of**: `Matchers.listWithAllOf(event matchers...)`

    해당 matcher는 주어진 이벤트 matcher가 실제 발생한 이벤트 중 하나 이상의 이벤트와 일치하면 성공 결과를 반환합니다. 여러 개의 matchers가 같은 이벤트에 대해 참을 반환하는지 혹은 이벤트 중 어떤 이벤트가 특정 matchers를 통해 참의 결과가 나오던지 상관없이 주어진 matcher들이 발생한 이벤트 중 하나 이상의 이벤트에 대해 일치한다는 결과를 반환하는지를 검증합니다.

-   **List with any of**: `Matchers.listWithAnyOf(event matchers...)`

    이 matcher는 주어진 이벤트 matcher 중 하나 혹은 그 이상이 실제 발생한 이벤트 중 하나 혹은 그 이상의 이벤트와 일치하면 성공 결과를 반환합니다. 몇 개의 matcher들은 다수의 이벤트에 일치하는 반면, 일부 matcher들은 전혀 일치하지 않을 수 있습니다.

-   **Sequence of Events**: `Matchers.sequenceOf(event matchers...)`

    실제 발생한 이벤트와 발생 순서가 주어진 이벤트 matcher와 같은 순서 인지를 검증하기 위해 이 matcher를 사용합니다. 개별 matcher는 이전 matcher가 일치 여부를 검증한 이벤트 다음의 이벤트를 검증하여 검증 결과를 반환합니다. 즉, 주어진 이벤트 matcher가 모두 검증에 성공하면 결과는 성공이 됩니다. 따라서 matcher와 일치하지 않는 이벤트가 나타날 가능성이 있습니다.

    만약 모든 이벤트를 검증한 후에도 matcher가 남아 있다면, 남아 있는 matcher들은 `null`과 일치 여부를 검증하게 됩니다. 따라서 남아 있는 matcher들이 `null`을 허용하는지 아닌지에 따라 성공 여부가 결정됩니다.

-   **Exact sequence of Events**: `Matchers.exactSequenceOf(event matchers...)`

    "Sequence of Events"의 변형된 matcher로 "Sequence of Events"에서는 matcher와 일치하지 않는 이벤트가 발생할 수 있었으나, 이 matcher는 일치하지 않는 이벤트를 허용하지 않습니다. 따라서 정확한 순서로 이벤트가 발생했는지를 검증할 수 있습니다.

편의상, 자주 사용되는 이벤트 matcher들만 살펴보았습니다. 다음은 단일 이벤트에 대한 matcher 메서드를 살펴보겠습니다.

-   **Equal Event**: `Matchers.equalTo(instance...)`

    해당 matcher는 주어진 객체와 발생한 이벤트가 의미상 같은지를 검증합니다. 두 객체 간 모든 속성을 비교하며 `null` 비교를 허용하여 문제없이 `null` 비교를 할 수 있습니다. 다시 말하면 `equals` 메서드를 구현하지 않은 이벤트들도 비교할 수 있습니다. 주어진 매개변수 객체의 속성들에 저장되어있는 객체들은 `equals` 메서드를 통해 비교하므로, 속성 객체는 `equals`를 정확히 구현해야 합니다.

-   **No More Events**: `Matchers.andNoMore()` or `Matchers.nothing()`

    위 matcher는 `null`값을 비교하는 matcher로 "Exact sequence of Events"의 가장 마지막에 사용되어 일치하지 않는 이벤트들이 남아 있지 않도록 합니다.

위의 matcher들은 이벤트의 목록에 대해 검증을 하는데, 때로는 메세지의 페이로드 부분만 검증해야 할 수 있습니다. 이런 경우 사용할 수 있는 matcher들은 아래와 같습니다.

-   **Payload Matching**: `Matchers.messageWithPayload(payload matcher)`

    실제 메시지의 페이로드와 주어진 페이로드 matcher가 일치하는지를 검증합니다.

-   **Payloads Matching**: `Matchers.payloadsMatching(list matcher)`

    실제 메시지들의 페이로드들이 주어진 페이로드들과 일치하는지를 검증합니다. 인자로 주어진 matcher는 메시지들의 페이로드를 포함하는 목록과 반드시 일치해야 합니다. "Payload Matching" matcher는 일반적으로 페이로드 matcher들의 반복을 방지하기 위한 페이로드 matcher들을 감싸는 외부 matcher로 사용이 됩니다.

아래의 예제 코드를 통해 위에서 살펴본 matcher들의 사용법에 대해 간단히 살펴보겠습니다. 아래의 예제를 통해 두개의 이벤트가 게시될 것으로 예상하며, 첫 번째 이벤트는 반드시 "ThirdEvent"이어야 하고 두 번째 이벤트는 "aFourthEventWithSomethingSpecialThings"이어야 합니다. 세 번째로 발생하는 이벤트는 없고 이를 검증하기 위해 "andNoMore" matcher를 사용하였습니다.

``` java
fixture.given(new FirstEvent(), new SecondEvent())
       .when(new DoSomethingCommand("aggregateId"))
       .expectEventsMatching(exactSequenceOf(
           // 페이로드 부분만 검증합니다.
           messageWithPayload(equalTo(new ThirdEvent())),
           // 메세지에 대한 검증을 합니다.
           aFourthEventWithSomeSpecialThings(),
           // 더 이상의 이벤트가 없음을 검증합니다.
           andNoMore()
       ));

        // 혹은 페이로드 부분만을 검증할 수 있습니다.
       .expectEventsMatching(payloadsMatching(
               exactSequenceOf(
                   // 페이로드만 검증하기때문에, equalTo를 바로 사용합니다.
                   equalTo(new ThirdEvent()),
                   // 아래의 코드 또한 페이로드 부분만 검증합니다.
                   aFourthEventWithSomeSpecialThings(),
                   // 더 이상의 이벤트가 없음을 검증합니다.
                   andNoMore()
               )
       ));
```

에노테이션 기반의 Saga 테스트 하기
=======================

명령 처리 컴포넌트와 비슷하게, Saga들도 이벤트에 응답하는 명확하게 정의된 인터페이스들을 가지고 있습니다. 반면에, Saga는 종종 시간의 개념을 가지고 있으며 이벤트 처리 과정의 일부로 다른 컴포넌트들과 연동할 수 있습니다. Axon Framework의 테스트 지원 모듈을 통해 Saga에 대한 테스트를 작성할 수 있습니다.

각각의 테스트 픽스쳐는 이전 장에서 설명한 명령 처리 컴포넌트와 유사한 세단계를 포함하고 있습니다.

-   given: 주어진 이미 발생한 특정 이벤트 들이 있고,(특정 aggregate로 부터 발생한 이벤트)
-   when: 이벤트가 도착했거나 시간이 경과하였을때,
-   expect: 특정 기대 행위 혹은 상태가 되어야 한다.

"given"과 "when" 단계들 모두 연동의 일부분으로 이벤트를 받습니다. "given" 단계 동안, 가능한 경우 생성된 명령들과 같은 모든 부작용(side-effect)들은 무시됩니다. 반면 "when" 단계 동안, Saga로부터 발생한 이벤트들과 명령들은 기록되고 확인할 수 있습니다.

 다음의 코드 예제를 통해 송장에 대한 결제가 30일 이내에 이루어지지 않았을 때 통지를 보내는 saga를 테스트하기 위한 테스트 픽스쳐의 사용법을 확인할 수 있습니다.

```java
FixtureConfiguration<InvoicingSaga> fixture = new SagaTestFixture<>(InvoicingSaga.class);
fixture.givenAggregate(invoiceId).published(new InvoiceCreatedEvent())
       .whenTimeElapses(Duration.ofDays(31))
       .expectDispatchedCommandsMatching(Matchers.listWithAllOf(aMarkAsOverdueCommand()));
       // 혹은 명령 메세지의 페이로드만 일치하는지 검증할 수 있습니다.
       .expectDispatchedCommandsMatching(Matchers.payloadsMatching(Matchers.listWithAllOf(aMarkAsOverdueCommand())));
```

Saga들은 명령 처리 결과을 받았을 때 실행되는 콜백(callback)을 사용하여 명령을 전송할 수 있습니다. 테스트를 진행할 때 실제로 완료되는 명령 처리는 없으므로, 명령을 보내는 행위는 `CallbackBehavior` 객체를 사용하여 정의합니다. `CallbackBehavior` 객체를 `setCallbackBehavior()` 메서드를 통해 픽스쳐에 등록하고 명령이 전달될 때 콜백을 호출해야 하는지 아닌지와 그 방법에 대해 정의합니다. `CallbackBehavior`는 커맨드 버스가 콜백 메서드와 함께 전송된 명령을 받았을 때 호출이 되며 `CallbackBehavior`의 `handle` 메서드의 반환 값은 해당 콜백의 `onSuccess` 메서드를 호출할 때 사용이 됩니다.

`CommandBus`를 직접 사용하지 않고, `CommandGateWay`를 사용할 수 있습니다. 사용법은 아래를 참조하면 됩니다.

종종, Saga들은 자원들을 필요로 합니다. 해당 자원들은 Saga 상태의 일부분은 아니지만, Saga가 생성되거나 로딩된 후에 주입됩니다. 테스트 픽스쳐를 통해 Saga가 필요로 하는 자원을 등록할 수 있습니다. 자원을 등록하기 위해서는, 등록할 자원을 인자로 주어 `fixture.registerResource(Object)` 메서드를 호출하면 테스트 픽스쳐는 적당한 setter 메서드 혹은 `@Inject` 에노테이션 필드를 찾아 호출하여 해당 자원을 주입합니다.

> **팁**
>
> Mockito 혹은 Easymock과 같은 Mock 객체들을 Saga의 자원으로 주입하여 사용하면 외부 자원과 Saga간의 연동이 제대로 이루어지는지를 확인할 수 있습니다.

Command GateWay를 통해 Saga에서 명령을 더욱 더 쉽게 전송할 수 있습니다. 재정의한 커맨드 게이트웨이를 사용하면, 테스트상에서 사용할 mock 혹은 stub 객체를 쉽게 만들어 사용할 수 있습니다. 하지만 mock 혹은 stub을 제공할 때, 실제 명령이 전달되지 않아 테스트 픽스쳐를 통해 전송된 명령을 확인할 수 없게 됩니다.

그러므로 테스트 픽스쳐가 제공하는 `registerCommandGateway(Class)`와 `registerCommandGateway(Class, stubImplementation)` 메서드를 통해 커맨드 게이트웨이 객체를 등록하고 추가로 커맨드 게이트웨이의 행위를 정의하는 mock(혹은 stub)을 허용하는 방법을 사용해야 합니다. 두 메서드 모두 사용할 게이트웨이를 나타내는 주어진 클래스의 인스턴스를 반환합니다. 이 반환된 인스턴스를 자원으로 등록할 수 있고 등록된 인스턴스는 주입하여 사용할 수 있습니다.

게이트웨이를 등록하기 위해 `registerCommandGateway(Class)`메서드를 사용하면, 픽스쳐가 관리하는 커맨드 버스에 명령을 전달합니다. 게이트웨이의 행위는 대부분 픽스쳐 상에 정의된 `CallbackBehavior`에 의해 정의됩니다. 만약 명시적인 `CallbackBehavior`가 없다면, 콜백들은 호출되지 않으므로 게이트웨이에 대한 어떤 반환 값도 제공되지 않습니다.

게이트웨이를 등록하기 위해 `registerCommandGateway(Class, stubImplementation)`메서드를 사용할 때, 두 번째 매개변수는 게이트웨이의 동작을 정의하는 데 사용됩니다.

테스트 픽스쳐는 가능하면 시스템 시간이 오래 걸리는 것을 제거하려고 합니다. 다시 말해, `whenTimeElapses() ` 메서드를 사용하여 명시적으로 선언하지 않는 한, 테스트 실행 동안 시각이 지나지 않는 것처럼 보입니다. 모든 이벤트는 테스트 픽스쳐가 생성된 시점의 타임스탬프를 가지고 있습니다.

테스트 동안 시각을 멈춰 놓으면, 예정에 맞춰 언제 이벤트가 게시될지 더 쉽게 예측할 수 있습니다. 만약 30초 이내에 게시될 예정인 이벤트를 검증하는 테스트가 있다면, 실제 예정과 테스트 실행 간에 걸리는 시간에 상관없이 30초가 걸릴 것입니다.

> **참고**
>
> 픽스쳐는 이벤트를 계획하고 시간을 앞당기는 등의 시간에 기반을 둔 활동을 하기 위해 `StubScheduler`를 사용합니다. 픽스쳐는 Saga 인스턴스에 보내진 모든 이벤트의 타임 스탬프를 `StubScheduler`의 시간으로 설정할 것입니다. 즉, 시간은 픽스쳐가 시작하게 되면 '멈춰진' 상태가 되고, `whenTimeAdvanceTo`와 `whenTimeElapses` 메서드를 사용하여 결정적으로 시간을 앞당겨집니다.

특정 시점에 발생할 이벤트를 테스트해야 한다면, 테스트 픽스쳐에 독립적인 `StubEventScheduler`를 사용할 수 있습니다. `EventScheduler`의 구현체인 `StubEventScheduler`를 통해 어떤 이벤트가 어느 시간으로 예정이 되어 있는지를 검증할 수 있고 시간의 경과를 조작할 수 있는 옵션들을 사용할 수 있습니다. 특정 `기간(Duration)`만큼 시간을 앞당겨 특정 날짜와 시간으로 조정하거나 다음에 예정된 이벤트로 시간을 앞당길 수 있습니다. 이 모든 작업은 진행된 일정내의 예정된 이벤트들을 반환합니다.
