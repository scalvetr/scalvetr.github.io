---
layout: post
title:  "Dual Writes Problem in Event Driven Architectures"
date:   2022-02-10 18:40:25 +0100
categories: eda 
---

## The problem

Dual write describes a situation when a service has to change data in two different systems. 
For instance a service changes state in its local database and also notifies other systems by publishing an event in Kafka.

If one of these two operations fails, you might end up with inconsistent data.


Example:

![Dual write problem](/assets/img/2022-02-10-dual-writes-problem-in-eda/dual-write-problem.png)

## Local transaction

There are strategies to minimize the chance of data inconsistency. For example by tying the event publishing to the BD 
transaction. With this approach the event would be published just before confirming the DB transaction, and in case the
event cannot be published, the local transactions would be rolled back.

![Dual write local transaction](/assets/img/2022-02-10-dual-writes-problem-in-eda/dual-write-local-transaction.png)

This approach minimizes the chance of data inconsistency, but it is still possible that the event is published but finally the
DB transaction is no confirmed, because of a network error, for instance.

## Outbox pattern

![Dual write outbox](/assets/img/2022-02-10-dual-writes-problem-in-eda/dual-write-outbox.png)



More information:
* [Avoiding dual writes in event-driven applications](https://developers.redhat.com/articles/2021/07/30/avoiding-dual-writes-event-driven-applications)
* [Dual Writes – The Unknown Cause of Data Inconsistencies](https://thorben-janssen.com/dual-writes/)
* [Dual write and data inconsistency](https://www.johnnyhashoul.com/post/dual-write-and-data-inconsistency)
* [Sending Reliable Event Notifications with Transactional Outbox Pattern](https://medium.com/event-driven-utopia/sending-reliable-event-notifications-with-transactional-outbox-pattern-7a7c69158d1b)