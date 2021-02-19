---
layout: post
title: Spring Boot Admin 1.X Quick Guide
date: 2018-08-09
tags: [ Spring, Java Melody ]
---
This post introduces how to setup Spring Boot Admin 1.X according to my experience within a dockerized environment. 

# Context
We are using some Spring Boot containers for different purposes. Each Spring Boot application uses third party systems to connect with. We also use [JavaMelody](https://github.com/javamelody/javamelody/wiki) to expose metrics for each service. 

In testing and production environments, we need to manually check each of these Spring Boot applications and Support is also challenging when we need to troubleshot issues. 

# Spring Boot Admin

Why Spring Boot Admin? We can monitor all our Spring Boot applications in all our nodes in one go. 
What can we monitor? LOT of things like system and app metrics. In this article, we'll focus on *logs*, *health checks* and *java melody information*.

Let's see first how to setup it. At the moment, even when Spring Boot Admin 2.X is available, we're still using Spring Boot 1.5.7, so we need to use Spring Boot Admin 1.X artifacts. 

Moreover, we'll assume all the Spring Boot applications are configured with the actuator. More in [here](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-endpoints.html). 

## How to Setup Spring Boot Admin Server

Let's create a new Spring Boot application using [Spring Initializr](https://start.spring.io/). Select Spring Boot 1.5.15 and the dependency of Spring Boot Admin (Server). The *pom.xml* maven file would look like as:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.example</groupId>
	<artifactId>demo</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<description>Demo project for Spring Boot</description>

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.15.RELEASE</version>
	</parent>

	<dependencies>
		<dependency>
			<groupId>de.codecentric</groupId>
			<artifactId>spring-boot-admin-starter-server</artifactId>
		</dependency>
	</dependencies>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>de.codecentric</groupId>
				<artifactId>spring-boot-admin-dependencies</artifactId>
				<version>1.5.7</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
</project>
```

And enable the admin service adding the @EnableAdminServer annotation in your Main application java class:

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

import de.codecentric.boot.admin.config.EnableAdminServer;

@SpringBootApplication
@EnableAdminServer
public class DemoApplication {

	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}
}
```

We also specify a concrete port via application.properties:

```
server.port=8181
```

## How to Setup Spring Boot Admin Client

In your existing Spring Boot application, you only need to add the Spring Boot Admin client dependency:

```xml
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-client</artifactId>
    <version>1.5.7</version>
</dependency>
```

And also specify the Spring Boot Admin Server url in the application properties:

```
spring.application.name=hive-gw
spring.boot.admin.url=http://localhost:8181
```

Spring Boot Admin will group all the Spring Boot applications with the same *spring.application.name*.

## How to Setup Spring Boot Admin in a dockerized environment

If we are running your docker containers on the same network (docker --net=XXX), you don't need to do anything. Otherwise, as it happens to us, you need to configure your Spring Boot Admin clients to use the IP rather than the host name. This can be done via configuration:

```
spring.application.name=hive-gw
spring.boot.admin.url=http://localhost:8181

spring.boot.admin.client.prefer-ip=true
```

## How to Enable Security

There are many articles that already cover this. For example: https://codecentric.github.io/spring-boot-admin/current/#securing-spring-boot-admin

# Configuration

At this moment, we already have our running Spring Boot Admin Server up with the clients connecting to. The information we can see right now is kind of basic. Let's tweak it.

## Info Actuator

The */info* end provides some basic information about our application.

### How Extend the Info Actuator

We can extend the information the */info* provides by adding an *InfoContributor* configuration class as:

```java
@Component
public class MyCustomConfiguration implements InfoContributor {

    @Override
    public void contribute(Builder builder) {
        builder.withDetail("key", "value");
    }

}
```

Now, the response from the */info* endpoint of our application will be:

```json
{
	key: "value"
}
```

### Build Info

All the Spring Boot applications should have the Maven plugin *spring-boot-maven-plugin*. Thanks to this plugin, we can expose the build info:

```xml
<plugin>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-maven-plugin</artifactId>
	<executions>
		<execution>
            <id>build-info</id>
            <goals>
                <goal>build-info</goal>
            </goals>
        </execution>
	</executions>
</plugin>
```
And finally, the */info* endpoint will be as:

```json
{
	build: {
		version: "0.0.1-SNAPSHOT",
		artifact: "demo",
		name: "demo",
		group: "com.example",
		time: 1533796864000
	},
	key: "value"
}
```

## Log and LogFile Actuator

These actuators heavily helped us when troubleshoting issues in our environments because we can enable/disable loggers at runtime in order to see *debug* information. Nevertheless, we did need still to search for the logger name and send a post using curl to the log actuator manually. So, this is still kind of manual stuff. Thanks to Spring Boot Admin, you can search for the logger names and change the log levels via the UI. And more, you even can see the logs via the UI.

In order to enable the log actuator, you don't need to do anything. For the LogFile actuator, you need to configure your Spring Boot client with the location of the traces of your app via the configuration file:

```
spring.application.name=hive-gw
spring.boot.admin.url=http://localhost:8181

spring.boot.admin.client.prefer-ip=true

logging.file=/opt/spring-boot/logs/hiveGateway.log
```

## Health Actuator

In real environments, when clients start complaining because something is not working, most of the times is caused by a third party system stopped working. 

As a resolution, we started adding health checks. As many as we needed. And exposing them via the */health* actuator using the *AbstractHealthIndicator*:

```java
@Configuration
public class MyHealthCheck extends AbstractHealthIndicator {

    private final String address = "my third party address"

    @Override
    protected void doHealthCheck(Builder builder) throws Exception {
        builder.withDetail("address", address);

        URL url = new URL(address);
        HttpURLConnection connection = (HttpURLConnection) url.openConnection();
        connection.setRequestMethod("GET");
        connection.connect();

        int httpCode = connection.getResponseCode();
        builder.withDetail("response", httpCode);

        if (HttpResponseCodes.SC_OK == httpCode) {
            builder.up();
        } else {
            builder.down();
        }

    }

}
```

Spring Boot Admin will manage when a client is down or up depending to the result of the health actuator. And it will allow us to implement circuit breakers to ensure the whole system will work in all scenarios. 

## JavaMelody

JavaMelody is great. It provides lot of metrics about your services, how long they took and the queries your app run. We can see even the sql explain plan that every query made. It also gives an UI where we can view all this. 

JavaMelody teams has developed a plugin to Spring Boot Admin 1.X to add a custom tab for each client. In order to enable it, you need to add these dependencies to the Spring Boot Admin server app:

```xml
<!-- Enable Java Melody -->
<dependency>
    <groupId>net.bull.javamelody</groupId>
    <artifactId>spring-boot-admin-server-ui-javamelody</artifactId>
    <version>1.5.7.1</version>
</dependency>
<dependency>
    <groupId>net.bull.javamelody</groupId>
    <artifactId>javamelody-core</artifactId>
    <version>1.73.1</version>
</dependency>
```

And enable the collect server in the properties too:

```
server.port=8181

javamelody.collectserver.enabled=true
javamelody.init-parameters.storage-directory=/tmp/javamelody
```

# Conclusion

We have configured a Spring Boot Admin application with multiple clients and demostrated how to make use of actuators to extend the information we see in the UI. We also configured JavaMelody.

There are also other configuration that is out of scope for now like notifications or deploying a cluster of Spring Boot Admin applications. 