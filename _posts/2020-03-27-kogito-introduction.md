---
layout: post
title: Kogito Introduction
date: 2020-03-27
tags: [ kogito, java, quarkus, drools ]
---

[Kogito](https://kogito.kie.org/) is the natural evalution of [Red Hat Automation Manager](https://sgitario.github.io/automation-manager/) to the cloud using [Quarkus](https://quarkus.io/) under [Graal VM](https://sgitario.github.io/graalvm-getting-started/).

Kogito provides the next features:

- Code Generation based on business assets
- Executable model for the process/rule/decision definitions

Kogito translates the Drools rules into Java model in order to be adapted to the [Graal VM](https://sgitario.github.io/graalvm-getting-started/) limitations (limited reflection and absence of dynamic class loading). Also, Kogito also translates the *kmodule* into Java model.

- Type safe data model that encapsulates variables
- REST api for each public business process/decision/rule

## Architecture

The high level architecture of Kogito based application consists of:

![Architecture]({{ site.url }}{{ site.baseurl }}/images/kogito_architecture_1.png)

Where:
- Business Process

This is our application (TBD).

- [Management Console](https://github.com/Sgitario/kogito-apps/blob/master/packages/management-console/README.md)
- [Data Index](https://github.com/kiegroup/kogito-runtimes/wiki/Data-Index-Service)

The Data Index consumes the Kafka events and *index* this information into the Infinispan as storage service. Technicallly, it is a Quarkus application based on [VertX](https://vertx.io/) and reactive messaging that exposes a [GraphQL](https://graphql.org/) endpoint.

- [Jobs Service](https://github.com/kiegroup/kogito-runtimes/wiki/Jobs-Service)

The Jobs Service is responsible for scheduling jobs that aim to be fired at a given time. The service does not execute the job itself, it triggers a callback that could be an HTTP request on a given endpoint specified on the job request, or any other callback that could be supported by the service. Technically, it is a Quarkus application based on [VertX](https://vertx.io/). 

- [Kafka](https://kafka.apache.org/)
- [Infinispan](https://infinispan.org/)

As said, Kogito is a cloud native solution and to ease the deployment, it's designed to be ruled by operators (supported by Openshift and Kubernetes):

- [Kogito Cloud Operator](https://github.com/kiegroup/kogito-cloud-operator): To create Management Console, Data Index, Business Processes and Jobs Service instances
- [Infinispan Operator](https://infinispan.org/infinispan-operator/master/operator.html): To Infinispan components
- [Strimzi Operator](https://strimzi.io): To Kafka cluster

To more information about how to get started with Kogito, go to [my next article](https://sgitario.github.io/kogito-developer-guide/).

## Projects

Find the github repository for each component:
- [Kogito Runtimes](https://github.com/kiegroup/kogito-cloud-operator): The framework
- [Kogito Apps](https://github.com/kiegroup/kogito-apps): Management Console, Data Index, Jobs Service
- [Kogito Cloud Operator](https://github.com/kiegroup/kogito-cloud-operator): Kogito Operator, Kogito CLI, Kogito Integration Tests
- [Kogito Tooling](https://github.com/kiegroup/kogito-tooling): Useful tools like the BPMN editor in Visual Code and Web.
- [Kogito Examples](https://github.com/kiegroup/kogito-examples): Kogito Examples

## Learn by example

- The Drool rule:

```drl
rule R when
        $r : Result()
        $p1 : Person(name == "Jose")
        $p2 : Person(name != "Jose", age > $p1.age)
    then
        $r.setValue($p2.getName() + "is older than " + $p1.getName());
end
```

- The configuration in *META-INF/kmodule.xml*:

```xml
<?xml version="1.0" encoding="UTF-8">
<kmodule xmlns="http://jboss.org/kie/6.0.0/kmodule">
    <kbase name="simpleKS" packages="com.sgitario.simple">
        <ksession name="simpleKS" default="true">
    </kbase>
</kmodule>
```

- The service:

```java
@ApplicationScoped
public class CheckPersonRuleService {

    @Named("simpleKS")
    RuleUnit<SessionMmeory> ruleUnit;

    public String run() {
        Result result = new Result();

        SessionMemory memory = new SessionMemory();
        memory.add(result);
        memory.add(new Person("Jose", 34));
        memory.add(new Person("Antonio", 37));
        memory.add(new Person("Felipe", 32));

        ruleUnit.evaluate(memory);

        return result.toString();
    }
}
```

- The endpoint:

```java
@Path("/check/Jose")
public class CheckPersonResource {

    @Inject
    CheckPersonRuleService service;

    @GET
    @Produces(MediaType.TEXT_PLAIN)
    public String checkPerson() {
        return service.run();
    }
}
```

Now, we can build and run our application by:

```bash
mvn clean package quarkus:dev
```

Then, we can call our endpoint:

```bash
curl http://localhost:8080/check/Jose
> Jose is older than Felipe
```

## What's next

I will develop the next topics soon:

- Rule bases modularization via Rule Units
- Rule Units orchestration through jBPM workflows
- Seamless integration with Camel routes, Kafka Streams, ...
- Automatic REST endpoint generation returning result of DRL queries

## Conclusions

This post is based on [this video](https://www.youtube.com/watch?v=tLz_aNLuCR0).
