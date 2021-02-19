---
layout: post
title: Update Custom Properties in Quarkus at Runtime
date: 2021-02-18
tags: [ Quarkus ]
---

Recently, I wanted to update some custom properties in order to dynamically change some inner behaviours in my application. As an example, let's see a simple example:

```java
@Path("/request-api")
public class GreetingResource {

    @ConfigProperty(name = "my.property")
    String property;

    @GET
    @Produces(MediaType.TEXT_PLAIN)
    public String hello() {
        return "Hello " + property;
    }
}
```

At service start-up time, the Quarkus application reads the `my.property` value from the `application.properties`:

```yaml
my.property=Default
```

And then the CDI framework in Quarkus (`Arc`) injects the value into my `GreetingResource` endpoint. So, by default, my service will return `Hello default` the first time we call the endpoint.

What about if I want to update this property `my.property` at runtime? I mean without needing to restart my application, I want to change the default value from `default` to `Jose`. 

Let's see first what we need to do to watch config changes and then how to change configuration dynamically.

# Watch Configuration Changes

We can do this either using the `@RequestScope` annotation or watching config change events. 

## Using `@RequestScope` annotation

By default, all the beans have application scope. This means that the CDI framework will instantiate our beans only once per application life. So, we need to mark our beans to request scope, so the CDI updates it accordingly everytime the endpoint is invoked:

```java
@RequestScoped // important!
@Path("/request-api")
public class GreetingResource {

    @ConfigProperty(name = "my.property")
    String property;

    @GET
    @Produces(MediaType.TEXT_PLAIN)
    public String hello() {
        return "Hello " + property;
    }
}
```

With our resource marked with the `@RequestScoped` annotation, the CDI framework will evaluate the property `my.property` everytime our `GreetingResource` bean is called and inject its most recent value.

## Watching config change events

If we can't mark our beans as request scoped. Another option is to watch for config change events:

```java
@Path("/request-api")
public class GreetingResource {

    private static final String PROPERTY = "my.property";

    @ConfigProperty(name = PROPERTY)
    String property;

    @GET
    @Produces(MediaType.TEXT_PLAIN)
    public String hello() {
        return "Hello " + property;
    }

    public void all(@Observes ChangeEvent changeEvent) {
        if (PROPERTY.equals(changeEvent.getKey()) && Type.UPDATE == changeEvent.getType()) {
            property = changeEvent.getNewValue();
        }
    }
}
```

# Update Configuration at Runtime

Quarkus uses the `microprofile-config` module that allows to use different config sources by providing a custom [`ConfigSource`](https://github.com/eclipse/microprofile-config/blob/master/api/src/main/java/org/eclipse/microprofile/config/spi/ConfigSource.java) implementation. The purpose of this interface is to cope with use cases where users work with different sources like injecting properties from GitHub, or NFS or any other source. 

For this post, what we would need is to have an in-memory source and have some exposed REST endpoints to read/update properties. Fortunately, this is exactly what It's implemented in the [microprofile-extensions/config-ext/configsource-memory](https://github.com/microprofile-extensions/config-ext/blob/master/configsource-memory) microprofile extension. So, this would just be a matter of configuring this extension in our Quarkus application.

## Use "configsource-memory" microprofile extension

First, let's add this extension into the `pom.xml` file:

```xml
<dependency>
    <groupId>org.microprofile-ext.config-ext</groupId>
    <artifactId>configsource-memory</artifactId>
    <version>1.0.11</version>
</dependency>
```

Also, we need to configure the CDI to use this `ConfigSource` implementation. For doing so, we need to add the file `org.eclipse.microprofile.config.spi.ConfigSource` into `src/main/resources/META-INF/services` with the content:

```
org.microprofileext.config.source.memory.MemoryConfigSource
```

Secondly, if we are not using it yet, we need to use a REST quarkus extension. In this post, we're going to use `resteasy`. Let's update the `pom.xml` file again by adding it:

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-resteasy-jackson</artifactId>
</dependency>
```

| Note that we could use simply `quarkus-resteasy` (without JSON support) or use `quarkus-resteasy-jsonb`. 

To simplify things, we can also add `swagger` in order to update the configuration via UI:

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-smallrye-openapi</artifactId>
</dependency>
```

And finally, we can start playing around the new endpoints!

```
mvn compile quarkus:dev
```

The first time we call our endpoint `http://localhost:8081/request-api` and we'll get `Hello Default`.
Let's update the property via Swagger. We need to browse to `http://localhost:8081/q/swagger-ui` and use the right endpoint as shown in:

![Swagger]({{ site.url }}{{ site.baseurl }}/images/quarkus_custom_config_source.png)

And finally, if we hit again into our endpoint, we'll get `Hello Jose`!

# Conclusions

We've seen how to dynamically update custom properties at runtime using either `@RequestScope` or obversing events. In order to update the configuration at runtime, we have made use of an existing Microprofile extension.

I also tried to use [Consul](https://quarkus.io/guides/consul-config) Quarkus extension to dynamically update configuration at runtime. Unfortunately, Quarkus is not watching for changes in Consul, so if we change the property in Consul, Quarkus will never get notified. Also, it seems this won't be supported: [issues/10287](https://github.com/quarkusio/quarkus/issues/10287).

What about Quarkus properties? To sum up, we can't dynamically update Quarkus properties at runtime. This is because Quarkus is fully focused on cloud-native application and does a lot of stuff to boost the application performance. Therefore, this feature is not supported.

Source Code: [Sgitario/quarkus-examples/quarkus-custom-config-source](https://github.com/Sgitario/quarkus-examples/tree/main/quarkus-custom-config-source)