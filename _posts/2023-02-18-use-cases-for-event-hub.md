---
layout: post
title: Use-Cases for Event Hub
---

I was working on a project recently where Event Hub was used in several different ways. I wanted to share these different use-cases and how I used Event Hubs to address those requirements. References to SDKs in this document refer to C#, though the concepts used here could be implemented in any language.

## Process Messages

This is the standard use-case that everyone should be familiar with. I had messages that needed to be processed by multiple processors. I used the `EventProcessorClient` SDK which provides an easy way to even distribute partitions between my processors. Each processor checkpoints messages as they are processed.

## Event Streaming

I had another use-case whereby events are coming into an Event Hub. Those events are staged into memory. Then, on a heartbeat, a large number of expressions are run that have tokens referring to the in-memory events. As you may notice, this use-case would also have been a good fit for Spark, but the customer did not want to take that dependency.

The requirements are:

- The events are partitioned such that all events required to process a set of scenarios are on the same partition.

- The window of events to consider is configurable but fixed (ex. 24 hours).

- Since a processor needs the full 24-hours of data, partitions should never move from one processor to another. The default implementation of `EventProcessorClient`, for instance, does not know how many partitions or processors there are so they can be reassigned.

To fulfill these requirements, I implemented the following:

- The configuration for the processors is set in such a way as to ensure the equitable distribution of partitions. For instance, if we have 32 partitions and 8 processors, then we set NUM_ASSIGNED_PARTITIONS to 4.

- I wrote an `AzureBlobDistributedLockManager` component. When it starts up, it creates a single 0-byte blob for each partition.

- Each processor will be running an `AzureBlobDistributedLockManager` component that will obtain an exclusive lease on a fixed number of blobs (NUM_ASSIGNED_PARTITIONS) and then continue to renew those leases. If for any reason, the processor looses the lease on a partition it immediately exits in error. When running with an orchestrator like Kubernetes, a new replica will be brought up shortly and it can try and obtain leases anew.

- Once a processor has obtained it's leases, it starts reading events from the Event Hub from the partitions it owns, using the `EventHubConsumerClient` SDK and `ReadEventsFromPartitionAsync` method. These events are read starting 24-hours back (or whatever the window is set to).

- The processors never checkpoint because if a processor is restarted, it would need to read events from 24-hours back (or whatever the window is set to) anyway.

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
