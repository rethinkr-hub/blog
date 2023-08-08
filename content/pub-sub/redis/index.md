---
title: "Publish Subscribe - Redis"
date: "2023-07-25"
draft: true
tags: ["Redis", "Pub/Sub"]
author: "Jason Grein"
description: "Evaluating Redis for Publish Subscribe Functionality"

cover:
  image: "cover-photo.png"
#  alt: "AI Generated Superhero Photo"
#  caption: "These guys are ready to duke it out for Data!"
#  relative: false # To use relative path for cover image, used in hugo Page-bundles

showToc: true
TocOpen: false
hidemeta: false
comments: false

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

Here's the first post in a series of Publish/Subscribe and Message Queue service evaluations. We're looking to identify whether Redis' native Pub/Sub functionality is a feasible option for creating a scalable data ingestion pipelines. In this demonstration, a Redis Publisher has been hooked into the Redis Logger Module Transport to stream log data to Redis Pub/Sub. We now have a plugin to stream raw log data for future processing, which is now where the Subscriber workers come into play.

At this point in time, it's important to distinguish the unique traits between some subtly different topics on data movement. In particular: data ingestion vs data sinks. The fundamental difference between a data sink and data ingestion lies in their roles and functionalities within a data processing system. A data sink refers to the endpoint or destination where data is collected, stored, or ultimately ends up after processing. It is the final stage of data flow in a system, often utilized for analysis, reporting, or archival purposes. On the other hand, data ingestion refers to the process of gathering and importing data from various sources into a data processing system. It serves as the initial step where raw data from diverse origins is collected, organized, and prepared for subsequent processing, analysis, or storage in the designated data sink. In essence, data ingestion focuses on bringing data into the system, while a data sink concentrates on receiving and holding the processed or refined data.

**Motivation**: Define a simple pipeline which utilizes Redis' Pub/Sub capabilities to benchmark it's suitability for a high caliber level data pipeline - ingesting log data from the Super Hero Combat Simulator and sinking them to local JSON files.

## Prerequisites

### Superhero Combat Data Simulator

