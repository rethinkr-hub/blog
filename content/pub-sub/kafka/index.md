---
title: "Message Queue - Kafka"
date: "2023-07-27"
draft: false
tags: ["Kafka", "Message Queue"]
author: "Jason Grein"
description: "Optimized Message Queue for Streaming Data"

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

So far in this series, we've evaluated our Data Simulator against Redis' Pub/Sub technology and then against RabbitMQ's Message Queue platform. We found that the Message Queue capabilities suits our data ingestion pipeline needs better than Publish/Subscribe. RabbitMQ was more then adequate when it came to streaming the Data Simulator's log data to our local data sink, but we have concerns about how RabbitMQ can scale with the inevitable increase in our log data's volume and velocity. Kafka is a well known alternative with better optimizations in scaling to Big Data levels of data streaming, and we're interested how easily Kafka could replace RabbitMQ in our Data Simulator.

Developing a robust and efficient data streaming ingestion pipeline to handle frequent and high volumes of data is a formidable challenge, but essential for modern data-driven applications. The pipeline must be designed with scalability, fault-tolerance, and low-latency in mind to accommodate the continuous influx of data. Utilizing technologies like Apache Kafka can help manage the streaming data flow, ensuring data is ingested, processed, and delivered in real-time. Additionally, employing a distributed processing architecture can enable parallel processing of data streams to maintain optimal performance even during peak data loads. Continuous monitoring and optimization of the pipeline will be imperative to keep up with the evolving demands of the data and ensure seamless data delivery for downstream applications.

**Motivation**: Integrate the Kafka Message Queue into our solution to benchmark it's suitability for a high caliber level data pipeline - ingesting log data from the Super Hero Combat Simulator and sinking them to local JSON files.

## Prerequisites

### Superhero Combat Data Simulator

