---
layout: post
title: Adding Cache Support to Spring Boot Application
date: 2018-11-06
tags: [ Spring Boot, Cache ]
---

There are lot of tutorials about how to enable Cache in your Spring Boot application. These a couple of examples:
- [Spring Docs](https://docs.spring.io/spring/docs/current/spring-framework-reference/integration.html#cache)
- [Baeldung](https://www.baeldung.com/spring-cache-tutorial)

However, I wanted to bring here all the information that was useful to me and also adding an approach to enable/disable the cache by configuration. 

# Dependencies

Add [the Spring Boot Cache dependency](https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-cache) in the pom.xml:
```xml
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-cache</artifactId>
   <version>2.1.0.RELEASE</version>
</dependency>
```

# Configuration

Let's configure now the cache configuration as:

```java
@Configuration
@ConditionalOnProperty(name = "spring.cache.names")
@EnableCaching
public class CacheConfiguration {

    @Value("${spring.cache.names}")
    public String[] cacheNames;

    @Inject
    public CacheManager cacheManager;

    @Bean
    public CacheManager cacheManager() {
        return new ConcurrentMapCacheManager(cacheNames); // Cache Vendor
    }

    @ConditionalOnProperty(name = "spring.cache.autoexpiry", value = "true")
    @Scheduled(fixedDelayString = "${spring.cache.expire.delay:500000}")
    public void cacheEvict() {
        cacheManager.getCacheNames().stream()
                .map(cacheManager::getCache)
                .forEach(Cache::clear);
    }
}
```
Note the *@EnableCaching* annotation will make our Spring application listen the *@Cacheable* annotations. 
Also, see we're using the very simple cache ConcurrentMapCacheManager as Cache Manager. In case we want to use EhCache or Hazelcast, we would only need to change the CacheManager implementation bean here.

This configuration provides the next benefits:
- Enable/Disable Cache by Configuration

This configuration will make the cache be enabled if and only if a property "spring.cache.names" exists in the app configuration. Something like:

```
spring.cache.names=cache1,cache2,cache3
```

This is equivalent to use the Spring native as:
```
spring.cache.type=none
```

- Evict All Cache every X seconds

Some cache vendors bring this feature by nature. However, this is a general approach for any cache vendor.

We'll register a scheduled action to evict all the cache entries by setting the property "spring.cache.autoexpiry" to true. By default, the evict of all the cache is run every 500 seconds. We can customize another value using the field "spring.cache.expire.delay":

```
spring.cache.expire.delay=750000
spring.cache.autoexpiry=true
```

# Usage

Annotate the methods you want to cache using the *@Cacheable* annotation:
```java
@Service
public class MyServiceImpl implements MyService {

    @Override
    @Cacheable(cacheNames = "languages", key = "#request.country")
    public Languages getLanguages(Request request) {
        // ...
    }
}
```

Note the key cache is how the cache needs to be structured in a better efficient way.

**If we're using Jersey, we cannot use the @Cacheable annotation in the Controllers since these are not managed within the bean context.**

So this annotation can be only used in beans annotated with *@Service*, *@Component*, etc.

Configuration the application as:
```
spring.cache.expire.delay=500000
spring.cache.autoexpiry=true
spring.cache.names=languages
```
**And that's it!**

# Metrics
We can see the cache sizes via the metrics actuator. For example: "http://myapp/metrics" returns:

```json
{
mem: 527435,
mem.free: 220078,
// ...
cache.languages.size: 1,
// ...
}
```
