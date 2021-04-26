---
title: 'Messaging: Idempotence and Delivery Types'
published: true
description: 'What happens when we have multiple consumers to a single message? And if the message arrives more than one time? In this post, Idempotence will be showcased, as well as its relation to three delivery types.'
tags: ['distributedsystems','architecture','messaging','dotnet']
# date: 2021-04-30
# canonical_url: 'https://mateuscviegas.net/2021/04/messaging-idempotence-and-delivery-types'
---

## *tl/dr*

This post is the third one of a series about basic concepts within distributed architectures.

1) [Messaging: An introduction](https://mateuscviegas.net/2021/02/messaging-an-introduction)
2) [Messaging: Point-to-point with System.Threading.Channels](https://mateuscviegas.net/2021/02/messaging-concepts-system-threading-channels)
3) [Messaging: Idempotence and Delivery Types](https://mateuscviegas.net/2021/04/messaging-idempotence-and-delivery-types)
4) Messaging: Consistency and Resiliency Patterns

## Competing Consumers

Since for point-to-point channels a message can only be consumed by a single receiver, what happens when we have more than one receiver attached ot it? This is when the concept of **competing consumers** comes in.

With only a single consumer per message, if there's more than one consumer connected to a point-to-point channel, the channels will compete for the messages and while one processes it, the others will be idle until a new message arrives and the competition starts again. In practice, messaging infrastructures have their own specific implementation on how this concurrency is executed.

This scenario is interesting to **increase the channel throughput**. If the messages are not being consumed as fast as they're being published to the channel, increasing the number of competing consumers would then clear the queue buffer faster as well. These could be achieved by either/both simply increasing the number of consumer threads in-memory or using horizontal scaling to increase the number of consumers externally.

Some considerations on this scenario is that *usually ordering is not guaranteed* and therefore the message consumption should be **idempotent**, that is, they should be independent of order.

## Idempotence

## Handling Deliveries

### Exact once

### At most once

### At least once