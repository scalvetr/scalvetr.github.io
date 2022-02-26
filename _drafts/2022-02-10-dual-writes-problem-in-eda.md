---
layout: post
title:  "Dual Writes Problem in Event Driven Architectures"
date:   2022-02-10 18:40:25 +0100
categories: eda cap cdc
---

## The problem

Dual write describes a situation when a service has to change data in two different systems. 
For instance a service changes state in its local database and also notifies other systems by publishing an event in Kafka.

If one of these two operations fails, you might end up with inconsistent data.

Example:

![Dual write problem](/assets/img/2022-02-10-dual-writes-problem-in-eda/dual-write-problem.png)

**Distributed Transaction**

One possible solution we might come up is to use a distributed transaction (2PC) to guarantee the atomicity of these two writes.

This strategy is not applicable because this kind of transaction is provably not supported by the event broker (for instance 
Kafka doesn't). But even if it is supported, it is not a good idea, the reason is that in distributed systems you 
don't want strong consistency between all participants because it does not scale, and according to the [CAP](https://en.wikipedia.org/wiki/CAP_theorem) 
theorem would lead to availability issues.


**Local Transaction**

There are strategies to minimize the chance of data inconsistency by using the local transaction. For example by tying 
the event publishing to the BD transaction. With this approach the event would be published just before confirming the 
DB transaction, and in case the event cannot be published, the local transactions would be rolled back.

![Dual write local transaction](/assets/img/2022-02-10-dual-writes-problem-in-eda/dual-write-local-transaction.png)

The sequence is like this:

1. Begin transaction
2. Update the database
3. Publish the event
4. Commit the transaction

This approach minimizes the chance of data inconsistency, but it is still possible that the event is published but finally the
DB transaction is no confirmed.

## Outbox pattern

The outbox pattern 

eventual consistency

Change data capture


![Dual write outbox](/assets/img/2022-02-10-dual-writes-problem-in-eda/dual-write-outbox.png)



More information:
* [Avoiding dual writes in event-driven applications](https://developers.redhat.com/articles/2021/07/30/avoiding-dual-writes-event-driven-applications)
* [Dual Writes – The Unknown Cause of Data Inconsistencies](https://thorben-janssen.com/dual-writes/)
* [Dual write and data inconsistency](https://www.johnnyhashoul.com/post/dual-write-and-data-inconsistency)
* [Sending Reliable Event Notifications with Transactional Outbox Pattern](https://medium.com/event-driven-utopia/sending-reliable-event-notifications-with-transactional-outbox-pattern-7a7c69158d1b)