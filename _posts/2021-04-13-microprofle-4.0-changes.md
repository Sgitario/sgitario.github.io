---
layout: post
title: MicroProfile 4.0 Changes
date: 2021-04-13
tags: [ Java ]
---

MicroProfile 4.0 was released on December 2020 and announced [here](https://microprofile.io/2020/12/23/microprofile-4-0-is-now-available/).

Let's see the most relevant changes between 3.0 and 4.0 releases!

## Changes that affect all modules

- **Updated to use [Jakarta EE 8](https://jakarta.ee/release/8/) dependencies**

## MicroProfile Config 2.0

- **Added bulk extraction of properties into POJO using @ConfigProperties**

Now, we can inject a group of properties with the same prefix into a POJO. 

For example, having these properties:

```
server.host = localhost
server.port=9080
server.endpoint=query
server.old.location=London
```

Then, we can map these properties onto a POJO class doing:

```java
@ConfigProperties(prefix="server")
@ApplicationScoped
public class Details {
    String host;
    int port;
    String endpoint;
    @ConfigProperty(name="old.location")
    String location;
}
```

- **Property Expression Enhancements**

The logic to resolve properties has been enhanced. For example, we can now do:

```
server.url=http://${server.host}/endpoint
server.host=example.org
```

And the property `server.url` will be `http://example.org/endpoint`.

Moreover, we can achieve the following syntact for property expressions:

```
server.url=http://${server.host:default.org}/endpoint # default properties
server.url=http://${host:${default-host}}/endpoint # nested properties
server.url=http://${server.host}/${path} # composed properties
server.url=\\${server.host} # no substitution: we would get "${server.host}"
```

| Note that this can be a breaking change, so if you need to disable it by default, you need to set `mp.config.property.expressions.enabled` to `false`.

Finally, we can inject directly `ConfigValue` in our Java classes:

```java
@Dependent
public class MyBean {
    @Inject
    @ConfigProperty(name = "server.host")
    private ConfigValue configValue;

    // configValue.getName() -> server.host
    // configValue.getValue() -> example.org
    // configValue.getSourceName() -> where the value is coming from
    // configValue.getSourceOrdinal() -> the ordinal of the injected value (depends on the source)
    // configValue.getRawValue() -> No substitution: ${server.host}
}
```

- **Added configuration profiles (ex: dev, testing, live)**

We can now control the profile we're using in MicroProfile using the property `mp.config.profile`. 
And then we can make use of this properties via the properties:

```
%dev.vehicle.name=car
%live.vehicle.name=train
%testing.vehicle.name=bike
vehicle.name=lorry
```

Or directly via the properties files:

```
META-INF\microprofile-config.properties
META-INF\microprofile-config-dev.properties
META-INF\microprofile-config-prod.properties
META-INF\microprofile-config-testing.properties
```

- **Changes in the ConfigSource interface**

By adding your own implementation of the `ConfigSource` interface, you can provide a new way to inject properties into your application.

As an example, we have implemented a in memory config source where we'll keep properties in a in memory map:

```java
public class CustomConfigSource implements ConfigSource {

    private static final int ORDINAL = 999;

    Map<String, String> customProperties = new HashMap<>();

    @Override
    public Map<String, String> getProperties() {
        return customProperties;
    }

    @Override
    public int getOrdinal() {
        return ORDINAL;
    }

    @Override
    public String getValue(String propertyName) {
        return customProperties.get(propertyName);
    }

    @Override
    public String getName() {
        return "Custom Config Source";
    }
}
```

The above implementation will not compile when using MicroProfile 4.x since now the method `getProperties()` is optional and the method `getPropertyNames` has to be provided:

```java
public class CustomConfigSource implements ConfigSource {

    private static final int ORDINAL = 999;

    Map<String, String> customProperties = new HashMap<>();

    @Override
    public Set<String> getPropertyNames() {
        return customProperties.keySet();
    }

    @Override
    public int getOrdinal() {
        return ORDINAL;
    }

    @Override
    public String getValue(String propertyName) {
        return customProperties.get(propertyName);
    }

    @Override
    public String getName() {
        return "Custom Config Source";
    }
}
```

- **Empty values are no longer valid**

Having this property:

```
server.url=
```

It will throw a `NoSuchElementException` exception now.

## MicroProfile Health 3.0

- **Default Readiness Statuses (UP/DOWN) for empty check responses**

Prior to MicroProfile 4.0, the status for an empty response was always DOWN. Now, we can control the default status for empty responses by setting:

```
mp.health.default.readiness.empty.response=UP
```

- **The annotation @Health has been removed**

If you were implementation a custom health check such as:

```java
@Health
@ApplicationScoped
public class GreetingHealthCheck implements HealthCheck {

    @Override
    public HealthCheckResponse call() {
        return HealthCheckResponse.builder().name("greeting").up().build();
    }
}
```

You would need to change the annotation `@Health` to either `@Readiness` or `@Liveness` depends on what you're doing in the health check.

- **HealthCheckResponse.state is renamed to status**

Before:

```java
@Produces
@Liveness
HealthCheck check() {
    return () -> HealthCheckResponse.named("heap-memory").state(getMemUsage() < 0.9).build();
}
```

After:

```java
@Produces
@Liveness
HealthCheck check() {
    return () -> HealthCheckResponse.named("heap-memory").status(getMemUsage() < 0.9).build();
}
```

## MicroProfile JWT Auth 1.2

- **New method has been added to get claims from JsonWebToken**

```java
JsonWebToken token = ...
long issuedAtTime = token.getClaim(Claims.iat);
Set<String> groups = token.getClaim(Claims.groups);
// ...
```

- **Support for JWT token cookies**

Before, MicroProfile only supported the header `Authentication` to set the Bearer JWT token. Now, this token can be retrieved from the cookies by setting:

```
mp.jwt.token.header=Cookie # where to find the token. Either `Authorization` or `Cookie`
mp.jwt.token.cookie=mycookiename # the cookie name where to get the JWT token. Default is `Bearer`.
```

## MicroProfile Metrics 3.0

- **Scopes: Base, Application, Vendor**

The following three sets of sub-resource (scopes) are exposed.
- `base`: metrics that all MicroProfile vendors have to provide and exposed under `/metrics/base`
- `vendor`: vendor specific metrics (optional) and exposed under `/metrics/application`
- `application`: application-specific metrics (optional) and exposed under `/metrics/vendor`

- **Timer now exposes total elapsed time duration as a metric value**

Having this bean with the `@Timer` annotation:

```java
public class TimedMethodBean {

    @Timed(name = "timedMethod")
    public void timedMethod() {
        // do something
    }
}
```

Now, we can get the elapsed time this way:

```java
public class TimedMethodBeanLookupTest {

    private final static String TIMER_NAME = MetricRegistry.name(TimedMethodBean.class, "timedMethod");

    private static MetricID timerMID;

    @Inject
    private MetricRegistry registry;

    @Inject
    private TimedMethodBean bean;

    @Test
    @InSequence(2)
    public void getElapsedTime() {
        Timer timer = registry.getTimers().get(timerMID);

        // Call the timed method and assert it's been timed
        bean.timedMethod();

        // Make sure that the timer has been called
        assertNotNull(timer.getElapsedTime());
    }

```

- **New REST metric for unmapped endpoints**

This is a super useful enhancement to retrieve the number of unmapped REST endpoints has been invoked. The new metric name is called `base_REST_request_unmappedException_total` and counts the occurrences of unmapped exceptions for each REST endpoint.

- **CDI producers annotated with @Metric no longer trigger metric registration**

Before, when having this producer:

```java
@Produces
@Metric(name = "hitPercentage")
@ApplicationScoped
protected Gauge<Double> createHitPercentage() {
    return // ...
}
```

The `hitPercentage` metric was automatically registered. Now, we need to explicitly register it using the `MetricRegistry`:

```java
@Inject
MeterRegistry registry;

@Inject
Gauge<Double> hitPercentageGauge;

// ...
registry.register("hitPercentage", hitPercentageGauge);
```

- **MetricRegistry changed from abstract class to interface**
- **Changed `Timer.update(long duration, java.util.concurrent.TimeUnit)` to `Timer.update(java.time.Duration duration)`**
- **Changes in the `MetadataBuilder` API**

| Old Method | New Method | Accepts Null Values |
| ------- | ----------- |
| withOptionalDisplayName | withDisplayName | Yes |
| withOptionalDescription | withDescription | Yes |
| withOptionalType | withType | Yes | 
| withOptionalUnit | withUnit | Yes | 

- **Changes in the `Metadata` API**

Changed existing `getDescription()` and `getUnit()` methods to return String (before they returned Optional<String>).
Added new methods `description()` and `unit()` to return Optional<String>.

## MicroProfile Rest Client 2.0

- **Added support for Server Sent Events**

[Server Sent Events](https://www.w3schools.com/html/html5_serversentevents.asp), part of the HTML 5 spec, enable a server to push data to a client asynchronously via events, over HTTP and is now supported in MicroProfile Rest Client!

The MicroProfile Rest Client specification uses the [Reactive Streams APIs](http://www.reactive-streams.org/) to consume events:

```java
@RegisterRestClient
public interface EventsClient {
    @GET
    @Path("/events/sse")
    @Produces(MediaType.SERVER_SENT_EVENTS)
    Publisher<String> getEvents();
}
```

- **Added support for configuring HTTP proxy servers**

If a REST endpoint needs to be accessed using a proxy server, we can now configure MicroProfile Rest Client to use it. Via properties:

```
com.sgitario.EventsClient/mp-rest/proxyAddress=myproxy.com:2000
```

Or programmatically:

```java
EventsClient client = RestClientBuilder.newBuilder()
                                    .proxyAddress("myproxy.com", 2000)
                                    //...
                                    .build(EventsClient.class);
```

- **Added support for automatically following redirect requests**

If a REST endpoint has been moved, we'll get a 300 HTTP code to indicate the new location. Now, we can configure the MicroProfile Rest Client to automatically redirect onto the new location by configurating the property:

```
com.sgitario.EventsClient/mp-rest/followRedirects=true
```

Or by doing it in Java:

```java
EventsClient client = RestClientBuilder.newBuilder()
                                    .followRedirects(true)
                                    //...
                                    .build(EventsClient.class);
```

## MicroProfile Fault Tolerance 3.0

- **Metric names and scopes changed**

There are two breaking changes in here: (1) the scope of the MicroProfile Fault Tolarence metrics have been moved from `application` to `base`; and (2) the method names are now a `method` tag.

Old metric:
```
application:ft.<name>.timeout.callsTimedOut.total
```

New metric:
```
base:ft.timeout.calls.total{method="<name>", timedOut="true"}
```

- **Lifecycle of circuit breakers and bulkheads is now Singleton**

Therefore, even when doing something like:

```java
@ApplicationScoped // or @RequestScoped
public class MyService {

    @CircuitBreaker(requestVolumeThreshold=2, failureRatio=0.5)
    public void doSomething() {
        // ...
    }
}
```

The `MyService` bean will use the specified scope (Application or Request or other), but the circuit breaker will be a Singleton.

## MicroProfile OpenAPI 2.0

- **Added @SchemaProperty annotation**

Previously, we had to use the same `@Schema` annotation to define properties, for example:

```java
@Schema(name = "Car", description = "Car entity")
public class Car {
    @Schema(description = "The vendor")
    String vendor;
    Engine engine;
}

@Schema(name = "Engine", description = "Engine entity")
public class Engine {
    String type;
}
```

It would generate the next OpenAPI definition:

```yaml
openapi: 3.0.3
info:
  title: Generated API
  version: "1.0"
paths:
components:
  schemas:
    Car:
      description: Car entity
      type: object
      properties:
        engine:
          $ref: '#/components/schemas/Engine'
        vendor:
          description: The vendor
          type: string
    Engine:
      description: Engine entity
      type: object
      properties:
        type:
          type: string
```

Now, we can do this using the new `@SchemaProperty` annotation in one go:

```java
@Schema(name = "Car", description = "Car entity", properties = {
        @SchemaProperty(name = "vendor", description = "The vendor"),
        @SchemaProperty(name = "engine", implementation = Engine.class)
})
public class Car {
    String vendor;
    Engine engine;
}
```

This can be useful when having some complex class hierarchy. 

- **Added @APIResponseSchema annotation**

These annotations ease the way to add descriptions to requests and responses. 

Previously, to define a list of objects in an operation, we had to do:

```java
@GET
@Operation(summary = "Retrieve all Cars")
@APIResponse(responseCode = "200", content = @Content(schema = @Schema(type = SchemaType.ARRAY, implementation = Car.class)))
public Response getCars() {
    // ...
}
```

Now:

```java
@GET
@Operation(summary = "Retrieve all Cars")
@APIResponseSchema(responseCode = "200", value = Car[].class) // or @APIResponseSchema(Car[].class)
public Response getCars() {
    // ...
}
```

- **Added @RequestBodySchema annotation**

For requests body, we can now define the schema this way:

```java
@POST
@Operation(summary = "Add Car")
@Produces(MediaType.TEXT_PLAIN)
public void updateCar(@RequestBodySchema(Car.class) Car car) {
    // ...
}
```

- **Added mp.openapi.schema MicroProfile Config property prefix**

What if we want to add some description to some sources that we can't modify like `java.time.Instant`? We can now do this using the `mp.openapi.schema` properties:

```
mp.openapi.schema.java.time.Instant = { \
        "name": "EpochSeconds", \
        "type": "number", \
        "format": "int64", \
        "title": "Epoch Seconds", \
        "description": "Number of seconds from the epoch of 1970-01-01T00:00:00Z" \
    }
```

- **A lot of removals and updates in the OpenAPI API**

If you would like to see the full list of breaking changes, go to [here](https://download.eclipse.org/microprofile/microprofile-open-api-2.0/microprofile-openapi-spec-2.0.html#_incompatible_changes).

## MicroProfile OpenTracing 2.0

**API deletions**:

- `ScopeManager.active()`: no alternative, the reference Scope has to be kept explicitly since the scope was created.
- `ScopeManager.activate(Span, boolean)`: no alternative auto-finishing has been removed.
- `Scope.span()`: use ScopeManager.activeSpan() or hold the reference to Span explicitly since the span was started.
- `SpanBuilder.startActive()`: use Tracer.activateSpan(Span) instead.
- `Tracer.startManual()`: use Tracer.start() instead.
- `AutoFinishScopeManager`: no alternative, auto-finishing has been removed.