---
layout: post
title: Transactions - ACID vs SAGA
date: 2020-01-28
tags: [ acid, saga ]
---

## ACID (Atomicy, consistency, isolation, durability 

Consensus:
- Two-phase commit protocol (2PC)

Check whether the transaction is ok by all the dependant services. 
If so (all services are prepared), commit the transaction in all the dependant services.
If not, abort transaction (cancel all the transactions in the dependant services)

The issue is that the resources are blocked between the services are awaiting to the two phase commit. If something fails (user doesn't pay), the resources take some time to be released.

- 3PC
- Paxos
- Raft

## SAGA

Saga tries to undercome the above issue by sending the transactions individually in each service. If a later service cannot process the transaction, it needs to cancel/compensate the transaction in the previous services.

The issue is that when the state of the transaction is inconsistent until the last service is complete (or it's fully compensated).

Mapping from ACID:

- Atomicity -> Basically Available
- Consistency -> Soft state
- Isolation -> Eventual consistency
- Durability

An implementation for SAGA transactions in Java is [MicroProfile LRA](https://github.com/eclipse/microprofile-lra).