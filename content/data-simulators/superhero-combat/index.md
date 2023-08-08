---
title: "Data Simulator - Super Hero Combat"
date: "2023-07-13"
draft: false
tags: ["Redis", "Data Simulator"]
author: "Jason Grein"
description: "Guide on how to use this complimentary Data Simulator"

cover:
  image: "cover-photo.png"
  #alt: "AI Generated Superhero Photo"
  #caption: "These guys are ready to duke it out for Data!"
  relative: false # To use relative path for cover image, used in hugo Page-bundles

images:
  - cover-photo.png

videos:
  - superhero-combat-simulator.webm

showToc: true
TocOpen: false
hidemeta: false
comments: true

disableHLJS: true # to disable highlightjs
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

To understand the concept of Data Architecture, we first need data! The post here-in describes this free-to-use Data Simulator for generating real-time data about a pseudo arcade style multi-player fighter game.

Data simulators play a crucial role in the professional world by providing a powerful tool for testing, modeling, and predicting outcomes in various domains. These simulators generate synthetic data that closely resembles real-world data, enabling organizations to conduct extensive simulations without the need for large-scale data collection or risking the integrity of sensitive or proprietary information.

**Motivation**: In this post, we'll use the Super Hero Combat Data Simulator to provide hands-on experience in working with data. Simulated data allows for controlled learning experiences, where users can practice data analysis, modeling, and decision-making without the need for expensive datasets.

## Prerequisites

### Docker

Docker is an open-source platform that enables software developers to package their applications and their dependencies into self-contained units called containers. Each container is isolated and includes all the necessary components, such as libraries, code, and system tools, to run the application reliably and consistently across different computing environments. Docker provides a lightweight, flexible, and efficient solution for deploying, scaling, and managing applications, as containers can be easily deployed on various operating systems and infrastructure setups, allowing for seamless development, testing, and deployment workflows.

Docker is required to run the Data Simulator and deploy all the service containers locally. All the services are also defined in a docker-compose YAML configuration, and should be spun up with docker-compose. It's not necessary to know how to script Dockerfiles or how to make custom configurations in order to use this Data Simulator. Docker-Compose commands should as they're coded in this post.

