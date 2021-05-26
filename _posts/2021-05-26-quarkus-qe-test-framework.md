---
layout: post
title: Quarkus QE Test Framework
date: 2021-05-26
tags: [ Quarkus ]
---

In my entire career, there is something I've always been very passionated about: frameworks. I think everything started the first time I met the Spring ecosystem or maybe when I was a .NET developer. Either way, I've always enjoyed really curated APIs and the usage of well known patterns to integrate different components. And that's also why I was so excited when I joined in [the Quarkus](https://github.com/quarkusio/quarkus) project as Quality Engineer. 

My contributions in Quarkus are related to ensure the quality of integrations and the framework. So far, we have a lot of test suites that cover:
- OpenShift deployments
- Baremetal testing
- Dev mode testing

However, from the beginning there was something I was missing from the beginning: an single test suite that allow us to write a single test to be run against any kind of environment (OpenShift/K8s/AWS/...) and easily deploy multiple Quarkus applications as part of the same test case. And this was my motivation to propose a new [Quarkus QE test framework](https://github.com/quarkus-qe/quarkus-test-framework).

I'm pleased to announce that [the first release](https://github.com/quarkus-qe/quarkus-test-framework/releases/tag/0.0.1) of the Quarkus QE test framework has been already released and the next release is already full of surprises!

The main features are:
- Easily deploy multiple Quarkus applications and third party components in a single test
- Write the test case once and run it everywhere (cloud, bare metal, etc)
- Developer and Test friendly
- Quarkus features focused (allow defining custom source classes, build/runtime properties, etc)
- Test isolation: for example, in OpenShift or Kubernetes, tests will be executed in an ephemeral namespace 

## Getting Started

In order to write Quarkus application in your tests, you first need to add the core dependency in your `pom.xml` file;

```xml
<dependency>
	<groupId>io.quarkus.qe</groupId>
	<artifactId>quarkus-test-core</artifactId>
</dependency>
```

Then, let's write our first test case:

```java
@QuarkusScenario
public class PingPongResourceIT {

    @QuarkusApplication(classes = PingResource.class)
    static final RestService pingApp = new RestService();

    @QuarkusApplication(classes = PongResource.class)
    static final RestService pongApp = new RestService();

    // will include all the classes within the current module including the ping and pong resources
    @QuarkusApplication
    static final RestService pingPongApp = new RestService();

    @Test
    public void shouldPingWorks() {
        ping.given().get("/ping").then().statusCode(HttpStatus.SC_OK).body(is("ping"));
        ping.given().get("/pong").then().statusCode(HttpStatus.SC_NOT_FOUND);
    }

    @Test
    public void shouldPongWorks() {
        pong.given().get("/pong").then().statusCode(HttpStatus.SC_OK).body(is("pong"));
        pong.given().get("/ping").then().statusCode(HttpStatus.SC_NOT_FOUND);
    }

    @Test
    public void shouldPingPongWorks() {
        pingpong.given().get("/ping").then().statusCode(HttpStatus.SC_OK).body(is("ping"));
        pingpong.given().get("/pong").then().statusCode(HttpStatus.SC_OK).body(is("pong"));
    }
}
```

This test spans three Quarkus applications in a single test case. 

We can run our test suite by:

```
mvn clean verify
```

We've writen our test case once, let's run it in different environments:

- Native

```
mvn clean verify -Dnative
```

The framework will build all the Quarkus applications in Native and run it. All you need to do is to add a Maven profile as stated in [the docs](https://github.com/quarkus-qe/quarkus-test-framework#native).

- OpenShift

If our test case supports OpenShift scenarios, we need to explicitly say it by extending our test case:

```java
@OpenShiftScenario
public class OpenShiftPingPongResourceIT extends PingPongResourceIT {
}
```

Which is even more useful, we can choose the deployment strategy to use between:
- `@OpenShiftScenario(deployment = OpenShiftDeploymentStrategy.Build)` - uses a custom template to deploy the components into OpenShift
- `@OpenShiftScenario(deployment = OpenShiftDeploymentStrategy.UsingContainerRegistry)` - will push the Quarkus application image into an intermediate container registry that will use OpenShift to pull the Quarkus application image from
- `@OpenShiftScenario(deployment = OpenShiftDeploymentStrategy.UsingOpenShiftExtension)` - will use the Quarkus OpenShift extension to deploy the Quarkus application (this method does not support to select custom classes)
- `@OpenShiftScenario(deployment = OpenShiftDeploymentStrategy.UsingOpenShiftExtensionAndDockerBuildStrategy)` - will use the Quarkus OpenShift extension and Docker build strategy to deploy the Quarkus application (this method does not support to select custom classes)

- Kubernetes

Similarly than in OpenShift:

```java
@KubernetesScenario
public class KubernetesPingPongResourceIT extends PingPongResourceIT {
}
```

### Containers

What about tests that need integration with third party components like databases or message brokers? We could easily use [Testcontainers](https://www.testcontainers.org/) to achieve this feature, however this test would not be compatible when running it in OpenShift or Kubernetes or other cloud infrastructure. So, test framework to the rescue!

First, we need an additional module:

```xml
<dependency>
	<groupId>io.quarkus.qe</groupId>
	<artifactId>quarkus-test-containers</artifactId>
</dependency>
```

Now, we can deploy services via docker:

```java
@QuarkusScenario
public class GreetingResourceIT {

    private static final String CUSTOM_PROPERTY = "my.property";

    @Container(image = "quay.io/bitnami/consul:1.9.3", expectedLog = "Synced node info", port = 8500)
    static final DefaultService consul = new DefaultService();

    @QuarkusApplication
    static final RestService app = new RestService();

    // ...
}
```

When running it on baremetal, internally it's using Testcontainers. But we have an abstraction layer of the container that makes the service worked also when extending the test suite in OpenShift and Kubernetes.

### Architecture

This framework is designed to follow **extension model** patterns. Therefore, we can extend any functionality just by adding other dependencies that extend the current functionality. As an example, Quarkus applications will be deployed locally, but if we add the OpenShift or the Kubernetes modules, we can run the same test it in OpenShift/K8s just by adding the `@OpenShiftScenario` or `@KubernetesScenario` as we have just seen above.

![Create a new Skill form]({{ site.url }}{{ site.baseurl }}/images/quarkusqe_test_framework_components.png)

## Future

There is still a lot of working in progress and some of the future enhancements are:

- Run tests in parallel (at the moment, this is limited by the usage of testcontainers and the OC CLI tool)
- Add AWS support
- Add Source to Image deployments support for OpenShift
- Add tracing of measurements for test execution and service start/stop performance