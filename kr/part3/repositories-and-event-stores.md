레퍼지토리와 이벤트 저장소
=============================

레퍼지토리는 aggregate들에 접근할 수 있도록 하는 메커니즘입니다. 레퍼지토리는 데이터를 저장하기 위해 사용되는 실제 저장 메커니즘에 대해 게이트웨이로 작동합니다. CQRS에선, 레퍼지토리들은 고유한 식별자를 통해 aggregate를 찾을 수 있는 기능만 있으면 됩니다. 다른 유형의 질의들은 조회를 위한 데이터베이스 상에서 실행이 되어야 합니다.

Axon Framework 상에서, 모든 레퍼지토리들은 반드시 `Repository` 인터페이스를 구현해야 합니다. `Repository` 인터페이스는 `load(identifier, version)`, `load(identifier)` 그리고 `newInstance(factoryMethod)` 메서드들을 규정하고 있습니다. `load` 메서드들은 레퍼지토리로부터 aggregate를 받아 오기 위한 메서드이며, 선택적으로 줄 수 있는 매개변수인 `version`은 동시변경(concurrent modification)을 감지하기 위해 사용 됩니다. (참고: [충돌 탐지 및 해결을 위한 향상된 방법](#충돌-탐지-및-해결을-위한-향상된-방법)) `newInstance` 메서드는 레퍼지토리에서 새롭게 생성된 aggregate를 등록하기 위해 사용합니다.

기반 영속화 저장소와 감사(auditing)의 필요에 따라, 대부분의 레퍼지토리가 필요로 하는 기본 기능을 제공하는 기본 구현체들은 많이 있습니다. Axon Framework은 aggregate의 현재 상태를 저장하는 레퍼지토리와 (참고 [표준 레퍼지토리](#표준-레퍼지토리)) aggregate의 이벤트들을 저장하는 레퍼지토리(참고 [이벤트 소싱 레퍼지토리](#이벤트-소싱-레퍼지토리))를 구분하여 사용합니다.

`Repository` 인터페이스는 `delete(identifier)` 메서드를 제공하지 않습니다. `AggregateLifecycle.markDeleted()` 메서드를 삭제하고자 하는 aggregate내에서 호출하여 해당 aggregate를 삭제 처리할 수 있습니다. aggregate를 삭제한다는 것은 다른 상태 변화와 마찬가지로 상태의 변경이지만 한 가지 차이점은 많은 경우에 다시 되돌릴 수 없다는 것입니다. aggregate의 상태를 "삭제"로 변경하는 메서드를 구현하여 해당 aggregate의 삭제에 따라 발생해야 하는 이벤트를 등록할 수 있습니다.

표준 레퍼지토리
---------------------

표준 레퍼지토리들은 Aggregate의 실제 상태를 저장합니다. 매번 상태가 변경될 때마다 새로운 상태 값으로 이전 상태 값을 덮어씁니다. 이렇게 함으로써, 질의 컴포넌트와 명령 컴포넌트들이 같은 정보를 사용하도록 합니다. 개발하는 애플리케이션의 특성에 따라 다를 수 있으나 이런 형태가 가장 간단한 형태입니다. 이런 형태가 문제가 된다면, Axon이 제공하는 빌딩 블록을 사용하여 레퍼지토리를 직접 구현할 수 있습니다.

Axon은 표준 레퍼지토리 구현체로 `GenericJpaRepository`를 기본 제공합니다. `GenericJpaRepository`를 사용하려면, Aggregate는 JPA 엔티티이어야 합니다. `GenericJpaRepository`는 레퍼지토리에 저장될 Aggregate의 실제 타입을 나타내는 클래스 그라고 `EntityManagerProvider`와 함께 설정하며, `EntityManagerProvider`는 실제 영속화를 관라하는 `EntityManager`를 제공합니다. 또한, Aggregate의 정적 메서드인 `AggregateLifeCycle.apply()`를 호출할 때 이벤트를 게시하는 `EventBus`를 `GenericJpaRepository`로 넘겨줄 수 있습니다.

레퍼지토리를 직접 구현하는 것도 어렵지 않습니다. 추상 레퍼지토리인 `LockingRepository`를 상속하여 구현하는 것이 가장 빠르게 레퍼지토리를 구현하는 방법입니다. aggregate의 래퍼 타입으로는 `AnnotatedAggregate`를 사용하는 것을 권장합니다. 상세한 예제는 `GenericJpaRepository`의 소스 코드를 살펴보세요.

이벤트 소싱 레퍼지토리
===========================

이벤트들을 기반으로 상태를 재구성할 수 있는 Aggregate 루트는 이벤트 소싱 레퍼지토리에 의해 로딩되도록 설정할 수 있습니다. 이런 이벤트 소싱 레퍼지토리들은 aggregate 자체를 저장하지 않지만, aggregate에 의해 발생하는 일련의 이벤트들을 저장합니다. 이런 이벤트들을 기반으로, 언제든 aggregate의 상태는 복원될 수 있습니다.

`EventSourcingRepository` 구현체는 Axon Framework 내의 이벤트 소싱 레포지토리가 필요로 하는 기본 기능들을 제공합니다. `EventStore`(참고:[이벤트 스토어 구현체들](#이벤트-스토어-구현체들))에 따라 이벤트를 저장하는 실제 메커니즘은 다를 수 있습니다.

선택적으로, `EventSourcingRepository`에 Aggregate 팩토리(Factory)를 제공할 수 있습니다. 제공할 `AggregateFactory`는 aggregate를 생성하는 방법을 명시해야 합니다. aggregate이 생성되면, `EventSourcingRepository`는 이벤트 스토어로부터 로드된 이벤트들을 사용하여 생성된 aggregate를 초기화합니다. Axon Framework은 사용할 수 있는 `AggregateFactory`들을 제공하며 제공된 `AggregateFactory`가 부족하면 직접 구현하여 사용할 수 있습니다.

*GenericAggregateFactory*

`GenericAggregateFactory`는 이벤트로부터 복원 가능한(event sourced) 모든 타입의 aggregate 루트에 사용할 수 있는 `AggregateFactory`의 특별한 구현체입니다. `GenericAggregateFactory`는 레퍼지토리가 관리하는 aggregate의 인스턴스를 생성합니다. 해당 aggregate 클래스는 추상 클래스일 수 없으며 바디가 비어 있는 기본 생성자를 반드시 선언하고 있어야 합니다.

GenericAggregateFactory는 aggregate에 비 직렬화 자원을 주입하지 않아도 되는 대부분 상황에서 사용할 수 있습니다.

*SpringPrototypeAggregateFactory*

애플리케이션의 아키텍처를 고려한 결정에 따라, 스프링을 사용하여 aggregate에 필요한 의존 객체들을 주입할 수 있습니다. 예를 들어, 질의 레퍼지토리를 aggregate에 주입하여 특정 값이 존재하는지(혹은 존재하지 않는지)를 확인하도록 할 수 있습니다.

aggregate에 의존 객체를 주입하기 위해선, 스프링 컨텍스트에 aggregate 루트 객체의 프로토타입 빈을 설정해야 하며 `SpringPrototypeAggregateFactory` 또한 스프링 컨텍스트에 설정해줘야 합니다. 생성자를 사용하여 일반적인 인스턴스를 생성하는 것 대신, 스프링 컨텍스트를 사용하여 aggregate들의 인스턴스들을 생성하며, 의존 객체들을 aggregate들에 주입합니다.

*AggregateFactory 직접 구현하기*

몇몇 경우, `GenericAggregateFactory`가 원하는 목적에 맞지 않을 수 있습니다. 예를 들어, 다른 여러 시나리오에 대해 다수의 구현체를 가지는 추상 aggregate 타입을 사용해야 하는 경우, (예, `PublicUserAccount`와 `BackOfficeAccount` 모두 `Account`를 상속하는 경우) 각각의 aggregate들 별로 다른 저장소를 생성하는 대신, 하나의 저장소를 사용하고 AggregateFactory는 각각의 다른 구현체들을 인식하도록 설정할 수 있습니다.

Aggregate Factory가 수행하는 작업 대부분은 초기화되지 않은 Aggregate 인스턴스를 생성하는 것입니다. Aggregate Factory는 주어진 aggregate의 식별자와 이벤트 스트림의 첫 번째 이벤트를 사용하여 aggregate 인스턴스를 생성합니다. 보통, 주어진 첫 번째 이벤트는 aggregate에 대한 생성 이벤트로 생성해야 할 aggregate의 타입에 대한 힌트를 포함하고 있습니다. 해당 힌트 정보를 사용하여 구현체를 선택하고 생성자를 호출할 수 있습니다. 해당 aggregate는 반드시 초기화되지 않은 상태이어야 하므로 생성자에 이벤트를 인자로 줄 수 없음을 명심해야 합니다.

지난 이벤트를 가지고 aggregate를 초기화하는 것은 레퍼지토리에서 직접 aggregate를 로딩하는 것에 비교하면 시간 소모적입니다. `CachingEventSourcingRepository`는 사용 가능하다면, aggregate를 로딩할 수 있는 캐시를 제공합니다.

이벤트 스토어 구현체들
===========================

이벤트 소싱 레포지토리들은 이벤트를 저장하고 aggregate로부터 발생한 이벤트들을 로딩하기 위한 이벤트 스토어(store)를 필요로 합니다. 이벤트 스토어는 이벤트 버스의 기능들을 제공하고, 게시된 이벤트들을 영속화(저장)하며 aggregate의 식별자에 기반을 두어 이벤트들을 꺼내올 수 있습니다.

Axon은 `EmbeddedEventStore`라는 이벤트 스토어를 기본으로 제공합니다. `EmbeddedEventStore`는 이벤트의 저장과 검색을 `EventStorageEngine`을 통해 수행합니다. 실제 이벤트의 저장과 검색은 `EventStorageEngine`를 통해 이루어집니다.

`EventStorageEngine`의 구현체는 `JpaEventStorageEngine`, `JDBCEventStorageEngine` 그리고 `MongoDBEventStorageEngine`등과 같이 여러 가지가 있습니다.

`JPA기반의 이벤트 저장 엔진`
-----------------------

`JpaEventStorageEngine`은 JPA와 호환 가능한 데이터 소스에 이벤트들을 저장합니다. JPA기반의 이벤트 저장소는 엔트리라고 불리는 곳에 이벤트들을 저장합니다. 이 엔트리들은 이벤트들을 직렬화된 형태로 가지고 있으며, 해당 엔트리들을 빠르게 찾을 수 있도록 메타 데이터가 저장된 추가 항목들 또한 가지고 있습니다. `JpaEventStorageEngine`을 사용하려면, JPA(`javax.persistance`) 에노테이션들을 클래스 패스에 추가해야만 합니다.

기본적으로 `DomainEventEntry`와 `SnapshotEventEntry` 클래스들을 포함하는 영속화 컨텍스트(persistence context)(예, `META-INF/persistence.xml` 파일에 정의된 내용)를 설정해야 합니다.

> 참고
>
> `DomainEventEntry`와 `SnapshotEventEntry`클래스들은 모두 `org.axonframework.eventsourcing.eventstore.jpa` 패키지안에 있습니다.

아래의 코드는 영속화 컨텍스트를 설정의 예제입니다.

``` xml
<persistence xmlns="http://java.sun.com/xml/ns/persistence" version="1.0">
    <persistence-unit name="eventStore" transaction-type="RESOURCE_LOCAL"> (1)
        <class>org...eventstore.jpa.DomainEventEntry</class> (2)
        <class>org...eventstore.jpa.SnapshotEventEntry</class>
    </persistence-unit>
</persistence>
```

1. 위의 예제에서, 이벤트 저장소에 대한 구체적인 영속화 단위를 볼 수 있습니다. 그렇지만, 세번째 라인에 다른 영속화 단위 설정을 추가할 수 있습니다.
1. 해당 코드를 통해 `JpaEventStore`가 사용하는 `DomainEventEntry`를 영속화 컨텍스트에 등록합니다.

> **참고**
>
> Axon은 두 개의 스레드가 동시에 같은 Aggregate에 접근하는 것을 방지하기 위해 락킹(Locking)을 사용합니다. 그렇지만, 같은 데이터베이스에 접근하는 다수의 JVM을 가지고 있다면, 락킹(Locking)은 도움이 되지 않습니다. 이런 경우, 데이터 충돌(conflict)을 발견하기 위해선 데이터베이스에 의존할 수밖에 없습니다. 테이블에 하나의 aggregate에 대해 고유한 일련번호를 가진 하나의 이벤트만을 저장할 수 있기 때문에, 이벤트 저장소에 동시에 접근하게 되면 키 제약사항 위반을 초래하게 될 것입니다. 이미 존재하는 aggregate에 대해 같은(이미 저장된) 일련번호를 가지는 두 번째 이벤트를 저장하게 되면 에러가 발생하게 됩니다.
>
> `JpaEventStorageEngine`은 위와 같은 에러를 발견하여 `ConcurrencyException`을 발생시킵니다. 하지만, 각각의 데이터베이스 시스템은 위와 같은 위반 사항을 다르게 보여줍니다. > 만약 `JapEventStore`에 `DataSource`를 등록하면, 데이터베이스 타입을 확인하고 키 제약사항 위반에 대한 에러코드가 무엇인지 파악합니다. 이에 대한 대안으로, `PersistenceExceptionTranslator` 인스턴스를 제공하여, 발생한 예외가 키 제약사항 위반을 나타내는지 알 수 있습니다.
>
> 만약 `DataSource` 혹은 `PersistenceExceptionTranslator`를 제공하지 않는다면, 데이터베이스 드라이버가 던진 예외 그대로 예외가 발생할 것입니다.

기본적으로, JPA 기반의 이벤트 저장 엔진은 `EventStorageEngine`이 사용할 `EntityManager` 인스턴스를 반환하는 ` EntityManagerProvider`의 구현체가 있어야 합니다. 이를 통해 애플리케이션은 관리할 수 있는 영속화 컨텍스트를 사용할 수 있습니다. `EntityManagerProvider`는 정확한 `EntityManager`를 제공하는 해야 합니다.

사용 가능한 `EntityManagerProvider`의 몇 가지 구현체가 있으며, 각기 다른 목적으로 사용할 수 있습니다. `SimpleEntityManagerProvider`는 단순하게 생성 시 주어진 `EntityManager` 인스턴스를 반환합니다. 이는 컨테이너가 관리하는 컨텍스트에 대한 간단한 옵션으로 구현체를 사용합니다. 다른 방법으로는, `ContainerManagedEntityManagerProvider`가 있으며, `ContainerManagedEntityManagerProvider`는 기본 영속화 컨텍스트를 반환하고 JPA 기반의 이벤트 저장소에 의해 사용됩니다.

`JpaEventStore`로 사용하고자 하는 "myPersistenceUnit"이라는 영속화 단위를 가지고 있다면, `EntityManagerProvider`에 대한 구현은 아래의 예제를 참고하면 될 것입니다.

``` java
public class MyEntityManagerProvider implements EntityManagerProvider {

    private EntityManager entityManager;

    @Override
    public EntityManager getEntityManager() {
        return entityManager;
    }

    @PersistenceContext(unitName = "myPersistenceUnit")
    public void setEntityManager(EntityManager entityManager) {
        this.entityManager = entityManager;
    }                
```

기본적으로, JPA 기반의 이밴트 저장소는 `DomainEventEntry`와 `SnapshotEventEntry`타입의 엔티티로 엔트리들을 저장합니다. 대부분은 이 방법으로 충분하지만, 이러한 엔티티가 제공하는 메터 데이터가 원하는 만큼 충분하지 않은 상황을 겪게 될 수 있습니다. 혹은 다른 aggregate 유형을 다른 테이블들에 저장하고 싶을 수도 있습니다.

만약 이런 상황이라면, `JpaEventStorageEngine`을 확장(상속)하여 사용할 수 있습니다. `JpaEventStorageEngine`은 상속 가능한(protected) 다수의 메서드를 가지고 있어서, 직접 재정의하여 원하는 기능을 추가할 수 있습니다.

> **경고**
>
하이버네이트와 같은 영속화 제공자들은 first-level 캐시를 사용하여 `EntityManager`를 구현한 것에 주의해야 합니다. 일반적으로 이는 질의에서 사용되거나 반환 된 모든 엔티티가 EntityManager에 캐시 되어 연결됨을 의미합니다. 해당 엔티티들은 엔티티가 포함된 트랜잭션이 커밋되거나 트랜잭션 내에서 명시적으로 "지움"이 수행될 때만 삭제됩니다. 이것은 특히 질의가 트랜잭션의 컨텍스트에서 실행되는 상황에 해당합니다.
>
> 이 문제를 우회적으로 해결하기 위해선, 비 엔티티(non-entity)객체에 대한 질의만을 반드시 사용해야 합니다. JPA의 "SELECT new SomeClass(parameters) FROM ..."과 같은 유형의 질의를 사용하여 문제를 해결할 수 있습니다. 다른 방법으로는, 일괄적으로 이벤트들을 가져온 후에, `EntityManager.flush()`와 `EntityManager.clear()` 메서드를 호출하세요. 그렇게 하지 않으면 상당한 양의 이벤트 스트림을 로딩할 경우, `OutOfMemoryException`가 발생할 수 있습니다.

JDBC 기반의 이벤트 저장 엔진
-------------------------

JDBC기반의 이벤트 저장 엔진은 JDBC Connection을 사용하여 JDBC를 사용해 연결한 데이터 저장소에 이벤트들을 저장합니다. 일반적으로, 데이터 저장소는 관계형 데이터베이스(RDMBS)입니다. 이론적으로, JDBC 드라이버를 제공하는 모든 데이터베이스는 JDBC 기반의 이벤트 저장 엔진과 사용할 수 있습니다.

JPA와 비슷하게, JDBC 기반의 이벤트 저장 엔진은 엔트리들로 이벤트를 저장합니다. 기본적으로 각 이벤트는 테이블의 하나의 열에 해당하는 단일 엔트리로 저장됩니다. 하나의 테이블은 이벤트들을 저장하기 위해 사용되고 다른 테이블은 스냅샵을 저장하기 위해 사용됩니다.

`JdbcEventStorageEngine`은 `ConnectionProvider`를 사용하여 데이터베이스 연결을 받아 옵니다. 일반적으로, 이런 연결들은 데이터 소스(DataSource)로부터 직접 받아 올 수 있습니다. 그렇지만, Axon은 이런 연결들을 작업 단위(Unit of Work)와 묶어서 사용합니다. 그래서 하나의 작업 단위는 하나의 연결을 사용합니다. 이것은 단일 트랜잭션을 사용하여 모든 이벤트를 저장할 수 있도록 보장하며, 심지어 다수의 작업 단위들이 같은 스레드 내에서 중첩될 수 있게 해줍니다.

> **참고**
>
> 스프링을 사용한다면, `DataSource`의 커넥션을 이미 존재하는 트랜잭션에 연결하기 위해 `SpringDataSourceConnectionProvider`를 사용하는 것을 권장합니다.

몽고DB 기반의 이벤트 저장 엔진
----------------------------

몽고DB는 도큐먼트 기반의 NoSQL 저장소입니다. 몽고DB의 규모 가변성 특징들은 이벤트 저장소로 사용하는 데 적합합니다. Axon은 `MongoDBEventStorageEngine`을 제공하며, `MongoDBEventStorageEngine`은 몽고DB를 기반 데이터베이스로 사용합니다. `MongoDBEventStorageEngine` 은 Axon 몽고 모듈에 포함되어 있으며, 메이븐 artifactId는 `axon-mongo` 입니다.

이벤트들은 두 개의 분리된 컬렉션에 저장이 됩니다. 하나는 실제 이벤트 스트림들을 저장하고, 다른 하나는 스냅샵들을 저장합니다.

기본적으로, `MongoDBEventStorageEngine`은 각각의 이벤트 유형을 독립된 도큐먼트에 저장합니다. 그렇지만 이벤트를 저장하는 방법에 대한 `StorageStrategy`를 변경할 수 있습니다. Axon에서 제공하는 다른 대안은 `DocumentPerCommitStorageStrategy`이며, `DocumentPerCommitStorageStrategy`은 하나의 도큐먼트를 생성하여 단일 커밋내에 포함된 모든 이벤트를 생성한 도큐먼트에 저장합니다. (예, 동일 `DomainEventStream`내의 이벤트들)

전체 커밋을 하나의 도큐먼트에 저장하는 것은 커밋을 자동으로 저장하게 되는 장점이 있습니다. 게다가, 모든 이벤트에 대해 단일 왕복(데이터베이스로 데이터를 요청하고 응답을 받는 과정)으로 가져올 수 있습니다. 단점으로는, 데이터베이스로 이벤트를 조회하기 위한 질의를 직접 실행하는 것이 힘들어집니다. 예를 들어, 이벤트들이 하나의 커밋 문서에 포함되어 있다면, aggregate에서 다른 aggregate로 이벤트들을 옮기는 것이 어려울 수 있습니다.

몽고DB는 많은 설정이 필요하지 않습니다. 이벤트를 저장하는 컬렉션의 참조만 있으면 됩니다. 운영 환경에선, 컬렉션에 설정된 인덱스들을 재확인해야 할 수 있습니다.

이벤트 저장 유틸리티
---------------------

Axon을 통해 특정 환경에서 유용하게 사용할 수 있는 이벤트 저장 엔진들을 사용할 수 있습니다.

### 여러 이벤트 저장소를 하나로 묶어 사용
`SequenceEventStorageEngine`은 두 개의 이벤트 저장 엔진을 묶어서 사용할 수 있게 해주는 래퍼입니다. 이벤트를 읽어 올 때, `SequenceEventStorageEngine`은 두 개의 이벤트 저장 엔진들에서 이벤트들을 반환합니다. 추가된 이벤트들은 두 번째 이벤트 저장소에만 추가됩니다. `SequenceEventStorageEngine`은 예를 들어, 성능상의 이유로 두 개의 이벤트 저장 구현체들을 사용하는 경우에 유용하게 사용할 수 있습니다. 두번째 구현체는 빠른 읽기와 쓰기를 지원하지만, 첫 번째 구현체는 큰 용량을 지원하지만 느린 이벤트 저장소일 때 `SequenceEventStorageEngine`를 사용할 수 있습니다.

### 필터링 이벤트 저장 엔진
`FilteringEVentStorageEngine`은 predicate를 사용하여 이벤트들을 걸러낼(필터링) 수 있습니다. 따라서 predicate에 일치하는 이벤트들만을 저장할 수 있습니다. 이벤트 저장소를 이벤트의 원천(source)으로 사용하는 이벤트 처리자들은 predicate에 일치하지 않는 이벤트들이 저장되지 않기 때문에, 해당 이벤트들을 받지 못할 수 있습니다.

### 메모리 기반 이벤트 저장 엔진
`InMemoryEventStorageEngine`은 메모리에 이벤트들을 저장하는 `EventStorageEngine`의 구현체입니다. 다른 이벤트 저장 엔진에 비교해 뛰어난 성능을 제공할 수 있지만, 오랜 기간 이벤트를 저장하는 용도로는 적합하지 않습니다. 그러나 이벤트가 있어야 하는 짧은 시간 동안 사용되는 도구 혹은 테스트 사용 시에 유용하게 사용할 수 있습니다.

직렬화 프로세싱
-------------------------------------

이벤트 저장소들은 이벤트들을 저장을 위한 준비를 하기 위해 이벤트를 직렬화하는 방법이 필요합니다. 기본적으로, Axon은 `XStreamSerializer`를 사용하며, `XStreamSerializer`은 [XStream](http://x-stream.github.io)을 사용하여 이벤트를 XML로 직렬화합니다. XStream은 Java 직렬화보다 아주 빠르며 보다 유연합니다. 게다가, XStream 직렬화의 결과물은 사람이 읽을 수 있습니다. 로깅과 디버깅용으로 상당히 유용합니다.

`XStreamSerializer`을 설정하여, 특정 패키지, 클래스 또는 필드에 사용해야 하는 별칭을 정의할 수 있습니다. 잠재적으로 긴 이름을 줄이는 좋은 방법일 뿐만 아니라, 이벤트의 클래스 정의가 변경될 때 별칭을 사용할 수도 있습니다. 별칭에 대한 자세한 내용은 XStream 웹 사이트를 방문하십시오.

다른 방법으로는, Axon은 이벤트를 JSON으로 직렬화하는 [Jackson](https://github.com/FasterXML/jackson)을 사용하는 `JacksonSerializer`를 제공합니다. `JacksonSerializer`는 보다 간결한 직렬화 형태를 생성하는 반면, 클래스를 Jackson의 요구 사항에 맞춰 작성해야 합니다.

`Serializer`를 구현하는 클래스를 생성하여, 직접 시리얼라이져를 구현하여 사용할 수 있습니다. 그리고 이벤트 저장소의 기본 객체 대신 구현체를 사용하는 이벤트 저장소를 설정해야 합니다.

### 이벤트 직렬화 대 다른 객체 직렬화 ###

Axon 3.1 이후, Axon이 직렬화해야 하는 다른 모든 객체들(커맨드들, 스냅샵, 사가(Saga) 등등)보다는 이벤트의 저장을 위해 다른 직렬화 객체를 사용할 수 있습니다. `XStreamSerializer`은 모든 객체를 직렬화하여 매우 적절한 기본값으로 만들지만, 항상 다른 애플리케이션과 공유하기 좋은 형태의 결과를 만드는 것은 아닙니다. `JacksonSerializer`은 훨씬 좋은 결과물을 만들지만, 직렬화를 위한 특정 형태를 가지는 객체들이 필요합니다. 이 구조는 일반적으로 이벤트에 나타나며, 이벤트 직렬화에 매우 적합합니다. 설정 API를 사용하여, 다음과 같이 간단하게 이벤트 직렬화 객체를 등록할 수 있습니다.

설정 API를 사용하여, 다음과 같이 간단하게 이벤트 직렬화 객체를 등록할 수 있습니다.

```java
Configurer configurer = ... // 초기화
configurer.configureEventSerializer(conf -> /* 직렬화 객체(시리얼라이져)를 생성합니다. */);
```

명시적으로 `eventSerializer`를 설정하지 않으면, 이미 구성된 메인 시리얼라이져를 사용하여 이벤트들을 직렬화합니다. (기본적으로 XStream 직렬화기를 기본값으로 사용합니다.).

이벤트 업 캐스팅
===============

소프트웨어 애플리케이션은 끊임없이 변경되는 속성을 가지고 있습니다. 이런 속성 때문에, 이벤트에 대한 정의 또한 시간이 지남에 따라 변경이 될 수 있습니다. 이벤트 저장소는 이벤트를 읽고 데이터 소스에 추가만 할 수 있는 것으로 간주하기 때문에, 이벤트가 언제 추가되었는지에 상관없이 모든 이벤트를 읽을 수 있어야 합니다. 이런 점 때문에 이벤트 업 캐스팅이 필요합니다.

원래 객체 지향 프로그래밍의 개념은 "하위 클래스는 필요에 따라 자동으로 상위 클래스로 캐스팅된다."를 포함하고 있습니다. 이런 업 캐스팅의 개념은 이벤트 소싱(sourcing)에도 적용될 수 있습니다. 이벤트를 업 캐스트한다는 것은 이벤트 원래의 구조를 새로운 구조로 변경한다는 것입니다. 객체지향(OOP)의 업 캐스팅과는 달리, 이벤트 업 캐스팅은 완전히 자동으로 이루어질 수 없습니다. 그 이유는 이전의 이벤트는 새로운 이벤트의 구조를 알 수 없기 때문입니다. 이전의 이벤트 구조를 새로운 구조로 변경하는 방법을 가진 업 캐스터들을 직접 작성하여 제공해야 합니다.

업 캐스터들은 수정 버전 x인 하나의 입력 이벤트를 가지며 수정 버전 x + 1의 새로운 이벤트들을 생산하거나 새로운 이벤트를 생산하지 않을 수 있는 클래스들입니다. 게다가, 업 캐스터들은 체인 형태로 처리됩니다. 즉, 하나의 업 캐스터의 결과물은 다음의 업 캐스터의 입력 값이 됩니다. 이렇게 체인 형태로 처리되는 것을 통해, 새롭게 수정된 각 버전의 이벤트들에 대한 업 캐스터들을 작게, 독립적으로 이해하기 쉽게 작성하여, 이벤트들을 계속해서(증분 방식으로) 갱신할 수 있습니다.

> **참고**
>
> 아마도 업 캐스팅의 가장 큰 장점은 기존 코드를 해치지 않고 리팩토링을 할 수 있다는 점입니다. 예를 들어, 전체 이벤트의 이력은 그대로 유지됩니다.

업 캐스터에게 어떤 버전의 직렬화된 객체가 전달 되는지를 알려 주기 위해서, 이벤트 저장소는 수정 번호와 함께 이벤트의 전체 이름 또한 저장합니다. 이 수정 번호는 `RevisionResolver`에 의해 생성되며, 직렬화 객체에 설정됩니다. Axon은 여러 `RevisionResolver`의 구현체를 제공합니다. 예를 들어, 이벤트의 페이로드에 사용된 `@Revision` 에노테이션을 확인하는 `AnnotationRevisionResolver`, Java 직렬화 API에 정의된 것과 같은 `serialVersionUID`를 사용하는 `serialVersionUIDRevisionResolver` 그리고 항상 이미 정의된 값을 반환하는 `FixedValueRevisionResolver`가 있습니다. `FixedValueRevisionResolver`를 사용하면, 현재 애플리케이션의 버전을 주입하여 특정 이벤트를 생성한 애플리케이션의 버전을 확인할 수 있습니다.

메이븐을 사용한다면, 프로젝트의 버전을 자동으로 사용하는 `MavenArtifactRevisionResolver`를 사용할 수 있습니다. `MavenArtifactRevisionResolver`는 버전 정보를 얻기 위해 프로젝트의 groupId와 artifactId를 사용하여 초기화됩니다. 이 방법은 메이븐을 사용하여 생성한 JAR 파일에만 적용할 수 있어서, 통합 개발 환경(IDE)이 항상 버전 정보를 파악할 수 없습니다. 만약 버전이 파악이 안 되면, `null`이 반환됩니다.

Axon의 업 캐스터들은 `EventMessage`에 직접 사용할 수 없지만, `IntermediateEventRepresentation`과 함께 사용할 수 있습니다. `IntermediateEventRepresentation`는 실제 업 캐스트(upcast) 기능들과 함께 `EventMessage`를 생성하기 위한 모든 필요한 항목들을 추출할 수 있는 기능을 제공합니다. (따라서 `EventMessage`또한 가능 합니다.) 업 캐스트 기능들은 기본적으로 이벤트의 페이로드, 페이로드 타입에 대한 조정과 이벤트의 메타 데이터의 추가만을 허용합니다. 업 캐스트 기능에서의 이벤트의 실제 표현은 사용된 직렬화 객체 혹은 원하는 연동 형식에 따라 달라질 수 있으므로, `IntermediateEventRepresentation`의 업 캐스트 기능을 통해 예상되는 표현 유형을 선택할 수 있습니다. 메시지, aggregate 식별자, aggregate 유형 그리고 타임스탬프 등과 같은 항목들은 `IntermediateEventRepresentation`에 의해 변경할 수 없습니다. 이런 항목들을 변경하는 것은 업 캐스터의 목적과 맞지 않기 때문에, `IntermediateEventRepresentation` 구현체를 통해 그런 항목들을 변경할 수 없습니다.

 Axon Framework에서의 기본 `Upcaster` 인터페이스는 `IntermediateEventRepresentation`의 `Stream`을 가지고 작동하며 `IntermediateEventRepresentation`의 `Stream`을 반환합니다. 따라서 업 캐스팅 프로세스는 업 캐스팅 기능의 최종 결과를 직접 반환하지는 않지만, `IntermediateEventRepresentation`를 스태킹(stacking)하여 하나의 수정 버전에서 다른 수정 버전까지 모든 업 캐스팅 기능을 체인으로 연결합니다. 이런 프로세스 과정을 거치고 최종 결과를 끌어내면, 실제 업 캐스팅 기능이 직렬화된 이벤트를 대상으로 수행된 것입니다.

추상 업 캐스터 구현체
------------------------------------------

이미 설명한 것처럼, `Upcaster` 인터페이스는 하나의 이벤트를 업 캐스트하지 않습니다. `Upcaster`는 하나의 `Stream`를 필요로 하며 `Stream`를 반환합니다. 하지만 업 캐스터는 보통 해당 스트림에서 단일 이벤트를 조정하기 위해 작성됩니다. 더 정교한 업 캐스팅 설정도 할 수 있습니다. 예를 들어, 하나의 이벤트에서 다수의 이벤트로 혹은 이전 이벤트에서 상태를 뽑아내어 이후에 발생한 이벤트에 밀어 넣을 수 있는 업 캐스터가 있을 수 있습니다. 이번 장은 원하는 추가 기능을 포함하는 업 캐스터를 구현하는데 사용할 수 있는 Axon에서 제공하는 이벤트 업 캐스터의 추상 구현체에 관해 설명합니다.

- `SingleEventUpcaster` - 이 업 캐스터는 이벤트 업 캐스터의 구현체 중 하나로, 추상 클래스 형태로 제공되므로 이 클래스를 상속받아 `canUpcast`와 `doUpcast` 메서드를 구현해야 합니다. `canUpcast` 메서드는 주어진 이벤트를 업 캐스트 할 수 있는지를 판단하는 메서드이고 `doUpcast` 메서드는 업 캐스트할 방법을 정의하는 메서드 입니다. 대부분의 이벤트 조정은 이벤트 자체에 포함된 데이터를 기반으로 일대일로 이루어지기 때문에, `SingleEventUpcaster`를 상속하여 업 캐스터를 구현하는 것이 가장 적합한 방법입니다.

- `EventMultiUpcaster` - 이 업 캐스터는 일대다를 위한 업 캐스터의 구현체로, `SingleEventUpcaster`와는 달리 `doUpcast` 메서드는 단일 `IntermediateEventRepresentation`` 대신 `Stream`을 반환합니다. `EventMultiUpcaster`를 통해, 단일 이벤트를 여러 개의 이벤트로 변환할 수 있습니다. 예를 들어, 복합적인 의미를 담고 있는 이벤트(fat event)를 여러 이벤트로 좀 더 세분하길 원할 때 유용하게 사용할 수 있습니다.

- `ContextAwareSingleEventUpcaster` - 일대일 변환을 위한 업 캐스터로, 변환 프로세스 동안 이벤트의 컨텍스트를 저장할 수 있습니다. `canUpcast`와 `doUpcast`외에, `ContextAwareSingleEventUpcaster`는 `buildContext` 메서드를 추가로 구현해야 합니다. `buildContext` 메서드는 컨텍스트를 생성하는 메서드로, 해당 컨텍스트는 업 캐스트를 거쳐 가는 이벤트들 간에 전달될 수 있습니다. `canUpcast`와 `doUpcast` 기능들은 `IntermediateEventRepresentation` 매개변수 다음으로 해당 컨텍스트를 두 번쩨 매개 변수로 받습니다. 컨텍스트는 이전 이벤트들에서 항목들을 뽑아내고 다음 이벤트를 채우기 위해 업 캐스팅 프로세스 내에서 사용될 수 있습니다. 즉, 컨텍스트를 통해, 하나의 이벤트내의 항목을 완전히 다른 이벤트로 전달할 수 있습니다.

- `ContextAwareEventMultiUpcaster` - 일대다를 위한 업 캐스터 구현체이며, 변환 프로세스 동안 이벤트의 컨텍스트를 저장할 수 있습니다. 이 구현체는 `EventMultiUpcaster`와 `ContextAwareSingleEventUpcaster`의 조합으로 만들어졌으며, `IntermediateEventRepresentations`의 컨텍스트를 유지할 수 있게 해주고 하나의 `IntermediateEventRepresentation`을 여러 개로 업 캐스팅 할 수 있게 해줍니다. 단순히 하나의 이벤트의 항목을 다른 이벤트로 복사하는 것이 아닌, 새로운 여러 이벤트들로 생성해야 할 때 사용할 수 있습니다.

업 캐스터 작성하기
-------------------
다음의 Java 코드들은 일대일 업 캐스터(`SingleEventUpcaster`)의 기본 예제 입니다.

이전 버전의 이벤트:
```java
@Revision("1.0")
public class ComplaintEvent {
    private String id;
    private String companyName;

    // 생성자 및 게터/세터
}
```

새로운 버전의 이벤트:
```java
@Revision("2.0")
public class ComplaintEvent {
    private String id;
    private String companyName;
    private String description; // 추가된 새로운 항목

    // 생성자 및 게터/세터
}
```

업 캐스터:
```java
// 1.0 버전의 이벤트를 2.0 버전의 이벤트로 업 캐스팅
public class ComplaintEventUpcaster extends SingleEventUpcaster {
    private static SimpleSerializedType targetType = new SimpleSerializedType(ComplainEvent.class.getTypeName(), "1.0");

    @Override
    protected boolean canUpcast(IntermediateEventRepresentation intermediateRepresentation) {
        return intermediateRepresentation.getType().equals(targetType);
    }

    @Override
    protected IntermediateEventRepresentation doUpcast(IntermediateEventRepresentation intermediateRepresentation) {
        return intermediateRepresentation.upcastPayload(
                new SimpleSerializedType(targetType.getName(), "2.0"),
                org.dom4j.Document.class,
                document -> {
                    document.getRootElement()
                            .addElement("description")
                            .setText("no complaint description"); // 기본값으로 설정
                    return document;
                }
        );
    }
}
```

스프링 부트 설정:
```java
@Configuration
public class AxonConfiguration {

    @Bean
    public SingleEventUpcaster myUpcaster() {
        return new ComplaintEventUpcaster();
    }

    @Bean
    public JpaEventStorageEngine eventStorageEngine(Serializer serializer,
                                                    DataSource dataSource,
                                                    SingleEventUpcaster myUpcaster,
                                                    EntityManagerProvider entityManagerProvider,
                                                    TransactionManager transactionManager) throws SQLException {
        return new JpaEventStorageEngine(serializer,
                myUpcaster::upcast,
                dataSource,
                entityManagerProvider,
                transactionManager);
    }
}
```

컨텐트 타입 변환
-----------------------

업 캐스터는 주어진 타입(예, dom4j Document)에 대해 작동하게 됩니다. 업 캐스터 간 추가적인 유연성을 향상 시키기 위해서, 서로 엮어있는(체인닝) 업 캐스터들 간의 컨텐트 타입들이 변할 수 있습니다. Axon은 `ContentTypeConverter`들을 사용하여 자동으로 컨텐트 타입 간의 변환을 합니다. Axon은 `x` 타입을 `y` 타입으로 변경하기 위한 가장 짧은 경로를 찾아, 변환을 수행하고 변경된 값을 요청된 업 캐스터로 전달합니다. 성능상의 이유로, 변환된 타입을 받을 업 캐스터의 `canUpcast`메서드가 참을 반환하는 경우에만 변환이 이루어집니다.

`ContentTypeConverter`들은 사용되는 직렬화 객체의 타입에 의존적 일 수 있습니다. `byte[]`를 dom4j의 `Document`로 변환을 시도하는 것은 사용된 `Serializer`가 이벤트를 XML로 쓰는 경우에만 수행될 수 있습니다. `UpcasterChain`이 직렬화 객체에 명시된 `ContentTypeConverter`들에 접근 권한을 가지게 하려면, `UpcasterChain`의 생성자에 직렬화 객체의 참조를 넘겨 주어야 합니다.

> **팁**
>
> 최고의 성능을 내기 위해선, 특정 업 캐스터의 결과가 다른 업 캐스터의 입력이 되는 같은 체인 내의 모든 업 캐스터들이 같은 컨텐트 타입에 대해 동작하도록 해야 합니다.

특정 컨텐트 타입을 Axon이 제공하는 `ContentTypeConverter`로 수행 할 수 없는 경우, 다른 경우와 마찬가지로 `ContentTypeConverter` 인터페이스를 직접 구현하여 사용할 수 있습니다.

`XStreamSerializer`는 Dom4j 뿐만 아니라 XOM을 XML 문서로 표현하는 것을 지원합니다. `JacksonSerializer`는 Jackson의 `JsonNode`를 지원합니다.

스냅샷 지정
============

Aggregate들이 오랫동안 살아 있으면서 상태가 계속 변경할 때, 해당 aggregate들은 많은 양의 이벤트들을 만들어 냅니다. Aggregate의 상태를 재구성하기 위해 이 모든 이벤트를 로딩해야 한다면, 성능상의 큰 문제를 일으킬 수 있습니다. 스냅숏 이벤트는 특정 목적을 가진 도메인 이벤트이며, 임의의 다수 이벤트를 요약하여 하나의 이벤트로 만들어 냅니다. 스냅숏 이벤트를 정기적으로 생성하고 저장하여, 이벤트 저장소가 많은 양의 이벤트들을 반환하지 않도록 합니다. 마지막 스냅숏 이벤트들과 스냅숏이 만들어진 후의 이벤트들만을 반환하면 됩니다.

예를 들어, 재고로 쌓여 있는 상품 수량은 자주 변경이 됩니다. 상품이 판매될 때마다, 상품 판매 이벤트는 재고량을 하나씩 줄여 갑니다. 새로운 상품들에 대한 적재가 이루어질 때마다, 재고량은 적재량만큼 증가하게 됩니다. 만약 하루에 100개의 상품을 판다고 하면, 매일 적어도 100개의 이벤트가 만들어지게 됩니다. 며칠이 지나면, 시스템은 단지 "ItemOutOfStockEvent"를 발생 시키기 위해 모든 이벤트를 읽어야 하고, 이로 인해 많은 시간이 소비됩니다. 하지만 하나의 스냅숏 이벤트가 이 많은 이벤트를 대체하게 되면, 단지 재고 상의 현재 상품 수만을 저장하면 됩니다.

스냅숏 생성
-------------------

스냅숏을 생성하는 것은 몇 가지 요소들을 통해 자동으로 이루어질 수 있습니다. 예를 들어, 마지막 스냅숏 이후 생성된 이벤트의 개수, 특정 기준점을 지난 aggregate을 초기화한 시간, 시간을 기반으로 한 방법 등이 있을 수 있습니다. 현재로는, Axon은 발생한 이벤트 개수를 기준으로 스냅숏을 생성할 수 있는 메커니즘을 제공합니다.

언제 스냅숏이 만들어져야 하는지에 대한 정의는 `SnapshotTriggerDefinition` 인터페이스를 통해 생성할 수 있습니다.

`EventCountSnapshotTriggerDefinition`은 이름처럼 aggregate를 로딩하는 데 필요한 이벤트의 개수가 특정 기준값을 초과할 때 스냅숏을 생성할 수 있는 메커니즘을 제공합니다. 만약 aggregate를 로딩하는 데 필요한 이벤트의 개수가 설정한 기준값을 초과하였다면, 트리거는 `Snapshotter`에게 해당 aggregate에 대한 스냅숏을 만들라고 말해 줍니다.

스냅숏 트리거는 이벤트를 저장하고 제공하는 레퍼지토리에 설정할 수 있고 `Snapshotter`와 `Trigger`와 같은 트리거링을 변경할 수 있도록 하는 속성들을 가지고 있습니다.

-   `Snapshotter`는 실제 스냅숏을 생성하는 snapshotter 인스턴스를 설정하며, 실제 스냅숏 이벤트를 생성하고 저장하는 책임을 지고 있습니다.

-   `Trigger`는 스냅숏 생성을 일으킬 기준값을 설정합니다.

Snapshotter는 실제 스냅숏을 생성하는 책임을 지고 있습니다. 일반적으로, 스냅숏을 생성하는 것은 최대한 운영 프로세스를 방해하지 말아야 하는 프로세스입니다. 따라서, 별도의 스레드에서 스냅숏을 생성하는 것을 권장합니다. `Snapshotter` 인터페이스는 단 하나의 메서드 `scheduleSnapshot()`를 선언하고 있으며, `scheduleSnapshot()`는 aggregate의 타입과 식별자를 매개변수로 받습니다.

Axon은 `AggregateSnapshotter`를 제공하며, `AggregateSnapshotter`는 `AggregateSnapshot` 인스턴스를 생성하고 저장합니다. `AggregateSnapshot`는 자신 내부에 실제 aggregate 인스턴스를 가지고 있는 특별한 형태의 스냅숏입니다. Axon이 제공하는 레퍼지토리들은 이와 같은 타입의 스냅숏을 인식하며, 새로운 aggregate 인스턴스를 만들지 않고 해당 스냅숏에서 aggregate를 추출할 수 있습니다. 스냅숏 이벤트 이후에 발생한 이벤트들은 로딩되어 추출된 aggregate 인스턴스로 스트리밍됩니다.

> **참고**
>
> 사용하고 있는 `Serializer` 인스턴스(기본값은 `XStreamSerializer`)가 정의한 aggregate를 직렬화할 수 있는지를 분명히 파악해야 합니다. `XStreamSerializer`를 사용한다면, Hotspot JVM을 사용하거나 혹은 정의한 aggregate이 접근 가능한 기본 생성자를 가지거나 `Serializable` 인터페이스를 구현해야 합니다.

`AbstractSnapshotter`는 스냅숏을 생성하는 방법을 변경(개조)할 수 있는 속성들, 예를 들어 `EventStore`와 `Executor`들과 같은 속성들을 제공합니다.

* `EventStore`를 통해, 지난 이벤트들을 가져오고 스냅숏을 저장하는데 사용되는 이벤트 저장소를 설정할 수 있습니다. 이벤트 저장소는 반드시 `SnapshotEventStore` 인터페이스를 구현해야 합니다.

* `Executor`는 `ThreadPoolExecutor`와 같은 Executor를 설정하며, 설정한 Executor를 통해 스냅숏 설정 프로세스를 실행할 스레드를 제공할 수 있습니다. 기본적으로, 스냅숏은 `scheduleSnapshot()`메서드를 호출하는 스레드 내에서 생성됩니다. 이 방법은 운영 환경에서 사용하지 않는 것이 좋습니다.

`AggregateSnapshotter`는 `AggregateFactories`라는 추가적인 속성 하나를 더 제공합니다.

* `AggregateFactories` 속성을 통해, aggregate 인스턴스를 생성할 수 있는 팩토리들을 설정할 수 있습니다. 다수의 aggregate 팩토리들을 설정한 후, 하나의 스냅숏 생성 객체를 사용하여 다양한 형태의 aggregate들에 대한 스냅숏을 생성할 수 있습니다. `EventSourcingRepository` 구현체들은 자신들이 사용하는 `AggregateFactory`에 접근할 수 있도록 허용해주기 때문에, 레퍼지토리에서 사용하는 aggregate 팩토리 객체들을 snapshotter에서도 사용할 수 있도록 설정할 수 있습니다.

> **참고**
>
> 다른 스레드에서 스냅숏을 생성하기 위해 executor를 사용할 때, 필요한 경우에 기반이 되는 이벤트 저장소를 위한 정확한 트랜잭션 관리자를 설정해야 합니다.
>
> 스프링을 사용하는 경우, `SpringAggregateSnapshotter`를 사용할 수 있습니다. `SpringAggregateSnapshotter`는 스냅숏을 생성할 필요가 있을 때, 애플리케이션 컨텍스트에서 스스로 필요한 `AggregateFactory`를 찾아냅니다.

스냅숏 이벤트 저장하기
-----------------------
스냅숏이 이벤트 저장소에 저장되면, 자동으로 해당 스냅숏을 사용하여 이전의 모든 이벤트를 요약하고 원래의 위치로 되돌립니다. 모든 이벤트 저장소 구현체들은 동시에 스냅숏들을 생성할 수 있게 합니다. 즉, 다른 프로세스가 같은 aggregate에 이벤트를 추가하는 동안, 스냅숏을 저장할 수 있습니다. 다시 말해, 스냅숏을 저장하는 프로세스는 별도의 프로세스로 다른 프로세스와 함께 실행됩니다.

> **참고**
>
> 보통, 스냅숏 이벤트의 일부분인 경우, 모든 이벤트를 저장할 수 있습니다. 이벤트 저장소는 일반적으로 이벤트를 다시 읽어 들이는 것처럼 저장된 이벤트들을 다시 읽을 수는 없습니다. 하지만, 스냅숏이 생성되기 이전의 aggregate의 상태로 복구하려면, 항상 최신의 이벤트를 유지해야 합니다.

Axon은 `AggregateSnapshot`이라는 특별한 형태의 스냅숏 이벤트를 제공합니다. `AggregateSnapshot`은 전체 aggregate를 하나의 스냅숏으로 저장합니다. 이렇게 하는 이유는 간단한데, 정의한 aggregate는 사업적인 결정을 내리는데 필요한 상태만을 가져야 하기 때문입니다. 이것이 바로 스냅숏으로 캡쳐하여 저장해야 할 정보입니다. Axon에서 제공하는 모든 이벤트 소싱 레퍼지토리들은 `AggregateSnapshot`을 인식할 수 있으며, aggregate를 추출할 수 있습니다. 이 스냅숏 이벤트를 사용하려면, 이벤트 직렬화 메커니즘을 통해 aggregate를 직렬화할 수 있어야 한다는 점에 유의하세요.

스냅숏 이벤트로 Aggregate 초기화하기
---------------------------------------------------

스냅숏 이벤트는 다른 이벤트들과 비슷합니다. 즉, 스냅숏 이벤트 또한 다른 도메인 이벤트들과 비슷한 방식으로 처리할 수 있습니다. 각 이벤트를 처리할 수 있는 처리자(handler)를 분명히 나누기 위해 에노테이션(```@EventHandler```)을 사용할 때, 스냅숏 이벤트를 가지고 전체 aggregate의 상태를 초기화할 수 있는 메서드에 에노테이션을 추가할 수 있습니다. 아래의 예제 코드를 통해 다른 도메인 이벤트를 다루는 것과 같이 스냅숏 이벤트를 다루는 방법을 살펴볼 수 있습니다.

``` java
public class MyAggregate extends AbstractAnnotatedAggregateRoot {

    // ... 간략한 코드를 위해 생략된 부분입니다.

    @EventHandler
    protected void handleSomeStateChangeEvent(MyDomainEvent event) {
        // ...
    }

    @EventHandler
    protected void applySnapshot(MySnapshotEvent event) {
        // 스냅숏 이벤트는 관련된 모든 상태를 포함하고 있어야 합니다.
        this.someState = event.someState;
        this.otherState = event.otherState;
    }
}                
```

`AggregateSnapshot`타입의 이벤트들은 다른 이벤트들과는 다르게 처리해야 합니다. 해당 유형의 스냅숏 이벤트는 실제 aggregate를 포함하고 있습니다. aggregate 팩토리는 `AggregateSnapshot` 타입의 이벤트를 인식하고 해당 스냅숏으로부터 aggregate를 뽑아낼 수 있습니다. 그런 다음 다른 모든 이벤트가 추출된 스냅숏에 다시 적용됩니다. 즉, aggregate들은 `AggregateSnapshot` 인스턴스들을 직접 다룰 필요가 없습니다.

충돌 탐지 및 해결을 위한 향상된 방법
==========================================

변경의 의미에 대해 명시적으로 표현하는 것의 주요 이점 중 하나는 충돌을 일으키는 변경 사항을 보다 정확하게 발견할 수 있다는 점입니다. 일반적으로, 이런 충돌이 나는 변경 사항들은 두 사용자가 같은 데이터에 (거의) 동시에 어떤 작업을 할 때 발생합니다. 두 사용자 모두 특정 버전의 데이터를 본다고 생각해보세요. 그리고 두 사용자 모두 해당 데이터를 바꾸려고 합니다. 그들 모두 데이터를 변경하기 위해 "버전 X에 해당하는 aggregate의 상태를 바꿔"와 같은 명령을 보낼 것입니다. 이때 X는 aggregate의 특정 버전을 말합니다. 그들 중 한 명은 원하는 대로 해당 버전에 적용된 변경 사항을 가질 것이고, 다른 한 명은 그렇지 못할 것입니다.

다른 프로세스에 의해 이미 aggregate가 변경된 경우, 모든 들어오는 명령들을 거부하는 대신, 사용자들의 의도와 보이지 않는 변경 사항들이 충돌하는지를 확인할 수 있습니다.

충돌 사항을 발견하기 위해선, 정의한 aggregate 내에 `@CommandHandler` 에노테이션이 사용된 메서드로 `ConflictResolver`를 매개 변수로 전달해야 합니다. `ConflictResolver`인터페이스는 `detectConflicts` 메서드를 선언하고 있으며, 해당 메서드를 통해 특정 타입의 명령을 수행할 때 충돌을 일으키는 이벤트들을 정의할 수 있습니다.

> **참고**
>
> 특정 기대 버전의 Aggregate가 로딩되었을 때, 잠재적으로 충돌을 일으키는 이벤트들만이 `ConflictResolver`에 포함됩니다. 특정 기대 버전의 Aggregate를 지정하기 위해서 `@TargetAggregateVersion` 에노테이션을 명령 객체의 특정 항목에 사용해야 합니다.

predicate를 통해 발견된 이벤트가 있다면, 예외가 발생합니다. 발생시킬 예외를 명시하려면, `detectConflicts` 메서드에 선택적으로 줄 수 있는 두 번째 매개 변수에 발생시키고자 하는 예외를 정의하면 됩니다. 아무런 이벤트도 발견되지 않는다면, 프로세스는 보통 때와 같이 진행됩니다.

`detectConflicts` 메서드 호출이 이루어지지 않지만, 잠재적인 충돌 이벤트들이 있다면, `@CommandHandler`는 실패하게 됩니다. 이런 경우는 기대 버전의 Aggregate가 제공되었지만, 사용 가능한 `ConflictResolver`가 `@CommandHandler`메서드의 매개변수에 없는 경우 일 수 있습니다. 
