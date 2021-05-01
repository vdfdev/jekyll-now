---
layout: post
title: Scaling FreeBoardGames.org backends horizontally
---

In this post, I describe how we were able to scale the [FreeBoardGames.org](https://freeboardgames.org)'s Nest and Boardgame.io backends horizontally using Pub/Sub.

[FreeBoardGames.org](https://freeboardgames.org) is a FOSS project focused on publishing free [boardgame.io](https://boardgame.io) games created by the community, without profits. During the COVID-19 pandemic, we saw significant growth on users (from ~ 1k to ~ 30k monthly active users, see [stats here](https://stats.freeboardgames.org)) and developers contributing to the project ([see contributors here](https://www.freeboardgames.org/about)).

However, this growth spurt brought me some worry that we might not be able to sustain another wave of growth with its current architecture.

- We have to run as cheaply as possible as we do not generate any revenue. It is better for us to focus on horizontally scaling as opposed to vertically scaling because machines get more expensive the less commodity they are, as a rule of thumb.
- We need high availability. We almost always have dozens of people online on the website playing games, most in multiplayer games. The majority of our traffic comes from organic search results, so low availability hurts growth.
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

## Adding Redis to our HELM chart

Foo

## Scaling fbg-server: Connecting GraphQL subscriptions to Redis

Reference:
https://dev.to/thisdotmedia/graphql-subscriptions-with-nest-how-to-publish-across-multiple-running-servers-15e

# Scaling bgio: Contributing upstream to boardgame.io