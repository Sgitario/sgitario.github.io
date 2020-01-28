# Transactions: Another flavour by @xstefank

See the difference between ACID and SAGA. Then, MicroProfile LRA as an implementation of SAGA in Java microservices.

## ACID
Atomicy, consistency, isolation, durability 

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

Talk: GOTO 2015 by Caite
Saga tries to undercome the above issue with:
The transaction is sent to one service each and complete the transaction individually. 
If a later service cannot process the transaction, it needs to cancel/compensate the transaction in the previous services.

The issue is that when the state of the transaction is inconsistent until the last service is complete (or it's fully compensated).

Mapping from ACID:

- Atomicity -> Basically Available
- Consistency -> Soft state
- Isolation -> Eventual consistency
- Durability

### MicroProfile LRA (Long Running Actions)

The implementation of SAGA in Java microservices.
Source: 

Quarkus extension Nayarana LRA (or something like this)