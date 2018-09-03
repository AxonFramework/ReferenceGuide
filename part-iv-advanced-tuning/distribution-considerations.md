# Distribution Considerations

One of the core properties of applications built using Axon is Location Transparency - being able to run application on several nodes increases scalability, responsiveness and resilience. This section summarizes things that should be taken into consideration while distributing Axon-based applications.

## Distributing Commands

More details on how Distributing the Command Bus works can be found [here](/part-iii-infrastructure-components/command-dispatching.md#distributing-the-command-bus). Two the most important components in a `DistributedCommandBus` are `CommandBusConnector` and `CommandRouter`. 

`CommandBusConnector` is a component that remotely connects multiple `CommandBus`es. Axon provides Spring Cloud and JGroups connectors which are free and open source. Of course, you can implement your own connector if provided are not sufficient.

`CommandRouter` is responsible for providing a member of a cluster based on a provided Command Message. This cluster member is the one which should handle the given command. Integral part of `CommandRouter` is a `RoutingStrategy`. `RoutingStrategy` is a mechanism that generates a routing key for a given command. Axon provides two strategies: `AnnotationRoutingStrategy` (provides a routing key based on `@TargetAggregateIdentifier` field in a Command Message) and `MetaDataRoutingStrategy` (provides a routing key based on provided Meta Data key).

In distributed environment commands might not be always delivered due to some network issues (partitions) or some other "hiccups". When that happens it is safe just to resend the command. Axon provides a `RetryScheduler` mechanism for this purposes.

> **Note** For distributing commands, there's an option to use the [AxonHub](https://axoniq.io/product-overview/axonhub).

## Distributing Events

Axon provides Spring AMQP and Apache Kafka support as free and open source solutions for distributing events. [AxonHub](https://axoniq.io/product-overview/axonhub) can be used as a mechanism for distributing events also. You can find details on how to distribute events [here](/part-iii-infrastructure-components/event-processing.md#distributing-events).

In order to increase performance you can process events in parallel using `TrackingEventProcessor`. Configuring `segmentSize` for `TrackingEventProcessor` enables events to be processed in several threads on a single node. Segments are claimed by processors thus preventing multiple processing of the same event by different processors. This property stands in distributed environment also (processors on different nodes will respect their segments).

Some events are not convenient to be processed in parallel. That's why Axon introduces `SequencingPolicy`. More details about `SequencingPolicy` [here](/part-iii-infrastructure-components/event-processing.md#parallel-processing).  

To increase interoperability you might want to consider [serialization](performance-tuning.md#event-serializer-tuning).

## Distributing Queries

Currently, running Queries in distributed environment is possible by using [AxonHub](https://axoniq.io/product-overview/axonhub) only.