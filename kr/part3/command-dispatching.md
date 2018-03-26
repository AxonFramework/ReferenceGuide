명령 전달 하기
===================

명시적인 명령 전달 메커니즘을 사용하면 좋은 점들이 있습니다. 그 중 첫 번째는 클라이언트의 의도를 명확히 묘사하는 단일 객체가 있다는 것입니다. 명령을 로깅 하여, 클라이언트의 의도와 관련 데이터들을 나중에 사용할 목적으로 저장할 수 있습니다. 명령 처리를 통해 예를 들어 웹 서비스 등을 통해 원격의 클라이언트에게 명령 처리 컴포넌트를 쉽게 노출 시킬 수 있습니다. 시작 상황(given), 실행할 명령(when) 그리고 기대 결과(then)로 이벤트들과 명령들을 열거하는 형식으로 테스트 스크립트를 정의할 수 있어서 테스트를 더 쉽게 수행할 수 있습니다([테스트](../part2/testing.md)를 참고하세요). 마지막 주요 이점은 동기와 비동기 간 변경뿐만 아니라 로컬 기반 명령 처리를 분산 환경의 명령 처리로의 변경을 매우 쉽게 처리할 수 있다는 것입니다.

This doesn't mean Command dispatching using explicit command object is the only way to do it. The goal of Axon is not to prescribe a specific way of working, but to support you doing it your way, while providing best practices as the default behavior. It is still possible to use a Service layer that you can invoke to execute commands. The method will just need to start a unit of work (see [Unit of Work](../part1/messaging-concepts.md#unit-of-work)) and perform a commit or rollback on it when the method is finished.

