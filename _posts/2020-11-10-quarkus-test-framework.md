---
layout: post
title: Quarkus Test Framework
date: 2020-11-10
tags: [ Quarkus ]
---

[Quarkus](https://quarkus.io/) is the most recent popular framework for Java applications aimed to be the reference for cloud-native applications. Its competitor is none other than the all-powerful [Spring Boot](https://spring.io/projects/spring-boot). So, Quarkus has a long path to engage users and for achieving this, the developer experience that needs to be rather than excellence and joyful. In this direction, this post is covering how we can test Quarkus apps as developers.

There are two guides of reference from the Quarkus documentation:

- [Getting Started with Testing](https://quarkus.io/guides/getting-started-testing)
- [Test Coverage](https://quarkus.io/guides/tests-with-coverage) using [JaCoCo](https://www.eclemma.org/jacoco/).

Basically, what Quarkus can do so far is summarized in the following table:

| Feature | Description |
| ------- | ----------- |
| [Use random port in Tests](https://quarkus.io/guides/getting-started-testing#controlling-the-test-port) | Via the `quarkus.http.test-port` property |
| [Inject Resources in Tests](https://quarkus.io/guides/getting-started-testing#injecting-a-uri) | Using the `@TestHTTPResource` annotation |
| [Inject Beans in Tests](https://quarkus.io/guides/getting-started-testing#injection-into-tests) | Using the `@Inject` annotation |
| [Support Transactions](https://quarkus.io/guides/getting-started-testing#applying-interceptors-to-tests) | |
| [Support Test Profiles](https://quarkus.io/guides/getting-started-testing#testing_different_profiles) | |
| [Support of Native testing](https://quarkus.io/guides/getting-started-testing#native-executable-testing) | Using the `@NativeImageTest` annotation |
| [Integration with RestAssured](https://quarkus.io/guides/getting-started-testing#restassured) | |
| [Integration with Mockito](https://quarkus.io/guides/getting-started-testing#mock-support) | We can provide our mocks using `QuarkusMock` or spy beans using `InjectSpy` |
| [Integration with Testcontainers](https://quarkus.io/guides/getting-started-testing#quarkus-test-resource) | Using `QuarkusTestResourceLifecycleManager` interface |

Having clear what the Quarkus test framework copes with, I'll try to cover more advanced scenarios in this post. 

If you are interested in more advanced features, you can use a new Quarkus Test Framework that the Quarkus QE team has released for:

| Feature |
| ------- |
| Allow to deploy multiple Quarkus applications |
| Allow to update build/runtime properties during the test case |
| Support start/stop for Quarkus applications and third parties containers |
| Allow to deploy scenarios into OpenShift and Kubernetes with zero changes |
| Support verification of application logs and metrics |
| Produce test execution traces and metrics |

And much more! To know more about this framework, please visit [the GitHub site](https://github.com/quarkus-qe/quarkus-test-framework) and [my blog post](https://sgitario.github.io/quarkus-qe-test-framework/).

## Getting Started

- Add the Quarkus JUnit 5 extension in your `pom.xml`:

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-junit5</artifactId>
    <scope>test</scope>
</dependency>
```

- Configure the logging support in surefire maven plugin:

```xml
<plugin>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>${surefire-plugin.version}</version>
    <configuration>
       <systemPropertyVariables>
          <java.util.logging.manager>org.jboss.logmanager.LogManager</java.util.logging.manager>
       </systemPropertyVariables>
    </configuration>
</plugin>
```

Now, we're ready to start writing our services and tests! See [this example](https://quarkus.io/guides/getting-started-testing#recap-of-http-based-testing-in-jvm-mode) as a reference.

## How to Test Several Quarkus Applications At Once

The idea would be to have several running Quarkus apps using the [JUnit 5 extensions](https://junit.org/junit5/docs/current/user-guide/#extensions), something like:

```java
public class MyTest {

    @RegisterExtension
    static final Extension firstQuarkusApp = // ...

    @RegisterExtension
    static final Extension secondQuarkusApp = // ...

    @Test
    public void test() {
        // ...
    }
}
``` 

Or:

```java
@ExtendWith({ FirstAppExtension.class, SecondAppExtension.class })
public class MyTest {

    @Test
    public void test() {
        // ...
    }
}
``` 

Let's recap first, the `quarkus-junit5` dependency provides functionality to test one Quarkus application:

```java
@QuarkusTest
public class GreetingResourceTest {

    @Test
    public void testHelloEndpoint() {
        given()
          .when().get("/hello")
          .then()
             .statusCode(200)
             .body(is("hello"));
    }
}
```

The hello endpoint must be in the source folder of our project. Even we can test our Quarkus application using different profiles:

```java
public class MockGreetingProfile implements QuarkusTestProfile {

    @Override
    public Map<String, String> getConfigOverrides() { 
        return Collections.singletonMap("quarkus.resteasy.path","/api");
    }

    // ...
}
```

```java
@QuarkusTest
@TestProfile(MockGreetingProfile.class)
public class GreetingResourceTest {

    @Test
    public void testHelloEndpoint() {
        given()
          .when().get("/api/hello")
          .then()
             .statusCode(200)
             .body(is("hello"));
    }
}
```

But what about to have several running Quarkus apps totally independently from each other? Quarkus has a really good build-in support for doing this using [ShrinkWrap](https://developer.jboss.org/docs/DOC-14138). Let's see how doing this:

First, we need to add an hidden dependency provided by Quarkus:

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-junit5-internal</artifactId>
    <scope>test</scope>
</dependency>
```

Note that this dependency is intended to be only used by Quarkus extension developers. However, we might want to build/test our custom Quarkus extension and might also want to have two running independent Quarkus applications!

Now, we have available these Quarkus extensions: 

| Extensions | Can Be Used For Quarkus Apps | Reload | Debug | Can Deploy Multiple Extensions | Documented |
| ------- | ----------- |
| [QuarkusUnitTest](https://quarkus.io/guides/writing-extensions#testing-extensions) | No | No | Yes | Yes | Yes |
| [QuarkusDevModeTest](https://quarkus.io/guides/writing-extensions#testing-hot-reload) | Yes | Hot reload (modify methods) | Yes | No | Yes |
| QuarkusProdModeTest | Yes | Restart (start/stop methods) | No | Yes | No |

The only extension that is fully focused for Quarkus extension development is QuarkusUnitTest, so I'll skip it as our purpose is to use it for testing several Quarkus Apps at once.

About the QuarkusDevModeTest, this class implements the [TestInstanceFactory](https://junit.org/junit5/docs/5.3.0/api/org/junit/jupiter/api/extension/TestInstanceFactory.html) interface and JUnit 5 does not support having multiple extensions that implement this interface in the same test. So, I'll skip this extension too.

So, the only possibility to achieve our goal is to use the QuarkusProdModeTest extension. Let's use it to have a ping pong example.

### Ping Pong Example

We're going to build two Quarkus Applications that:
- Pong application with a REST endpoint that returns "pong":

```java
@Path("/pong")
public class PongResource {

    @GET
    @Produces(MediaType.TEXT_PLAIN)
    public String getPong() {
        return "pong";
    }
}
```

And the `pong.properties`:

```
# Quarkus
quarkus.http.port=8082
```

- Ping application with a REST endpoint that returns "ping" plus the output of the Pong application, so it should return "ping pong" in the end:

```java
@Path("/ping")
public class PingResource {

    @Inject
    @RestClient
    PongClient pongClient;

    @GET
    @Produces(MediaType.TEXT_PLAIN)
    public String getPing(@Context HttpHeaders headers) {
        return "ping " + pongClient.getPong();
    }
}
```

```java
@RegisterRestClient
public interface PongClient {
    @GET
    @Path("/pong")
    @Produces(MediaType.TEXT_PLAIN)
    String getPong();

}
```

And the `ping.properties`:

```
# Quarkus
quarkus.http.port=8081

# RestClient
io.quarkus.qe.ping.PongClient/mp-rest/url=http://localhost:8082 
io.quarkus.qe.ping.PongClient/mp-rest/scope=javax.inject.Singleton 
```

And this is how would look like the test running both applications independently:

```java
public class PingPongResourceTest {

    private static final int PING_PORT = 8081;
    private static final int PORT_PORT = 8082;

    @RegisterExtension
    static final QuarkusProdModeTest pongApp = new QuarkusProdModeTest()
            .setArchiveProducer(() -> ShrinkWrap.create(JavaArchive.class)
                    .addClasses(PongResource.class))
            .setRun(true)
            .setRuntimeProperties(PropertiesUtils.toMap("pong.properties"));

    @RegisterExtension
    static final QuarkusProdModeTest pingApp = new QuarkusProdModeTest()
            .setArchiveProducer(() -> ShrinkWrap.create(JavaArchive.class)
                    .addClasses(PingResource.class, PongClient.class))
            .setRun(true)
            .setRuntimeProperties(PropertiesUtils.toMap("ping.properties"));

    @Test
    public void testPong() {
        given()
                .when().port(PORT_PORT).get("/pong")
                .then().statusCode(HttpStatus.SC_OK)
                .body(is("pong"));
    }

    @Test
    public void testPingPong() {
        given()
                .when().port(PING_PORT).get("/ping")
                .then().statusCode(HttpStatus.SC_OK)
                .body(is("ping pong"));
    }
}
```

We'll see that both applications are running as expected:

```
 ____  __  _____   ___  __ ____  ______ 
 --/ __ \/ / / / _ | / _ \/ //_/ / / / __/ 
 -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \   
--\___\_\____/_/ |_/_/|_/_/|_|\____/___/   
10:52:30 INFO  traceId=, parentId=, spanId=, sampled= [io.quarkus] (main) Quarkus 1.9.2.Final on JVM started in 1.954s. Listening on: http://0.0.0.0:8081
10:52:30 INFO  traceId=, parentId=, spanId=, sampled= [io.quarkus] (main) Profile prod activated. 
10:52:30 INFO  traceId=, parentId=, spanId=, sampled= [io.quarkus] (main) Installed features: [cdi, rest-client, resteasy]
__  ____  __  _____   ___  __ ____  ______ 
 --/ __ \/ / / / _ | / _ \/ //_/ / / / __/ 
 -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \   
--\___\_\____/_/ |_/_/|_/_/|_|\____/___/   
10:52:36 INFO  traceId=, parentId=, spanId=, sampled= [io.quarkus] (main) Quarkus 1.9.2.Final on JVM started in 1.664s. Listening on: http://0.0.0.0:8082
10:52:36 INFO  traceId=, parentId=, spanId=, sampled= [io.quarkus] (main) Profile prod activated. 
10:52:36 INFO  traceId=, parentId=, spanId=, sampled= [io.quarkus] (main) Installed features: [cdi, rest-client, resteasy]
```

Note that we have used `setRuntimeProperties` because of an issue with the QuarkusProdModeTest extension (see conclusions). Another issue is that we can't debug our endpoints using the IDE (we should attach the pid manually).

## Conclusions

- Quarkus is tied up with [the JBoss Logging](https://docs.jboss.org/hibernate/orm/4.3/topical/html/logging/Logging.html) framework which makes odd having to define the logging manager in surefire/failsafe plugins to see the traces:

```xml
<plugin>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>${surefire-plugin.version}</version>
    <configuration>
       <systemPropertyVariables>
          <java.util.logging.manager>org.jboss.logmanager.LogManager</java.util.logging.manager>
          <maven.home>${maven.home}</maven.home>
       </systemPropertyVariables>
    </configuration>
</plugin>
```

- The QuarkusDevModeTest JUnit Quarkus extension implements the TestInstanceFactory interface. 
- The QuarkusProdModeTest JUnit Quarkus extension is not documented at all.
- The QuarkusProdModeTest JUnit Quarkus extension is ignoring the application.properties:

Using QuarkusDevModeTest, we can do this:

```java
@RegisterExtension
static final QuarkusDevModeTest config = new QuarkusDevModeTest()
        .setArchiveProducer(() -> ShrinkWrap.create(JavaArchive.class)
            .addClasses(PongResource.class)
            .addAsResource("pong.properties", "application.properties"));

@Test
public void testPong() {
    given()
    .when().port(PongApplicationExtension.PORT).get("/pong")
    .then().statusCode(HttpStatus.SC_OK)
    .body(is("pong"));
}
```

This example works fine, but using QuarkusProdModeTest, it's not working:

```java
@RegisterExtension
static final QuarkusProdModeTest config = new QuarkusProdModeTest()
        .setArchiveProducer(() -> ShrinkWrap.create(JavaArchive.class)
            .addClasses(PongResource.class)
            .addAsResource("pong.properties", "application.properties"))
        .setRun(true);

@Test
public void testPong() {
    given()
    .when().port(PongApplicationExtension.PORT).get("/pong")
    .then().statusCode(HttpStatus.SC_OK)
    .body(is("pong"));
}
```

It fails because the `pong.properties` is ignored. Also, using `.withConfigurationResource("pong.properties")` is ignored. Therefore, the only workaround that I found is:

```java
@RegisterExtension
static final QuarkusDevModeTest config = new QuarkusDevModeTest()
        .setArchiveProducer(() -> ShrinkWrap.create(JavaArchive.class)
            .addClasses(PongResource.class))
        .setRun(true)
        .setRuntimeProperties(PropertiesUtils.toMap("ping.properties"));

@Test
public void testPong() {
    given()
    .when().port(PongApplicationExtension.PORT).get("/pong")
    .then().statusCode(HttpStatus.SC_OK)
    .body(is("pong"));
}
```

- Source Code can be found [here](https://github.com/Sgitario/quarkus-examples/tree/main/quarkus-opentracing).