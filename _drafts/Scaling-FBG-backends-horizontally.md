---
layout: post
title: Scaling FreeBoardGames.org backends horizontally
---

In this post, I describe how we were able to scale the [FreeBoardGames.org](https://freeboardgames.org) backends horizontally.

[FreeBoardGames.org](https://freeboardgames.org) is a FOSS project focused on publishing free board games created by the community, without profits. During the COVID-19 pandemic, we saw a good amount of user growth (from ~ 1k to ~ 30k monthly active users, see [stats here](https://stats.freeboardgames.org)) and developers contributing to the project ([see contributors here](https://www.freeboardgames.org/about)).

However, this growth spurt brought me some worry that we might not be able to sustain another wave of growth with its current architecture.

- We have to run as cheaply as possible as we do not generate any revenue. It is better for us to focus on horizontally scaling as opposed to vertically scaling because machines get more expensive the less commodity they are, as a rule of thumb.
- We need high availability. We almost always have dozens of people online on the website playing games, most in multiplayer games. The majority of our traffic comes from organic search results, so not being available will hurt future user growth.
- Bottlenecks became apparent when the single instance of _fbg-server_ crashed without failing its liveness probe and users were unable to start matches (see [issue #773](https://github.com/freeboardgames/FreeBoardGames.org/issues/773)).  

# Bottlenecks for horizontally scaling


# Scaling fbg-server: NestJS with GraphQL subscriptions

Reference:
https://dev.to/thisdotmedia/graphql-subscriptions-with-nest-how-to-publish-across-multiple-running-servers-15e

# Scaling bgio: Contributing upstream to boardgame.io