명령 모델 알아보기
===============

CQRS 기반의 애플리케이션에서, 도메인 모델(에릭 에반스와 마틴 파울러에 의해 정의된)은 통해 상태 변경의 검증과 실행에 관련된 복잡성을 극복하기 위한 강력한 메커니즘을 제공합니다. 비록 일반적인 도메인 모델은 많은 수의 빌딩 블록들을 가지고 있지만, 빌딩 블록 중 하나인 Aggregate는 굉장히 중요한 역할을 담당합니다.

애플리케이션 내에서 상태의 변경은 명령으로부터 시작됩니다. 모든 명령은 표현 의도와 그 의도에 따라 행동을 취하는 데 필요한 정보의 조합입니다. 명령 모델은 들어오는 명령을 처리하기 위해 사용되며, 수신한 명령을 검증하고 반환 값을 정의합니다. 이 모델을 사용하면, 명령 처리자는 특정 유형의 명령을 처리해야 할 책임을 지고 명령에 포함된 정보들을 가지고 조처를 합니다.

Aggregate
---------
Aggregate는 항상 일정한 상태로 유지되는 하나의 엔티티이거나 엔티티들의 그룹입니다. 최상위 Aggregate(Aggregate Root)는 aggregate 트리 상의 최상위 객체이며, 일정한 상태를 유지하는 책임을 지고 있습니다. 이런 점들이 Aggregate를 CQRS 기반의 애플리케이션에서 명령 모델을 구현하기 위한 가장 중요한 빌딩 블록으로 만듭니다.

> **참고**
>
> "**Aggregate**"라는 용어는 도메인 주도 개발에서 다음과 같이 에릭 에반스가 정의한 aggregate을 말합니다.
> 데이터 변경에 대해 하나의 단위 묶음으로 취급되는 도메인 객체들의 묶음이라고 할 수 있습니다. 예를 들어 주문과 주문 상품들은 각각 독립된 객체들이지만 이들을 하나의 주문으로 처리할 수 있습니다. 즉, 주문과 상품들을 한데 묶은 단일 객체인 Aggregate로 취급할 수 있습니다. 그리고 Aggregate는 Aggregate Root라는 컴포넌트를 가집니다. Aggregate Root를 통해 외부 객체가 Aggregate에 대해 참조하도록 하여, Aggregate의 정합성과 무결성을 보장해야 합니다.

예를 들어, 하나의 "연락처" aggregate는 "연락처"와 "주소"라는 두 개의 엔티티를 포함할 수 있습니다. 전체 aggregate를 일관된 상태로 유지하기 위해선, 하나의 연락처로써 주소를 추가하는 행위는 연락처 엔티티를 통해서 이루어져야 합니다. 이 경우, 연락처 엔티티를 최상위 aggregate로 지정합니다.

Axon에선, aggregate는 Aggregate 식별자에 의해 식별이 됩니다. 식별자의 유형에 제한은 없지만, 어떤 객체든 식별자로 사용할 수 있습니다. 하지만 좋은 식별자를 구현하기 위해 아래와 같은 지침들을 살펴보겠습니다.

-   다른 인스턴스와의 비교에서 정확한 동치 성을 보장하기 위해 `equals`와 `hasCode`메서드를 구현해야 합니다.

-   일관된 결과를 제공하는 `toString()` 메서드를 구현해야 합니다. 동치 성 비교에 사용되는 항목들의 내용은 항상 일관되어야 하며, `toString` 메서드를 통해서도 일관된 내용이 표시되어야 합니다.

-   `Serializable` 인터페이스를 구현하는 것을 권장합니다.

테스트 픽스쳐([테스트](testing.md)를 살펴 보세요)는 이런 조건들을 검증하고 Aggregate가 호환되지 않는 식별자를 사용할 경우 테스트는 실패하게 됩니다. `String`, `UUID` 그리고 숫자 형태의 식별자를 사용한다면 아무런 문제가 되지 않습니다. 원시 타입의 변수는 늦은 초기화(lazy initialization)를 허용하지 않기 때문에, 원시(primitive) 타입의 값을 식별자로는 사용이 불가합니다. 일부 환경에서, Axon은 원시 타입 변수의 기본값을 식별자의 값으로 잘못 가정할 수 있습니다.

