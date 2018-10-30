---
layout: post
title: Architecture Comparison for IDOC to CSV - Stream vs. Batch
---

This article compares an IDOC receiver and processor built as a stream processor using containers and a batch processor using Functions.

# Introduction

The customer wants to accept IDOC documents from an SAP server, determine a schema, and parse the document per the schema, and write the values to a CSV file. I originally wrote the application using a stream processing pattern but later the customer decided they wanted a batch process for business reasons. Since I had written the same functionality a couple of different ways, I thought it would make an interesting article to compare the architectures of these applications.

# Stream

https://github.com/plasne/SAP-receiver

The stream processing application benefits from being a very simple design. The code is written in Node.js and there is a single server.js file that contains all the logic. The application presents an HTTP endpoint that can accept POSTs containing IDOC bodies. The application leverages a folder of schema files that it automatically monitors for changes. Whenever an IDOC file comes in, the following happens:

1. A POST request containing an XML body is received.
1. The timeslice folder name is determined.
1. The raw XML file is saved in the timeslice folder. A GUID is used as part of the name to ensure the filenames are unique.
1. The schemas are examined to determine if anything needs to be extracted to a CSV file.
1. Any extracted rows are written to the appropriate CSV files.
1. The response is sent as 200 if all files were written and a 500 if there was an error.

Notice that the primary functional difference with this implementation is that files are _processed as they are received_.

This application could be run as a single threaded process, but for higher volumes, you can put this application in a container, scale to any number of instances, and put those instances behind a load balancer. The CSV files are Append Blobs so any number of writers can concurrently add rows.

# Batch

https://github.com/plasne/SAP-receiver-func

The batch processing application is a bit more complicated. Rather than use containers, it is built for Azure Functions v2 (TypeScript/Node.js).

There are 4 Functions that work together to satisfy the scenario:

- **Receiver**: This function listens for incoming documents in an HTTP message body and saves them to Azure Blob Storage within a time-partitioned "folder".

- **Start**: This function starts a batch process given a partition. This does not do any processing of the actual files but rather creates the CSV append blobs and then chunks up the processing into queued messages.

- **Status**: This function returns the status of batch processing.

- **Processor**: This function monitors the processing queue and and will process messages. Messages are a collection of filenames that will be loaded, matched to schemas, and then appended to CSV files.

Notice that the primary functional difference with this implementation is that files are _received without validation_ and then _processed as a batch_ after everything has been received.

This article assumes that this application will run on a Consumption AppService Plan (perhaps more commonly known as "server-less") configuration.

# Containers vs. Functions

Why use containers for the first application and functions for the second? First off, either application could be designed for either platform.

The details are below, but to summarize here: I like containers when I need to control the scalability and Functions when I want a black-box.

## Predictable Scalability: Containers

The stream processing application scales linearly the more instances added (for example, if 1 container could handle 1k req/sec, then 4 containers could handle 4k req/sec). Given that the receiver and processor functionality is bundled into a single process, the scalability to receive is restricted by the scalability to process. This design makes it very important to understand the required scale and have capacity to fulfill it. Any container orchestrator will give this explicit control - if you know you need 4 containers then you can have 4 containers. Azure Functions are "magic", they _should_ scale up as you need more instances, but you don't have any control over that scaling process or even a good way to tell how many instances you have.

In the case of the batch processing application, the receiver does no processing (simply saves the file as is). While it is still important that there are enough instances to handle the inbound load, each receiver instance should be able to handle a higher volume making this less of a concern.

## Gateway Services: Functions

There are services provided by a gateway for Functions that make some scenarios easier.

- **Load Balancing**: As your number of Function instances increases, they are automatically placed behind a load-balancer. With containers spanning multiple VMs, you would need to put a load balancer in front of the nodes.

- **SSL**: Functions provide an SSL endpoint (custom domain name or not). With containers, you would likely deploy a gateway service to handle this termination or use an ingress controller.

- **Authentication**: Functions can provide some simple methods of authentication out-of-the-box. Specifically, the batch processing application uses a Function Key, whereas the stream processing application uses Basic Authentication in the code.

Likely, if you were running the stream processing application in production, you would need to put it behind Azure Application Gateway to handle SSL-offload and load balancing. Alternatively, you could deploy this in the Azure Kubernetes service and use an ingress controller that provides SSL offload.

## Bindings: Functions

Functions have the ability to bind input and/or output. Generally I am pretty skeptical of this saving any time or effort, but for a design pattern that uses a queue to handle the scalability of the processor (as the batch processing application does), it can be useful.

# REST vs. SDKs

Both applications read/write data to Azure Storage. For the stream processing application I make all those storage calls using REST, whereas for the batch processing application I use the Microsoft storage SDK.

There were 2 reasons for using REST:
- The Node.js Storage SDK does not support multiple concurrent append writers.
- Since the stream processor must receive and process messages in realtime, I wanted the fastest possible method to write.

In the batch processing application, speed is less important so using an SDK is fine. In fact, the Azure Storage SDK is wrapped inside my Storage Streaming API. This is one additional layer of abstraction but allows for controlling the concurrency of storage transactions - for instance, up to 10 concurrent threads are used for flushing writes. My Storage Streaming API also uses REST for append blob calls to ensure that multiple writers are supported as well.

# Outbound Connections

One thing you must always be cautious of is the number of outbound threads being used because many hosting methods impose a tight limit on them. You must be careful to use KeepAlive and MaxSockets settings to ensure a connection pool is maintained of the proper size.

The container solution uses the REST API directly so it is easy enough to create a new Agent and implement the proper settings.

```javascript
const agent = new agentKeepAlive.HttpsAgent({
    maxSockets: 40,
    maxFreeSockets: 10,
    timeout: 60000,
    freeSocketKeepAliveTimeout: 30000
});
```

Since Functions use the Storage SDK, and the Storage SDK does not have a way to specify an HTTP(S) Agent, we can only specify to use the Global Agent and modify that to specify keepAlive and maxSockets.

```typescript
const httpAgent: any = http.globalAgent;
httpAgent.keepAlive = true;
httpAgent.maxSockets = 30;
const httpsAgent: any = https.globalAgent;
httpsAgent.keepAlive = true;
httpsAgent.maxSockets = 30;

const queue = new AzureQueue({
    connectionString: AZURE_WEB_JOBS_STORAGE,
    encoder: 'base64',
    useGlobalAgent: true
});

if (obj.useGlobalAgent) this.service.enableGlobalHttpAgent = true;
```
