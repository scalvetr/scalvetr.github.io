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
DB transaction cannot be confirmed.

## Outbox pattern

### Eventual consistency

As we've seen the two systems cannot be updated in a strong consistent manner, the logical solution to avoid the dual 
write problem is to just update one them.

One system is updated, but what happens with the second one?

A process must be put in place to take the responsibility to eventually update the other system.

![Dual write eventually consistent](/assets/img/2022-02-10-dual-writes-problem-in-eda/dual-write-eventually-consistent.png)

In the diagram, the local transaction only involves `System A`, and the `Update` process is responsible 
that after enough `System B` will be updated.

### Database first

In the example above you could choose either to:
* *Publish the event* and eventually *update the database*.
* *Update the database* and eventually *publish the event*.

In the microservice context, what usually makes more sense is to choose the second one: Update first the database
and then publish the event. That's because the database is usually the internal store for the domain data, you want to 
keep it consistent so your business logic can easily access to the current state of your data.

This way the local database is source of truth for the domain data, and the event broker is an eventual consistent copy.

However, with this approach the local database is source of truth for the domain data, and the event broker is an 
eventual consistent copy. So this is something you must consider while designing your process.

### Change data capture

Databases always offer an API to access to the current state of the data they store.  This is what they are for no mather 
if we are talking about `RDMS` or `NoSQL`.
For instance in a *relational* database you can use `SQL` statement to query the current state.

What is not so well known is that most of them also stores a log of all tha changes, in a similar way that and Event 
Broker does.

Change data capture consists on a process listening to changes on the database log and publishing them to the Event Broker.

![Dual write cdc](/assets/img/2022-02-10-dual-writes-problem-in-eda/dual-write-cdc.png)

By applying this technic, all changes in a database can be easily consumed by an external system.

### The outbox pattern

In the case of relational databases the event might contain information from different tables.
Imagine the customer information modeled as follows in a relational database:

![Customer Data Model](/assets/img/2022-02-10-dual-writes-problem-in-eda/data_model_customer.png)

Although this two tables can be modified within a single transaction, they will produce two events and there's no 
guarantee on the order or the time between the two events triggered from the `CUSTOMER` and `ADDRESSES` tables.

So how would and event containing information about this two tables be produced? It would be provably very complex.

There's where the outbox pattern comes into the picture.

The outbox pattern consists on writing the event to an outbox in the first system within the same transaction that you 
write the data, that ensures reliable delivery to the event broker.

The data model would be like this:

![Customer Outbox Data Model](/assets/img/2022-02-10-dual-writes-problem-in-eda/data_model_customer_outbox.png)

In this case `CUSTOMER_EVENTS` is the outbox table.

The process would be like this:

![Dual write outbox](/assets/img/2022-02-10-dual-writes-problem-in-eda/dual-write-outbox.png)