> **참고**
>
> 순차적인 값을 사용하는 것보다 무작위로 생성된 값을 식별자로 사용하는 것이 모범 사례로 생각됩니다. 순차적인 값을 식별자로 사용하게 되면 애플리케이션의 규모 가변성을 크게 해치게 됩니다. 왜냐면 애플리케이션 인스턴스 간 마지막으로 사용된 순차적인 값을 최신으로 유지를 해야 하기 때문입니다. 8.2 × 10<sup>11</sup>  UUID를 사용한다면, UUID의 값이 중복될 확률은 매우 낮습니다. (중복될 확률: 10<sup>−15</sup>)
>
> 기능적 식별자는 변경될 가능성이 있어, 적절하게 애플리케이션에 적용하기가 쉽지 않습니다. 따라서 기능적 식별자를 사용할 때는 조심해서 사용해야 합니다.


Aggregate 구현체들
-------------------------
Aggregate는 Aggregate 루트(최상위 Aggregate)인 단일 엔티티를 통해서만 접근할 수 있습니다. 보통 해당 엔티티의 이름은 Aggregate의 이름과 완전히 같습니다. 예를 들어, 주문 Aggregate는 주문 엔티티와 주문 상품 항목 엔티티로 구성될 수 있으며, 주문과 주문 상품 함께 Aggregate를 구성하게 됩니다.

Aggregate은 일반 객체이며, 상태와 상태를 변경할 수 있는 메서드를 포함합니다. 비록 CQRS 원칙을 완벽히 따르는 것은 아닐지라도, setter(accesor) 메서드를 통해 aggregate의 상태를 변경하는 것은 가능합니다.

Aggregate 루트 객체는 Aggregate 식별자를 포함하는 항목을 반드시 선언해야 합니다. 이 식별자는 최소한 첫번째 이번트가 발생되었을때 초기화 되어야 합니다. `@AggregateIdentifier` 에노테이션을 반드시 해당 식별자에 사용해야 합니다. JPA와 JPA 에노테이션들을 aggregate에 사용한다면, `@Id`에노테이션을 사용하여 식별자를 나타낼 수 있습니다.

Aggregate은 게시(publication)할 이벤트를 등록하기 위해 `AggregateLifeCycle.apply()` 메서드를 사용할 수 도 있습니다. EventMessage로 메시지들을 감싸는  `EventBus`와 다르게, `AggregateLifeCycle.apply()` 메서드는 페이로드(payload) 객체를 바로 사용할 수 있습니다. 다음의 예제코드는 aggregate에 JPA, JPA 에노테이션의 사용과 `AggregateLifeCycle.apply()`메서드의 사용 예 입니다.

``` java
import static org.axonframework.commandhandling.model.AggregateLifecycle.apply;

@Entity // JPA 엔티티
public class MyAggregate {

    @Id // JPA @Id 에노테이션을 사용할 경우, @AggregateIdentifier 에노테이션은 필요하지 않습니다.
    private String id;

    // 상태를 나타내는 멤버 필드...

    @CommandHandler
    public MyAggregate(CreateMyAggregateCommand command) {
        // ... 상태 변경
        apply(new MyAggregateCreatedEvent(...));
    }

    // JPA가 필요롤 하는 생성자
    protected MyAggregate() {
    }
}
```

Aggregate내의 엔티티는 `@EventHandler` 에노테이션이 사용된 메서드를 통해 Aggregate가 게시하는 이벤트들을 수신할 수 있습니다. 외부 핸들러로 전파되기 전에, 해당 메서드들은 이벤트가 게시되었을 때 호출이 됩니다.

이벤트로부터 aggregate 복구
------------------------
Aggregate의 현재 상태를 저장하는 것 외에, 이미 게시된 지난 이벤트들을 가지고 Aggregate의 상태를 복원하는 것 또한, 가능합니다. 이것을 가능하게 하기 위해선, 모든 상태변경은 이벤트로 표현이 되어야 합니다.