This exercise requires a Data Simulator to generate data in order to stream logging data through the Pub/Sub & MQ Services. We will utilize the [Super Hero Combat Data Simulator](/data-simulators/superhero-combat) offered in this earlier post. Please refer back to [This post](/data-simulators/superhero-combat#objective) to familiarize yourself with how to operate the Data Simulator.

### Logging and Transport

The generation of data is possible via the logging message transport. The Data Simulator is purposefully designed in this experiment to customize the logging emit hook and transport messages to a variety of down-stream applications. a recap can be found in the [Pub/Sub & MQ Comparison Article](/pub-sub/comparison). The logging transport allows us to seamlessly decouple the application development work-streams separately from the data streaming capabilities in order to focus on the Data Architecture and value creation from data insights.

### Redis

[Redis](https://redis.io/docs/getting-started/) memory cache is a high-performance, in-memory data structure store that serves as a key-value database, allowing for lightning-fast access to frequently used data. As a cache, Redis keeps frequently requested information readily available in RAM, significantly reducing the need to retrieve data from slower disk-based storage systems. This enables applications to achieve remarkable speed and efficiency, making Redis an ideal solution for accelerating data-intensive operations and improving overall application performance. We will extend on the learning from the Super Hero Combat Data Simulator.

## Install

Clone this repository to local computer **[py-superhero-pubsub](https://github.com/jg-ghub/py-superhero-pubsub)** {{< svg-icon "github" >}}
```bash
git clone https://github.com/jg-ghub/py-superhero-pubsub
```

## Services
{{< details "Schematic" >}}
  {{< get-html "content/pub-sub/redis/worker-schematic.html" >}}
{{</ details >}}

### Worker

**Description**
Here we expand on the [Original Worker](/data-simulators/superhero-combat#workers) service to not only clean up stale records, but also persist the log messages to a local data sink. At the end of each game, the Server publishes a message to the `lib.server.lobby` channel in Redis where this worker subscribes to. The worker subscribes messages containing `'Cleaning Game Records'`, which signifies that the game has completed and can proceed to extract and load the game state from Redis to JSON in the local data sink.

We already touched on the [Logging Transport](#logging-and-transport) in the [Prerequisites](#prerequisites), which links to the more in-depth article to explain of how messages are published for the worker service to listens to. Now, we'll look more closely into our worker's Subscription operations. We can first start off with our worker's [Factory Method](https://realpython.com/factory-method-python/) pattern. This design pattern allows us to interchange a Subscription class like `Redis_Subscriber` and a common routine into a worker script which continuously waits for instructions.
{{< details "Github Code" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fjg-ghub%2Fpy-superhero-pubsub%2Fblob%2Fmain%2Fredis%2Fworker.py&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

The Subscriber class will connect to its respective messaging application (in this case Redis), and wait for messages to be delivered. Once a message is received, the Factory Method `callback_function` is executed against the message, and in this case the message is log data which is passed to an ingestion pipeline.
{{< details "Github Code" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fjg-ghub%2Fpy-superhero-pubsub%2Fblob%2Fmain%2Fredis%2Flib%2Fpubsub%2Fredis.py&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{< /details >}}

This design pattern allows us to interchange a Subscription class like `Redis_Subscriber` and a common routine into a worker script which continuously waits for instructions. This pattern helps us integrate a common routine, like ETL, and ingestion pipelines into different Pub/Sub applications with their own unique Subscription class. Now that we've found a simple way to decouple different Pub/Sub applications with our Factory Method class and script, next let's look at what our common data ingestion pipeline looks like.

Like mentioned earlier, we've expanded on the original worker service so the first function `clean` is coming from our original worker script. Further down, we've created some new common data cleaning functions to run streaming ETL tasks across sub-categories of game state like Game Meta Data & Game Podium Data. These functions help to organize/de-duplicate specific game state from redis so we can have cleaner data for analysis from our data sink. Finally, we have the `sub_routine` function which is our common method in the worker factory. This helps us to transform/decode the messages from bytes to JSON content before cleaning and then loading into the local data sink.
{{< details "Github Code" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fjg-ghub%2Fpy-superhero-pubsub%2Fblob%2Fmain%2Fredis%2Flib%2Fworker%2F__init__.py&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

Behind the scenes for these common data ingestion routines are some Parsing libraries which pre-filter/manipulate the game state in order to enhance the log data for future pipelines, but is out-of-scope for this analysis.

#### Local Deployment
To test/debug the worker service, we can start up the services required to begin generating data for consumption by executing the following locally in bash.

```bash
export $(cat redis/.env) && \
export PLAYERS=10 && \
docker-compose -f redis/compose/docker-compose.redis.yml up -d redis build_db superhero_server superhero_nginx player --scale player=${PLAYERS}
```

Then we can begin deploying worker services locally for debugging with

```bash
export $(cat redis/.env) && \
export WORKER_CHANNEL=lib.server.lobby && \
python3 worker.py
```

## Objective

In this demonstration, we want to evaluate how well Redis can handle Pub/Sub message passing & processing with the Data Simulator. Redis will be implemented into the Logger Transport to publish messages to the `lib.server.lobby` channel, and similarly subscribed to transform and pipe data to a local data sink for future processing. We're less concerned about performance benchmarking opposed to suitably meeting our needs. We need a framework which is going to be able to stream data efficiently with scale and provide transactional data integrity (no lost messages).

Redis Pub/Sub offers a very simplistic and easy to use messaging framework out-of-the-box. It's already acting as the Super Hero Combat Data Simulator's DB Key Store on the application side, so making use of it's Pub/Sub feature is a *no-brainer*. We can deploy a trial experiment running a [Production Deployment](#production-deployment---single-worker) with a single worker following the script below. 

#### Production Deployment - Single Worker
{{< details "Video Demonstration" >}}
  {{< google-drive-video "1hDwU8IpLlkZx-xlH2DI_W0QfYhxF05mP" >}}
{{< /details >}}

Spinning up a production application with docker-compose is simple. Follow the commands below, replacing ${PLAYERS} with the required participants.

```bash
export PLAYERS=10
docker-compose --env-file redis/.env -f redis/compose/docker-compose.redis.yml build && \
docker-compose --env-file redis/.env -f redis/compose/docker-compose.redis.yml up --scale player=${PLAYERS}
```

Running this application on a local environment, using a small amount of Player Services, with Redis' built-in Pub/Sub works wonders - it handles the payload effectively. 

Next we need to be able to scale the Deployment with multiple workers. we can deploy a similar trial experiment run a [Product Deployment](#production-deployment---multiple-workers) with multiple workers. We can scale the workers with the following

#### Production Deployment - Multiple Workers
{{< details "Video Demonstration" >}}
  {{< google-drive-video "1-z9myZXMu-H8CnC2XOGjmYeUjSy4M8-d" >}}
{{< /details >}}

```bash
export PLAYERS=10
export WORKERS=4
docker-compose --env-file redis/.env -f redis/compose/docker-compose.redis.yml build && \
docker-compose --env-file redis/.env -f redis/compose/docker-compose.redis.yml up --scale worker=${WORKERS} --scale player=${PLAYERS}
```

This trial experiment unfortunately suffers from scalability issues when we try to add more workers attempting to create some efficiency. Redis' simplicity effectively breaks the out-of-the-box solution for our needs. The Redis Pub/Sub mechanism is a Broadcast message to all Subscribers listening to our channel `lib.server.lobby`, which means that all workers receive the same message. This doesn't help provide us more efficiency when we scale our workers - instead it causes duplication.

We can verify the duplications in filename pattern `%GAME TOKEN%_%HOST%` running the following

```bash {linenos=table,hl_lines=["2","24"]}
docker exec -it compose-worker-1 ls data/superhero/GAME_PODIUM
00cb4536-10ff-4df5-a358-a0eca45451cf_c434711b159f.gz  3719ced6-1746-4615-a9da-9136a76814f4_c434711b159f.gz
8412f6ab-b8ac-48be-b3af-a120254d9941_c434711b159f.gz  aa06fe55-72eb-4a12-a6b5-329f1fb3298c_c434711b159f.gz
0cf6a6f1-9c12-471b-ad64-7e1ba359d13c_c434711b159f.gz  3b3a1816-4f6b-47d0-80a9-6540b6c3a72d_c434711b159f.gz
87802b81-e513-4bf8-9843-f6868419ed61_c434711b159f.gz  b4de9241-a71b-4c4e-8afd-7fd93ecaa527_c434711b159f.gz
1119e940-e6af-46ca-8c8a-9b5b2486e511_c434711b159f.gz  43ed338a-2d08-47f4-9c78-d5fbb875a4b8_c434711b159f.gz
8cdf6df0-ce14-4228-a94c-b47ed44d6842_c434711b159f.gz  ba829501-6f88-4d39-83ba-3a50ade19ad6_c434711b159f.gz
163d7ec9-c04c-4c84-b5a8-a95dbbe2dff6_c434711b159f.gz  4911add9-3734-43fa-8e7b-afebc4e1fb1a_c434711b159f.gz
8d85d184-cca5-4551-9f21-df4225febc34_c434711b159f.gz  cb60288e-9d3c-4219-bd36-06381ac74854_c434711b159f.gz
18c16771-37a3-4ea8-b294-b8e8200d98ca_c434711b159f.gz  4c6b65c0-b638-4828-83ff-9e36a21af199_c434711b159f.gz
95dc03d2-8939-4158-ad06-dc5d468439e3_c434711b159f.gz  cd29510f-c235-46f3-b9a8-d00b376d433d_c434711b159f.gz
22500bbc-dcbc-47c6-87f5-df4bd847c55b_c434711b159f.gz  520ee98d-2801-40c9-91ba-bd7c60db252d_c434711b159f.gz
9b0d3f76-5b20-4266-99b9-91fa66d65170_c434711b159f.gz  d2b845d7-3c68-4c30-99eb-851589575d17_c434711b159f.gz
26b99668-b4e9-4780-9cb9-55f0aab4a274_c434711b159f.gz  5f0c8b9c-9361-4540-9edf-0827a8dc3e37_c434711b159f.gz
9c04e4de-1072-446e-8283-7947ccf60a30_c434711b159f.gz  dc2b9a65-19c9-44ba-b571-4d433d9ff55c_c434711b159f.gz
317a5d68-597e-423d-94ad-66ce255d19f0_c434711b159f.gz  65b0b2c3-e99f-450b-8fd3-04de803c0ea7_c434711b159f.gz
9c0f0b94-9f3c-424a-b003-e22911a998bd_c434711b159f.gz  e64c32dc-c198-4115-86bf-f37a0b675864_c434711b159f.gz
323d24f9-d123-4d0c-b962-2b89e76e761d_c434711b159f.gz  6934141d-9f87-49b6-90b5-5ad28e5e7ccd_c434711b159f.gz
9cdd2577-5a2c-4eeb-b513-3eaed3c76cbc_c434711b159f.gz  f22883ec-43f9-4392-95de-84f939b598f2_c434711b159f.gz
345927c1-77fa-4104-b1ff-1751d9a6f693_c434711b159f.gz  7588cad0-4f8f-4807-bdbc-0ae304e21ed9_c434711b159f.gz
a40bec3f-6267-4e46-9e28-71ff08ece46c_c434711b159f.gz  f81e79d5-379b-4540-b98b-3ce49424dc0a_c434711b159f.gz

docker exec -it compose-worker-2 ls data/superhero/GAME_PODIUM
00cb4536-10ff-4df5-a358-a0eca45451cf_4123ba141fe8.gz  3719ced6-1746-4615-a9da-9136a76814f4_4123ba141fe8.gz
8412f6ab-b8ac-48be-b3af-a120254d9941_4123ba141fe8.gz  aa06fe55-72eb-4a12-a6b5-329f1fb3298c_4123ba141fe8.gz
0cf6a6f1-9c12-471b-ad64-7e1ba359d13c_4123ba141fe8.gz  3b3a1816-4f6b-47d0-80a9-6540b6c3a72d_4123ba141fe8.gz
87802b81-e513-4bf8-9843-f6868419ed61_4123ba141fe8.gz  b4de9241-a71b-4c4e-8afd-7fd93ecaa527_4123ba141fe8.gz
1119e940-e6af-46ca-8c8a-9b5b2486e511_4123ba141fe8.gz  43ed338a-2d08-47f4-9c78-d5fbb875a4b8_4123ba141fe8.gz
8cdf6df0-ce14-4228-a94c-b47ed44d6842_4123ba141fe8.gz  ba829501-6f88-4d39-83ba-3a50ade19ad6_4123ba141fe8.gz
163d7ec9-c04c-4c84-b5a8-a95dbbe2dff6_4123ba141fe8.gz  4911add9-3734-43fa-8e7b-afebc4e1fb1a_4123ba141fe8.gz
8d85d184-cca5-4551-9f21-df4225febc34_4123ba141fe8.gz  cb60288e-9d3c-4219-bd36-06381ac74854_4123ba141fe8.gz
18c16771-37a3-4ea8-b294-b8e8200d98ca_4123ba141fe8.gz  4c6b65c0-b638-4828-83ff-9e36a21af199_4123ba141fe8.gz
95dc03d2-8939-4158-ad06-dc5d468439e3_4123ba141fe8.gz  cd29510f-c235-46f3-b9a8-d00b376d433d_4123ba141fe8.gz
22500bbc-dcbc-47c6-87f5-df4bd847c55b_4123ba141fe8.gz  520ee98d-2801-40c9-91ba-bd7c60db252d_4123ba141fe8.gz
9b0d3f76-5b20-4266-99b9-91fa66d65170_4123ba141fe8.gz  d2b845d7-3c68-4c30-99eb-851589575d17_4123ba141fe8.gz
26b99668-b4e9-4780-9cb9-55f0aab4a274_4123ba141fe8.gz  5f0c8b9c-9361-4540-9edf-0827a8dc3e37_4123ba141fe8.gz
9c04e4de-1072-446e-8283-7947ccf60a30_4123ba141fe8.gz  dc2b9a65-19c9-44ba-b571-4d433d9ff55c_4123ba141fe8.gz
317a5d68-597e-423d-94ad-66ce255d19f0_4123ba141fe8.gz  65b0b2c3-e99f-450b-8fd3-04de803c0ea7_4123ba141fe8.gz
9c0f0b94-9f3c-424a-b003-e22911a998bd_4123ba141fe8.gz  e64c32dc-c198-4115-86bf-f37a0b675864_4123ba141fe8.gz
323d24f9-d123-4d0c-b962-2b89e76e761d_4123ba141fe8.gz  6934141d-9f87-49b6-90b5-5ad28e5e7ccd_4123ba141fe8.gz
9cdd2577-5a2c-4eeb-b513-3eaed3c76cbc_4123ba141fe8.gz  f22883ec-43f9-4392-95de-84f939b598f2_4123ba141fe8.gz
345927c1-77fa-4104-b1ff-1751d9a6f693_4123ba141fe8.gz  7588cad0-4f8f-4807-bdbc-0ae304e21ed9_4123ba141fe8.gz
a40bec3f-6267-4e46-9e28-71ff08ece46c_4123ba141fe8.gz  f81e79d5-379b-4540-b98b-3ce49424dc0a_4123ba141fe8.gz
```

In this example, the ETL jobs were run once on each worker service. In the highlighted rows above, the same `%GAME TOKEN%` JSON is seen in multiple workers meaning that the task was processed more than one. So, this solution isn't viable for our needs. Instead, we would like to have our workers take tasks from a queue whereby they run unique operations simultaneously, but never the identical task. We've validated some other popular services within the blog for you to explore.
