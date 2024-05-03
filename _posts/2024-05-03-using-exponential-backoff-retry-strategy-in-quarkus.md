---
layout: post
title: Using Exponential Backoff Retry strategy in Quarkus
date: 2024-05-03
tags: [ Java, Quarkus ]
---

One of the most important parts when implementing a microservice architecture is to think about resilience strategies in your applications. How will your service behave after failures? There are a few very well known patterns to ensure the resillience of your services like Circuit Breakers, Fallbacks, Timeouts and Retries. In this post, we'll focus on the Exponential Backoff Retry pattern in Quarkus services.

# Introduction

What is the Exponential Backoff Retry pattern about? The simple Retry strategy will attempt to perform one logic every X seconds when an exception occurs. For example, we want to call one service and if an exception happens, I want to retry up to 3 times every 1 second. But what if the service we're trying to reach out, it's very unstable/can only handle a few requests/needs long startup times, in this scenario is where the Exponential Backoff Retry strategy comes to play. The strategy will retry with a delay among retries that will exponentially increase up to a max duration. Let's say that I want to retry up to 7 times with a backoff factor of 2, with a delay of 3 seconds and a max duration of 30 seconds, then:

1. attempt at 3 seconds (because the delay was set to 3 seconds)
2. attempt at 6 seconds (doing factor x previous delay = 2 x 3 = 6 seconds)
3. attempt at 12 seconds
4. attempt at 24 seconds
5. attempt at 30 seconds (because the max duration limit is 30 seconds)
6. attempt at 30 seconds
7. attempt at 30 seconds

And no more retries will be made since the max attempts was set to 7. 

I hope the concept is clear enough. Otherwise, there is much more information about it online with many more examples. 

# Quarkus service

Let's now see how to use the Exponential Backoff Retry strategy in Quarkus. Our use case is really simple and easy to understand: we're going to implement a couple of REST services: one GET resource that will simply return "Hello, <name>" and another GET resource that will use a REST Client to invoke the first GET resource. For doing this, we're going to use the following Quarkus extensions:

