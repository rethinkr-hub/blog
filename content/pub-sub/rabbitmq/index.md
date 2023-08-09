---
title: "Message Queue - RabbitMQ"
date: "2023-07-26"
draft: false
tags: ["RabbitMQ", "Message Queue"]
author: "Jason Grein"
description: "Introduction to the Message Queue"

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

The last two articles evaluated how Redis could be used to manage streaming our Data Simulator's data ingestion pipeline with Pub/Sub. While Redis was functional in streaming data with our simulator, it did have its downfalls in terms of fail-over support and message resiliency. So, instead of relying on Redis to act as both the Mem DB and the Pub/Sub service, we can scope Redis to only service the Mem DB for the Data Simulator solution and look for an alternative open source solution to handle the data streaming Pub/Sub activities. The first open source technology we'll evaluate is the tried-and-true RabbitMQ. RabbitMQ is one of the more well known applications which has helped to pioneer the more common Message Queue pattern.

Message Queue and Pub/Sub are both messaging patterns used in distributed systems, but they differ in their fundamental communication models. A Message Queue is a point-to-point communication pattern where messages are sent from a single sender to one or more specific receivers. The sender and receiver are decoupled, meaning the sender doesn't need to know the identity of the receivers. On the other hand, Pub/Sub is a one-to-many communication pattern where messages are published to a topic or channel, and multiple subscribers interested in that topic receive the messages. The key distinction is that in a Message Queue, each message is consumed by only one receiver, while in Pub/Sub, a single message can be consumed by multiple subscribers. This makes Pub/Sub well-suited for broadcasting information to multiple interested parties while Message Queues are more suitable for point-to-point communication and load balancing scenarios.

**Motivation**: Integrate the RabbitMQ message queue into our solution to benchmark it's suitability for a high caliber level data pipeline - ingesting log data from the Data Simulator and sinking them to local JSON files.

## Prerequisites

### Superhero Combat Data Simulator

