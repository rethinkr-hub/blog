---
title: "Publish Subscribe - Interchangeable Patterns for Evaluating Open Source Services"
date: "2023-07-15"
draft: false
tags: ["Logging Transport", "Pub/Sub"]
author: "Jason Grein"
description: "Design Pattern for interchanging Pub/Sub Services"

cover:
  image: "cover-photo.png"
#  alt: "AI Generated Superhero Photo"
#  caption: "These guys are ready to duke it out for Data!"
#  relative: false # To use relative path for cover image, used in hugo Page-bundles

showToc: true
TocOpen: false
hidemeta: false
comments: true

disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
---

In the ever-evolving world of technology, it's crucial to assess the suitability of various services for specific applications. This blog post focuses on the evaluation of Pub/Sub and Message Queue services using the Super Hero Combat Data Simulator. We will explore how efficiently these services synergize with the streaming logs from the Data Simulator, aiming to provide replicatable demonstrations and facilitate service interchangeability.

**Motivation**: establish a foundation for utilizing popular Open Source Pub/Sub and Message Queue (MQ) services as a plugin to the Logging Transport that follows a similar pattern style. This approach enables us to easily switch between services, ensuring simplicity and flexibility in our system architecture.

## Prerequisites

### Superhero Combat Data Simulator