This exercise requires a Data Simulator to generate data in order to stream logging data through the Pub/Sub & MQ Services. We will utilize the [Super Hero Combat Data Simulator](/data-simulators/superhero-combat) offered in this earlier post. Please refer back to [This post](/data-simulators/superhero-combat#objective) to familiarize yourself with how to operate the Data Simulator.

### Logging and Transport

The generation of data is possible via the logging message transport. The Data Simulator is purposefully designed in this experiment to customize the logging emit hook and transport messages to a variety of down-stream applications. a recap can be found in the [Pub/Sub & MQ Comparison Article](/pub-sub/comparison). The logging transport allows us to seamlessly decouple the application development work-streams separately from the data streaming capabilities in order to focus on the Data Architecture and value creation from data insights.

### Kafka

[Kafka](https://kafka.apache.org/documentation/) is a distributed, fault-tolerant, and high-throughput event-streaming system designed to handle real-time data streams efficiently. Developed by LinkedIn and later open-sourced, Kafka operates on a producer-consumer model, allowing producers to send records to topics, which are then consumed by workers. It provides scalable and durable storage for data, acting as a buffer between various components in data pipelines and facilitating seamless data integration across applications. Kafka's ability to handle massive amounts of data in a fault-tolerant manner has made it a fundamental component of modern data architectures, supporting diverse use cases such as log aggregation, event streaming, and real-time analytics.

## Install

Clone this repository to local computer **[py-superhero-pubsub](https://github.com/jg-ghub/py-superhero-pubsub)** {{< svg-icon "github" >}}
```bash
git clone https://github.com/jg-ghub/py-superhero-pubsub
```

## Services
{{< details "Schematic" >}}
  {{< get-html "content/pub-sub/kafka/worker-schematic.html" >}}
{{</ details >}}

### Worker

**Description**
In this assessment comparing Kafka as the message distribution technology, we draw on our prior experience with the [Logging Transport](#logging-and-transport). To start using Kafka effectively, our initial step is to integrate the Kafka producer functionality into the Server logging transport. Just as we did with [Redis](/pub-sub/comparison#logging-transport), we create a new Kafka logger transport that generates messages and sends them to Kafka using the logging emit hook.
{{< details "Github Code" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Frethinkr-hub%2Fpy-superhero-pubsub%2Fblob%2Fmain%2Fkafka%2Flib%2Futils%2Floggers%2Fkafka.py&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details>}}

Similar to [RabbitMQ](/pub-sub/rabbutmq), we instantiate the Kafka logger `lib.utils.logger kafka.py` module and start an asynchronous `aiokafka` package to deliver messages to Kafka in its own IO loop. This Kafka logging transport will run asynchronously and alleviate any potential bottlenecks in the Data Simulator.

The Producer and Consumer classes can be found in the `lib.pubsub.kafka` module. Notice how the `Kafka_Consumer` class has been designed with `callback_func` so we can easily plug-and-play the original data ingestion pipeline, which follows the [Factory Method](/pub-sub/redis#worker) pattern we discussed in earlier articles.

{{< details "Github Code" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Frethinkr-hub%2Fpy-superhero-pubsub%2Fblob%2Fmain%2Fkafka%2Flib%2Fpubsub%2Fkafka.py&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

This allows us to use the same data ingestion pipeline for storing game state from Redis to our local data sink.
{{< details "Github Code" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Frethinkr-hub%2Fpy-superhero-pubsub%2Fblob%2Fmain%2Fkafka%2Fworker.py&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

#### Local Deployment
To test/debug the worker service, we can start up the services required to begin generating data for consumption by executing the following locally in bash. Once the server is up and running, we can deploy as many players as we'd like

**Note** make sure that Kafka is not only up, but accepting publications as well. The HeartBeat for Kafka uptime isn't a proper signal that it's ready to receive Topic publications.

```bash
export $(cat kakfka/.env) && \
export PLAYERS=10 && \

docker-compose --env-file kafka/.env -f kafka/compose/docker-compose.kafka.yml up -d zookeeper kafka kafka-ui
#Wait for Kafka-ui to be up and connected to Kafka cluster
docker-compose -f kafka/compose/docker-compose.kafka.yml up -d redis build_db superhero_server superhero_nginx player --scale player=${PLAYERS}
```

Then we can begin deploying worker services locally for debugging with

```bash
export $(cat kafka/.env) && \
export WORKER_CHANNEL=lib.server.lobby && \
python3 kafka/worker.py
```

**Note** 

Kafka's performance can be measured with the Kafka UI application. Kafka UI is spun up along side Kafka in the provided docker-compose, and can be accessed with this link [localhost:8081](localhost:8081)

## Objective

Here we're going to evaluate how Kafka operates within our Data Simulation. We can deploy all the same trial experiments run against Rabbit MQ with these [Production Deployment](#production-deployment) scenarios below

#### Production Deployment
{{< details "Video Demonstration" >}}
  {{< google-drive-video "1-QHW33LD6SOpVINdn4NWLvnnSizJC6dD" >}}
{{< /details >}}

Spinning up a production application with docker-compose is simple. Follow the commands below, replacing ${PLAYERS} with the required participants.

```bash
export PLAYERS=10
docker-compose --env-file kafka/.env -f kafka/compose/docker-compose.kafka.yml build && \
docker-compose --env-file kafka/.env -f kafka/compose/docker-compose.kafka.yml up -d zookeeper kafka kafka-ui
#Wait for Kafka-ui to be up and connected to Kafka cluster
docker-compose --env-file kafka/.env -f kafka/compose/docker-compose.kafka.yml up --scale player=${PLAYERS}
```
#### Production Deployment with Multiple Workers
```bash
export PLAYERS=10
export WORKERS=4
docker-compose --env-file kafka/.env -f kafka/compose/docker-compose.kafka.yml up --scale worker=${WORKERS} --scale player=${PLAYERS}
```

#### Production Deployment with Dead Workers\
{{< details "Video Demonstration" >}}
  {{< google-drive-video "1psVs04ij2zKJ1lhhd5_0FDbsXwiXDzY7" >}}
{{< /details >}}

```bash
export PLAYERS=10
docker-compose --env-file kafka/.env -f kafka/compose/docker-compose.kafka.yml up --scale worker=0 --scale player=${PLAYERS}
```

*After the simulation has completed*
```bash
export WORKERS=4
docker-compose --env-file kafka/.env -f kafka/compose/docker-compose.kafka.yml up --scale worker=${WORKERS}
```

Unsurprisingly, Kafka has performed the same as RabbitMQ against the same types of scenarios. In order to truly understand which Message Queue system is truly better with our Data Simulator, we would really need to scale up operations. Perhaps in a future article we can attempt to find who's more performant. Until then we can say that both satisfy our current requirements, but do provide different strengths and weaknesses in their own. Many of Kafka's favorable traits include

 * **Scalability and Performance** Kafka is designed to handle high-throughput, real-time data streams and is known for its exceptional performance and scalability. It can efficiently handle millions of messages per second across distributed environments. Kafka's architecture is based on distributed commit logs, allowing it to scale easily by adding more brokers to the cluster.

 * **Storage and Retention** Kafka retains messages for a configurable period of time, even after they have been consumed. This characteristic makes Kafka suitable for use cases where data needs to be stored for longer periods, such as log aggregation and data analytics. RabbitMQ, on the other hand, typically relies on external systems for message persistence and may not retain messages in the same way as Kafka.

 * **Data Stream Processing** Kafka provides built-in support for stream processing through its Kafka Streams API and integration with Apache Flink, Apache Spark, and other stream processing frameworks. This makes it a popular choice for real-time data processing and analytics applications.

 * **Message Durability** Kafka offers strong durability guarantees by replicating messages across multiple brokers in the cluster. This ensures that messages are not lost even if some brokers fail. RabbitMQ, by default, relies on message acknowledgments and relies on the backing storage system for durability.

 * **Partitioning** Kafka partitions data across multiple brokers, which allows for parallel processing of messages. This enables better load balancing and can improve overall system performance. RabbitMQ does not offer the same level of partitioning natively.

 * **Use Cases** Kafka is particularly well-suited for use cases involving real-time event streaming, log aggregation, complex event processing, and building data pipelines. Its Message Queue model and distributed architecture are ideal for scenarios where high scalability and low latency are critical.

It's essential to note that while Kafka excels in certain areas, RabbitMQ shines in other use cases. RabbitMQ is a mature, easy-to-use message broker that works well for traditional messaging patterns. It's often preferred when you need a simple and straightforward message queuing system without the complexities of handling massive data streams.

Ultimately, the choice between Kafka and RabbitMQ depends on your specific requirements, existing infrastructure, and the nature of the applications you're building. Many organizations use both technologies within their systems to leverage the strengths of each platform where they are best suited.