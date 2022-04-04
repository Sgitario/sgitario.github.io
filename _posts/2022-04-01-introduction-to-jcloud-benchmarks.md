---
layout: post
title: Introduction to jCloud Benchmarks
date: 2022-04-01
tags: [ Java, jCloud, JMH, Benchmark ]
---

Nowadays, the [Java Microbenchmark Harness (JMH)](https://github.com/openjdk/jmh) tool is a very popular and well-known tool to write benchmarks in Java. Also, this tool is maintained by [OpenJDK](https://openjdk.java.net/projects/code-tools/jmh/) and you can find thousands of guides on the web about how to write all kinds of benchmarks - from throughput to memory benchmarks. For example, a very simple benchmark would look like as:

```java
int x = 1;
int y = 2;

@Benchmark
public int measure() {
    return (x + y);
}
```

This benchmark will measure the sum operation of two integers.

But how can we use JMH to benchmark our application as a whole? Let's say we want to write the next benchmark to call my application via a REST endpoint:

```java
@Benchmark
public String measureRest() {
    // call http://host:port/path from my application
}
```

**Note**: Using REST endpoints to run benchmarks will indirectly add some lags due to the network layer. 

To write this benchmark, we would first need to start/stop our application before and after running the benchmark to ensure we have a fresh run and hence have more realistic results. So, how can we do this in Java and that works fine with JMH? This is where [jCloud](https://github.com/Sgitario/jcloud-unit/) comes to play here to ease up these things.

[jCloud](https://github.com/Sgitario/jcloud-unit/) is a JUnit 5 extension that I've been working in recently. This testing framework is the evolution of the [Quarkus QE Testing Framework](https://github.com/quarkus-qe/quarkus-test-framework) which much more features more focused on the cloud world rather than only [Quarkus](https://quarkus.io/). For example, using jCloud we can easily write test scenarios like:

```java
@JCloud
public class KeycloakGreetingResourceIT {
    private static final String CLIENT_ID = "test-application-client";
    private static final String CLIENT_SECRET = "test-application-client-secret";

    @Container(image = "quay.io/keycloak/keycloak:14.0.0", expectedLog = "Admin console listening", ports = 8080)
    static DefaultService keycloak = new DefaultService().withProperty("KEYCLOAK_IMPORT", "resource::/keycloak-realm.json");

    @Quarkus
    static RestService secured = new RestService().withProperty("quarkus.oidc.auth-server-url", this::getRealmUrl)
            withProperty("quarkus.oidc.client-id", CLIENT_ID)
            .withProperty("quarkus.oidc.credentials.secret", CLIENT_SECRET);

    @Test
    public void testSecuredEndpointWithInvalidToken() {
        secured.given().get("/hello").then().statusCode(HttpStatus.SC_UNAUTHORIZED);
    }

    // ...
}
```

**Note**: The above example can be found [here](https://github.com/Sgitario/jcloud-unit/blob/main/examples/quarkus-oidc/src/test/java/io/jcloud/examples/quarkus/oidc/KeycloakGreetingResourceIT.java).

This test will start a Keycloak container using the `quay.io/keycloak/keycloak` image and also a Quarkus application which sources are placed in the current module. To run this test using Maven, we simply run `mvn verify -Dit.test=KeycloakGreetingResourceIT` and the container and the Quarkus application will be started locally. But the greatest feature in jCloud is that if we annotate the `KeycloakGreetingResourceIT` class with `RunOnKubernetes` or we provide the system property `-Dts.scenario.target=kubernetes` to the Maven command, the test will start the container and the Quarkus application in Kubernetes (you need to be logged into a Kubernetes cluster beforehand). 

At this moment, jCloud supports deployments on local and Kubernetes of Quarkus, Spring Boot and containers applications.

After having introduced the jCloud testing framework, wouldn't it be great if we could use the services that we deploy as part of the test scenarios to run benchmarks? For example:

```java
@JCloud
public class GreetingResourceIT {   

    @Quarkus
    static RestService app = new RestService();

    @Benchmark
    public Object measure() {
        return app.given().get("/hello");
    }
}
```

This is exactly what [the jCloud benchmark](https://github.com/Sgitario/jcloud-unit#jcloud-benchmarks) extension is for.
For using it, after installing this dependency, you simply need to implement the `EnableBenchmark` interface:

```java
@JCloud
public class GreetingResourceIT implements EnableBenchmark {   

    @Quarkus
    static RestService app = new RestService();

    @Benchmark
    public Object measure() {
        return app.given().get("/hello");
    }
}
```

And then all the JMH features and annotations will work as part of the scenario execution.

The benchmark results are saved at target/benchmarks-results/GreetingResourceIT.json. We can visualize the benchmark results by submitting this file in a JMH visualizer like [this](https://jmh.morethan.io/).

## Runtime Application Benchmarks

In the previous section, we have seen how to implement JMH benchmarks and deploying services using the jCloud tool. 

Let's now write a benchmark to measure the performance of runtime frameworks like Spring Boot and Quarkus. The idea would be to compare the performance of the same applications written in Quarkus and Spring Boot.

### The Ping Pong REST application

What is the easiest possible REST application? One with a GET endpoint that returns a string. Let's implement this application in the different runtime frameworks.

- Quarkus:

```java
import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;

@Path("/ping")
public class PingResource {

    @GET
    @Produces(MediaType.TEXT_PLAIN)
    public String ping() {
        return "pong";
    }
}
```

Internally, Quarkus can be configured to use [Resteasy classic](https://quarkus.io/guides/rest-json) or [Resteasy Reactive](https://quarkus.io/guides/resteasy-reactive). 
You can find these implementations in [here](https://github.com/Sgitario/frameworks-benchmarks/tree/main/rest-benchmark).

- Spring Boot:

For Spring, we need two classes, the controller and the application:

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class GreetingApplication {

    public static void main(String[] args) {
        SpringApplication.run(GreetingApplication.class, args);
    }
}

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class GreetingResource {

    @GetMapping("/ping")
    public String pong() {
        return "pong";
    }
}
```

You can find these implementations in [here](https://github.com/Sgitario/frameworks-benchmarks/tree/main/rest-benchmark).

### The Throughput Benchmark

Let's now measure the overall throughput (operations per second) of these applications: Quarkus using Resteasy Classic, Quarkus using Resteasy Reactive and Spring Boot Web.

jCloud also supports the deployment of services that are in other source locations and this is what will be doing here. 
The source locations of our applications are [here](https://github.com/Sgitario/frameworks-benchmarks/tree/main/rest-benchmark), so let's start writing our benchmark:

```java
@JCloud
public class ThroughputBenchmarks implements EnableBenchmark {

    @Quarkus(location = "../quarkus-resteasy-reactive")
    public static HttpService quarkusReactive = new HttpService();

    @Quarkus(location = "../quarkus-resteasy-classic")
    public static HttpService quarkusClassic = new HttpService();

    @Spring(location = "../spring-boot-web")
    public static HttpService springWeb = new HttpService();

    @Benchmark
    public String quarkusResteasyReactive() {
        return runBenchmark(quarkusReactive);
    }

    @Benchmark
    public String quarkusResteasyClassic() {
        return runBenchmark(quarkusClassic);
    }

    @Benchmark
    public String springWeb() {
        return runBenchmark(springWeb);
    }

    private String runBenchmark(HttpService service) {
        HttpResponse<String> response = service.getString("/ping");
        assertEquals("pong", response.body());
        return response.body();
    }
}
```

**Note**: The `HttpService` implementation uses internally the new Java Http Client that is more suitable for performance testing than the RestAssured framework.

The problem with the above benchmarks is that we're running the three applications at the same time, so the measurements might be compromised. 
Thankfully, jCloud also supports auto-start of the services by using `.setAutoStart(false)`, so the services will be by default stopped and we can now use JMH states to start and stop the services before and after each benchmark by using the `ServiceState` implementation from jCloud:

```java
@Quarkus(location = "../quarkus-resteasy-reactive")
public static HttpService quarkusReactive = new HttpService().setAutoStart(false);

public static class QuarkusResteasyReactiveState extends ServiceState<HttpService> {

    public QuarkusResteasyReactiveState() {
        super(quarkusReactive);
    }
}
```

Where `ServiceState` is simply an utility to avoid repeating the same code for all the services:

```java
@State(Scope.Benchmark)
public class ServiceState<T extends Service> {

    private final T service;

    public ServiceState(T service) {
        this.service = service;
        if (service.isRunning()) {
            throw new IllegalStateException(
                    "Service is already running! You need to declare services using `.setAutoStart(false)`");
        }
    }

    public T getService() {
        return service;
    }

    @Setup(Level.Trial)
    public void doSetup() {
        service.start();
    }

    @TearDown(Level.Trial)
    public void doTearDown() {
        service.stop();
    }
}
```

And now, we can inject these states in our benchmark (all this is done by JMH):

```java
@Benchmark
public String quarkusResteasyReactive(QuarkusResteasyReactiveState state) {
    return runBenchmark(state);
}
```

The final version of this benchmark can be found [here](https://github.com/Sgitario/frameworks-benchmarks/blob/main/rest-benchmark/benchmarks/src/test/java/io/sgitario/benchmarks/rest/ThroughputBenchmarks.java).

After running the benchmark using the Maven command `mvn clean install`, the benchmark results will be generated at `target/benchmarks-results/ThroughputBenchmarks.json` and if we visualize it using the JMH visualizer, we would see something along this plot:

![benchmark results]({{ site.url }}{{ site.baseurl }}/images/quarkus-spring-rest-performance.png)

## Conclusions

The idea of the jCloud benchmark was to ease up the implementation of benchmarks for applications and this can be useful for other users as well. 

For the future, I will include more relevant and useful applications to measure and more different benchmarks in [https://github.com/Sgitario/frameworks-benchmarks](https://github.com/Sgitario/frameworks-benchmarks).