---
layout: post
title: Quarkus Testing with Test Containers
date: 2020-11-16
tags: [ quarkus, test ]
---

In my [previous post](https://sgitario.github.io/quarkus-test-framework/), I covered how to test several Quarkus applications using internal Quarkus tooling. This approach works perfectly fine specially when we only want to cope with a single module. However, if we are working in a multi-modular project, there is an easier approach using [Testcontainers](https://www.testcontainers.org/) and [Docker](https://www.docker.com/). Let's see how to do this!

## Context

Let's start with a multi-module project similar to:

Parent
└── Quarkus App 1
└── Quarkus App 2
└── Integration Tests

The goal is to have running Docker images at the Integration Tests stage, so we can use Testcontainers to start these images.

## The Quarkus Container Image JIB extension

In order to configure Quarkus to build a docker image when compiling each Quarkus Apps project, we need to use [the Quarkus Container Image JIB](https://quarkus.io/guides/container-image) extension.

To enable this extension, we need to add it into the `pom.xml`:

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-container-image-jib</artifactId>
</dependency>
```

And configure the Docker image attributes in the `application.properties`:

```
# Docker
quarkus.container-image.build=true # This property will enable the extension to build the image into Docker
quarkus.container-image.group=my-group
quarkus.container-image.name=quarkus-app-one
```

## Set Up the Integration Tests

We are writing integration tests, so we need to use [the failsafe](https://maven.apache.org/surefire/maven-failsafe-plugin/) Maven plugin. The first thing we need to do is to ensure we have built our Quarkus Apps modules by configuring the failsafe plugin dependencies in our `pom.xml`:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-failsafe-plugin</artifactId>
            <dependencies>
                <!-- ensure modules were built, but not added in the 
                    classpath -->
                <dependency>
                    <groupId>org.sgitario.quarkus.examples</groupId>
                    <artifactId>my-app-one</artifactId>
                    <version>${project.version}</version>
                    <scope>runtime</scope>
                </dependency>
                <dependency>
                    <groupId>org.sgitario.quarkus.examples</groupId>
                    <artifactId>my-app-one</artifactId>
                    <version>${project.version}</version>
                    <scope>runtime</scope>
                </dependency>
            </dependencies>
            <executions>
                <execution>
                    <id>run-tests</id>
                    <phase>integration-test</phase>
                    <goals>
                        <goal>integration-test</goal>
                        <goal>verify</goal>
                    </goals>
                </execution>
            </executions>
            <configuration>
                <systemPropertyVariables>
                    <container.image.my-app-one>my-group/quarkus-app-one:${project.version}</container.image.my-app-one>
                    <container.image.my-app-two>my-group/quarkus-app-two:${project.version}</container.image.my-app-two>
                </systemPropertyVariables>
                <classesDirectory>${project.build.outputDirectory}</classesDirectory>
            </configuration>
        </plugin>
    </plugins>
  </build>
```

| Note that this is not required when we have a multi module project, but better be safe.

Then we need to add the testcontainers dependencies:

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.7.0</version>
    <scope>test</scope>
</dependency>
```

Next, let's configure our full deployment by writing a custom JUnit 5 extension:

```java
public class MyFullDeploymentExtension implements BeforeAllCallback, AfterAllCallback {

    public static final String MY_FIRST_APP_ALIAS = "firstApp";

    private Network network;
    private MyCustomServiceContainer myFirstService;
    private MyCustomServiceContainer mySecondService;

    @Override
    public void beforeAll(ExtensionContext context) throws Exception {
        network = Network.newNetwork();

        myFirstService = new MyCustomServiceContainer("one", 8081);
        myFirstService.withNetwork(network);
        myFirstService.withNetworkAliases(MY_FIRST_APP_ALIAS);
        myFirstService.start();

        mySecondService = new MyCustomServiceContainer("two", 8082);
        mySecondService.withNetwork(network);
        mySecondService.start();
    }

    @Override
    public void afterAll(ExtensionContext context) throws Exception {
        closeSilently(mySecondService);
        closeSilently(myFirstService);
    }

    public int getMyFirstServicePort() {
        return myFirstService.getMappedPort();
    }

    public int getMySecondServicePort() {
        return mySecondService.getMappedPort();
    }

    private static final void closeSilently(AutoCloseable object) {
        // ...
    }
}
```

We are configuring our services to be running in the same network, so each one can interact with each other. We should do the same if running another third services like databases or message brokers.

Where `MyCustomServiceContainer` is:

```java
public class MyCustomServiceContainer extends GenericContainer<BaseServiceContainer> {

    private static final Logger LOGGER = LoggerFactory.getLogger(BaseServiceContainer.class);

    private static final String IMAGE = "container.image.";

    private final int port;

    public MyCustomServiceContainer(String name, int port) {
        super(System.getProperty(IMAGE + name, getDefaultImageValue(name)));
        this.port = port;

        withLogConsumer(new Slf4jLogConsumer(LOGGER));
        waitingFor(Wait.forLogMessage(".*Listening on:.*", 1));
        withExposedPorts(port);
    }

    public int getMappedPort() {
        return getMappedPort(port);
    }

    private static final String getDefaultImageValue(String name) {
        // This is to IDE compability only.
        return String.format("my-group/quarkus-app-%s:1.0.0-SNAPSHOT", name);
    }
}
```

Also, we can overwrite the default properties by using environment variables.

## Let's Write The Test

Let's write a test to verify that the health endpoint returns OK for all our Quarkus apps:

```java
@Testcontainers
public class HealthIT {

    private static final String HEALTH_PATH = "/health";

    @RegisterExtension
    static final MyFullDeploymentExtension deployment = new MyFullDeploymentExtension();

    @BeforeAll
    public static void beforeAll() {
        RestAssured.defaultParser = Parser.JSON;
    }

    @Test
    public void myFirstServiceHealthEndpointShouldBeOk() {
        givenMyFirstQuarkusAppEndpoint().get(HEALTH_PATH).then().statusCode(HttpStatus.SC_OK);
    }

    @Test
    public void mySecondServiceHealthEndpointShouldBeOk() {
        givenMySecondQuarkusAppEndpoint().get(HEALTH_PATH).then().statusCode(HttpStatus.SC_OK);
    }

    private RequestSpecification givenMyFirstQuarkusAppEndpoint() {
        return given().port(deployment.getMyFirstServicePort());
    }

    private RequestSpecification givenMySecondQuarkusAppEndpoint() {
        return given().contentType(ContentType.JSON).accept(ContentType.JSON).port(deployment.getMySecondServicePort());
    }
} 
```

Simply like this! :)