**Install**
More details can be found in
  * [Docker Install](https://docs.docker.com/engine/install/?ref=rethinkr.ghost.io)
  * [Docker-Compose Install](https://docs.docker.com/compose/install/?ref=rethinkr.ghost.io)

### Redis

Redis is an open-source, in-memory data structure store that serves as a versatile and high-performance database and cache. It excels in providing lightning-fast data retrieval and manipulation by keeping the entire dataset in memory, ensuring swift access and low latencies. Redis supports various data structures, including strings, hashes, lists, sets, and sorted sets, enabling developers to build complex applications and implement advanced functionalities such as caching, real-time analytics, pub/sub messaging, and distributed locking. Its advanced features, such as persistence options, high availability, and cluster support, make Redis a reliable and scalable solution for handling large-scale data needs while maintaining remarkable performance.

Redis plays an important role in the Data Simulator both as a data store, and a pub/sub service all in one. To get a better understanding of all the data structures in this simulator, it's important to consult the Redis documentation and command index.

**Commands**
More details on Redis commands can be found in
  * [Redis Commands](https://redis.io/?ref=rethinkr.ghost.io)

## Install

Clone this repository to local computer **[py-superhero](https://github.com/jg-ghub/py-superhero)** {{< svg-icon "github" >}}
```bash
git clone https://github.com/jg-ghub/py-superhero
```

Next, register at [Super Hero API](https://www.superheroapi.com/?ref=rethinkr.ghost.io) to acquire a token in order to pull their Super Hero repository into the Data Simulator staging steps. Their API is free to register with, and a special thanks to this team for allowing free public use of their API!

Once an API token has been successfully acquired from Super Hero API, add it to the .env file in the root folder of `py-superhero` pulled from github earlier.
```bash
cd py-superhero
echo API_TOKEN=*[INSERT API TOKEN HERE]* > .env
```

## Services

### Servers
{{< details "Schematic" >}}
  {{< get-html "content/data-simulators/superhero-combat/player-server-schematic.html" >}}
{{< /details >}}

**Description**
The server instance will connect players together to begin combat. It will wait for players to login, and provide the number of contestants the player is willing to fight simultaneously. On login, the server will sort players into their own lobby room by registering them to a game token. If no lobby room exists with the configured amount of participants the player is requesting, then one will be created. Once the number of participants is satisfied, the server will call all players to begin play, and iteratively let each player know when their turn is.

The server instance also offers the meat of the logging transport capabilities. Two different logging transports are created at spin-up to log messages to the stdout stream, and publish messages to the Redis pubsub channels based on logger names. The worker tasks are published to the `lib.server.lobby` channel where the worker service will action its routines.

Upon completion of the game, the server will call all participants to indicate the game has finished. The players are then signaled to close connection. At this time, all that is left is to clean up the DB registries which was monitoring game activities.

**Local Deployment**
To test/debug these services, each service can be started locally by executing the script in bash. A Redis service must be running, and can easily be started separately from the other docker services. The following commands will have the server application running locally
```bash
docker-compose -f compose/docker-compose.prod.yml up -d redis && \
export $(cat .env) && \
python3 build_db.py && \
python3 server.py
```

The server will sit idle waiting to players to communicate. We can scale some players to initiate communication with the local server with docker-compose
```bash
export PLAYERS=10
docker-compose --env-file .env \
  -f compose/docker-compose.prod.yml up \
  --scale player=${PLAYERS}
```

### Players

**Description**
The player instance will connect to the server awaiting combat. The player provides a user identification, along with how many opponents it wants to play against in this current round. On login, if no user exists then the server will register the user. The user then waits for the server to slot all participants into the game and waits for acknowledgement that the game can start.

The user picks an action to send to the server, and then waits for the outcome of the combat against the selected opponent. After acknowledgement from the server in regard to the action, the user then waits for signal from the server for its next turn. If all opponents have been eliminated, then the server will broadcast back to all participants that the game has completed, and then the client will disconnect.

**Local Deployment**
We can start the required services with the following, and run some player instances locally with the following
```bash
docker-compose --env-file .env \
  -f compose/docker-compose.prod.yml up \
  -d redis build_db server &&
export $(cat .env) && python3 client.py
```

Multiple players are required to join a lobby before a game is initiated, so we can repeat the local player instance multiple times to get the players fighting
```bash
export $(cat .env) && python3 client.py
```

### Workers
{{< details "Schematic" >}}
  {{< get-html "content/data-simulators/superhero-combat/db-cache-schematic.html" >}}
{{< /details >}}

**Description**
This is where the clean up work is done. At the end of each game, the Server publishes a message to the `lib.server.lobby` channel in Redis where this worker subscribes to. The worker consumes messages from this channel, and applies expiry times to all Redis Keys (found in the API section) while deleting all other objects.

This worker has incredibly simple tasks for the purpose of illustrating the native Pub/Sub capability in Redis. We aim to expand on this application's worker service in later articles.

**Local Deployment**
To begin fanning out messages to the workers locally, we can initialize a session with
```bash
docker-compose --env-file .env \
  -f compose/docker-compose.prod.yml up \
  -d redis build_db server
```

Redis should be up at this point which the workers can listen to while they wait for pub messages from the server. It's important to note that the workers need to be listening to the pub/sub channel before messages are fanned out because Redis has no message queue nor acknowledgement capabilities to retain messages when there are no active listeners. In this scenario, missed messages would lead to a bloated cache.
```bash
export $(cat .env) && python3 worker.py
```

Once the workers are actively listening to Redis, we can deploy players to start the combat
```bash
export PLAYERS=10 && \
docker-compose --env-file .env \
  -f compose/docker-compose.prod.yml up \
  -d player \
  --scale player=${PLAYERS}
```

Multiple workers can be consuming from Redis' pub/sub simultaneously. However, messages are consumed by all listeners native to Redis in this fashion which is inefficient for what we're trying to accomplish.
```bash
export $(cat .env) && python3 worker.py
```

### Build DB

**Description**
A simple script to pull Super Hero attributes from the Super Hero API, clean the data and store in the Redis DB in the superheros Hash. The script by default looks for the superhero.json file found in this repository to reduce the pulls required from the API.

**Local Deployment**
In the case where we want to get an updated version of the Super Hero collection, then we can run the following script to re-pull and store in superhero.json.
```bash
export $(cat .env) && python3 build_db.py pull
```

## Objective

### Free Data Simulator
{{< details "Video Demonstration" >}}
  {{< google-drive-video "1SktxT6KKz82M48WZAtG3vI9K-R9sn584" >}}
{{< /details >}}

We now have the means to generate some streaming data with this Super Hero Combat Data Simulator at no expense. Spinning up a local application with docker-compose is simple. Follow the commands below, replacing `${PLAYERS}` with the required participants.
```bash
export PLAYERS=10
docker-compose --env-file .env \
  -f compose/docker-compose.prod.yml build && \
docker-compose --env-file .env \
  -f compose/docker-compose.prod.yml up \
  --scale player=${PLAYERS}
```
We can even load balance the server with the following
```bash
export PLAYERS=10
export LOADBALANCE=3
docker-compose --env-file .env \
  -f compose/docker-compose.prod.yml build && \
docker-compose --env-file .env \
  -f compose/docker-compose.prod.yml up \
  --scale superhero_server=${LOADBALANCE} \
  --scale player=${PLAYERS}
```

### Redis
{{< details "Video Demonstration" >}}
  {{< google-drive-video "16H3IV7iXzJSCfrYDcbzzjagzMZv94f9p" >}}
{{< /details >}}

This Data Simulator can provide us a great starting point for digging into memory-cache state and exploring Redis' commands to play with its data structures. More details in regard to all the data structures from this Data Simulator can be found in the [README API Documentation](https://github.com/jg-ghub/py-superhero/tree/main?ref=rethinkr.ghost.io#api). For now, let's run a short simulation with the workers off (so they don't clean up all the state), and journey through some of the data structures.
```bash
docker-compose --env-file .env \
  -f compose/docker-compose.prod.yml up \
  --scale player=10 \
  --scale worker=0
```
After the simulation finishes, let's look at how many 2 player games were created
```bash
redis-cli -h localhost -n 0 \
  LLEN games:players:2

> 65
```

We can pick one game at random
```bash
redis-cli -h localhost -n 0 \
  LRANGE games:players:2 12 12

> 1) "60618947-9079-4afd-be1d-f215171a6014"
```

Check if this game had completed, and who the players were
```bash
export GAME_TOKEN=$(redis-cli -h localhost -n 0 \
  LRANGE games:players:2 12 12)
redis-cli -h localhost -n 0 \
  GET games:${GAME_TOKEN}:status & \
redis-cli -h localhost -n 0 \
  SMEMBERS games:${GAME_TOKEN}

> "Completed"
> 1) "6b51cf16-90e6-4083-810f-aee09409b6a5"
> 2) "069dbae2-a030-4424-8764-db5e15e0916f"
```

Now that we know the players in this game, let's check a random member's superhero in-game stats
```bash
export PLAYER_TOKEN=$(redis-cli -h localhost -n 0 \
  SRANDMEMBER games:${GAME_TOKEN})
redis-cli -h localhost -n 0 \
  HGETALL games:${GAME_TOKEN}:superheros:${PLAYER_TOKEN}

> 1) "id"
> 2) "191"
> 3) "attack"
> 4) "40992"
> 5) "health"
> 6) "0"
```

If the Health is greater than 0, then that player won the match and the hash is the final state of that player's attributes when the game finished. We can reverse lookup the superhero this player used during combat
```bash
export SUPERHERO_ID=$(redis-cli -h localhost -n 0 \
  HGET games:${GAME_TOKEN}:superheros:${PLAYER_TOKEN} id)
redis-cli -h localhost -n 0 \
  HGET superheros ${SUPERHERO_ID} | \
python3 -c "import sys, json; print(json.load(sys.stdin)['biography']['full-name'])"

> Crystallia Amaquelin
```

### Pub/Sub

Our final point of interest is building a working pub/sub model into our Data Simulator Architecture. Publish-subscribe microservice architecture is a distributed system design where services communicate through a messaging infrastructure. In this architecture, services act as either publishers or subscribers, decoupling them from direct dependencies on each other. Publishers send messages to topics or channels, and subscribers express their interest in specific topics, receiving messages that match their subscriptions. This asynchronous communication model allows for loose coupling, scalability, and flexibility, as services can dynamically join or leave the system without impacting others. It enables event-driven and real-time processing, facilitating efficient and decoupled communication among microservices.

As described already, this Simulator has a much simpler consumer function which is only responsible for cleaning the Redis state from stale data (More explicit detail on the worker functionality can be found in the [code](https://github.com/jg-ghub/py-superhero/blob/main/lib/pubsub/__init__.py)). However, the worker's potential is far greater with the benefit of flexibility and scalability to manage professional caliber data ingestion and data sinks. We expand on this functionality in later articles as we move deeper into the world of Data Architecture.

## Schematic

{{< get-html "content/data-simulators/superhero-combat/full-schematic.html" >}}