가장 중요한 부분은, 다른 aggregate들과 마찬가지로 식별자를 가져야 하고 `apply`메서드를 통해 이벤트를 게시하는 점에서 이벤트로 기반 Aggregate들은 '일반' aggregate들과 유사합니다. 그렇지만 이벤트 기반 Aggregate들의 상태 변경(예를 들어 필드 값의 변경)과 식별 값의 설정은 *전적으로* `@EventSourcingHandler` 에노테이션이 사용된 메서드를 통해서만 적용이 되어야 합니다.

Aggregate가 게시하는 가장 첫 번째 이벤트는 보통 생성 이벤트이며, Aggregate가 게시한 가장 첫 번째 이벤트를 처리할 `@EventSourcingHandler` 메서드를 통해 반드시 Aggregate의 식별 값을 설정해야 합니다.

이벤트 기반 Aggregate의 Aggregate 루트는 인자를 가지지 않는 기본 생성자를 반드시 포함해야 합니다. Axon Framework는 지난 이벤트를 사용하여 Aggregate를 초기화하기 전에, 기본 생성자를 사용하여 기본적인 Aggregate 인스턴스를 생성합니다. Aggregate에 기본 생성자가 없는 경우, Aggregate 로딩 시 예외가 발생하게 됩니다.

``` java
public class MyAggregateRoot {

    @AggregateIdentifier
    private String aggregateIdentifier;

    // 상태를 나타내는 멤버 필드...

    @CommandHandler
    public MyAggregateRoot(CreateMyAggregate cmd) {
        apply(new MyAggregateCreatedEvent(cmd.getId()));
    }

    // 복원을 위해 필요한 기본 생성자
    protected MyAggregateRoot() {
    }

    @EventSourcingHandler
    private void handleMyAggregateCreatedEvent(MyAggregateCreatedEvent event) {
        // 식별자 값은 항상 올바르게 초기화 되어야 합니다.
        this.aggregateIdentifier = event.getMyAggregateIdentifier();

        // ... 상태를 변경합니다.
    }
}                
```

