---
layout: post
title: Migration Guide from Guice to Spring
date: 2018-08-16
tags: [ Spring, Guice ]
---

[Guice](https://github.com/google/guice) and [Spring DI](https://docs.spring.io/spring-boot/docs/current/reference/html/using-boot-spring-beans-and-dependency-injection.html) are by large the most used DI frameworks in Java world. I don't pretend saying this is better than the other but, to be honest, I really like Spring frameworks ecosystem and all the benefits around. 

This post is purely a guide about how to migrate from Guice to Spring DI.

# How to Bind Components

Both Guice and Spring DI works with @Inject annotations from Java. Nevertheless, the way to register your components in each framework is different. By default, Guice will scan all the classpath and inject as much as possible. In Spring, we can configure our application to scan your components driven by annotations (@Component, @Controller, @Repository, @Service, ...), but we can achieve the same as Guice in Spring just doing:

- Via configuration:

```xml
<context:component-scan base-package="my.package" use-default-filters="true">
	<context:include-filter type="regex" expression="my\.package\..*"/>
</context:component-scan> 
```

- Via Spring Boot:

```java
@SpringBootApplication
@ComponentScan(basePackages = "my.package", includeFilters = @Filter(type = FilterType.REGEX, pattern="my\\.package\\..*"))
public class Application {
}
```

Moreover, we can configure Spring to scan other types of annotations apart from the default ones by:

```java
@SpringBootApplication
@ComponentScan(basePackages = "my.package", includeFilters = @Filter(type = FilterType.ANNOTATION, classes=MyCustomAnnotation.class))
public class Application {
}
```

We also can exclude components using the similar "excludeFilters" in the component scan element.

# Pluggable Configuration

In guice, we can split our application in modules. This concept is like pluggable configuration. In Spring, we use the @Configuration annotation to declare modules:

- Guice:

```java
public class MyModule extends AbstractModule {
    @Override
    protected void configure() {
		bind(MyInterface.class).to(MyImplementation.class);
    }

	@Provides
	@Inject
	public MyAnotherInterface providesAnotherInterface(MyInterface dependency) {
		return new MyAnotherImplementation();
	}
}
```

In Guice, we can use the bind() method to register our components and the @Provides annotation when there are dependencies. In Spring, we always use the @Bean annotation. 

- Spring:

```java
@Configuration
public class MyModule {
	@Bean
	public MyInterface myInterface() {
		return new MyImplementation();
	}

	@Bean
	@Inject
	public MyAnotherInterface providesAnotherInterface(MyInterface dependency) {
		return new MyAnotherImplementation();
	}
}
```

Moreover, Spring provides lot of possibilities to configure your module to behave purely as auto pluggable component:

```java
@Configuration
@ConditionalOnProperty(prefix = "my.config", name = "enabled", matchIfMissing = false)
public class MyModule {
    @Bean
    public MyInterface myInterface() {
		return new MyImplementation();
    }
}
```

So, we configured our module to be registered if and only if there is a property "my.config.enaled" set to true. 

# Bind Properties

As far I know, Spring does not support the @Named annotation from Java as Guice does. In order to inject properties, we need to use the @Value annotation with SPel.

- Guice:

```java
public class MyComponent {
    @Inject
    public MyComponent(@Named("my.property") String value) {
		// ...
    }
}
```

- Spring:

```java
@Component
public class MyComponent {
    @Inject
    public MyComponent(@Value("${my.property}") String value) {
		// ...
    }
}
```

# Interceptors

The interceptors is a really nice feature in Guice where you can bind aspect interceptors to methods very easily using AspectJ:

```java
public class MyModule extends AbstractModule {
    @Override
    protected void configure() {
		binder().bindInterceptor(Matchers.any(), Matchers.annotatedWith(Transactional.class), new MethodInterceptor() {

            @Override
            public Object invoke(MethodInvocation invocation) throws Throwable {
                // ...
            }
            
        });
    }
}
```

The alternative in Spring is to use the @Aspect annotation as:

```java
@Component
@Aspect
public class MyMethodInterceptor {

    @Pointcut("@annotation(my.package.Transactional)")
    public void methods() {
    };

    @Around("readOnlyTxMethods()")
    public void methods(ProceedingJoinPoint pjp) throws Throwable {
	    // ...
	}
}
```

# Working with Proxy classes
Another important note for the migration is that Guice works quite well when working with proxy classes. A proxy class is a runtime class made by AspectJ / CGlib or by methods annotated with @Transactional. Also, it affects to methods annotated with @Cache and @Async. 

This happens with we inject implementation classes rather than the interfaces. Spring does not allow this since we should work with interfaces instead (which is correct). However, we can make Spring to work with proxy classes as Guice does by:

```java
@SpringBootApplication
@ComponentScan(basePackages = "my.package", /* ... */)
@EnableAspectJAutoProxy(proxyTargetClass = true) // If we use @Aspect
@EnableTransactionManagement(mode = AdviceMode.ASPECTJ, proxyTargetClass = true) // If we use @Transactional
@EnableAsync(proxyTargetClass = true) // If we use @Async
@EnableCaching(proxyTargetClass = true) // If we use @Cache
public class Application {
}
```

# @ImplementedBy to @Primary
Guice also provides an annotation to specify a default implementation for an interface:

```java
@ImplmementedBy(MyImplementation.class)
public interface MyInterface {
    // ...
}
```

Where in Spring this is done via the @Primary annotation but in the implementation. Therefore, when two or more implementations are found for the same interface, it will use the one marked with the @Primary annotation:

```java
@Component
@Primary
public MyImplementation implements MyInterface {
    // ...
}
```

I personally do not recommend using either the @ImplementedBy or the @Primary annotation. When we need it, it sounds like a code smell in the configuration that needs to be refactored.