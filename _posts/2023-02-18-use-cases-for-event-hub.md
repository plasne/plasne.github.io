---
layout: post
title: Use-Cases for Event Hub
---

I was working on a project recently where Event Hub was used in several different ways. I wanted to share these different use-cases and how I used Event Hubs to address those requirements. References to SDKs in this document refer to C#, though the concepts used here could be implemented in any language.

## Process Messages

This is the standard use-case that everyone should be familiar with. I had messages that needed to be processed by multiple processors. I used the `EventProcessorClient` SDK which provides an easy way to even distribute partitions between my processors. Each processor checkpoints messages as they are processed.

### Alternatives

There are many messaging services that could fit in this role, most notably Kafka or Azure Service Bus. Kafka would be a direct replacement. Service Bus would be a good alternative if the success or failure of individual messages is important and if the performance requirements could still be met.

## Event Streaming

I had another use-case whereby events are coming into an Event Hub. Those events are staged into memory. Then, on a heartbeat, a large number of expressions are run that have tokens referring to the in-memory events.

The requirements are:

- The events are partitioned such that all events required to process a set of scenarios are on the same partition.

- The window of events to consider is configurable but fixed (ex. 24 hours).

- Since a processor needs the full 24-hours of data, partitions should never move from one processor to another. The default implementation of `EventProcessorClient`, for instance, does not know how many partitions or processors there are so they can be reassigned.

To fulfill these requirements, I implemented the following:

- The configuration for the processors is set in such a way as to ensure the equitable distribution of partitions. For instance, if we have 32 partitions and 8 processors, then we set ASSIGN_TO_X_PARTITIONS to 4.

- I wrote a custom EventProcessor component called [EventHubFixedPartitionProcessor](https://github.com/plasne/fixed-partitioned-event-hub).

  - When it starts up, it creates a single 0-byte blob for each partition.

  - The component will then obtain an exclusive lease on a fixed number of blobs (ASSIGN_TO_X_PARTITIONS) and then continue to renew those leases. If for any reason, the processor looses the lease on a partition it immediately exits in error. When running with an orchestrator like Kubernetes, a new replica will be brought up shortly and it can try and obtain leases anew.

  - Once a processor has obtained it's leases, it starts reading events from the Event Hub from the partitions it owns in batches. These events are read starting 24-hours back (or whatever the window is set to).

- The processors never checkpoint because if a processor is restarted, it would need to read events from 24-hours back (or whatever the window is set to) anyway.

### Alternatives

The customer could have used Spark for this use-case, however, they were already using Event Hubs heavily and didn't want to take a new dependency.

## Change Feed

The application is a microservices solution with APIs that own specific entities (like the expression configurations mentioned above, information on how to partition, or the configuration of the environment, etc.) and processors that handle messages in various Event Hubs. Those processors need to get information from the APIs but it would be inefficient to do that on every heartbeat. Instead, those processors cache data until it is evicted by a message on a change feed.

The APIs write a message to a change feed which is just another Event Hub saying "this stuff has changed". The processors consume those messages and evict those things from cache. On the next heartbeat, the processors fetch whatever they need from the APIs that is no longer in cache.

The requirements are:

- All processor gets the same notifications so each can evict anything that has changed.

I implemented the following:

- I used 1 partition for the Event Hub. We want as few as possible as every processor will need to get messages from every partition.

- I used the `EventHubConsumerClient` SDK and `ReadEventsFromPartitionAsync` method so that I could read from all partitions on each processor.

- Each processor starts reading from the tail (latest) of each partition. If a notification happened when the processor wasn't running, there is obviously nothing for it to evict.

- There is no checkpointing, if the processor starts up again, it will start reading from the tail again.

### Consumer Groups

The above implementation means that each replica for each service is considered a [non-epoch consumer](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-event-processor-host#no-epoch), and is therefore limited such that no consumer group may have more than 5 consumers. For instance, if you have 3 services with 3 replicas each, you will have 9 consumers and therefore need at least 2 consumer groups. If a service is not going to support more than 5 replicas, then you could simply have a consumer group dedicated to that service. If you have a service that may have more than 5 replicas, then you will need to have multiple consumer groups for that service and some way to load balance between them. Based on tier, there is also a maximum number of consumer groups per Event Hub, which can be found [here](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits#basic-vs-standard-vs-premium-vs-dedicated-tiers).

If the limit is exceeded, the EventHubsException.FailureReason.QuotaExceeded exception will be thrown by the client when trying to read messages from a partition. This exception is not thrown when accessing the Event Hub in other ways, for instance, reading metadata.

For this customer, several services required more than 5 replicas, so I implemented the following:

- Each service had multiple dedicated consumer groups. For example, serviceA had serviceA0, serviceA1, serviceA2, etc.

- In the configuration, each service specified the list of consumer groups it was capable of using.

- At startup, the code will pick a random consumer group from the list and start reading events.

- Whenever messages are read and the QuotaExceeded exception is thrown, the code will pick a new random consumer group from the list.

### Alternatives

I don't believe there are great alternatives, but a few possibilities were mentioned by others:

- Event Grid. The problem with Event Grid is that it pushes event messages, rather than allowing the individual replicas to pull them. Therefore, Event Grid would need to be configured with the IP address or host name of every replica, both of which can change over the lifetime of the deployment.

- Service Bus Topic/Subscriptions. Services could publish changes to the topic but allow many subscribers to get those change notifications. The challenge is subscriptions have to be provisioned for new replica and de-provisioned as they are no longer needed. This is possible via the ARM REST API, but increases complexity.
