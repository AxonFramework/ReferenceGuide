# Distribution Considerations

One of the core properties of applications built using Axon is Location Transparency - being able to run application on several nodes thus increasing scalability, responsiveness and resilience. This section summarizes things that should be taken into consideration while distributing Axon-based applications.

## Distributing Commands

More details on how Distributing the Command Bus works can be found [here](/part-iii-infrastructure-components/command-dispatching.md#distributing-the-command-bus). Two the most important components in a `DistributedCommandBus` are `CommandBusConnector` and `CommandRouter`. 

`CommandBusConnector` is a component that remotely connects multiple `CommandBus`es. Axon provides Spring Cloud and JGroups connectors which are free and open source and AxonHub which is a paid component. Of course, you can implement your own connector if provided are not sufficient.

`CommandRouter` is responsible for providing a member of a cluster based on a provided Command Message. This cluster member is the one which should handle the given command. Integral part of `CommandRouter`s is a `RoutingStrategy`. `RoutingStrategy` is a mechanism that generates a routing key for a given command. Axon provides two strategies: `AnnotationRoutingStrategy` (provides a routing key based on `@TargetAggregateIdentifier` field in a Command Message) and `MetaDataRoutingStrategy` (provides a routing key based on provided Meta Data key).

In distributed environment commands might not be always delivered due to some network issues (partitions) or some other "hiccups". When that happens it is safe just to resend the command. Axon provides a `RetryScheduler` mechanism for this purposes.

## Distributing Events

## Distributing Queries