This exercise requires a Data Simulator to generate data in order to stream logging data through the Pub/Sub & MQ Services. We will utilize the [Super Hero Combat Data Simulator](/data-simulators/superhero-combat) offered in this earlier post. Please refer back to [This post](/data-simulators/superhero-combat#objective) to familiarize yourself with how to operate the Data Simulator.

### Logging and Transport

The generation of data is possible via the logging message transport. The Data Simulator is purposefully designed in this experiment to customize the logging emit hook and transport messages to a variety of down-stream applications. a recap can be found in the [Pub/Sub & MQ Comparison Article](/pub-sub/comparison). The logging transport allows us to seamlessly decouple the application development work-streams separately from the data streaming capabilities in order to focus on the Data Architecture and value creation from data insights.

### Rabbit MQ

[RabbitMQ](https://www.rabbitmq.com/documentation.html) is a robust and flexible message queue system that serves as an intermediary for communication between distributed applications. Acting as a middleman, RabbitMQ enables seamless data exchange, allowing various components of a system to asynchronously exchange messages with each other. It follows the Advanced Message Queuing Protocol (AMQP) and employs a "producer-consumer" model, where message producers send data to the queue, and message consumers retrieve and process the messages at their own pace. This decoupling of components enhances system scalability, resilience, and reliability, making RabbitMQ a vital tool for building efficient and loosely-coupled microservices architectures and other distributed systems.

## Install

Clone this repository to local computer **[py-superhero-pubsub](https://github.com/jg-ghub/py-superhero-pubsub)** {{< svg-icon "github" >}}
```bash
git clone https://github.com/jg-ghub/py-superhero-pubsub
```

## Services
{{< details "Schematic" >}}
  {{< get-html "content/pub-sub/rabbitmq/worker-schematic.html" >}}
{{</ details >}}

### Worker

**Description**
{{< details "Github Code" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fjg-ghub%2Fpy-superhero-pubsub%2Fblob%2Fmain%2Frabbitmq%2Flib%2Futils%2Floggers%2Frabbit.py&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

In this evaluation against RabbitMQ as the message distributor technology, we rely once again on our development work with the [Logging Transport](#logging-and-transport). Specifically, the first requirement to begin utilizing RabbitMQ is to incorporate the RabbitMQ producer functionality into the Server logging transport. Similarly to how we accomplished this with [Redis](/pub-sub/comparison#logging-transport), we create a new RabbitMQ logger transport which produces messages and delivers to RabbitMQ with the logging emit hook.

Prior to the RabbitMQ logger being instantiated, the `lib.utils.logger rabbit.py` module also starts an asynchronous `rmqio` package to deliver messages to RabbitMQ in its own IO loop. Prior to this, we were publishing message with Redis synchronously in the Server application meaning we had to wait for the log message to be published before proceeding to the next step in code. The synchronous development with the Redis Pub/Sub logging transport can likely be a bottleneck on the actual Server process in this Data Simulator as the server needs to wait for the delivery of the log message before proceeding with higher priority actions against Redis because they both utilize the same Redis connection. The asynchronous nature will alleviate this potential bottleneck in the Data Simulator application which is a great benefit to moving away from Redis Pub/Sub.

This asynchronous IO loop is started in the logger module with

```python
from lib.pubsub.rabbit import Rabbit_Publisher
import asyncio

# Setup
publisher = Rabbit_Publisher()
asyncio.create_task(publisher.connect())
```

The Producer and Consumer classes can be found in the `lib.pubsub.rabbit` module. Notice how the `Rabbit_Consumer` class has been designed with `callback_func` so we can easily plug-and-play the original data ingestion pipeline, which follows the [Factory Method](/pub-sub/redis#worker) pattern we discussed in earlier articles.
{{< details "Github Code">}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fjg-ghub%2Fpy-superhero-pubsub%2Fblob%2Fmain%2Frabbitmq%2Flib%2Fpubsub%2Frabbit.py&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

This allows us to use the same data ingestion pipeline for storing game state from Redis to our local data sink.
{{< details "Github Code" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fjg-ghub%2Fpy-superhero-pubsub%2Fblob%2Fmain%2Frabbitmq%2Fworker.py&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

#### Local Deployment
To test/debug the worker service, we can start up the services required to begin generating data for consumption by executing the following locally in bash.

```bash
export $(cat rabbitmq/.env) && \
export PLAYERS=10 && \
docker-compose -f rabbitmq/compose/docker-compose.rabbitmq.yml up -d redis rabbit build_db superhero_server superhero_nginx player --scale player=${PLAYERS}
```

Then we can begin deploying worker services locally for debugging with

```bash
export $(cat rabbitmq/.env) && \
export WORKER_CHANNEL=lib.server.lobby && \
python3 rabbitmq/worker.py
```

**Note** 

RabbitMQ's performance can be measured, and visualized, with the RabbitMQ Management UI. When spinning up RabbitMQ with Docker, the Management UI can be accessed by visiting [localhost:15672](localhost:15672). The default credentials are

 * User: guest
 * Pass: guest

## Objective

Here we're going to evaluate how RabbitMQ operates within our Data Simulation. We can deploy a trial experiment running a [Production Deployment](#production-deployment) with a single worker following the script below

#### Production Deployment
{{< details "Video Demonstration" >}}
  {{< google-drive-video "1hnetnp5bKAYSgB3ZAEMR8D7gooQHhc6u" >}}
{{< /details >}}

Spinning up a production application with docker-compose is simple. Follow the commands below, replacing ${PLAYERS} with the required participants.

```bash
export PLAYERS=10
docker-compose --env-file rabbitmq/.env -f rabbitmq/compose/docker-compose.rabbitmq.yml build && \
docker-compose --env-file rabbitmq/.env -f rabbitmq/compose/docker-compose.rabbitmq.yml up --scale player=${PLAYERS}
```
#### Production Deployment with Multiple Workers
{{< details "Video Demonstration" >}}
  {{< google-drive-video "1beJfHWy4BDQfQ4UMVDcf8kPgoiMINhzv" >}}
{{< /details >}}

```bash
export PLAYERS=10
export WORKERS=4
docker-compose --env-file rabbitmq/.env -f rabbitmq/compose/docker-compose.rabbitmq.yml up --scale worker=${WORKERS} --scale player=${PLAYERS}
```

We can see this first trial run has met our expectations, which are the ingestion pipelines ran at most once when we scaled the worker services so no duplication has occurred. However, our past experiment with [Redis and Narrowcasting](/pub-sub/redis-narrowcast#objetive) was also successful in running its data ingestion pipeline at most once, but failed to resiliently process messages when no Subscriber was actively listening to the channel. Let's experiment with RabbitMQ how resilient the Message Queue can be by mocking a scenario where all workers are dead while the Data Simulation is running, but return once it's complete.

#### Production Deployment with Dead Workers\
{{< details "Video Demonstration" >}}
  {{< google-drive-video "1Ld8GtZ5Oy-GxQefBEL4ijxBhXxZMNMS-" >}}
{{< /details >}}

```bash
export PLAYERS=10
docker-compose --env-file rabbitmq/.env -f rabbitmq/compose/docker-compose.rabbitmq.yml up --scale worker=0 --scale player=${PLAYERS}
```

*After the simulation has completed*
```bash
export WORKERS=4
docker-compose --env-file rabbitmq/.env -f rabbitmq/compose/docker-compose.rabbitmq.yml up --scale worker=${WORKERS}
```

In this scenario, we can observe all the messages produced during the Data Simulation were successfully stored in RabbitMQ while workers were waiting to start listening to the Topic. After the workers scaled back up, the queue began to drop as log messages were processed by the workers/consumers. This is a great example of the resiliency we're looking for when needing to process messages at most once. We can conclude that, given the volume and velocity of messages produced and consumed in this Data Simulation, RabbitMQ can handle our needs without issues.

Can RabbitMQ scale to higher magnitudes of volume and velocity? This kind of exercise is out-of-scope (for now) for our evaluation trials, but it's important to think about future constraints during the development stage. Next we'll explore a more popular open source Message Queue platform in the Apache Software Foundation called Kafka.