`@EventSourcingHandler` 에노테이션이 달린 메서드들은 특정 규칙에 따라 호출됩니다. 이 규칙들은 [에노테이션 기반 이벤트 처리자](./event-handling.md#이벤트-처리자-정의하기)에서 상세히 설명할 `@EventHandler` 에노테이션이 달린 메서드들에 적용되는 규칙과 같습니다.

> **Note**
>
> JVM의 보안 설정을 Axon Framework가 메서드의 접근 제한을 변경할 수 있도록 하는 한 이벤트 처리 메서드를 private로 선언할 수도 있습니다. 이벤트 처리 메서드를 private로 선언하면 aggregate의 이벤트를 생성하는 메서드를 노출하는 공용 API와 이벤트를 처리하는 내부 로직을 분리할 수 있습니다.
>
> 대부분의 IDE는 "사용하지 않는 private 메서드"에 대한 경고를 무시할 수 있는 옵션을 가지고 있어서, 이벤트 처리 메서드를 private로 변경하여도 경고를 표시하지 않도록 할 수 있습니다. 아니면 `@SuppressWarnings("Unused-Declaration")` 에노테이션을 사용하면, 실수로 이벤트 처리 메서드를 삭제하지 않도록 할 수 있습니다.

몇몇 경우에, 특히 aggregate의 구조가 한두 개의 엔티티를 넘어 커질 경우, 같은 aggregate내의 다른 엔티티로 게시되는 이벤트들을 처리하는 것이 더욱 더 깔끔한 코드를 작성할 수 있는 방법입니다. 그러나 이벤트 처리 메서드는 aggregate의 상태를 재구성할 때도 호출이 되기 때문에, 반드시 특별한 주의가 필요합니다.

이벤트 처리 메서드내에서 `apply`메서드를 통해 새로운 이벤트를 발생시킬 수 있습니다. 이렇게 하면, 엔티티 B가 특정 이벤트를 받았을 때, 엔티티 A로 하여금 무언가를 하도록 하기 위해 다른 이벤트를 발생 시킬 수 있습니다. 단 이미 기록된 이벤트들을 재현할 때는 `apply()` 메서드 호출은 무시됩니다. `apply()` 메서드를 통해 발생되는 이벤트는 모든 엔티티가 첫 번째 이벤트를 수신한 이후에 해당 엔티티들에만 전달이 되는 점에 유의해야 합니다. `apply()` 메서드를 통해 발생한 이벤트가 적용된 이후의 엔티티 상태에 기반을 두어 더 많은 이벤트를 발생시켜야 한다면, `apply(...).andThenApply(...)` 형태로 코드를 작성해야 합니다.

`AggregateLifecycle.isLive()` 메서드를 사용하여 현재 aggregate가 활성 상태인지를 확인할 수 있습니다. 기본적으로 지난 이벤트들의 재현을 모두 마쳤다면 aggregate는 활성 상태로 간주 됩니다. 지난 이벤트들을 재현하는 동안에는 `isLive()`메서드는 false를 반환합니다. `isLive` 메서드를 사용하면 새로 생성된 이벤트를 처리할 때만 수행해야 하는 작업을 수행할 수 있습니다.

복잡한 Aggregate의 구조
----------------------------
복잡한 비즈니스 로직을 처리하기 위해서는 때때로 단 하나의 aggregate 루트가 제공하는 aggregate, 그 이상이 필요합니다. 이런 경우, 복잡성은 aggregate내의 다수의 엔티티에 걸쳐 있다는 것이 중요합니다. 이벤트 소싱을 사용할 때, aggregate뿐만 아니라 aggregate내의 엔티티들도 이벤트를 통한 상태 변경을 할 필요가 있습니다.

> **참고**
>
> 'Aggregate는 상태를 노출해서는 안 된다.'라는 것을 보통 '어떤 엔티티도 상태에 대한 접근 메서드를 가져선 안 된다'라고 오해합니다. 그렇지 않으며 사실은 aggregate *내부의* 엔티티들이 같은 aggregate내의 다른 엔티티들에 상태를 노출하는 것은 Aggregate를 작성하는 데 많은 도움이 됩니다. 그렇지만 Aggregate *외부의* 엔티티들에 상태를 노출하는 것은 권장하지 않습니다.

Axon을 통해 복잡한 aggregate 구조 내에서의 이벤트 소싱을 처리할 수 있습니다. Aggregate 루트와 같은 엔티티들은 단지 객체일 뿐입니다. 하위 엔티티를 참조하는 필드(field)에 반드시 `@AggregateMember` 에노테이션을 달아 주어야 합니다. Axon은 `@AggregateMember`을 통해 명령 및 이벤트 처리자에게 해당 필드의 타입 클래스를 검사하도록 합니다.

Aggregate 루트를 포함한 엔티티가 이벤트를 적용할 때, Aggregate 루트가 먼저 해당 이벤트를 처리하고 난 다음에, 모든 `@AggregateMember` 에노테이션이 달린 항목들과 해당 항목의 하위 항목들로 차례로 이동하게 됩니다.

하위 항목을 포함하는 필드에는 반드시 `@AggregateMember` 에노테이션을 달아 주어야 합니다. `@AggregateMember` 에노테이션은 다음과 같은 유형의 필드에 사용합니다.

-   필드에 직접 사용된 엔티티 타입 (예 `private CustomAggregateMember customAggregateMember`);

-   `Iterable` 타입의 필드, `Set`, `List`와 같은 모든 컬렉션을 포함합니다.;

-   `java.util.Map` 타입의 항목의 값(value)

### Aggregate내에서 명령 처리

명령 처리자가 해당 작업을 수행하는데 Aggregate의 상태는 필요하지 않지만, 명령 처리를 위한 상태를 가지고 있는 Aggregate내에 직접 명령 처리자를 정의하는 것을 권장합니다.

Aggregate에 명령 처리자를 정의하기 위해선, 단지 명령 처리를 위한 메서드에 `@CommandHandler` 에노테이션을 달아 주기만 하면 됩니다. 다른 일반적인 핸들러 메서드에 적용되는 규칙을 동일하게 `@CommandHandler` 에노테이션 대상 메서드에도 적용하면 됩니다. 그렇지만 명령은 명령 객체가 포함하고 있는 페이로드에 의해서면 분배되지는 않습니다. 명령 메시지들은 명령 객체의 정규화된 클래스 명을 기본값으로 하는 이름을 가지고 있습니다.

기본적으로,  `@CommandHandler` 에노테이션 메서드는 다음과 같은 매개변수 타입을 받습니다.

- 첫 번째 매개변수는 명령 메시지의 페이로드입니다. `@CommandHandler` 에노테이션에 처리자가 처리할 명령 객체의 이름을 명시적으로 선언하였다면, 첫 번째 매개변수는 `Message` 혹은 `CommandMessage`가 될 수도 있습니다. 기본적으로, 명령 이름은 명령 메시지의 페이로드 객체의 정규화된 클래스의 이름이 됩니다.

- `@MetaDataValue` 에노테이션과 에노테이션에 명시된 키에 해당하는 메타 데이터값을 인자로 받을 수 있습니다. (예 `@MetaDataValue("userId") String id")`. `@MetaDataValue` 에노테이션에 `required` 속성을 기본값인 `false`로 주게 되면, 없는 메타 데이터를 `null`로 처리하여 넘겨줍니다. 반면 `required`가 `true`이고 해당 메타 데이터가 존재하지 않으면, 해당 메서드는 호출되지 않습니다.

- `MetaData` 타입의 매개변수는 주입된 `CommandMessage`의 전체 `MetaData`를 포함하게 됩니다.

- `UnitOfWork` 타입의 매개변수는 현재의 작업 단위(Unit of Work) 객체를 가지게 됩니다. 넘겨받은 `UnitOfWork` 매개변수를 통해, 명령 처리자는 작업 단위 객체의 특정 단계에서 실행되어야 할 작업을 등록할 수 있거나 이미 등록된 자원을 사용할 수 있습니다.

- `Message` 혹은 `CommandMessage` 타입의 매개변수는 페이로드와 메타 데이터를 포함한 완전한 메시지를 가지게 됩니다. 이를 통해 메타 데이터의 필드 혹은 래핑 메시지의 다른 속성들을 사용할 수 있습니다.

Aggregate 타입의 명령 메시지를 처리해야하는 인스턴스를 알기 위해서, 아래의 예제 코드처럼 명령 객체 내의 Aggregate 식별자를 포함하는 속성에 반드시 `@TargetAggregateIdentifier` 에노테이션을 붙여야 합니다. `@TargetAggregateIdentifier` 애노테이셔은 필드나 속성에 접근하는 메서드(예, getter 메서드)에 사용해야 합니다.

비록 Aggregate 인스턴스를 생성하는 명령에 Aggregate 식별자 에노테이션을 사용하도록 권장하지만, 인스턴스를 생성하는 명령은 대상 aggregate 식별자를 확인할 필요가 없습니다.

명령을 분배하기 위해 다른 메커니즘을 사용하길 원한다면, 사용자 정의 `CommandTargetResolver`를 제공하여 해당 기능을 재정의 할 수 있습니다. 재정의한 `CommandTargetResolver`는 Aggregate의 식별자와 가능하다면 기대 버전을 반환해야 합니다..

> **참고**
>
> Aggregate 생성자에 `@CommandHandler` 에노테이션을 사용하면, 해당 명령은 aggregate의 새로운 인스턴스를 생성하고 생성된 aggregate를 저장소에 저장합니다. 이러한 명령들은 특정 aggregate 인스턴스를 대상으로 할 필요가 없습니다. 그러므로, 이러한 명령들은 `@TargetAggregateIdentifier` 혹은  `@TargetAggregateVersion` 에노테이션을 필요로 하지 않으며, `CommandTargetResolver`가 이런 명령들을 처리할 필요가 없습니다.  
>
> 명령으로부터 aggregate 인스턴스를 생성할 때, 해당 명령 처리가 정상적으로 수행되면 명령에 대한 콜백에 aggregate 식별자를 전달합니다.

```java
import static org.axonframework.commandhandling.model.AggregateLifecycle.apply;

public class MyAggregate {

    @AggregateIdentifier
    private String id;

    @CommandHandler
    public MyAggregate(CreateMyAggregateCommand command) {
        apply(new MyAggregateCreatedEvent(IdentifierFactory.getInstance().generateIdentifier()));
    }

    // Axon에서의 처리를 위한 기본 생성자.
    MyAggregate() {
    }

    @CommandHandler
    public void doSomething(DoSomethingCommand command) {
        // 해당 명령에 대한 처리 수행
    }

    // 간단한 예제코드이기 때문에, MyAggregateCreatedEvent를 수신하여 id값을 설정하는 코드는 생략합니다.
}

public class DoSomethingCommand {

    @TargetAggregateIdentifier
    private String aggregateId;

    // 간단한 예제코드이므로, 이하 코드는 생략합니다.

}
```

Aggregate 설정을 위한 Axon Configuration API 사용에 대한 예제 코드입니다.

```java
Configurer configurer = ...
// 기본 사항 사용을 위함.
configurer.configureAggreate(MyAggregate.class);

// 사용자 정의 허용
configurer.configureAggregate(
        AggregateConfigurer.defaultConfiguration(MyAggregate.class)
                           .configureCommandTargetResolver(c -> new CustomCommandTargetResolver())
);
```

`@CommandHandler` 에노테이션은 Aggregate 루트에만 사용할 수 있는 것은 아닙니다. 모든 명령 처리자들을 Aggregate 루트에 정의하게 되면 너무 많은 메서드들이 생길 수 있으며, 그 중 많은 수의 메서드는 단순히 하위 엔티티 중 하나로 호출을 전달합니다. 이런 것이 문제가 된다면, `@CommandHandler` 에노테이션을 하위의 엔티티 중 하나의 엔티티의 메서드에 선언하여 사용할 수 있습니다. Axon이 `@CommandHandler` 에노테이션이 사용된 메서드를 찾도록 하기 위해서 해당 하위 엔티티를 aggregate 루트의 속성 필드에 반드시 `@AggregateMember` 에노테이션을 사용해야 합니다. `@AggregateMember` 에노테이션이 사용된 필드만이 명령 처리자를 위한 검사 대상이 되기 때문입니다. 해당 엔티티에 대한 명령을 수신했을 때, 만약 해당 엔티티 필드의 값이 `null`이라면 예외가 발생하게 됩니다.

```java
public class MyAggregate {

    @AggregateIdentifier
    private String id;

    @AggregateMember
    private MyEntity entity;

    @CommandHandler
    public MyAggregate(CreateMyAggregateCommand command) {
        apply(new MyAggregateCreatedEvent(...);
    }

    // Axon에서의 처리를 위한 기본 생성자.
    MyAggregate() {
    }

    @CommandHandler
    public void doSomething(DoSomethingCommand command) {
        // 해당 명령에 대한 처리 수행.
    }

    // 간단한 예제코드이기 때문에, MyAggregateCreatedEvent를 수신하여 id 값을 설정하는 코드는 생략합니다.
    // 그리고 생애주기 상의 특정 지점에서, DoSomethingInEntityCommand commands를 처리하기 위해 "entity" 필드에 대한 값을 설정을 반드시 해줘야 합니다.
}

public class MyEntity {

    @CommandHandler
    public void handleSomeCommand(DoSomethingInEntityCommand command) {
        // entity에 대한 DoSomethingInEntityCommand 처리 수행
    }
}
```

> **참고**
>
> Aggregate내에 개별 명령에 대해 처리자는 오로지 하나만 존재해야 합니다. 즉, 같은 타입의 명령을 처리할 수 있는 다수의 엔티티(루트, 루트가 아닌 엔티티 모두 포함)에 `@CommandHandler`를 사용할 수 없습니다. 만약 조건에 따라서 명령을 특정 엔티티로 분배해야 한다면, 이런 엔티티들의 부모 객체가 해당 명령을 처리해야 하고 조건에 따라 해당 명령을 전달해야 합니다.
>
> 실행 시점의 필드 타입과 선언 시점의 필드 타입이 반드시 일치해야 하는 것은 아니지만, `@AggregateMember`로 선언된 필드의 타입으로 `@CommandHandler`를 적용합니다.

엔티티들을 포함하는 컬렉션(Collection)과 맵(Map) 타입의 필드에도 `@AggregateMember` 에노테이션을 사용할 수 있습니다. 맵을 경우 값들에는 엔티티들을 포함하도록 해야 하고 키는 값에 대한 참조로 사용될 값을 가지면 됩니다.

명령을 정확한 인스턴스로 분배해야 하므로, 이런 인스턴스들은 반드시 정확하게 식별이 되어야 합니다. `@EntityId` 에노테이션을 인스턴스들의 "id" 필드에 반드시 사용해야 합니다. 메시지를 받아야 할 엔티티를 찾기 위한 용도로 사용되는 명령의 속성은 기본적으로 `@EntityId` 에노테이션이 사용된 필드의 이름으로 지정이 됩니다. 예를 들어, "myEntityId"필드에 `@EntityId` 에노테이션을 사용했을 때, 해당 명령은 반드시 동일한 이름의 속성을 가지고 있어야 합니다. 다시 말하면, 명령 객체는 `getMyEntityId` 혹은 `myEntityId()` 메서드들 중 하나를 반드시 포함해야 합니다. 만약 필드 명과 분배 기준이 되는 속성이 다르다면, `@EntityId(routingKey = "customRoutingProperty" )`와 같이 `routingKey`를 사용하여 명시적으로 분배 기준이 되는 항목 이름을 지정해야 합니다.

`@AggregateMember` 에노테이션이 사용된 컬렉션이나 맵에서 엔티티가 발견되지 않는다면, aggregate이 해당 시점에 수신된 명령을 처리할 수 없으므로 `IllegalStateException`이 발생하게 됩니다.

> **참고**
>
> `@AggregateMember` 에노테이션이 사용된 컬렉션이나 맵 타입의 필드를 선언할 때, 내부 요소의 타입을 식별할 수 있게 해주는 제너릭을 사용해야 합니다. 만약 제너릭을 사용할 수 없다면, `@AggregateMember` 에노테이션의 속성인 `entityType`을 사용하여 엔티티의 타입을 명시적으로 선언해 주어야 합니다.

### 외부 명령 처리자

명령을 Aggregate 인스턴스로 직접 분배할 수 없는 경우가 있는데, 이런 경우 명령 처리자를 등록해야 합니다.

 명령 처리자 객체는 `@CommandHandler` 에노테이션이 붙은 메서드를 가지는 일반 객체입니다. Aggregate의 경우와는 다르게, 오로지 단 하나의 명령 처리자 객체가 생성되어 해당 명령 처리자 객체에 선언된 모든 명령을 처리할 수 있습니다.

```java
public class MyAnnotatedHandler {

    @CommandHandler
    public void handleSomeCommand(SomeCommand command, @MetaDataValue("userId") String userId) {
        // 명령을 받아 처리할 로직 수행
    }

    @CommandHandler(commandName = "myCustomCommand")
    public void handleCustomCommand(SomeCommand command) {
       // 명령을 받아 처리할 로직 수행
    }

}

// 위의 명령 처리자를 커맨드 버스(command bus)에 등록
Configurer configurer = ...
configurer.registerCommandHandler(c -> new MyAnnotatedHandler());
```

### 명령 처리자로부터 결과 반환하기
몇몇 경우, 명령을 전달하는 컴포넌트는 명령 처리 결과에 대한 정보가 있어야 합니다. 명령 처리자는 명령 처리 메서드를 통해 결괏값을 반환할 수 있는데, 해당 결괏값은 명령을 보낸 객체에 전달 됩니다.

Aggregate의 생성자에 `@CommandHandler`를 사용한 경우는 위의 경우에 대한 예외가 됩니다. 이런 경우, Aggregate 자신의 메서드를 통해 결괏값을 반환하는 대신, `@AggregateIdentifier` 에노테이션이 사용된 항목의 값을 대신 반환합니다.

> **참고**
>
> 명령 처리 후 결과를 반환하는 것이 가능하지만, 자주 사용하는 것을 권장하지 않습니다. 명령에 포함된 의도는 특정 값을 반환하는 것이 되어서는 안 됩니다. 특정 명령을 통해 결괏값을 반환받아야 한다면, 질의 메시지를 통해 원하는 결괏값을 받도록 설계해야 합니다. 명령 처리에 대한 결괏값을 반환하는 일반적인 예는 새로이 생성된 엔티티의 식별자를 반환하는 것입니다.