This exercise requires a Data Simulator to generate data in order to stream logging data through the Pub/Sub & MQ Services. We will utilize the [Super Hero Combat Data Simulator](/data-simulators/superhero-combat) offered in this earlier post. Please refer back to [This post](/data-simulators/superhero-combat#objective) to familiarize yourself with how to operate the Data Simulator.

### Logging

Application logging provides numerous benefits to developers, system administrators, and business stakeholders. First and foremost, logging allows for the detection and diagnosis of issues within an application, facilitating efficient debugging and troubleshooting. By capturing and storing relevant information about events, errors, and user actions, logs enable developers to pinpoint the root causes of problems and implement effective solutions. Logging also plays a critical role in monitoring application performance and identifying bottlenecks, allowing for timely optimization and scalability. Moreover, logs offer valuable insights into user behavior and usage patterns, supporting data-driven decision-making, and improving the overall user experience. Additionally, logs are invaluable for auditing and compliance purposes, providing a historical record of system activities, user interactions, and security-related events. Overall, application logging fosters enhanced reliability, maintainability, and security, empowering organizations to deliver robust and efficient software solutions.

Building logging capabilities into tools is likely a familiarity with all the readers of this blog. However, we can go a step deeper and build a customer logger to offer us the flexibility to scope this exercise against different Pub/Sub services in order to evaluate their performance and suitability. This custom logging capability is built on top of [Python's native logging module](https://docs.python.org/3/library/logging.html).

## Design for interoperability

To achieve seamless interoperability during the evaluation trials, we opted for a design that prioritizes compatibility. The primary focus was to develop a milestone pattern for logging transports, which holds significant value and deserves recognition. The result of this effort is a modular container design that builds upon the `superhero_server` service, allowing the introduction of additional logging transports. This design minimizes repetitive work by leveraging the evaluation components and streamlines them into the logging transport pattern.

### Logging Transport

In order to better understand the Logging Transport functionality, let's take a better look at *what's under the hood* in the Superhero Combat Data Simulator application. For instance, this simulator comes with 2 built-in loggers already configured.

#### Default Logger
{{< details "Github Code" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fjg-ghub%2Fpy-superhero%2Fblob%2Fmain%2Flib%2Futils%2Floggers%2Fdefault.py&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

A basic logging configuration which outputs log messages to STDOUT with a specific format of [LogRecord Attributes](https://docs.python.org/3/library/logging.html#logrecord-attributes) `%(asctime)s - %(name)s - %(levelname)s - %(message)s`. It's also important to note that this logger is created with the [`getLogger`](https://docs.python.org/3/library/logging.html#logger-objects) constructor, and at this time it registers the name of this logger to `default`. Now we can instantiate children of the `default` logger for each of our services by calling the `getLogger` constructor with the parent node being `default`. Examples

 * `default.lobby` is the Logger for the lobby channel using the **Default Logger** handler
 * `default.game` is the Logger for the game channel using the **Default Logger** handler

#### Redis Logger
{{< details "Github Code" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fjg-ghub%2Fpy-superhero%2Fblob%2Fmain%2Flib%2Futils%2Floggers%2Fredis.py&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on">}}
{{</ details >}}

Custom logging configuration which emits any log messages with a `queue` or `task` attribute to the [`redis_publisher`](https://github.com/jg-ghub/py-superhero/blob/main/lib/pubsub/redis.py#L10-L14) function, whereby it passes the message to a specific channel in Redis' Pub/Sub message store. We also pass along the same format and streaming handler equipped in the **Default Logger** to the **Redis Logger** so the application will continue logging to STDOUT for monitoring purposes.

Each service must [explicitly declare](https://github.com/jg-ghub/py-superhero/blob/main/server.py#L24C1-L24C52) which pre-designed Logger it's going to use on start-up in the `LOGGER_MODULE` Environment Variable. The **Default** logger is enabled by default.

By overriding the emit hook (example [Here](https://docs.python.org/3/howto/logging-cookbook.html#speaking-logging-messages)) in our Redis Handler example, we can utilize a standardized component in any application - Logging - to also stream important data, as a Publisher, into a Pub/Sub Framework. The logging transport allows us to seamlessly decouple the application development work-streams separately from the data streaming capabilities in order to focus on the Data Architecture and value creation from data insights.

### Docker

The Dockerfiles have been simplified thanks to [Docker's Multi-Stage Build](https://docs.docker.com/build/building/multi-stage/) capabilities. The first step involves sourcing the upstream `python/datasim/superhero image` to build on-top of the Data Simulator. Afterward, we inject the Logging Transport Hooks, Pub/Sub service, and Output Model into the image. This multi-stage build allows us to interchange logging transports and Pub/Sub Services with a couple line changes to the Image.

```docker {linenos=table,hl_lines=["1","6-7"]}
FROM python/datasim/superhero

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY ./lib/utils/loggers/redis.py ./lib/utils/loggers
COPY ./lib/pubsub/redis.py ./lib/pubsub
COPY ./lib/model ./lib/model
COPY ./lib/worker ./lib/worker
COPY ./worker_fanout.py .
COPY ./worker.py .
```

Furthermore, multiple logging transports and Pub/Sub services can be introduced into the same image, and flipped between selection by changing the `LOGGER_MODULE` Environment Variable. For example, we can choose to include both the redis and Kafka pub/sub services into the image by adding a couple more lines.

```docker {linenos=table,hl_lines=[1,"6-9"]}
FROM python/datasim/superhero

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY ./lib/utils/loggers/kafka.py ./lib/utils/loggers
COPY ./lib/utils/loggers/redis.py ./lib/utils/loggers
COPY ./lib/pubsub/kafka.py ./lib/pubsub
COPY ./lib/pubsub/redis.py ./lib/pubsub
COPY ./lib/model ./lib/model
COPY ./lib/worker ./lib/worker
COPY ./worker_fanout.py .
COPY ./worker.py .
```

Then we can select the Redis option to start in the Docker-Compose container configurations.

```docker-compose {linenos=table,hl_lines=[9]}
...

  superhero_server:
    image: python/datasim/superhero/pubsub/redis_kafka
    environment:
      - REDIS_HOST=redis
      - SERVER_SLEEP=0.2
      - WEBSOCKET_HOST=0.0.0.0
      - LOGGER_MODULE=redis
    env_file:
      - ../.env
    depends_on:
      - redis
    expose:
      - ${WEBSOCKET_PORT}

...

```

## Objective

Now that we've successfully decoupled the log streaming from the Superhero Combat Data Simulator in order to flexibly evaluate different Pub/Sub & MQ services with little code changes, we're now ready to start the evaluation. The following trials will compare how a handful of highly reputable Pub/Sub & MQ services manage streaming data to a local data sink in JSON files for future topics on Data Flows and Data Warehouses.

### Pub/Sub

#### Redis

{{< refcard "/pub-sub/redis" >}}

Redis Pub/Sub offers a very simplistic and easy to use messaging framework out-of-the-box. It's already acting as our DB Key Store on the application side, so making use of it's Pub/Sub feature is a *no-brainer*. Running this application on a local environment, using a small amount of Player Services, with Redis' built-in Pub/Sub works wonders - it handles the payload effectively. However, Redis Pub/Sub suffers from scalability issues when we try to add more workers attempting to create some efficiency.

Redis' simplicity effectively breaks the out-of-the-box solution for our application's needs. The Redis Pub/Sub mechanism is a Broadcast message to all subscribers listening to our channel `lib.server.lobby`, which means that all workers receive the same message. This doesn't help provide us more efficiency when we scale our workers - instead it causes duplication.

In this example, the same data ingestion was run once on each worker service. This solution isn't viable for our needs. Instead, we would like to have our workers take tasks from a queue whereby they run unique operations simultaneously, but never the identical task.

#### Redis Narrowcast

{{< refcard-alt "/pub-sub/redis_narrowcast" >}}

The Narrowcast service is a middleware subscriber we dedicate to receiving all messages in the `lib.server.lobby` channel, and then randomly publishing back to a worker's unique channel - following the pattern `lib.server.lobby-*`. Middleware is a common technique used to insert logic between the Publisher and Subscriber.

Here we've come one step closer to building a sustainable Pub/Sub service which processes independent tasks scaled over multiple workers. The Narrowcast service was a quick way to spread out tasks amongst the workers, so that nothing was being processed more than once. This example doesn't actually replicate a message queue system, but instead sends out a broadcast message to a narrower focus group.

In terms of sustainability, this approach still doesn't meet other service's levels of fail-over support and resiliency in message delivery/acknowledgment. This is where other popular solutions will be used, instead of trying to recreate the wheel with Redis.

### Message Queue

#### RabbitMQ

{{< refcard "/pub-sub/rabbitmq" >}}

On this run we have 100% acknowledgment of messages consumed successfully with no issues consuming the velocity of messages produced/sec. This can be inferred by the Consumer Ack (Green) Message Rate overlapping the Produced (Yellow) in ths image below.

*/Insert Image Here/*

A significant benefit to RabbitMQ is its smart broker technology. in contrast to Pub/Sub, a published message is immediately sent out to a subscriber once received by the broker. If no subscriber is available to receive the published message, then the message has lost its opportunity for subscriber processing. However, with RabbitMQ the produced message is persisted in the Queue until fetched by the consumer. The produced message can be consumed at a future time, and will only be removed from the queue once a consumer has acknowledged the completion of message processing.

### Kafka

{{< refcard-alt "/pub-sub/kafka" >}}

Kafka UI can provide us great detail about the number of messages sitting in a Topic, what those message's content looks like and how many subscribers are consuming messages within that topic. However, it doesn't provide any useful charts for monitoring Topic throughput, nor what messages were actually executed successfully.

A significant benefit to Kafka is its native batch processing capabilities. This demonstration uses an agnostic worker to transfer Redis logs into JSON files, but really isn't necessary with Kafka's streaming buffer technology. The logs are published to Kafka Topics, but are never actually removed from the Topic (unlike Message Queue Platforms). Instead, the logs are given an expiry period before they're discarded or archived in Cold storage.

This enables the logs in Kafka Topics to be processable through batch or stream style processing. The agnostic worker is purely for demonstrating the ability to extend the same processing across different Pub/Sub and Message Queue architectures. Batch execution would be the preferred ingestion methodology for this style of data processing.

## Supporting Documents

A great document on the two popular services (RabbitMQ & Kafka) can be found here [Apache Kafka Vs RabbitMQ: Main Differences You Should Know](https://www.simplilearn.com/kafka-vs-rabbitmq-article#:~:text=Deciding%20Between%20Kafka%20and%20RabbitMQ&text=While%20Kafka%20is%20best%20suited,for%20both%20Kafka%20and%20RabbitMQ.). This is a very well thought out comparison of the two tools, and those most prevalent realization to this evaluation was the description on broker "smartness"

The document mentions that RabbitMQ is considered a Smart Broker for dumb consumers, and Kafka is a dumb broker for smart consumers. These evaluations prove how simple it is to start up using Rabbit MQ, and how much the Broker handles the queue for message consumption. Where I don't see the same simplicity in Kafka, I do see better suitability for the later ingestion of data given:

 * batch processing capability
 * message retention

This allows for more efficient ingestion procedures to be built into the workers which is the goal of this evaluation.