- [Resteasy Reactive](https://quarkus.io/guides/resteasy-reactive): REST support
- [REST Client Reactive](https://quarkus.io/guides/rest-client): REST Client support

First of all, we need to create the Quarkus application from zero using the Quarkus Maven plugin:

```s
mvn io.quarkus.platform:quarkus-maven-plugin:3.10.0:create -Dextensions=resteasy-reactive,rest-client-reactive
```

After providing your group ID and the artefact name, go to the generated folder.

If everything is correct, you can run the application in DEV mode by doing:

```s
mvn quarkus:dev
```

## Add the Hello resource:

Let's add the hello resource:

```java
@Path("/hello")
public class GreetingResource {

    @GET
    @Path("/{name}")
    @Produces(MediaType.TEXT_PLAIN)
    public String hello(@RestPath String name) {
        return "Hello, " + name;
    }
}
```

And when calling to this resource, you should see:

```s
curl localhost:8080/hello/Jose
> Hello, Jose
```

## Add the REST Client service:

Next, we're going to create our client to call our Hello resource:

```java
@RegisterRestClient(configKey = "greeting-client")
public interface GreetingClient {
    @Path("/hello/{name}")
    @GET
    @Produces(MediaType.TEXT_PLAIN)
    String hello(String name);
}
```

Then, configure the client to use the same service:

```
quarkus.rest-client."greeting-client".url=http://localhost:8080
```

And expose the client via another resource:

```java
@Path("/client")
public class RestClientResource {

    @RestClient
    GreetingClient client;

    @GET
    @Path("/{name}")
    @Produces(MediaType.TEXT_PLAIN)
    public String helloViaClient(@RestPath String name) {
        return "From client: " + client.hello(name);
    }
}
```

Now, if we call to the new resource "/client", we should see: 

```s
curl localhost:8080/client/Jose
> From client: Hello, Jose
```

## Make our Hello service faulty

Let's introduce a bug in our hello service by simply throwing an exception in the Hello resource:

```java
@Path("/hello")
public class GreetingResource {

    @GET
    @Path("/{name}")
    @Produces(MediaType.TEXT_PLAIN)
    public String hello(@RestPath String name) {
        Log.info("Handling hello request: " + name);
        if ("jose".equalsIgnoreCase(name)) {
            throw new InternalServerErrorException("Wrong!");
        }

        return "Hello, " + name;
    }
}
```

Now, if we tick the resources again, we'll see that the call failed with "HTTP/1.1 500 Internal Server Error" and in the service logs, you should see:

```
2024-05-03 10:13:03,191 INFO  [io.sgi.GreetingResource] (executor-thread-1) Handling hello request: Jose
```

## Let's retry!

Let's configure our application to retry up to 5 times. For doing this, we are going to add [the Smallrye Fault Tolerance](https://quarkus.io/guides/smallrye-fault-tolerance) quarkus extension:

```s
mvn quarkus:add-extension -Dextensions='smallrye-fault-tolerance'
```

And add the `@Retry` annotation in our REST Client interface:

```java
@RegisterRestClient(configKey = "greeting-client")
public interface GreetingClient {
    @Path("/hello/{name}")
    @GET
    @Produces(MediaType.TEXT_PLAIN)
    @Retry(maxRetries = 5) // from the smallrye fault tolerance quarkus extension
    String hello(String name);
}
```

When calling the client resource again `curl -v localhost:8080/client/Jose`, we should now see several attempts in the service logs:

```
2024-05-03 10:17:02,367 INFO  [io.sgi.GreetingResource] (executor-thread-2) Handling hello request: Jose
2024-05-03 10:17:02,381 INFO  [io.sgi.GreetingResource] (executor-thread-2) Handling hello request: Jose
2024-05-03 10:17:02,442 INFO  [io.sgi.GreetingResource] (executor-thread-2) Handling hello request: Jose
2024-05-03 10:17:02,449 INFO  [io.sgi.GreetingResource] (executor-thread-2) Handling hello request: Jose
2024-05-03 10:17:02,455 INFO  [io.sgi.GreetingResource] (executor-thread-2) Handling hello request: Jose
2024-05-03 10:17:02,463 INFO  [io.sgi.GreetingResource] (executor-thread-2) Handling hello request: Jose
```

But this is the simple retry strategy, let's now configure the exponential backoff retry strategy. For doing this, we need to simply add the `@ExponentialBackoff` annotation along with the `@Retry` annotation:

```java
@RegisterRestClient(configKey = "greeting-client")
public interface GreetingClient {
    @Path("/hello/{name}")
    @GET
    @Produces(MediaType.TEXT_PLAIN)
    @Retry(maxRetries = 7, delay = 1000) // this is 1 second since the default unit is milliseconds
    @ExponentialBackoff(factor = 2, maxDelay = 60000) // use the exponential backoff retry strategy
    String hello(String name);
}
```

And finally, when calling again to our client resource, we should see the attempts in the service logs at the expected time:

```
2024-05-03 10:37:09,335 INFO  [io.sgi.GreetingResource] (executor-thread-1) Handling hello request: Jose // first call
2024-05-03 10:37:18,643 INFO  [io.sgi.GreetingResource] (executor-thread-2) Handling hello request: Jose // 1 second of delay
2024-05-03 10:37:21,834 INFO  [io.sgi.GreetingResource] (executor-thread-2) Handling hello request: Jose // factor x previous delay = 3 seconds
2024-05-03 10:37:27,828 INFO  [io.sgi.GreetingResource] (executor-thread-2) Handling hello request: Jose // factor x previous delay = 6 seconds
2024-05-03 10:37:39,971 INFO  [io.sgi.GreetingResource] (executor-thread-2) Handling hello request: Jose // factor x previous delay = 12 seconds
2024-05-03 10:38:04,154 INFO  [io.sgi.GreetingResource] (executor-thread-2) Handling hello request: Jose // factor x previous delay = 24 seconds
2024-05-03 10:38:52,047 INFO  [io.sgi.GreetingResource] (executor-thread-3) Handling hello request: Jose // max delay is 60 seconds
2024-05-03 10:39:52,062 INFO  [io.sgi.GreetingResource] (executor-thread-4) Handling hello request: Jose // max delay is 60 seconds
2024-05-03 10:40:52,067 ERROR [io.qua.ver.htt.run.QuarkusErrorHandler] (executor-thread-1) HTTP Request to /client/Jose failed, error id: 1061e587-8ca9-4cbe-bf94-bf36d3e6bb07-2: org.jboss.resteasy.reactive.ClientWebApplicationException: Received: 'Internal Server Error, status code 500' when invoking: Rest Client method: 'io.sgitario.GreetingClient#hello'
```

**Note** that the times are not fully exact since we're seeing the log times, not the CPU timer. 

## Limitations of the Smallrye Fault Tolerance API to configure the retry settings at runtime

The previous example configured the retry using fixed settings which means that for example we can't increase or decrease the max retries at runtime using a property. 

Unfortunately, the API provided by the Smallrye Fault Tolerance quarkus extension lacks of the property binding to be used directly in the `@Retry` annotations like we can use in other frameworks. So, we can't do the following:

```java
@RegisterRestClient(configKey = "greeting-client")
public interface GreetingClient {
    @Path("/hello/{name}")
    @GET
    @Produces(MediaType.TEXT_PLAIN)
    @Retry(maxRetries = "${my.custom.property.max-retries:10}") // compilation failure since maxRetries must be an integer
    String hello(String name);
}
```

There is a dedicated section in the [the Smallrye Fault Tolerance extension guide](https://quarkus.io/guides/smallrye-fault-tolerance#runtime-configuration), where it explains how to do it using the fully-qualified-class-name/method-name as:

```
io.sgitario.GreetingClient/hello/Retry/maxRetries=3
```

The problem is that the REST Client internally uses an auto-generated resource and hence this is not working for REST Clients (and also other frameworks that auto-generate code which is quite common in Quarkus).

To make it to work, we need to move the `@Retry` and `@ExponentialBackoff` annotations from the GreetingClient interface to the RestClientResource resource:

```java
@Path("/client")
public class RestClientResource {

    @RestClient
    GreetingClient client;

    @Retry
    @ExponentialBackoff
    @GET
    @Path("/{name}")
    @Produces(MediaType.TEXT_PLAIN)
    public String helloViaClient(@RestPath String name) {
        return "From client: " + client.hello(name);
    }
}
```

And add the following properties:

```
io.sgitario.RestClientResource/helloViaClient/Retry/maxRetries=8
io.sgitario.RestClientResource/helloViaClient/ExponentialBackoff/maxDelay=10000
```

Then, after calling the client resource again, we should see the following service logs:

```
2024-05-03 10:59:22,536 INFO  [io.sgi.GreetingResource] (executor-thread-2) Handling hello request: Jose
2024-05-03 10:59:28,688 INFO  [io.sgi.GreetingResource] (executor-thread-2) Handling hello request: Jose
2024-05-03 10:59:40,577 INFO  [io.sgi.GreetingResource] (executor-thread-2) Handling hello request: Jose
2024-05-03 11:00:04,586 INFO  [io.sgi.GreetingResource] (executor-thread-2) Handling hello request: Jose
2024-05-03 11:00:52,406 INFO  [io.sgi.GreetingResource] (executor-thread-3) Handling hello request: Jose
...
```

But wait... the `Retry/maxRetries=8` works, but not the `ExponentialBackoff/maxDelay=10000` (it's using the default of 60 seconds). 

This is indeed another limitation when configuring the retries at runtime.

## Solution to configure the retry settings at runtime

As we saw in the previous section, at the moment, the API has a few limitations to configure the retry settings at runtime:
- it does not work for REST Clients
- doesn't support property binding (the alternative of using fully qualified class names is trick and prone to errors)
- can't configure the ExponentialBackoff properties

Let's address all the above limitations by creating our custom annotation and using the programmatically API from the Smallrye Fault Tolerance extension. 

First, let's define the annotation we would like to use:

```java
@InterceptorBinding
@Retention(RUNTIME)
@Target({TYPE, METHOD})
public @interface ReplyWith {
    @Nonbinding String maxRetries() default "";
    @Nonbinding String delay() default "";
    @Nonbinding ExponentialBackoff exponentialBackoff() default @ExponentialBackoff;
}

@Retention(RUNTIME)
@Target({TYPE, METHOD})
public @interface ExponentialBackoff {
    @Nonbinding String enabled() default "";
    @Nonbinding String factor() default "";
    @Nonbinding String maxDelay() default "";
}
```

**Note** that the `@Nonbinding` annotation is needed so the interceptor gets triggered and that all the properties are string based because we want to use a property binding here.

Next, the interceptor binding implementation for the `@ReplyWith` annotation:

```java
@Interceptor
@ReplyWith
@Priority(jakarta.interceptor.Interceptor.Priority.PLATFORM_BEFORE + 1)
public class ReplyWithInterceptor {

    // ...

    @AroundInvoke
    public Object intercept(InvocationContext context) throws Exception {
        Optional<ReplyWith> interceptionContext = getArcCacheInterceptionContext(context);
        if (interceptionContext.isEmpty()) {
            return context.proceed();
        }

        var binding = interceptionContext.get();
        var retry = FaultTolerance.create().withRetry();
        getOptionalValue(binding.maxRetries(), Integer.class)
                .ifPresent(retry::maxRetries);
        var duration = getOptionalValue(binding.delay(), Duration.class);
        if (duration.isPresent()) {
            retry.delay(duration.get().toMillis(), ChronoUnit.MILLIS);
        }

        if (isExponentialBackoffStrategySet(binding)) {
            var exponentialBackoff = retry.withExponentialBackoff();
            getOptionalValue(binding.exponentialBackoff().factor(), Integer.class)
                    .ifPresent(exponentialBackoff::factor);
            getOptionalValue(binding.exponentialBackoff().maxDelay(), Duration.class)
                    .ifPresent(maxDelay -> exponentialBackoff.maxDelay(maxDelay.toMillis(), ChronoUnit.MILLIS));
            retry = exponentialBackoff.done();
        }

        return retry.done().build().call(context::proceed);
    }

    // ...
}
```

You can find the full implementation of the interceptor [here](https://github.com/Sgitario/quarkus-exponential-backoff-retry-example/blob/main/src/main/java/io/sgitario/custom/ReplyWithInterceptor.java).

Next, we need to use the new `@ReplyWith` annotation in the REST Client interface:

```java
@RegisterRestClient(configKey = "greeting-client")
public interface GreetingClient {
    @Path("/hello/{name}")
    @GET
    @Produces(MediaType.TEXT_PLAIN)
    @ReplyWith(maxRetries = "my.custom.max-retries",
            delay = "my.custom.delay",
            exponentialBackoff = @ExponentialBackoff(
                    factor = "my.custom.factor",
                    maxDelay = "my.custom.max-delay"
            ))
    String hello(String name);
}
```

And provide the properties (if not set, it will use the defaults):

```
my.custom.max-retries=7
my.custom.delay=1s
my.custom.factor=2
my.custom.max-delay=60s
```

Finally, the retry settings will use this configuration and we will be able to customise it at runtime by providing different property values.

# Concusion

We have seen how ease is to implement the Exponential Backoff Retry resillience pattern in Quarkus thanks to the Smallrye Fault Tolerance extension and also its limitations when using the annotations API. I've proposed a solution to address all the limitations by using one annotation interceptor and the Smallrye Config to bind the properties. 