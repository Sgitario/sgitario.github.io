---
layout: post
title: Applying Event Sourcing and CQRS in a legacy project
date: 2016-04-10
tags: [ Architecture ]
---

# Introduction
We are told to work on a legacy project about a Bicycle management system which architecture looks quite like to:

![Legacy Architecture]({{ site.url }}{{ site.baseurl }}/images/qcrs_legacy_architecture.png)

As a very brief summary: “Mule ESB is listening events raised by Third Party Services, then fulfills these events via Gateway Services and sends updates to Application Services.”

More:
- Mule ESB consists in a couple of applications that listen events raised by Third Party Systems.
- Mule ESB uses Rabbit MQ for queueing messages between the applications.
- Mule ESB fulfills the raised events via Gateway Services and sends updates to Application Services.
- Application Services and Gateway Services have been designed following tier architecture with Application, Services and Data Access tiers.
- Application Services writes the updates and reads the data from DB.
- UI: Consumes Application Services for reporting purposes to manage the current state of bicycles (from design, fabric, distribution …).
- DB: General purpose design where we have a table for bicycle and another table for attributes. There are other tables for Resellers, Producers, …

I'm sure this kind architecture may sound familiar to most of you. So, let’s go for the drawbacks behind:
- [ ] UI slowness: The user is able to select a report of bicycles (as an example: the bicycles that are pending to distribution). Each time the user selects a report, the Application Services has to query to DB and this query may involve several tables. When there are lot of data in the DB and many involved tables, then our report will behave very slowly for sure.
- [ ] Concurrency: Application Services has to deal with concurrent updates from Mule ESB for the same bicycles.
- [ ] Audit: Application Services has to manually audit the changes against every entity in the DB.
- [ ] Traceability: Not possible to figure out the sequence of changes against a bicycle: don't have proper snapshots and data history. 
- [ ] Scalability: To achieve scalability, we need to scale either the whole Application Services app or Gateway Services app or the whole applications within Mule ESB.

Most of the above drawbacks can be easily sorted them out by using the Event Sourcing and CQRS architecture approach. But, first, we need to answer two questions:

* What about eventual consistency?

Eventual consistency means that the changes will not be displayed UI directly when the Application Services receives an update. There will be always some minutes of delay. For this legacy project, the architecture was using eventual consistency already, but it may be something to take into account in other projects.

* What about validation?

Assuming we are using optimistic locking to detect DB collisions (this is always a good practice), what to do next? Also, define retries.

The nature of this kind of apps matches exactly with the features in an Event Sourcing and CQRS architecture. So, let’s try to refactor this legacy architecture in three steps:

# ESB replacement
Why do we need a Mule ESB instance to orchestrate the incoming events? If you find a very good reason, go for it; otherwise replace your Mule ESB instance into micro-services and orchestrate the logic only with Rabbit MQ. I see the next advantages:

- Easier to deploy/scale: Just adds a new consumer for a queue.
- More app granularity: For example, if a consumer/producer app is not working fine, we can detect this easily.
- Technology agnostic: We can decide to implement an app consumer/producer using a framework which may work better for some purposes.

Therefore, thanks to Rabbit MQ, we still can offer high availability and fault tolerance.

![New Architecture]({{ site.url }}{{ site.baseurl }}/images/qcrs_events.png)

# Event Sourcing
By refactoring the ESB layer into a micro-services approach we did not face yet any of the above drawbacks. The main bottleneck is still in the database where we use the same application Application Services for write and read operations. For example, anytime when the state of a bicycle is updated, the Bicycle and Attributes tables have to be updated. Therefore, how to allow more multiple concurrent write operations? Using Event Store tables:

![Event Store]({{ site.url }}{{ site.baseurl }}/images/qcrs_event_store.png)

These Event Store tables can be within a single database instance or in multiple database instances. The important bit is that the Event Store stores only events. So the Application Services insert these events (instead of updating entities). For example, in the Attributes Event Store, we have:

* Event 1: Bicycle X contains an attribute distributionStatus with value “Pending”.

If the Application Services receives that the Bicycle X is ready to be distributed, then the Attributes Event Store would have the new event 2:

* Event 2: Bicycle X contains an attribute distributionStatus with value “Ready”.
* Event 1: Bicycle X contains an attribute distributionStatus with value “Pending”.

This way we also can have data history for free and possibility to run a replay of data and traceability. 

It’s important to remark that we should use optimistic locking to update the Event Store. A solution for doing this is to add another column “Version” and check always if the new event we want to add, matches with the version we did read at the beginning of the event transaction. More info [here](https://docs.jboss.org/hibernate/orm/4.0/devguide/en-US/html/ch05.html).

# CQRS (Command Query Responsability Segregation)
Application Service application is also used as reporting purposes by the Reporting UI in order to read data about bicycles. Now, when using Event Store databases, these read operations will have to read all the events for a single entity, so probably the latency will be much worse. We might deploy more Application Service applications, but still we will lock the Event Store for reading and the purpose of Event Store is only to write events and allow more concurrency operations by decoupling business areas (see more in [Domain Driven Design](https://en.wikipedia.org/wiki/Domain-driven_design)). Therefore we need to separate Application Service into two components: Query components (for read operations) and Command components (write operations).

The Command applications will manage the incoming events and store these events into the Event Store. The Query applications will read the data from the Read Storage. The Read Storage is like a “user” friendly database of the Event Store. Example, we have 10 events for different attributes of an entity in the Event Store, and in the Read Storage, we will have these 10 events mapped to either a single row or a table with two foreign keys, etc. For this, we need a new component Event Handler who manages the mapping between Event Store <-> Read Storage.

![Final Architecture]({{ site.url }}{{ site.baseurl }}/images/qcrs_final_architecture.png)

There are many options to implement the Event Handler component and you can configure it to define the frequency for invalidate/refresh your data in the Read Storage which at the end will mean the eventual consistency in your solution.