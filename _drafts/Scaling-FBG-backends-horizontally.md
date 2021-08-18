---
layout: post
title: Scaling FreeBoardGames.org backends horizontally
---

In this post, I describe how we were able to scale the [FreeBoardGames.org](https://freeboardgames.org)'s [NestJs](https://nestjs.com) and [boardgame.io](https://boardgame.io) backends horizontally using Pub/Sub.

[FreeBoardGames.org](https://freeboardgames.org) is a FOSS project focused on publishing free [boardgame.io](https://boardgame.io) games created by the community, without profits. During the COVID-19 pandemic, we saw significant growth on users (from ~ 1k to up to ~ 30k monthly active users, see [stats here](https://stats.freeboardgames.org)) and developers contributing to the project ([see contributors here](https://www.freeboardgames.org/about)).

However, this growth spurt brought me some worry that we might not be able to sustain another wave of growth with its current architecture.

- We have to run as cheaply as possible as we do not generate any revenue. It is better for us to focus on horizontally scaling as opposed to vertically scaling because machines get more expensive the less commodity they are, as a rule of thumb.
- We need high availability. We almost always have dozens of people online on the website playing games, most in multiplayer games. If we don't have redundancy on a single server, we can't update its software without downtime. The majority of our traffic comes from organic search results, so low availability also hurts growth.
- Bottlenecks became apparent when the single instance of _fbg-server_ crashed without failing its liveness probe and users were unable to start matches (see [issue #773](https://github.com/freeboardgames/FreeBoardGames.org/issues/773)).  
- It is also fun to over-engineer side-projects and learn new technologies :).

# Previous Architecture

FBG started with the *web* server responsible for rendering the React page, and later we migrated it to [Next.js](https://nextjs.org/). The JavaScript on this webpage would then send http requests directly to the boardgame.io (*bgio*) server in order to create new matches. After a match was created, the browser connects with the server using a websocket. This websocket is used for broadcasting moves and state between players after validating them in the *bgio* server.

<pre>
______________        _________
|            |  http  |       |
|   Browser  | <----> |  web  |
|            |        |       |
^^^^^^^^^^^^^^        ^^^^^^^^^
      ^               __________
      |  websocket    |        |
      +-------------->|  bgio  |
         http         |        |
                      ^^^^^^^^^^
</pre>

Initially, the state of all matches were kept in-memory on the *bgio* server. However, this had the unfortunate side-effect that restarting the server meant losing all active matches state. In order to avoid the ire of players by losing their games, we would only restart the server when nobody was playing. This constraint made it hard to deploy new versions of the software when we reached a few hundred monthly active users, as at any given time we would have some users playing. So, in order to avoid this issue, we made the *bgio* server write and recover matches state to a database, which currently is Postgres.

<pre>
______________        _________
|            |  http  |       |
|   Browser  | <----> |  web  |
|            |        |       |
^^^^^^^^^^^^^^        ^^^^^^^^^
      ^               __________       ______________
      |  websocket    |        |  SQL  |            |
      +-------------->|  bgio  | <---> |  Postgres  |
         http         |        |       |            |
                      ^^^^^^^^^^       ^^^^^^^^^^^^^^
</pre>

Over time, we wanted to experiment with new infrastructure that, at the time, was not supported by the [boardgame.io](https://boardgame.io) framework. While initially we tried to contribute upstream most of these changes, we realized that some of these features were particular to our use case, or would require significant effort to generalize for the boardgame.io library. For instance, we wanted [a public lobby](https://cdn.discordapp.com/attachments/547617829005033483/836995367572471868/Screenshot_20210428-090019.png), and we wanted live updates to the room state, while boardgame.io only supported pooling REST APIs.

So we created the *fbg-server*, a service that would _wrap_ the boardgame.io server, handling authentication, rooms, and allowing us to create the public lobby on the home page, and later a [chat too](https://github.com/freeboardgames/FreeBoardGames.org/issues/428). It was built using [NestJs](https://nestjs.com) as it provided a powerful and yet simple dependency injection framework. We also decided to use [GraphQL](https://graphql.org/) mainly to leverage [Subscriptions](https://docs.nestjs.com/graphql/subscriptions) that made it easy to write pages with live updates. In order to persist its state, we used [TypeORM](https://docs.nestjs.com/recipes/sql-typeorm#sql-typeorm) and connected this new service to the existing Postgres database.

<pre>
______________        _________
|            |  http  |       |
|   Browser  | -----> |  web  |
|            |        |       |
^^^^^^^^^^^^^^        ^^^^^^^^^
      ▲               __________       ______________
      |  websocket    |        |  SQL  |            |
      +-------------->|  bgio  | ----> |  Postgres  |
      |               |        |       |            |
      |               ^^^^^^^^^^       ^^^^^^^^^^^^^^
      |                 ▲                     ▲
      |                 | http                |
      |                 |                     |
      |               ________________        |
      |  websocket    |              |   SQL  |
      +-------------->|  fbg-server  | -------+
         http         |              |
                      ^^^^^^^^^^^^^^^^
</pre>

# Bottleneck and New Architecture

Meanwhile, we also migrated to [Kubernetes](https://kubernetes.io/) and [coded our infrastructure](https://github.com/freeboardgames/FreeBoardGames.org/tree/master/helm) in [Helm](https://helm.sh/). This made it easy for contributors to be able to [easily run a production-like environemnt](https://www.freeboardgames.org/docs/?path=/story/documentation-running-prod--page) using [minikube](https://minikube.sigs.k8s.io/docs/) and to scale multiple replicas of our _web_ server, as it is stateless.

However, we could not scale the _bgio_ and the _fbg-server_ services because the websocket connection makes them stateful. This means that if we naively ran two (or more) instances of these services, some users on a particular instance would not be able to receive the updates generated by other users on a separate instance (i.e. a move in a game), unless they reloaded the whole page (which forces the state to be fetched from Postgres).

The solution to this is a [Pub/Sub service](https://cloud.google.com/pubsub/docs/overview). Each room (for *fbg-server*), and each match (for *bgio*) can be a topic/channel that each instance subscribes and pushes updates to, bridging the communication between multiple server instances. For that role, we picked [Redis](https://redis.io/topics/pubsub), as it a FOSS project too which also avoids any technology lock-in, making our architecture look like the diagram below.

<pre>
______________        _________
|            |  http  |       |
|   Browser  | -----> |  web  |
|            |        |       |
^^^^^^^^^^^^^^        ^^^^^^^^^
      ▲                                         Pub/sub
      |                 +------------------------------
      |                 ▼                             |
      |               __________       ______________ |
      |  websocket    |        |  SQL  |            | |
      +-------------->|  bgio  | ----> |  Postgres  | |
      |               |        |       |            | |
      |               ^^^^^^^^^^       ^^^^^^^^^^^^^^ |
      |                 ▲                     ▲       |
      |                 | http                |       |
      |                 |                     |       |
      |               ________________        |       |
      |  websocket    |              |   SQL  |       |
      +-------------->|  fbg-server  | -------+       |
         http         |              |                |
                      ^^^^^^^^^^^^^^^^                |
                        ▲                             |
                        | Pub/sub                     |
                        ▼                             |
                      ___________                     |
                      |         |<--------------------+
                      |  Redis  |
                      |         |
                      ^^^^^^^^^^^
</pre>

# Implementation

## New Redis server and connecting it to fbg-server

The first step was creating a new Redis service. This was also the easiest one because we use Helm, so we just needed to add the [bitnami redis](https://github.com/bitnami/charts/tree/master/bitnami/redis) dependency to our `Chart.yaml`:

```
dependencies:
(...)
- name: redis 
  version: "14.1.0"
  repository: "https://charts.bitnami.com/bitnami"
```

The only configuration needed was setting the password on `values.yaml`:

```
redis:
  auth:
    password: REDIS_FBG_PASSWORD 
```

Then, we needed to connect the existing `fbg-server` to Redis to allow it to horizontally scale:

<pre>
 ________________
 |              |
 |  fbg-server  |
 |              |
 ^^^^^^^^^^^^^^^^ 
 ▲
 | Pub/sub
 ▼   
 ___________ 
 |         |
 |  Redis  |
 |         |
 ^^^^^^^^^^^
</pre>

Under the hood, our `fbg-server` uses [NestJS](https://nestjs.com) and [GraphQL subscriptions](https://docs.nestjs.com/graphql/subscriptions). [This article](
https://dev.to/thisdotmedia/graphql-subscriptions-with-nest-how-to-publish-across-multiple-running-servers-15e) was a good step-by-step guide on how to scale horizontally this setup. 

In summary, we had to use [graphql-redis-subscriptions](https://github.com/davidyaha/graphql-redis-subscriptions) npm library to create a new `FbgPubSubModule`, which provides an specific implementation for GraphQL's `PubSub` interface. After that, we had to replace all the previous injections of the default GraphQL `PubSub` to use the new one, by annotating them with `    @Inject(FBG_PUB_SUB)` and removing the previous PubSub module. Lastly, we had to make sure we provided the correct host name and port of the redis service to the deployment running `fbg-server`. You can [see the full PR here](https://github.com/freeboardgames/FreeBoardGames.org/pull/794/files).

## Scaling bgio: Contributing upstream to boardgame.io

All in all, scaling NestJS and the GraphQL subscriptions was a painless experience. I knew [boardgame.io](https://boardgame.io/) did not have built-in horizontally scaling support, because, well, nobody ever needed it (do we ?). Therefore, I was expecting a bit more work there as we would probably need to change some APIs ...

However, I also knew that [boardgame.io uses socket.io](https://github.com/boardgameio/boardgame.io/blob/main/src/server/transport/socketio.ts) under the hood. And there are [plenty of guides online](https://socket.io/docs/v4/redis-adapter/) on how to use Redis pub/sub to scale socket.io to multiple servers.

<pre>
 __________
 |        |
 |  bgio  |
 |        |
 ^^^^^^^^^^ 
 ▲
 | Pub/sub
 ▼   
 ___________ 
 |         |
 |  Redis  |
 |         |
 ^^^^^^^^^^^
</pre>

I found out a way to pass the [redis adapter](https://github.com/socketio/socket.io-redis-adapter) to the bgio `Server`. It connected succesfully with Redis, and... **Nothing worked :(**. No messages went through the pub/sub channels that were created by socket.io adapter. Testing with two servers, a player would do the move in one server and the other player would not receive the move, making the game state diverge between them. 

I sent a message to the boardgame.io gitter channel:
```
hello folks... I am trying to scale boardgame.io horizontally, and even though I was able to make the socket.io broadcast to multiple servers using a redis adapter, it doesnt work. 

My hypothesis is that we are keeping some state on the memory of the server, and that state is not being updated with new messages coming from peer socket.io servers, only when the client is directly connected to it. This would make the in-memory state of different replicas of the same server to drift apart. 

Does anybody know what state we keep in-memory on the server? (...)
```

And bingo, the always helpful [Chris Swithinbank](https://github.com/delucis/) replied with this:
```
Ohhh, might it be to do with the way we emit update events to everyone? Basic schema is:

1. The server (each instance in your case) has a Map of match IDs called roomInfo. Each match ID maps to a set of client IDs (socket IDs in this case). (source)

     roomInfo = {
         matchID -> [ clientID, clientID, ... ]
         ...
     }

2. When a client connects to the server, we add its ID to the set of client IDs for the relevant match (source). This will only happen on the server instance they connect to.

3. When a client makes an action, we run it through the reducer etc., then emit the updated state to connected clients. But we do this by saying: give me the set of client IDs for this match ID; now send the update to each client ID (source).

Say there are two servers you might have a situation like this:

Server 1                      Server 2

roomInfo = {                  roomInfo = {
  matchA -> [ client1 ]         matchA -> [ client2 ]
}                             }

In this case, server 1 doesn’t know about client 2 and server 2 doesn’t know about client 1, so the two clients won’t actually emit to each other.
```

But... Why wasn't boardgame.io using a [socket.io room](https://socket.io/docs/v3/rooms/index.html) per match instead of sending a separate message to each connected player? If that was the case, everything would work out of the box. Chris replied:

```
(...) One thing we do is store the player ID for each client so that we can run the playerView for each of them when updating state. (...)
```

*playerView* is the function that allows a very popular feature in boardgame.io: [Secret state](https://boardgame.io/documentation/#/secret-state?id=secret-state). It inhibits cheating by only sending the relevant subset of the state for each player. For instance, if you were playing poker, we would not send the poker hands of your adversaries to your browser. This would only be known by the server.

However, this feature/requirement made so each player receives a different message from the server, and its implementation abandoned the usage of socket.io rooms. 

The problem is illustred below. Previously when `Player 0` made a move: (G is the game state)
<pre>
 ______________
 |            |
 |  Postgres  |
 |            |
 ^^^^^^^^^^^^^^
      |
      | previousG
      |
      ▼
newG = previousG + move
  __________
  |        | --- playerView(newG, 0) ---> Player 0
  |  bgio  | --- playerView(newG, 1) ---> Player 1
  |        | --- playerView(newG, 2) ---> Player 2
  ^^^^^^^^^^ 
</pre>

The single server would know all the players connected to the match and send a specific messages to each, the result of the `playerView` function. Naively using multiple servers:

<pre>
 ______________
 |            |
 |  Postgres  |
 |            |
 ^^^^^^^^^^^^^^
      |
      | previousG
      |
      ▼
newG = previousG + move
  __________
  |        | --- playerView(newG, 0) ---> Player 0
  |  bgio  | --- playerView(newG, 1) ---> Player 1
  |   0    | 
  ^^^^^^^^^^ 

  __________
  |        | <- x -> Player 2
  |  bgio  | 
  |   1    |
  ^^^^^^^^^^   
</pre>

`Player 2` which is connected *only* to `bgio 1` would not receive any updates from its server, as `bgio 0` that received the move is not aware at all about the existence of `Player 2`.


To fix this, my initial reaction was to move the metadata about which players that were connected to which servers to the database. However, after thinking a little bit, it was clear that this was not enough, we would still need to send messages across servers, having only the metadata would not enable `bgio 0` to send a message to `player 2`, even if it knew it was connected to `bgio 1`. At that point, I got lazy and stopped working in this project for some months, as it was shaping to be much more work than expected.

Then, looking at the [boardgame.io code](https://github.com/boardgameio/boardgame.io/pull/966/files), it was clear that the easiest way forward would be the solution below:

<pre>
 ______________
 |            |
 |  Postgres  |
 |            |
 ^^^^^^^^^^^^^^
      |
      | previousG
      |
      ▼
newG = previousG + move
  __________
  |        | 
  |  bgio  |
  |   0    | 
  ^^^^^^^^^^ 
      | Pub (newG)
      ▼   
 ___________ 
 |         |
 |  Redis  |
 |         |
 ^^^^^^^^^^^ 
 | Sub  | Sub (newG)
 |      ▼   
 |  __________
 |  |        | --- playerView(newG, 0) ---> Player 0
 |  |  bgio  | --- playerView(newG, 1) ---> Player 1
 |  |   0    |
 |  ^^^^^^^^^^   
 |  __________
 |  |        | --- playerView(newG, 2) ---> Player 2
 +->|  bgio  | 
    |   1    |
    ^^^^^^^^^^
</pre>

This had the advantage that we would not need to query the database regarding the metadata of each connection: Each server would only know about the players directly connected to them, and apply the `playerView` to any new state received for a match the player is in. It also meant that we could *always* publish a `newG` to the pub/sub interface, and *always* calculate the `playerView` from its subscriptions, and the pub/sub interface could be generic enough to run in-memory for the (most common) case that the boardgame.io user does not need to scale horizontally.

We implemented this idea by first [refactoring the existing interfaces](https://github.com/boardgameio/boardgame.io/pull/966) to postpone the `playerView` calculation to the transport layer, and then [creating a generic pub/sub service and the default in-memory implementation](https://github.com/boardgameio/boardgame.io/pull/978). Then, the [@boardgame.io/redis-pub-sub](https://github.com/boardgameio/redis-pubsub) library was created to hold the pub/sub adapter to redis, which avoided adding any redis dependency directly to the boardgame.io library.

Finally, we were able to scale the boardgame.io deployment to multiple pods in production a few days ago :tada::
```
$ kubectl get deployments -n fbg-prod
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
fbg-prod-bgio         2/2     2            2           2d2h
fbg-prod-fbg-server   2/2     2            2           2d2h
fbg-prod-web          2/2     2            2           2d2h
```
