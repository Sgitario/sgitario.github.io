---
layout: post
title: Red Hat Automation Manager
date: 2020-03-26
tags: [ java, jbpm ]
---

The Automation Manager product from Red Hat consists of two versions:

- [Red Hat Process Automation Manager](https://developers.redhat.com/products/rhpam/overview) uses [jBPM](https://sgitario.github.io/jbpm-introduction/) and has its own admin site to manage the processes.
- [Red Hat Decision Manager](https://www.redhat.com/en/technologies/jboss-middleware/decision-manager) uses [Drools](https://sgitario.github.io/drools-introduction/) and intended to be used in middleware layer only

Without digging into the differences of these versions, its architecture consists of:

![Architecture]({{ site.url }}{{ site.baseurl }}/images/redhat-automation-manager-architecture.png)

Where:

- Business/Decision Manager: a site where we can configure users/projects/tasks/...
- Kie Server: the execution server where our processes will run
- Smart Router: allows to group a number of Kie Servers

Also, this product allows to be configured for:

- Persistence: using any kind of Database (mysql, Oracle, Postgresql, ...)
- High Availability
- Message Broker