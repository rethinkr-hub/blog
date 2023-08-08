---
title: "Publish Subscribe - Redis Narrowcast"
date: "2023-07-26"
draft: true
tags: ["Redis", "Pub/Sub", "Narrowcast"]
author: "Jason Grein"
description: "Pushing Redis Closer to a Proper Pub/Sub Messaging Service"

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

In the last article, we ran a trial experiment using Redis' out-of-the-box Pub/Sub capabilities to build a streaming data ingestion pipeline to a local data sink. Unfortunately, this capability didn't reach our expectations when we noticed that the streaming JSON files were duplicated across all the workers. We aim to achieve "At Most Once" data processing in our data ingestion pipeline. 

Without giving up immediately on using Redis as our solution, we can try to add on-top of the last solution with a "Middleware" worker which can narrowcast messages to unique workers instead of the natively broadcasting to all workers. Middleware refers to software that acts as an intermediary layer between different applications or systems, facilitating communication and data exchange in a seamless and standardized manner. Its primary purpose is to enhance interoperability and efficiency in distributed computing environments. One of the key capabilities of middleware is its ability to narrowcast messages, which means delivering information selectively to a targeted group of recipients rather than broadcasting it to all connected devices or applications. This targeted message dissemination ensures that only the intended recipients receive and process the information, reducing network congestion, improving data security, and optimizing resource utilization in large-scale distributed systems. Narrowcasting through middleware is particularly advantageous in scenarios where specific data needs to be delivered to a limited set of users, devices or services without causing unnecessary overhead on the network or persistence layer.

**Motivation**: Define a less simple pipeline which builds on-top of Redis' Pub/Sub capabilities with a middleware narrowcasting worker to benchmark it's suitability for a high caliber level data pipeline - ingesting log data from the Super Hero Combat Simulator and sinking them to local JSON files.

## Prerequisites

### Superhero Combat Data Simulator

