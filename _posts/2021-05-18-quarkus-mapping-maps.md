---
layout: post
title: How to bind properties into a Map in Quarkus
date: 2021-05-18
tags: [ Quarkus ]
---

As Spring developer, binding some properties into a Map in Java is something that I have eventually needed when coding. Basically, having these properties:

```
custom.map.A=Value 1
custom.map.B=Value 2
```

I would like to map the values with prefix `custom.map` into a Map with values `A=Value 1` and `B=Value 2`. 

In Spring, binding the above properties is fairly simple:

```java
@Configuration
public class Config {

    @Bean
    @ConfigurationProperties(prefix = "custom.map")
    public Map<String, String> customMap() {
        return new HashMap<>();
    }
}
```

But what about Quarkus? 

First of all, [Quarkus Config](https://quarkus.io/guides/config) is based on [Smallrye Config](https://smallrye.io/docs/smallrye-config/index.html) which is a framework also based on [Microprofile Config](https://github.com/eclipse/microprofile-config). Surprisingly, I could not find any information or example about how to achieve this use case neither in Quarkus site nor Smallrye nor Microprofile. But as you can imagine, there is a way! And due to the lack of examples, I wanted to write this post.

## The `@ConfigMapping` annotation

In Quarkus, the config values are injected via the `@ConfigProperty` annotation. But there is another annotation called `@ConfigMapping` to map more complex relations like binding maps.

Therefore, for binding Maps, we need to create an interface:

```java
import java.util.Map;

import io.smallrye.config.ConfigMapping;

@ConfigMapping(prefix = "custom")
public interface ConfigMapInterface {
    Map<String, String> map();
}
```

And then we can inject it in our bean:

```java
@Path("/config-mapping")
public class ConfigMappingResource {

    @Inject
    ConfigMapInterface configMapInterface;

    @GET
    public String getKeyA() {
        return configMapInterface.get("A");
    }
}
```

Moreover, we can override the prefix by annotating the field using the `@ConfigMapping` annotation again:

```java
@Path("/config-mapping")
public class ConfigMappingResource {

    @Inject
    @ConfigMapping(prefix = "another.custom")
    ConfigMapInterface configMapInterface;

    @GET
    public String getKeyA() {
        return configMapInterface.get("A");
    }
}
```

## Conclusion

Quarkus is going on the right path and trying to cope with a lot of use cases and, at the same time, giving a really good developer experience. Being said this, Quarkus has a few of things to improve and I think this is an example of one. The good side is that Quarkus is a open-source framework and in some sense, we can owe it and help to became a better framework. This's why we need to raise issues everytime we see something that needs improvising. About binding maps in Quarkus, I raised this feature request: [https://github.com/quarkusio/quarkus/issues/17269](https://github.com/quarkusio/quarkus/issues/17269).

My suggestion is to ease the binding by simply doing:

```java
@ConfigProperty("custom.map")
Map<String, String> myMap;
```

I see this is more than doable in Quarkus and it will provide a much better experience for users.