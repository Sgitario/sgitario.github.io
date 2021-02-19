---
layout: post
title: Kogito Introduction
date: 2020-03-27
tags: [ Kogito ]
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

- [Kogito Runtime](https://docs.jboss.org/kogito/release/latest/html_single/#chap-kogito-deploying-on-openshift)

The Kogito Runtime applications keep our business logic. These are our Kogito services we are going to implement with our business logics.

- [Management Console](https://docs.jboss.org/kogito/release/latest/html_single/#con-management-console_kogito-developing-process-services)

The Kogito Management Console is a user interface for viewing the state of all available Kogito services and managing process instances.

- [Data Index](https://docs.jboss.org/kogito/release/latest/html_single/#proc-kogito-travel-agency-enable-data-index_kogito-deploying-on-openshift)

The Data Index consumes the Kafka [cloud events](https://cloudevents.io/) and *index* this information into the storage service. Technicallly, it is a Quarkus application based on [VertX](https://vertx.io/) and reactive messaging that exposes a [GraphQL](https://graphql.org/) endpoint.

- [Jobs Service](https://docs.jboss.org/kogito/release/latest/html_single/#con-kogito-operator-with-jobs-service_kogito-deploying-on-openshift)

The Jobs Service is responsible for scheduling jobs that aim to be fired at a given time. The service does not execute the job itself, it triggers a callback that could be an HTTP request on a given endpoint specified on the job request, or any other callback that could be supported by the service. Technically, it is a Quarkus application based on [VertX](https://vertx.io/). 

- [Trusty Service](https://docs.jboss.org/kogito/release/latest/html_single/#con-kogito-operator-with-trusty-service_kogito-deploying-on-openshift)

Kogito provides a Trusty Service that stores all Kogito tracing events related to decisions made in Kogito services. The Trusty Service uses the Kafka [cloud events](https://cloudevents.io/) from Kogito services, and then processes the tracing events and stores the data in the storage service.

- [Explainability Service](https://docs.jboss.org/kogito/release/latest/html_single/#con-kogito-operator-with-explainability-service_kogito-deploying-on-openshift)

The Explainability service is a complementary service for the Trusty Service to identify why the decision were made.

- [Kafka](https://kafka.apache.org/)
- [Infinispan](https://infinispan.org/)
- [MongoDB](https://www.mongodb.com/)

As said, Kogito is a cloud native solution and to ease the deployment, it's designed to be ruled by operators (supported by Openshift and Kubernetes):

- [Kogito Cloud Operator](https://github.com/kiegroup/kogito-cloud-operator): To create Management Console, Data Index, Business Processes and Jobs Service instances
- [Infinispan Operator](https://infinispan.org/infinispan-operator/master/operator.html): To Infinispan components
- [Strimzi Operator](https://strimzi.io): To Kafka cluster

To more information about how to get started with Kogito, go to [my next article](https://sgitario.github.io/kogito-developer-guide/).

## Projects

Find the github repository for each component:
- [Kogito Runtimes](https://github.com/kiegroup/kogito-cloud-operator): The framework
- [Kogito Apps](https://github.com/kiegroup/kogito-apps): Management Console, Data Index, Jobs Service, Trusty Service, Explainability Service, ...
- [Kogito Cloud Operator](https://github.com/kiegroup/kogito-cloud-operator): Kogito Operator, Kogito CLI, Kogito Integration Tests
- [Kogito Tooling](https://github.com/kiegroup/kogito-tooling): Useful tools like the BPMN editor in Visual Code and Web.
- [Kogito Examples](https://github.com/kiegroup/kogito-examples): Kogito Examples

## Learn by example

For running our new Kogito business application, we first need to deploy our Kogito ecosystem (see [this post](https://sgitario.github.io/kogito-developer-guide/) as a guide to have Kogito running in Openshift).

Then, let's create our first business application:

```sh
mvn io.quarkus:quarkus-maven-plugin:1.3.1.Final:create \
    -DprojectGroupId=org.sgitario \
    -DprojectArtifactId=kogito-persons-example \
    -Dextensions="kogito"
cd kogito-persons-example
```

| We're using Quarkus, however Spring Boot is also supported.

Next, we create a rule file *person-age.drl* inside the src/main/resources/org/sgitario/kogito folder of the generated project:

```drl
package org.sgitario.kogito

rule R when
        $r : Result()
        $p1 : Person(name == "Jose")
        $p2 : Person(name != "Jose", age > $p1.age)
    then
        $r.setOlder(true);
end
```

Also, as seen in the rule file, the *Person.java* and *Result.java*:

```java
package org.sgitario.kogito;

public class Person {
	private String name;
	private int age;
	
    // constructors
	// getters and setters
}
```

```java
package org.sgitario.kogito;

public class Result {
	private boolean older;
	
	// getters and setters
}
```

Add the kmodule in *META-INF/kmodule.xml*:

```xml
<kmodule xmlns="http://www.drools.org/xsd/kmodule" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"/>
```

Define our REST endpoint:

```java
package org.sgitario.kogito;

import org.kie.api.runtime.KieSession;
import org.kie.kogito.rules.KieRuntimeBuilder;

import javax.inject.Inject;
import javax.ws.rs.Consumes;
 
import javax.ws.rs.POST;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;

@Consumes(MediaType.APPLICATION_JSON)
@Produces(MediaType.APPLICATION_JSON)
@Path("/check-age")
public class CheckAgeResource {
	
	private static final Person JOSE = new Person("Jose", 18);
 
    @Inject
    KieRuntimeBuilder runtimeBuilder;
 
    @POST
    @Produces(MediaType.TEXT_PLAIN)
    public boolean checkPerson(Person p) {
    	Result result = new Result();
        KieSession ksession = runtimeBuilder.newKieSession();
        ksession.insert(JOSE);
        ksession.insert(p);
        ksession.insert(result);
        ksession.fireAllRules();
        return result.isOlder();
 
    }
}
```

Add the tests to check our application is working fine:

```java
package org.sgitario.kogito;

import static io.restassured.RestAssured.given;
import static org.hamcrest.core.Is.is;
import org.junit.jupiter.api.Test;
import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;

@QuarkusTest
public class CheckAgeResourceTest {

	@Test
	public void testPass() {
		given()
			.body("{\"name\":\"Juan\", \"age\": " + 20 + "}")
			.contentType(ContentType.JSON)
			.when().post("/check-age")
			.then().statusCode(200).body(is("true"));
	}

	@Test
	public void testShouldFail() {
		given()
		.body("{\"name\":\"Juan\", \"age\": " + 15 + "}")
		.contentType(ContentType.JSON)
		.when().post("/check-age")
		.then().statusCode(200).body(is("false"));
	}

}
```

Now, we can build and run our application by:

```bash
mvn clean package quarkus:dev
```

Finally, we can deploy our application into Openshift. We'll be doing this using the [Kogito CLI](https://github.com/kiegroup/kogito-cloud-operator#kogito-cli) (instructions about how to install it [here](https://sgitario.github.io/kogito-developer-guide/)). At the moment, Kogito CLI only supports deploying a service if the sources are available in a Git repository, so the first thing we need to do is to push our service into our GitHub account.

| In the future release 1.0.0, it will be possible to deploy a service directly from our localhost using [S2I](https://docs.openshift.com/container-platform/3.6/creating_images/s2i.html) feature from OpenShift.

For this example, the sources are in [my GitHub account](https://github.com/Sgitario/kogito-persons-example), so we can now deploy our service:

```sh
kogito deploy-service kogito-persons-example https://github.com/Sgitario/kogito-persons-example
```

Now, our service is running internally in our cluster and we could make use of it in our businesses. However, this is not all. In this example, we have created a Kie Session and expose the service "manually", however Kogito is designed to do this for you in a cloud native approach. In my next post, I will provide a full example where we'll fully see all the advantages of Kogito. 

## What's next

I will develop the next topics soon:

- Rule bases modularization via Rule Units
- Rule Units orchestration through jBPM workflows
- Seamless integration with Camel routes, Kafka Streams, ...
- Automatic REST endpoint generation returning result of DRL queries

## Conclusions

We can find more information about Kogito in [the official site](https://kogito.kie.org/).