This exercise requires a Data Simulator to generate data in order to stream logging data through the Pub/Sub & MQ Services. We will utilize the [Super Hero Combat Data Simulator](/data-simulators/superhero-combat) offered in this earlier post. Please refer back to [This post](/data-simulators/superhero-combat#objective) to familiarize yourself with how to operate the Data Simulator.

### Logging and Transport

The generation of data is possible via the logging message transport. The Data Simulator is purposefully designed in this experiment to customize the logging emit hook and transport messages to a variety of down-stream applications. a recap can be found in the [Pub/Sub & MQ Comparison Article](/pub-sub/comparison). The logging transport allows us to seamlessly decouple the application development work-streams separately from the data streaming capabilities in order to focus on the Data Architecture and value creation from data insights.

### Redis

Redis memory cache is a high-performance, in-memory data structure store that serves as a key-value database, allowing for lightning-fast access to frequently accessed data. As a cache, Redis keeps frequently requested information readily available in RAM, significantly reducing the need to retrieve data from slower disk-based storage systems. This enables applications to achieve remarkable speed and efficiency, making Redis an ideal solution for accelerating data-intensive operations and improving overall application performance. We will extend on the learning from the Super Hero Combat Data Simulator.

## Install

Clone this repository to local computer **[py-superhero-pubsub](https://github.com/jg-ghub/py-superhero-pubsub)** {{< svg-icon "github" >}}
```bash
git clone https://github.com/jg-ghub/py-superhero-pubsub
```

## Services
{{< details "Schematic" >}}
  {{< get-html "content/pub-sub/redis-narrowcast/worker-schematic.html" >}}
{{</ details >}}

### Worker

**Description**
{{< details "Github Code" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Fjg-ghub%2Fpy-superhero-pubsub%2Fblob%2Fmain%2Fredis-fanout%2Fworker_fanout.py&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

Building on the last solution, we're now creating a middleware worker to distribute Redis' natively broadcasted messages to individual workers, which is being coined a *Narrowcast* in this solution. How does this work? Like a normal worker in the last solution, we create a Subscriber to listen to the `lib.server.lobby` channel in Redis' Pub/Sub. Once a message is received, it scans all possible workers dedicated to streaming data, and narrowly publish to a random worker channel.

The data streaming task is exactly the same as the old worker's task. However, the only difference is that it's listening to its own unique channel

```python
subscriber.run(channel='%s.%s' % (WORKER_CHANNEL, HOSTNAME))
```

#### Local Deployment
To test/debug the worker service, we can start up the services required to begin generating data for consumption by executing the following locally in bash.

```bash
export $(cat redis_narrowcast/.env) && \
export PLAYERS=10 && \
docker-compose -f redis_narrowcast/compose/docker-compose.redis-narrowcast.yml up -d redis build_db superhero_server superhero_nginx player --scale player=${PLAYERS}
```

Then we can begin deploying worker services locally for debugging with

```bash
export $(cat redis_narrowcast/.env) && \
export WORKER_CHANNEL=lib.server.lobby && \
python3 redis_narrowcast/worker_narrowcast.py
```

```bash
export $(cat redis_narrowcast/.env) && \
export WORKER_CHANNEL=lib.server.lobby && \
python3 redis_narrowcast/worker.py
```

## Objective

We understand the short comings of Redis' Pub/Sub natively broadcasting messages to all Subscribers, but now we can validate how a Narrowcast middleware does to evenly distribute work amongst the workers to end the cycle of duplication. We can deploy a trial experiment running a [Production Deployment](#production-deployment---single-worker) with a single worker following the script below. 

#### Production Deployment - Single Worker
Spinning up a production application with docker-compose is simple. Follow the commands below, replacing ${PLAYERS} with the required participants.

```bash
export PLAYERS=10
docker-compose --env-file redis_narrowcast/.env -f redis_narrowcast/compose/docker-compose.redis-narrowcast.yml build && \
docker-compose --env-file redis_narrowcast/.env -f redis_narrowcast/compose/docker-compose.redis-narrowcast.yml up --scale player=${PLAYERS}
```

Running this application on a local environment, using a small amount of Player Services, provides a similar result set to the previous test with Redis native Pub/Sub. This is good as the middleware is working without scale.

Next we need to be able to scale the Deployment with multiple workers. We can deploy a similar trial experiment by running a [Product Deployment](#production-deployment---multiple-workers) with multiple workers listening to a middleware Narrowcaster. We can scale the workers with the following

### Production Deployment - Multiple Workers
{{< details "Video Demonstration" >}}
  {{< google-drive-video "1BKw4PlDsoYeGXnM7nGaexGigN_aTVLVL" >}}
{{< /details >}}

```bash
export PLAYERS=10
export WORKERS=4
docker-compose --env-file redis_narrowcast/.env -f redis_narrowcast/compose/docker-compose.redis-narrowcast.yml build && \
docker-compose --env-file redis_narrowcast/.env -f redis_narrowcast/compose/docker-compose.redis-narrowcast.yml up --scale worker=${WORKERS} --scale player=${PLAYERS}
```

This trial run has observed some positive results!

```bash
docker exec -it compose-worker-1 ls data/superhero/GAME_PODIUM
028cc172-ce6e-444a-ad7f-d0f60651aee4_016cea10bfcb.gz  20eacc76-c0b4-4968-8963-149a0a0ebf4d_016cea10bfcb.gz
d16d2f54-022a-4eab-a7ed-060bad6753ec_016cea10bfcb.gz  fbfc2120-f178-4e8c-85b5-99f0b91a9aae_016cea10bfcb.gz
0892bc91-834b-439b-b382-08882b7ae489_016cea10bfcb.gz  6351bcba-3e18-45b4-aab9-625faf2db56a_016cea10bfcb.gz
d245d11a-b2f1-4ddc-abc9-e08400b3e2a9_016cea10bfcb.gz  146d1f50-da16-47e4-bdcc-639ab8d25229_016cea10bfcb.gz
afdc2bb6-c302-4847-b8f0-fe75f2bb5f90_016cea10bfcb.gz  f49fb27d-0c16-4d71-8c9b-56df195f8d0c_016cea10bfcb.gz

docker exec -it compose-worker-2 ls data/superhero/GAME_PODIUM
24d206e8-f8d2-4c09-a591-0e7edbe27d6b_6265457077b3.gz  7a071df1-044c-4e58-ad9d-e10e4ac4e3f4_6265457077b3.gz
a3c4d894-5248-4abf-9827-1463eb59f780_6265457077b3.gz  5d97cd46-285d-47cc-a1ac-e290c310b571_6265457077b3.gz
80ecd360-ab17-4a4d-a99b-6d34ee34264d_6265457077b3.gz  dcac55d8-c5c9-4cb6-bd78-83cf736484df_6265457077b3.gz
64822852-1164-4619-a7ce-1fe956c9e231_6265457077b3.gz  8a0d081f-8abe-4fdf-8a4d-ccc9381617cf_6265457077b3.gz
f4084ab7-9335-471f-aa73-c377ecbb16cb_6265457077b3.gz

docker exec -it compose-worker-3 ls data/superhero/GAME_PODIUM
0bb9bbde-d329-425c-adc7-b56d9b7a5869_6327963e7360.gz  60489b45-dd8c-4414-b49c-2c47215fbe28_6327963e7360.gz
8e0b34b7-b535-4ac8-b823-541977b0fe26_6327963e7360.gz  cb214239-233d-45bf-9a35-204fd0ecb730_6327963e7360.gz
566acc9c-f48a-48ab-aec5-b1e524f85b6f_6327963e7360.gz  71bc9451-a7d9-4c87-bf94-d4290e6da661_6327963e7360.gz
b067a742-71ab-4f87-b821-794cff69e777_6327963e7360.gz  f0c047bb-c371-46ce-afc4-75d56edbadf4_6327963e7360.gz

docker exec -it compose-worker-4 ls data/superhero/GAME_PODIUM
01085fd7-e3a4-48a5-bbbb-04da114a4146_46e3cbfe99f2.gz  01500a78-1ab4-4df5-84b7-e0e6c2b951fb_46e3cbfe99f2.gz
141dbb59-d1c7-4a00-a08e-6fc97510fafb_46e3cbfe99f2.gz  23a977dc-5dfa-4d04-ac64-8c61b9488109_46e3cbfe99f2.gz
6be21d0f-6610-4ed3-8a2b-f3ffdba5fd5d_46e3cbfe99f2.gz  9cbef64e-65e1-4297-8ca1-b481fd2caad1_46e3cbfe99f2.gz
a5e579f3-d208-4df3-a13a-c4da2a4c3748_46e3cbfe99f2.gz  c0b87cc3-d4df-4e68-b9c6-cd28860bba9b_46e3cbfe99f2.gz
cede7812-39cd-4a44-a203-1aaa68fa65cd_46e3cbfe99f2.gz  d5838d7a-d10c-486b-9d0a-9c691ce498b7_46e3cbfe99f2.gz
dcccc8a5-e7b0-4269-891d-e87bce6550d0_46e3cbfe99f2.gz
```

We can see that each worker has been tasked with unique `%GAME TOKEN%`s that allowed them to stream data ingestion pipelines to the local data sink independently. This means the log data is processed only once by a single worker, and we no longer see this solution suffering from duplicative efforts. We've come one step closer to building a Pub/Sub service which can scale our streaming data ingestion pipeline. However, this demonstration doesn't actually replicate a message queue system which is an issue in terms of sustainability.

We can see better fail-over support and resiliency in message delivery/acknowledgement with other open source solutions. Remember, if no worker is listening to the Broadcast/Narrowcast, then the message is never processed and lost forever. This is where other popular solutions will be evaluated and chosen, instead of trying to recreate the wheel with Redis. With that said, this Narrowcast solution is incredibly light weight and could be incredibly useful for applications which can withstand some message loss.

Next we'll explore some open source solutions and how they can leverage the Data Simulator's streaming data ingestion pipeline.
