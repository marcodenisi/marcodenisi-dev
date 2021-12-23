---
title: "Fantastic bottlenecks and where to find them"
date: 2021-12-23T12:21:26+01:00
draft: false
categories: ["Development"]
tags: ["java", "monitoring", "datadog", "performance", "dynamodb"]
---

In Klarna, I work in a team called Post Purchase Decisioning (aka PPD). We take care of a product called **Skan**, which is used by internal stakeholders (called fraud agents) to work on all the orders in Klarna that can be fraudolent.

This is a story all about how ~~my life got flipped-turned upside down~~ we approached the so called Peak Season (the range of time between the Black Friday and Christmax) and in detail, how we discovered and fixed a bottleneck. Enjoy!

![PPD Approaching Peak Season](/img/articles/bottlenecks/fantastic-beasts.jpg)

# TL;DR

- We wanted to load test our service used to ingest data from kafka;
- Turned out the V1 Java DynamoDB client was the bottleneck;
- Changing to asynchronous version improved metrics up to 75%;

# Some Context

The Peak Season is just around the corner, and as everyone in Klarna we need to be sure we won’t blow up like a balloon.

The SKAN backend can be roughly split in two different parts:

- The ingestion part, used to consume data from all our sources (aka Kafka Topics) and to save everything in a DynamoDB;
- The serverless part, triggered by DynamoDB Streams, enriches the data and moves everything in ElasticSearch, ready to be consumed by the frontend.

![Simplified SKAN Architecture](/img/articles/bottlenecks/be.jpg)

What we wanted to do, is to start checking if our ingestion was good enough to keep up with the expected load: 9K kafka messages per second during peak seconds. This is our environment setup:

- **Load produced**: 9K events per second
- **Instances Size**: Medium (m5-generation instance, 1vCPU, 3GB memory)
- **Instances Min/Max**: Min 8, Max 32

# First Run

We started producing 9K events per second using cheetah, an internal framework based on [locust](https://locust.io/), and that’s the result:
![Datadog dashboard consumer lag](/img/articles/bottlenecks/run1_0.png)
![Datadog dashboard consumer write capacity dynamodb](/img/articles/bottlenecks/run1_1.png)
![Datadog dashboard container cpu load](/img/articles/bottlenecks/run1_2.png)

To sum up:
- 16K consumer lag at peak
    * And 4 minutes to recover
- DynamoDB write units at almost 11K at peak
- We started scaling to 14 instances only at the end of the computation, when the max CPU reached up to 92%

We wanted to minimize the consumer lag though. So, under the hypothesis that more instances would mean ingesting more messages, we decided to warm up the pool and then to hit it with everything we’ve got.

Another thing we did was increase the number of partitions of the kafka topic we were using to mimic the production environment and better balance the consumers/partitions ratio.

Spoiler alert: that didn’t work.

![Totally unexpected](/img/articles/bottlenecks/totally_unexpected.jpeg)

# Second Run

Without further ado, here’s the results of the warming up experiment plus the actual load test:

![Datadog dashboard consumer lag](/img/articles/bottlenecks/run2_0.png)
![Datadog dashboard consumer write capacity dynamodb](/img/articles/bottlenecks/run2_1.png)
![Datadog dashboard container cpu load](/img/articles/bottlenecks/run2_2.png)

You can clearly see the warm up run and then the actual load test run by looking at the metrics above.

Again, let’s put together some considerations:

- The consumer lag went up to 29K 
    - increasing the instances didn’t do much
- Again, DynamoDB write units at 11K at peak
    - This starts to be suspicious
- The actual load test run was done against 19 instances

The most suspicious thing is DynamoDB write units being “locked” to 11K per minute. DynamoDB can write way more than this, so it can’t be a real bottleneck. But clearly, something’s going on there.

# A deeper look at DynamoDB Java Client

The focus then moved to how we persist data in DynamoDB. One interesting metric to check is the SuccessfulRequestLatency, which, according to the AWS documentation:

> The successful requests to DynamoDB or Amazon DynamoDB Streams during the specified time period
> 
> ....
> 
> SuccessfulRequestLatency reflects activity only within DynamoDB or Amazon DynamoDB Streams, and does not take into account network latency or client-side activity.

And while the average value was ok-ish, that’s what we got when we looked for the maximum value:

![Datadog dashboard dynamodb request latency average](/img/articles/bottlenecks/dynamodb_latency_avg.png)
![Datadog dashboard dynamodb request latency max](/img/articles/bottlenecks/dynamodb_latency.png)

We got one peak of over 2 seconds and multiple peaks over 1 second! There was of course some issue with that. We used the DynamoDB client offered by the AWS SDK v1, which we discovered being synchronous and blocking! This means that each and every thread performing http calls to AWS will wait for the response. And this means a lot of wasted time doing basically nothing.

Thus we decided to try the shiny new [DynamoDB Enhanced Client](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/dynamodb-enhanced-client.html). As stated in the official documentation, the main difference is

>The AWS SDK for Java 2.x features truly non blocking asynchronous clients that implement high concurrency across a few threads. The AWS SDK for Java 1.x has asynchronous clients that are wrappers around a thread pool and blocking synchronous clients that don’t provide the full benefit of nonblocking I/O.

This would mean that even if the DDB latency can have peaks, we no longer wait for each call to finish, instead we offload this “I/O” operation to another thread. Let’s see this new client in action.

![Datadog dashboard consumer lag](/img/articles/bottlenecks/run3_0.png)
![Datadog dashboard consumer write capacity dynamodb](/img/articles/bottlenecks/run3_1.png)
![Datadog dashboard container cpu load](/img/articles/bottlenecks/run3_2.png)

# Conclusions

Results are really promising. To sum up:

- We still have consumer lag, but we have a reduction of the 75%
- We manage to unleash the potential of DynamoDB with 25K write capacity units per minute

The only drawback is that we almost broke DynamoDB. In fact, we reached an upper limit for which DynamoDB had to scale. We were not resilient to that scenario (aka we didn’t retry on that error), so this is something we definitely need to address and something we wouldn't have noticed if we hadn't done performance tests.