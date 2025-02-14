---
layout: post
title: Propagating MDC Context Across REST Endpoints and Kafka Consumers/Producers
date: 2025-02-14
tags: [ Java ]
---

In this post, we will explore how to propagate the MDC (Mapped Diagnostic Context) among REST endpoints using the JAX-RS API standard and Kafka consumers/producers in Quarkus and Spring Boot services.

# Motivation

Why should we propagate the MDC context across REST endpoints and Kafka?

There are multiple use cases, but one of the most important is troubleshooting. A common scenario is writing integration tests that involve multiple REST endpoints, where we want to track which actions are triggered by each test. By propagating the MDC context, we gain better observability and debugging capabilities.

If your system already has an observability setup (e.g., using OpenTelemetry), tracing context is automatically propagated across REST and Kafka in most frameworks like Spring Boot and Quarkus. For SmallRye Reactive Messaging in Quarkus, you can check [these changes](https://github.com/smallrye/smallrye-reactive-messaging/pull/659) to understand how it works.

# Scenario

Let's introduce the scenario that we'll use during this post. We'll have the following components:
- API service: REST + kafka producer service:

```java
@Path("/api")
public class Resource {

    @Channel("out")
    Emitter<String> emitter;

    @POST
    public void submitAuthor(@RestPath String payload) {
        Log.info("API received message: " + payload);
        emitter.send(payload);
    }
}
```

- Worker service: Kafka consumer service:

```java
@ApplicationScoped
public class MessageConsumer {

    @Incoming("in")
    void consume(String message) {
        Log.infov("Received new message from topic with name `{0}`", message);
        // do something with the message
    }
}
```

If integration tests are poorly designed, data pollution may occur, causing random test failures. Identifying the root cause becomes difficult without proper tracking. To improve troubleshooting, we need a way to track which test triggered each action.

## Creating a test context

To solve this, we introduce a test context, using the test name as a unique identifier. We pass this identifier via an HTTP header ("x-it-test").

## MDC Context

Instead of modifying all log traces, we propagate the "x-it-test" header using the MDC context. To log this header, we configure the logging format:

- For Quarkus:

```
quarkus.log.console.format=[it-test=%X{x-it-test}] %d{yyyy-MM-dd HH:mm:ss,SSS} %-5p [%c{3.}] (%t) %s%e%n
```

More information [here](https://quarkus.io/guides/logging#use-mdc-to-add-contextual-log-information).

- For Spring Boot, in logback-spring.xml:

```xml
<appender name="ConsoleAppender" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
        <pattern>[it-test=%X{x-it-test}] %d{ISO8601} [thread=%thread] [%-5p] [%c] %X{user}- %m%n%ex</pattern>
    </encoder>
</appender>
```

## JAX WS RS API filters

The test must add the "x-it-test" header when calling the API service. The API service then needs a request filter to store this header in the MDC context:

```java
@Component // this annotation for Spring Boot only!
@Provider
public class MdcLoggingContainerRequestFilter implements ContainerRequestFilter {

  @Override
  public void filter(ContainerRequestContext requestContext) throws IOException {
    String iqeTest = requestContext.getHeaderString("x-it-test");
    MDC.put("x-it-test", iqeTest);
  }
}
```

More information about Quarkus [here](https://quarkus.io/guides/rest#the-jakarta-rest-way).

## Kafka Consumer/Producers interceptors

The implementation varies between frameworks:

### For Quarkus

Quarkus uses the Smallrye Messaging Reactive implementation for Kafka, so this is not really about Quarkus, but the Smallrye Messaging framework.

We need the outgoing interceptor to propagate the header from the MDC context to the Kafka message header:

```java
@Default
@ApplicationScoped
public class MdcOutgoingInterceptor implements OutgoingInterceptor {
  @Override
  public Message<?> beforeMessageSend(Message<?> message) {
    String test = MDC.get("x-it-test");
    if (!StringUtil.isNullOrEmpty(test) && message instanceof KafkaMessage kafkaMessage) {
      kafkaMessage.getHeaders().put("x-it-test", test);
    }

    return message;
  }

  @Override
  public void onMessageAck(Message<?> message) {}

  @Override
  public void onMessageNack(Message<?> message, Throwable failure) {}
}
```

And the incoming interceptor to propagate the header from the Kafka message header to the MDC context back:

```java
@Default
@ApplicationScoped
public class MdcIncomingInterceptor implements IncomingInterceptor {

  @Override
  public Message<?> afterMessageReceive(Message<?> message) {
    if (message instanceof KafkaMessage kafkaMessage) {
      MDC.put("x-it-test", kafkaMessage.getHeaders().get("x-it-test"));
    }

    return message;
  }

  @Override
  public void onMessageAck(Message<?> message) {}

  @Override
  public void onMessageNack(Message<?> message, Throwable failure) {}
}
```

Important considerations:
- Smallrye Messaging Reactive only supports one IncomingInterfactor and one OutgoingInterceptor. If you have multiple implementations, it will use only one, the one with more priority. I'm not sure if this was designed like this for some purpose, but checking the implementation, I don't see a reason for this strong limitation...

- The `@Default` annotation is to configure these interceptors for all the channels. An alternative is to use the `@Identifier("channel-a")` annotation and the interceptor would only work for the channel `channel-a`. 

More information [here](https://smallrye.io/smallrye-reactive-messaging/4.26.0-RC2/concepts/decorators/#intercepting-incoming-and-outgoing-messages).

- For Spring Boot

The idea is basically the same, but the interfaces change a bit.

We need the producer interceptor to propagate the header from the MDC context to the Kafka message header:

```java
public class MdcProducerInterceptor implements ProducerInterceptor<Object, Object> {
  @Override
  public ProducerRecord<Object, Object> onSend(ProducerRecord<Object, Object> record) {
    String test = MDC.get("x-it-test");
    if (!StringUtils.isNullOrEmpty(test)) {
      record.headers().add("x-it-test", test.getBytes(StandardCharsets.UTF_8));
    }

    return record;
  }

  @Override
  public void onAcknowledgement(RecordMetadata metadata, Exception exception) {}

  @Override
  public void close() {}

  @Override
  public void configure(Map<String, ?> configs) {}
}
```

And configure the producer to use this interceptor:

```java
@NotNull
  public static Map<String, Object> getProducerProperties(KafkaProperties kafkaProperties) {
    Map<String, Object> properties = kafkaProperties.buildProducerProperties(null);
    // ...
    properties.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG, MdcProducerInterceptor.class.getName());
    return properties;
  }
```

Also, we need to record interceptor to propagate the header from the Kafka message header to the MDC context back:

```java
public class MdcRecordInterceptor<K, V> implements RecordInterceptor<K, V> {
  @Override
  public ConsumerRecord<K, V> intercept(ConsumerRecord<K, V> record, Consumer<K, V> consumer) {
    var header = record.headers().lastHeader("x-it-test");
    if (header != null) {
      MDC.put("x-it-test", new String(header.value()));
    }

    return record;
  }
}
```

And configure the consumer factory to use this interceptor:

```java
@Bean
ConcurrentKafkaListenerContainerFactory<String, String>kafkaContainerFactory() {
    var factory = new ConcurrentKafkaListenerContainerFactory<String, String>();
    // ...
    factory.setRecordInterceptor(new IqeTestMdcRecordInterceptor<>());
    return factory;
}
```

# Conclusion

By propagating the MDC context across REST endpoints and Kafka consumers/producers, we enhance observability and troubleshooting. This approach enables us to track integration test executions and debug issues more efficiently. Whether using Quarkus or Spring Boot, implementing this mechanism ensures a consistent tracing context across distributed services.