명시적인 명령 객체를 사용한 명령 전달 처리만이 위와 같은 처리를 하는데 유일한 방법은 아닙니다. Axon의 목적은 특정 방법을 규정하는 것이 아니라 기본적 구현으로서 모범 사례를 제공하면서 사용자 나름의 방법을 지원하는 것입니다. 따라서 명령 실행을 위한 서비스 계층(layer)을 사용할 수 있으며 서비스 계층의 메서드는 작업 단위를 시작하고([작업 단위](../part1/messaging-concepts.md#unit-of-work)를 참고하세요) 메서드의 종료에 따라서 커밋 혹은 롤백을 수행합니다.

다음 장에서는, Axon Framework을 사용하여 명령 전달 기반 구조의 구성과 관련된 작업의 개요를 살펴볼 것입니다.

커멘드 게이트웨이
===================

커멘드 게이트웨이는 명령 전달 메커니즘에 대한 인터페이스입니다. 명령을 전달하기 위해 게이트웨이를 사용할 필요는 없지만, 일반적으로 게이트웨이를 사용하는 것이 가장 쉬운 방법입니다.

커멘드 게이트웨이를 사용하기 위한 두 가지 방법 중, 첫 번째 방법은 Axon에서 제공하는 `CommandGateway` 인터페이스와 `DefaultCommandGateway` 구현체를 사용하는 것입니다. Command Gateway를 통해 명령을 전송하고 결과를 동기적으로 기다리도록 처리할 수 있고, 시간제한 및 비동기적으로도 처리할 수 있습니다.

다음으로 사용할 수 있는 가장 유연한 방법으로, `CommandGateWayFactory`를 사용하여 거의 모든 인터페이스를 Command Gateway로 변경하는 방법입니다. 정확한 타입 정보와 비즈니스 관련 예외를 사용하여 정의된 애플리케이션 인터페이스를 정의하여, Axon을 통해 해당 인터페이스를 런타임(실행 시점)에 자동으로 해당 인터페이스에 대한 구현체를 생성할 수 있습니다.

커멘드 게이트웨이 설정하기
-------------------------------

사용자 정의 게이트웨이와 Axon에서 제공되는 게이트웨이 모두 커맨드 버스(command bus)에 접근할 수 있도록 설정이 필요합니다. 또한, 커맨드 게이트웨이는 `RetryScheduler`, `CommandDispatchInterceptor` 그리고 `CommandCallback`을 통한 설정을 해야 합니다.

`RetryScheduler`을 통해 명령 실행이 실패했을 경우, 명령 처리에 대한 재시도에 대한 스케쥴링을 할 수 있습니다. `RetryScheduler`의 구현체인 `IntervalRetryScheduler`를 통해, 명령 처리가 성공할 때까지 일정한 간격으로 주어진 명령 처리를 재시도하거나, 최대 재시도 횟수만큼 명령을 처리를 재차 시도할 수 있습니다. 하지만 일시적인 예외가 아닌 예외 사항으로 명령 처리에 실패한 경우에는 재시도는 이루어지지 않습니다. `RetryScheduler`는 명령 처리가 `RuntimeException`으로 실패한 경우에만 호출됩니다. 확인된 예외(Checked exceptions)는 "비즈니스 예외"로 간주하여 절대 재시도를 하지 않습니다. 분산 환경에서 커맨드 버스를 사용하여 명령 처리를 할 때 주도 `RetryScheduler`를 사용합니다. 한 노드가 실패했을 때, `RetryScheduler`를 사용하여 다른 사용 가능한 노드로 해당 명령을 분산하여 명령이 처리될 수 있도록 조치합니다. (참조: [분산 커맨드 버스](#distributing-the-command-bus))

`CommandDispatchInterceptor`를 통해, 커맨드 버스로 명령이 전달되기 전에 명령 메세지(`CommandMessage`)를 변경할 수 있습니다. `CommandDispatchInterceptor`는 `CommandBus`에 설정이 되지만, 메시지들이 게이트웨이를 통해서 전달되는 경우에만 설정된 인터셉터들은 호출됩니다. 예를 들어, 인터셉터들을 사용하여 메타 데이터를 추가하거나 검증을 수행할 수 있습니다.

`CommandCallback`은 각각 전송된 명령에 대해 호출이 됩니다. 명령의 타입에 상관없이, 게이트웨이를 통해 전송되는 모든 명령에 대해 일반적인 기능을 수행할 수 있습니다.

사용자 정의 Command Gateway 생성하기
---------------------------------

Command Gateway를 재정의한 인터페이스를 사용할 수 있습니다. 인터페이스에 정의된 각각의 메서드들은 매개변수, 반환 타입 그리고 선언된 예외 사항으로 어떤 기능을 하는지 알 수 있습니다. 인터페이스로 선언된 게이트웨이를 사용하면 사용의 간편성뿐만 아니라 mock을 사용하여 테스트를 더 쉽게 진행할 수 있도록 해줍니다.

다음을 통해 매개변수들이 Command Gateway의 기능(행위)에 어떤 영향을 미치는지 살펴보겠습니다.

-   Command Gateway의 `send`, `sendAndWait` 메서드들의, 첫 번째 매개변수는 전달될 실제 명령 객체이어야 합니다.

-   `@MetadataValue` 에노테이션이 사용된 매개변수의 값은 에노테이션 매개변수로 전달된 식별자를 통해 메타데이터 필드에 할당이 됩니다.

-   `MetaData` 타입의 매개변수는 커멘드 메시지(Command Message)의 `MetaData`와 합쳐지게 되며, 같은 키를 가지는 항목은 이후에 정의된 메타 데이터 항목으로 덮어쓰게 됩니다.

-   `CommandCallback` 타입의 매개변수는 명령 처리 이후 호출되는 `onSuccess` 혹은 `onFailure` 메서드를 가집니다. 하나 이상의 콜백을 전달할 수 있으며, 처리된 명령과 결과 값을 인자로 콜백 객체에 전달하여 호출합니다.

-   마지막 두 개의 매개변수들은 각각 `long`(혹은 `int`) 그리고 `TimeUnit` 타입일 것입니다. 이 경우, 매개변수들이 나타내는 시간 동안 해당 메서드는 블럭킹되어 메서드가 완료될 때까지 대기합니다. 타임아웃 발생에 대한 반응 및 처리는 메서드에 선언된 예외에 따라 달라집니다. 메서드의 다른 속성들이 블럭킹을 방지하게 되면, 타임아웃은 절대 발생하지 않습니다.

메서드에 선언된 반환 타입 또한 아래와 같이 메서드의 행위에 영향을 미칩니다.

- `Void`반환 타입은 해당 메서드에 타임아웃 혹은 선언된 예외 사항 같은 대기해야 한다는 지시가 없다면, 해당 메서드가 즉시 반환하도록 합니다.

- `Future`, `CompletionStage` 그리고 `CompletableFuture`의 반환 타입은 메서드가 즉시 반환되도록 합니다. 메서드가 반환하는 `CompletableFuture` 인스턴스를 통해 명령 처리자의 결과에 접근할 수 있습니다. 메서드에 선언된 예외나 타임아웃은 무시됩니다.

- 다른 반환 타입들을 사용하면, 해다 메서드는 사용 가능한 결과가 나올 때까지 블럭킹 되어 결과를 기다리게 됩니다. 결괏값은 반환 타입으로 캐스팅되며, 타입이 맞지 않으면 `ClassCastException`이 발생하게 됩니다.

메서드에 선언된 예외 사항들은 아래와 같은 영향을 미칩니다.

- 명령 처리자(혹은 인터셉터)에서 확인된 예외(checked exception)를 던진다면, 같은 예외가 해당 메서드에서도 발생 될 것입니다. 만약 해당 예외를 선언하지 않는다면 발생한 예외는 `RuntimeException`인 `CommandExecutionException`로 감싸져서 발생하게 됩니다.

- 타임아웃이 발생하면, 해당 메서드는 `null`을 기본으로 반환합니다. `null`을 반환하는 대신, `TimeoutException`을 선언하여 `TimeoutException`을 대신 던지도록 할 수 있습니다.

- 결과를 기다리는 동안 대기 중인 스레드가 중단(interrupted)된다면, 해당 메서드는 기본적으로 `null`을 반환합니다. 이 경우, 스레드가 중단된 것을 나타내는 값(interrupted flag)을 해당 스레드에 설정합니다. 해당 메서드에 `InterruptedException`을 선언하여, interrupted flag가 설정되는 것 대신 `InterruptedException` 예외를 발생시킬 수 있습니다. 예외가 발생하였을 경우, interrupted flag 값은 Java specification에 따라 삭제됩니다.

- 다른 런 타임 예외들을 메서드에 선언하면, API를 사용하는 사용자에게 설명해 주는 것 외에 다른 효과는 없습니다.

마지막으로, 에노테이션들을 사용할 수 있습니다.

- `@MetaDataValue` 에노테이션을 사용하여 메타 데이터값들을 메서드의 인자로 받아 볼 수 있습니다. 메타 데이터의 키값은 에노테이션의 매개변수로 선언하여 사용합니다.

- `@Timeout` 에노테이션을 메서드에 사용하면, 최대 에노테이션에 지정된 시간만큼 메서드는 블럭킹되어 결과를 기다리게 됩니다. 단 메서드에 타임아웃 매개변수를 선언하면 이 에노테이션은 무시됩니다.

- `@Timeout`을 클래스에 사용하면, 해당 클래스에 선언된 모든 메서드들에 해당 에노테이션이 적용됩니다. 따라서 클래스의 모든 메서드들은 최대 에노테이션에 지정된 시간만큼 메서드는 블럭킹되어 결과를 기다리게 됩니다. 단 메서드에 선언된 `@Timeout` 에노테이션이 우선 적용됩니다.

``` java
public interface MyGateway {

    // 전송하고 잊어버리는 전략입니다.
    void sendCommand(MyPayloadType command);

    // "userId"라는 메터 데이터를 가지고 10초의 타임아웃을 가지는 메서드입니다.
    @Timeout(value = 10, unit = TimeUnit.SECONDS)
    ReturnValue sendCommandAndWaitForAResult(MyPayloadType command,
                                             @MetaDataValue("userId") String userId);

    // 타임아웃이 발생했을때, 예외를 던지는 메서드입니다.
    @Timeout(value = 20, unit = TimeUnit.SECONDS)
    ReturnValue sendCommandAndWaitForAResult(MyPayloadType command)
                         throws TimeoutException, InterruptedException;

    // 아래의 메서드를 호출하는 client가 timeout 값을 지정합니다.
    void sendCommandAndWait(MyPayloadType command, long timeout, TimeUnit unit)
                         throws TimeoutException, InterruptedException;
}

// 게이트웨이 설정:
CommandGatewayFactory factory = new CommandGatewayFactory(commandBus);
// commandBus는 'configurer.buildConfiguration()' 메서드가 반환하는 Configuration객체를 통해 얻을 수 있습니다.
MyGateway myGateway = factory.createGateway(MyGateway.class);
```

The Command Bus
===============

The Command Bus is the mechanism that dispatches commands to their respective Command Handlers. Each Command is always sent to exactly one command handler. If no command handler is available for the dispatched command, a `NoHandlerForCommandException` exception is thrown. Subscribing multiple command handlers to the same command type will result in subscriptions replacing each other. In that case, the last subscription wins.

Dispatching commands
--------------------

The CommandBus provides two methods to dispatch commands to their respective handler: `dispatch(commandMessage, callback)` and `dispatch(commandMessage)`. The first parameter is a message containing the actual command to dispatch. The optional second parameter takes a callback that allows the dispatching component to be notified when command handling is completed. This callback has two methods: `onSuccess()` and `onFailure()`, which are called when command handling returned normally, or when it threw an exception, respectively.

The calling component may not assume that the callback is invoked in the same thread that dispatched the command. If the calling thread depends on the result before continuing, you can use the `FutureCallback`. It is a combination of a `Future` (as defined in the java.concurrent package) and Axon's `CommandCallback`. Alternatively, consider using a Command Gateway.

If an application isn't directly interested in the outcome of a Command, the `dispatch(commandMessage)` method can be used.

SimpleCommandBus
----------------

The `SimpleCommandBus` is, as the name suggests, the simplest implementation. It does straightforward processing of commands in the thread that dispatches them. After a command is processed, the modified aggregate(s) are saved and generated events are published in that same thread. In most scenarios, such as web applications, this implementation will suit your needs. The `SimpleCommandBus` is the implementation used by default in the Configuration API.

Like most `CommandBus` implementations, the `SimpleCommandBus` allows interceptors to be configured. `CommandDispatchInterceptor`s are invoked when a command is dispatched on the Command Bus. The `CommandHandlerInterceptor`s are invoked before the actual command handler method is, allowing you to do modify or block the command. See [Command Interceptors](#command-interceptors) for more information.

Since all command processing is done in the same thread, this implementation is limited to the JVM's boundaries. The performance of this implementation is good, but not extraordinary. To cross JVM boundaries, or to get the most out of your CPU cycles, check out the other `CommandBus` implementations.

AsynchronousCommandBus
----------------------

As the name suggest, the `AsynchronousCommandBus` implementation executes Commands asynchronously from the thread that dispatches them. It uses an Executor to perform the actual handling logic on a different Thread.

By default, the `AsynchronousCommandBus` uses an unbounded cached thread pool. This means a thread is created when a Command is dispatched. Threads that have finished processing a Command are reused for new commands. Threads are stopped if they haven't processed a command for 60 seconds.

Alternatively, an `Executor` instance may be provided to configure a different threading strategy.

Note that the `AsynchronousCommandBus` should be shut down when stopping the application, to make sure any waiting threads are properly shut down. To shut down, call the `shutdown()` method. This will also shutdown any provided `Executor` instance, if it implements the `ExecutorService` interface.

DisruptorCommandBus
-------------------

The `SimpleCommandBus` has reasonable performance characteristics, especially when you've gone through the performance tips in [Performance Tuning](../part4/performance-tuning.md#performance-tuning). The fact that the `SimpleCommandBus` needs locking to prevent multiple threads from concurrently accessing the same aggregate causes processing overhead and lock contention.

The `DisruptorCommandBus` takes a different approach to multithreaded processing. Instead of having multiple threads each doing the same process, there are multiple threads, each taking care of a piece of the process. The `DisruptorCommandBus` uses the Disruptor (<http://lmax-exchange.github.io/disruptor/>), a small framework for concurrent programming, to achieve much better performance, by just taking a different approach to multi-threading. Instead of doing the processing in the calling thread, the tasks are handed off to two groups of threads, that each take care of a part of the processing. The first group of threads will execute the command handler, changing an aggregate's state. The second group will store and publish the events to the Event Store.

While the `DisruptorCommandBus` easily outperforms the `SimpleCommandBus` by a factor of 4(!), there are a few limitations:

-   The `DisruptorCommandBus` only supports Event Sourced Aggregates. This Command Bus also acts as a Repository for the aggregates processed by the Disruptor. To get a reference to the Repository, use `createRepository(AggregateFactory)`.

-   A Command can only result in a state change in a single aggregate instance.

-   When using a Cache, it allows only a single aggregate for a given identifier. This means it is not possible to have two aggregates of different types with the same identifier.

-   Commands should generally not cause a failure that requires a rollback of the Unit of Work. When a rollback occurs, the `DisruptorCommandBus` cannot guarantee that Commands are processed in the order they were dispatched. Furthermore, it requires a retry of a number of other commands, causing unnecessary computations.

-   When creating a new Aggregate Instance, commands updating that created instance may not all happen in the exact order as provided. Once the aggregate is created, all commands will be executed exactly in the order they were dispatched. To ensure the order, use a callback on the creating command to wait for the aggregate being created. It shouldn't take more than a few milliseconds.

To construct a `DisruptorCommandBus` instance, you need an `EventStore`. This component is explained in [Repositories and Event Stores](repositories-and-event-stores.md).

Optionally, you can provide a `DisruptorConfiguration` instance, which allows you to tweak the configuration to optimize performance for your specific environment:

-   Buffer size: The number of slots on the ring buffer to register incoming commands. Higher values may increase throughput, but also cause higher latency. Must always be a power of 2. Defaults to 4096.

-   ProducerType: Indicates whether the entries are produced by a single thread, or multiple. Defaults to multiple.

-   WaitStrategy: The strategy to use when the processor threads (the three threads taking care of the actual processing) need to wait for each other. The best WaitStrategy depends on the number of cores available in the machine, and the number of other processes running. If low latency is crucial, and the DisruptorCommandBus may claim cores for itself, you can use the `BusySpinWaitStrategy`. To make the Command Bus claim less of the CPU and allow other threads to do processing, use the `YieldingWaitStrategy`. Finally, you can use the `SleepingWaitStrategy` and `BlockingWaitStrategy` to allow other processes a fair share of CPU. The latter is suitable if the Command Bus is not expected to be processing full-time. Defaults to the `BlockingWaitStrategy`.

-   Executor: Sets the Executor that provides the Threads for the `DisruptorCommandBus`. This executor must be able to provide at least 4 threads. 3 of the threads are claimed by the processing components of the `DisruptorCommandBus`. Extra threads are used to invoke callbacks and to schedule retries in case an Aggregate's state is detected to be corrupt. Defaults to a `CachedThreadPool` that provides threads from a thread group called "DisruptorCommandBus".

-   TransactionManager: Defines the Transaction Manager that should ensure that the storage and publication of events are executed transactionally.

-   InvokerInterceptors: Defines the `CommandHandlerInterceptor`s that are to be used in the invocation process. This is the process that calls the actual Command Handler method.

-   PublisherInterceptors: Defines the `CommandHandlerInterceptor`s that are to be used in the publication process. This is the process that stores and publishes the generated events.

-   RollbackConfiguration: Defines on which Exceptions a Unit of Work should be rolled back. Defaults to a configuration that rolls back on unchecked exceptions.

-   RescheduleCommandsOnCorruptState: Indicates whether Commands that have been executed against an Aggregate that has been corrupted (e.g. because a Unit of Work was rolled back) should be rescheduled. If `false` the callback's `onFailure()` method will be invoked. If `true` (the default), the command will be rescheduled instead.

-   CoolingDownPeriod: Sets the number of seconds to wait to make sure all commands are processed. During the cooling down period, no new commands are accepted, but existing commands are processed, and rescheduled when necessary. The cooling down period ensures that threads are available for rescheduling the commands and calling callbacks. Defaults to 1000 (1 second).

-   Cache: Sets the cache that stores aggregate instances that have been reconstructed from the Event Store. The cache is used to store aggregate instances that are not in active use by the disruptor.

-   InvokerThreadCount: The number of threads to assign to the invocation of command handlers. A good starting point is half the number of cores in the machine.

-   PublisherThreadCount: The number of threads to use to publish events. A good starting point is half the number of cores, and could be increased if a lot of time is spent on IO.

-   SerializerThreadCount: The number of threads to use to pre-serialize events. This defaults to 1, but is ignored if no serializer is configured.

-   Serializer: The serializer to perform pre-serialization with. When a serializer is configured, the `DisruptorCommandBus` will wrap all generated events in a `SerializationAware` message. The serialized form of the payload and meta data is attached before they are published to the Event Store.

Command Interceptors
====================

One of the advantages of using a command bus is the ability to undertake action based on all incoming commands. Examples are logging or authentication, which you might want to do regardless of the type of command. This is done using Interceptors.

There are different types of interceptors: Dispatch Interceptors and Handler Interceptors. Dispatch Interceptors are invoked before a command is dispatched to a Command Handler. At that point, it may not even be sure that any handler exists for that command. Handler Interceptors are invoked just before the Command Handler is invoked.

Message Dispatch Interceptors
-----------------------------

Message Dispatch Interceptors are invoked when a command is dispatched on a Command Bus. They have the ability to alter the Command Message, by adding Meta Data, for example, or block the command by throwing an Exception. These interceptors are always invoked on the thread that dispatches the Command.

### Structural validation

There is no point in processing a command if it does not contain all required information in the correct format. In fact, a command that lacks information should be blocked as early as possible, preferably even before any transaction is started. Therefore, an interceptor should check all incoming commands for the availability of such information. This is called structural validation.

Axon Framework has support for JSR 303 Bean Validation based validation. This allows you to annotate the fields on commands with annotations like `@NotEmpty` and `@Pattern`. You need to include a JSR 303 implementation (such as Hibernate-Validator) on your classpath. Then, configure a `BeanValidationInterceptor` on your Command Bus, and it will automatically find and configure your validator implementation. While it uses sensible defaults, you can fine-tune it to your specific needs.

> **Tip**
>
> You want to spend as few resources on an invalid command as possible. Therefore, this interceptor is generally placed in the very front of the interceptor chain. In some cases, a Logging or Auditing interceptor might need to be placed in front, with the validating interceptor immediately following it.

The BeanValidationInterceptor also implements `MessageHandlerInterceptor`, allowing you to configure it as a Handler Interceptor as well.

Message Handler Interceptors
----------------------------

Message Handler Interceptors can take action both before and after command processing. Interceptors can even block command processing altogether, for example for security reasons.

Interceptors must implement the `MessageHandlerInterceptor` interface. This interface declares one method, `handle`, that takes three parameters: the command message, the current `UnitOfWork` and an `InterceptorChain`. The `InterceptorChain` is used to continue the dispatching process.

Unlike Dispatch Interceptors, Handler Interceptors are invoked in the context of the Command Handler. That means they can attach correlation data based on the Message being handled to the Unit of Work, for example. This correlation data will then be attached to messages being created in the context of that Unit of Work.

Handler Interceptors are also typically used to manage transactions around the handling of a command. To do so, register a `TransactionManagingInterceptor`, which in turn is configured with a `TransactionManager` to start and commit (or roll back) the actual transaction.

Distributing the Command Bus
============================

The CommandBus implementations described in earlier only allow Command Messages to be dispatched within a single JVM. Sometimes, you want multiple instances of Command Buses in different JVMs to act as one. Commands dispatched on one JVM's Command Bus should be seamlessly transported to a Command Handler in another JVM while sending back any results.

That's where the `DistributedCommandBus` comes in. Unlike the other `CommandBus` implementations, the `DistributedCommandBus` does not invoke any handlers at all. All it does is form a "bridge" between Command Bus implementations on different JVM's. Each instance of the `DistributedCommandBus` on each JVM is called a "Segment".

![Structure of the Distributed Command Bus](distributed-command-bus.png)

> **Note**
>
> While the distributed command bus itself is part of the Axon Framework Core module, it requires components that you can find in one of the *axon-distributed-commandbus-...* modules. If you use Maven, make sure you have the appropriate dependencies set. The groupId and version are identical to those of the Core module.

The `DistributedCommandBus` relies on two components: a `CommandBusConnector`, which implements the communication protocol between the JVM's, and the `CommandRouter`, which chooses a destination for each incoming Command. This Router defines which segment of the Distributed Command Bus should be given a Command, based on a Routing Key calculated by a Routing Strategy. Two commands with the same Routing Key will always be routed to the same segment, as long as there is no change in the number and configuration of the segments. Generally, the identifier of the targeted aggregate is used as a routing key.

Two implementations of the `RoutingStrategy` are provided: the `MetaDataRoutingStrategy`, which uses a Meta Data property in the Command Message to find the routing key, and the `AnnotationRoutingStrategy`, which uses the `@TargetAggregateIdentifier` annotation on the Command Messages payload to extract the Routing Key. Obviously, you can also provide your own implementation.

By default, the RoutingStrategy implementations will throw an exception when no key can be resolved from a Command Message. This behavior can be altered by providing a UnresolvedRoutingKeyPolicy in the constructor of the MetaDataRoutingStrategy or AnnotationRoutingStrategy. There are three possible policies:

-   ERROR: This is the default, and will cause an exception to be thrown when a Routing Key is not available

-   RANDOM\_KEY: Will return a random value when a Routing Key cannot be resolved from the Command Message. This effectively means that those commands will be routed to a random segment of the Command Bus.

-   STATIC\_KEY: Will return a static key (being "unresolved") for unresolved Routing Keys. This effectively means that all those commands will be routed to the same segment, as long as the configuration of segments does not change.

JGroupsConnector
----------------

The `JGroupsConnector` uses (as the name already gives away) JGroups as the underlying discovery and dispatching mechanism. Describing the feature set of JGroups is a bit too much for this reference guide, so please refer to the [JGroups User Guide](http://www.jgroups.org/ug.html) for more details.

Since JGroups handles both discovery of nodes and the communication between them, the `JGroupsConnector` acts as both a `CommandBusConnector` and a `CommandRouter`.

> **Note**
>
> You can find the JGroups specific components for the `DistributedCommandBus` in the `axon-distributed-commandbus-jgroups` module.

The JGroupsConnector has four mandatory configuration elements:

-   The first is a JChannel, which defines the JGroups protocol stack. Generally, a JChannel is constructed with a reference to a JGroups configuration file. JGroups comes with a number of default configurations which can be used as a basis for your own configuration. Do keep in mind that IP Multicast generally doesn't work in Cloud Services, like Amazon. TCP Gossip is generally a good start in such type of environment.

-   The Cluster Name defines the name of the Cluster that each segment should register to. Segments with the same Cluster Name will eventually detect each other and dispatch Commands among each other.

-   A "local segment" is the Command Bus implementation that dispatches Commands destined for the local JVM. These commands may have been dispatched by instances on other JVMs or from the local one.

-   Finally, the Serializer is used to serialize command messages before they are sent over the wire.

> **Note**
>
> When using a Cache, it should be cleared out when the `ConsistentHash` changes to avoid potential data corruption (e.g. when commands don't specify a `@TargetAggregateVersion` and a new member quickly joins and leaves the JGroup, modifying the aggregate while it's still cached elsewhere.)

Ultimately, the JGroupsConnector needs to actually connect, in order to dispatch Messages to other segments. To do so, call the `connect()` method.

``` java
JChannel channel = new JChannel("path/to/channel/config.xml");
CommandBus localSegment = new SimpleCommandBus();
Serializer serializer = new XStreamSerializer();

JGroupsConnector connector = new JGroupsConnector(channel, "myCommandBus", localSegment, serializer);
DistributedCommandBus commandBus = new DistributedCommandBus(connector, connector);

// on one node:
commandBus.subscribe(CommandType.class.getName(), handler);
connector.connect();

// on another node, with more CPU:
commandBus.subscribe(CommandType.class.getName(), handler);
commandBus.subscribe(AnotherCommandType.class.getName(), handler2);
commandBus.updateLoadFactor(150); // defaults to 100
connector.connect();

// from now on, just deal with commandBus as if it is local...
```

> **Note**
>
> Note that it is not required that all segments have Command Handlers for the same type of Commands. You may use different segments for different Command Types altogether. The Distributed Command Bus will always choose a node to dispatch a Command to that has support for that specific type of Command.

If you use Spring, you may want to consider using the `JGroupsConnectorFactoryBean`. It automatically connects the Connector when the ApplicationContext is started, and does a proper disconnect when the `ApplicationContext` is shut down. Furthermore, it uses sensible defaults for a testing environment (but should not be considered production ready) and autowiring for the configuration.

Spring Cloud Connector
----------------------

The Spring Cloud Connector setup uses the service registration and discovery mechanism described by [Spring Cloud](http://projects.spring.io/spring-cloud/) for distributing the Command Bus. You are thus left free to choose which Spring Cloud implementation to use to distribute your commands. An example implementations is the Eureka Discovery/Eureka Server combination.

 > **Note**
 >
 > The `SpringCloudCommandRouter` uses the Spring Cloud specific `ServiceInstance.Metadata` field to inform all the nodes in the system of its message routing information. It is thus of importance that the Spring Cloud implementation selected supports the usage of the `ServiceInstance.Metadata` field. If the desired Spring Cloud implementation does not support the modification of the `ServiceInstance.Metadata` (e.g. Consul), the `SpringCloudHttpBackupCommandRouter` is a viable solution. See the end of this chapter for configuration specifics on the `SpringCloudHttpBackupCommandRouter`.

Giving a description of every Spring Cloud implementation would push this reference guide to far. Hence we refer to their respective documentations for further information.

The Spring Cloud Connector setup is a combination of the `SpringCloudCommandRouter` and a `SpringHttpCommandBusConnector`, which respectively fill the place of the `CommandRouter` and the `CommandBusConnector` for the `DistributedCommandBus`.

> **Note**
>
> When using the `SpringCloudCommandRouter`, make sure that your Spring application is has heartbeat events enabled. The implementation leverages the heartbeat events published by a Spring Cloud application to check whether its knowledge of all the others nodes is up to date. Hence if heartbeat events are disabled the majority of the Axon applications within your cluster will not be aware of the entire set up, thus posing issues for correct command routing.

The `SpringCloudCommandRouter` has to be created by providing the following:

- A "discovery client" of type `DiscoveryClient`. This can be provided by annotating your Spring Boot application with `@EnableDiscoveryClient`, which will look for a Spring Cloud implementation on your classpath.

- A "routing strategy" of type `RoutingStrategy`. The `axon-core` module currently provides several implementations, but a function call can suffice as well. If you want to route the Commands based on the 'aggregate identifier' for example, you would use the `AnnotationRoutingStrategy` and annotate the field on the payload that identifies the aggregate with `@TargetAggregateIdentifier`.

Other optional parameters for the `SpringCloudCommandRouter`  are:

- A "service instance filter" of type `Predicate<ServiceInstance>`. This predicate is used to filter out `ServiceInstances` which the `DiscoveryClient` might encounter which by forehand you know will not handle any command messages. This might be useful if you've got several services within the Spring Cloud Discovery Service set up which you do not want to take into account for command handling, ever.  

- A "consistent hash change listener" of type `ConsistentHashChangeListener`. Adding a consistent hash change listener provides you the opportunity to perform a specific task if  new members have been added to the known command handlers set.

The `SpringHttpCommandBusConnector` requires three parameters for creation:

- A "local command bus" of type `CommandBus`. This is the Command Bus implementation that dispatches Commands to the local JVM. These commands may have been dispatched by instances on other JVMs or from the local one.

- A `RestOperations` object to perform the posting of a Command Message to another instance.

- Lastly a "serializer" of type `Serializer`. The serializer is used to serialize the command messages before they are sent over the wire.

> **Note**
>
> The Spring Cloud Connector specific components for the `DistributedCommandBus` can be found in the `axon-distributed-commandbus-springcloud` module.

The `SpringCloudCommandRouter` and `SpringHttpCommandBusConnector` should then both be used for creating the `DistributedCommandsBus`. In Spring Java config, that would look as follows:

```java
// Simple Spring Boot App providing the `DiscoveryClient` bean
@EnableDiscoveryClient
@SpringBootApplication
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }

    // Example function providing a Spring Cloud Connector
    @Bean
    public CommandRouter springCloudCommandRouter(DiscoveryClient discoveryClient) {
        return new SpringCloudCommandRouter(discoveryClient, new AnnotationRoutingStrategy());
    }

    @Bean
    public CommandBusConnector springHttpCommandBusConnector(@Qualifier("localSegment") CommandBus localSegment,
                                                             RestOperations restOperations,
                                                             Serializer serializer) {
        return new SpringHttpCommandBusConnector(localSegment, restOperations, serializer);
    }

    @Primary // to make sure this CommandBus implementation is used for autowiring
    @Bean
    public DistributedCommandBus springCloudDistributedCommandBus(CommandRouter commandRouter,
                                                                  CommandBusConnector commandBusConnector) {
        return new DistributedCommandBus(commandRouter, commandBusConnector);
    }

}
```
``` java
// if you don't use Spring Boot Autoconfiguration, you will need to explicitly define the local segment:
@Bean
@Qualifier("localSegment")
public CommandBus localSegment() {
    return new SimpleCommandBus();
}

```
> **Note**
>
> Note that it is not required that all segments have Command Handlers for the same type of Commands. You may use different segments for different Command Types altogether. The Distributed Command Bus will always choose a node to dispatch a Command to that has support for that specific type of Command.

##### Spring Cloud Http Back Up Command Router

Internally, the `SpringCloudCommandRouter` uses the `Metadata` map contained in the Spring Cloud `ServiceInstance` to communicate the allowed message routing information throughout the distributed Axon environment. If the desired Spring Cloud implementation however does not allow the modification of the `ServiceInstance.Metadata` field (e.g. Consul), one can choose to instantiate a `SpringCloudHttpBackupCommandRouter` instead of the `SpringCloudCommandRouter`.

The `SpringCloudHttpBackupCommandRouter`, as the name suggests, has a back up mechanism if the `ServiceInstance.Metadata` field does not contained the expected message routing information. That back up mechanism is to provide an HTTP endpoint from which the message routing information can be retrieved and by simultaneously adding the functionality to query that endpoint of other known nodes in the cluster to retrieve their message routing information. As such the back up mechanism functions is a Spring Controller to receive requests at a specifiable endpoint and uses a `RestTemplate` to send request to other nodes at the specifiable endpoint.

To use the `SpringCloudHttpBackupCommandRouter` instead of the `SpringCloudCommandRouter`, add the following Spring Java configuration (which replaces the `SpringCloudCommandRouter` method in our earlier example):

```java
@Configuration
public class MyApplicationConfiguration {
    @Bean
    public CommandRouter springCloudHttpBackupCommandRouter(DiscoveryClient discoveryClient,
                                                            RestTemplate restTemplate,
                                                            @Value("${axon.distributed.spring-cloud.fallback-url}") String messageRoutingInformationEndpoint) {
        return new SpringCloudHttpBackupCommandRouter(discoveryClient, new AnnotationRoutingStrategy(), restTemplate, messageRoutingInformationEndpoint);
    }
}
```
