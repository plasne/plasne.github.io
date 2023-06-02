---
layout: post
title: Use-Cases for Event Hub
---

Our team was working on a project recently where Azure Event Hub was used in several different ways. We wanted to share these different use-cases and how we used Event Hubs to address those requirements.

References to SDKs in this document refer to C#, though the concepts used here could be implemented in any language. We refer to "processors" in this document. A processor is just a component that reads messages from an Event Hub and does something with them. For instance, it could be a component that validates messages, enriches them, and then writes a record to a database. We assume that each processor will have multiple replicas hosted on a container orchestrator such as Azure Kubernetes Service.

## Process Messages

This is the standard use-case that everyone should be familiar with. We had events that needed to be processed by replicas. We used the `EventProcessorClient` SDK which provides an easy way to evenly distribute partitions between those consumers. Each processor checkpoints events as they are processed.

### Alternatives (Process Messages)

There are many messaging services that could fit in this role, most notably Kafka or Azure Service Bus. Kafka would be a direct replacement. Service Bus would be a good alternative if the success or failure of individual messages is important and if the performance requirements could still be met.

## Event Streaming

We had another use-case whereby events were partitioned such that each partition would have all the data needed for processing a slice of the whole problem. Each replica of the processor would get 1 or more of those partitions and stage the data into memory. On a heartbeat, a large number of expressions were run that have tokens referring to the in-memory events.

The requirements are:

- The events are partitioned such that all events required to process a set of expressions are on the same partition.

- The window of events to consider is configurable but fixed (ex. 24 hours).

- Since a replica needs the full 24-hours of data, partitions should never move from one replica to another. The default implementation of `EventProcessorClient`, for instance, does not know how many partitions or consumers there are so they can be reassigned.

To fulfill these requirements, we implemented the following:

- The configuration for the processor is set in such a way as to ensure the equitable distribution of partitions. For instance, if we have 32 partitions and 8 replicas, then we set ASSIGN_TO_X_PARTITIONS to 4.

- We wrote a custom EventProcessor component called EventHubFixedPartitionProcessor. This is available for use under an MIT license [here](https://github.com/plasne/fixed-partitioned-event-hub).

  - When it starts up, it creates a single 0-byte blob for each partition.

  - The component will then obtain an exclusive lease on a fixed number of blobs (ASSIGN_TO_X_PARTITIONS) and then continue to renew those leases. If for any reason, the replica looses the lease on a partition it immediately exits in error. When running with an orchestrator like Kubernetes, a new replica will be brought up shortly and it can try and obtain leases anew.

  - Once a replica has obtained it's leases, it starts reading events from the Event Hub from the partitions it owns. These events are read in batches starting 24-hours back (or whatever the window is set to).

- The processors never checkpoint because if a processor is restarted, it would need to read events from 24-hours back (or whatever the window is set to) anyway.

### Alternatives (Event Streaming)

The customer could have used Spark for this use-case, however, they were already using Event Hubs heavily and didn't want to take a new dependency.

## Change Feed

The application is a microservices solution with APIs that own specific entities (like the expression configurations mentioned above, information on how to partition, or the configuration of the environment, etc.). As previously mentioned, it also includes processors that handle events in various Event Hubs. Those processor replicas need to get information from the APIs but it would be inefficient to do that on every heartbeat. Instead, those replicas cache data until it is evicted by a message on a change feed.

The APIs write a message to a change feed which is just another Event Hub saying "this stuff has changed". The processor replicas consume those messages and evict those things from cache. On the next heartbeat, the replicas fetch whatever they need from the APIs that is no longer in cache.

The requirements are:

- Each replica gets the same notifications so each can evict anything that has changed.

We implemented the following:

- We used 1 partition for the Event Hub. We want as few as possible as every replica will need to get events from every partition.

- We used the `EventHubConsumerClient` SDK and `ReadEventsFromPartitionAsync` method so that we could read from all partitions on each processor.

- Each replica starts reading from the tail (latest) of each partition. If a notification happened when the replica wasn't running, there is obviously nothing for it to evict.

- There is no checkpointing, if the replica starts up again, it will start reading from the tail again.

An implementation of this change feed under an MIT license can be found [here](https://github.com/plasne/change-feed).

### Consumer Groups

The above implementation means that each replica for each processor is considered a [non-epoch consumer](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-event-processor-host#no-epoch), and is therefore limited such that no consumer group may have more than 5 consumers. For instance, if you have 3 processors with 3 replicas each, you will have 9 consumers and therefore need at least 2 consumer groups. If a service is not going to support more than 5 replicas, then you could simply have a consumer group dedicated to that processor. If you have a processor that may have more than 5 replicas, then you will need to have multiple consumer groups for that processor and some way to load balance between them. Based on tier, there is also a maximum number of consumer groups per Event Hub, which can be found [here](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits#basic-vs-standard-vs-premium-vs-dedicated-tiers).

If the limit is exceeded, the EventHubsException.FailureReason.QuotaExceeded exception will be thrown by the client when trying to read messages from a partition. This exception is not thrown when accessing the Event Hub in other ways, for instance, reading metadata.

For this customer, several processors required more than 5 replicas, so I implemented the following:

- Each processor had multiple dedicated consumer groups. For example, serviceA had serviceA0, serviceA1, serviceA2, etc.

- In the configuration, each processor specified the list of consumer groups it was capable of using.

- At startup, each replica will pick a random consumer group from the list and start reading events.

- Whenever events are read and the QuotaExceeded exception is thrown, the replica will pick a new random consumer group from the list.

### Alternatives (Change Feed)

We didn't find great alternatives, but a few possibilities that were mentioned by others:

- Event Grid. The problem with Event Grid is that it pushes events, rather than allowing the individual replicas to pull them. Therefore, Event Grid would need to be configured with the IP address or host name of every replica, both of which can change over the lifetime of the deployment.

- Service Bus Topic/Subscriptions. Services could publish changes to the topic but allow many subscribers to get those change notifications. The challenge is subscriptions have to be provisioned for new replica and de-provisioned as they are no longer needed. This is possible via the ARM REST API, but increases complexity.
