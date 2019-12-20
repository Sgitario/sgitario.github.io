---
layout: post
title: Kogito Introduction
date: 2019-12-20
tags: [ kogito, java, quarkus, drools ]
---

[Kogito](https://kogito.kie.org/) is the natural evalution of [Drools](https://www.drools.org/) (to implement rules and decisions) and [jBPM](https://www.jbpm.org/) (to design processes and cases) to the cloud using [quarkus](https://quarkus.io/) under [Graal VM](https://sgitario.github.io/graalvm-getting-started/).

It provides the next features:

- Code Generation based on business assets
- Executable model for the process/rule/decision definitions

Kogito translates the Drools rules into Java model in order to be adapted to the [Graal VM](https://sgitario.github.io/graalvm-getting-started/) limitations (limited reflection and absence of dynamic class loading). Also, Kogito also translates the *kmodule* into Java model.

- Type safe data model that encapsulates variables
- REST api for each public business process/decision/rule

## Learn by example

- The Drool rule:

```drools
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
