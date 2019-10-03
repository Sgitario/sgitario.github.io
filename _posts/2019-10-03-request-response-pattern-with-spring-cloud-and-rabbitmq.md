---
layout: post
title: Request-Response pattern with Spring Cloud
date: 2019-10-03
tags: [ Spring Boot, Spring Cloud, RabbitMQ ]
---

Let's write a hands on tutorial about how to implement the request-response message pattern with Spring Cloud. In this tutorial, I've used RabbitMQ but Kafka is also supported.

## RabbitMQ installation with Docker

Write the *docker-compose.yml* file with:

```yml
version: '3.5'
services:
  rabbitq:
    image: "rabbitmq"
    container_name: rabbit
    environment:
      RABBITMQ_ERLANG_COOKIE: "SWQOKODSQALRPCLNMEQG"
      RABBITMQ_DEFAULT_USER: "guest"
      RABBITMQ_DEFAULT_PASS: "guest"
      RABBITMQ_DEFAULT_VHOST: "/"
    ports:
      - "15672:15672"
      - "5672:5672"
```

And run rabbitmq:

```bash
> docker-compose up
```

Now, we have our RabbitMQ instance up and running on port 5672.

## Spring Cloud Application

Let's write a very basic spring cloud application using the next *pom.xml*:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.crud.springboot</groupId>
    <artifactId>architecture-test-performance</artifactId>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.5.RELEASE</version>
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <java.version>1.8</java.version>
        <spring.cloud.version>2.1.2.RELEASE</spring.cloud.version>
    </properties>

    <dependencies>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-stream</artifactId>
			<version>${spring.cloud.version}</version>
		</dependency>
        <dependency>
		    <groupId>org.springframework.cloud</groupId>
		    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
		    <version>${spring.cloud.version}</version>
		</dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <scope>provided</scope>
        </dependency>

    </dependencies>
</project>
```

And the main application:

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.stream.messaging.Processor;

@SpringBootApplication
@EnableBinding(Processor.class)
public class Application {
	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}
}
```

The @EnableBinding annotation will make all the magic behind to have a sink processor that process in and out messages.

### Consumer

```java
@Slf4j
@Component
public class Consumer {

	@StreamListener(Processor.INPUT)
	@SendTo(Processor.OUTPUT)
	public String load(int messageIndex) {
		log.info("Received load message index {}", messageIndex);
		return "Message " + messageIndex;
	}
}
```

The simpler the better. This consumer listens messages from our INPUT queue and replies to an OUTPUT queue. We receive an integer and generate a string with this index.

Internally, Spring Boot Cloud tags the message with a replyTo header, so the response will be handled only by the origin producer.

### Producer

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class RequestAndResponseProducer {

	private final Processor queue;
	private final ThreadLocal<CountDownLatch> barrier = new ThreadLocal<>();
	private final ThreadLocal<List<String>> messages = new ThreadLocal<>();

	public List<String> getMessages(int num) throws InterruptedException {
		messages.set(new ArrayList<>());
		barrier.set(new CountDownLatch(num));

		for (int index = 0; index < num; index++) {
			queue.input().send(MessageBuilder.withPayload(index).build());
		}

		barrier.get().await();

		return messages.get();
	}

	@StreamListener(Processor.OUTPUT)
	public void getMessage1(String msg) {
		log.info("Received message {}", msg);
		messages.get().add(msg);
		barrier.get().countDown(); // This should NOT go to production ever! See conclusions.
	}

}
```

The producer will produce *num* messages and will await until all these messages have been processed by all the consumers.

By design, the service will block the current request until all the responses have been fetch.

## Conclusions

What about if the producer is shutdown? None will receive the message will be sent. Indeed, Spring will raise the next exceptions:

```
2019-10-03 09:36:33.261  INFO 80288 --- [o-auto-1-exec-1] o.s.a.r.c.CachingConnectionFactory       : Attempting to connect to: [localhost:5672]
2019-10-03 09:36:33.273  INFO 80288 --- [o-auto-1-exec-1] o.s.a.r.c.CachingConnectionFactory       : Created new connection: rabbitConnectionFactory.publisher#16a3c8d8:0/SimpleConnection@6469658e [delegate=amqp://admin@127.0.0.1:5672/, localPort= 62283]
2019-10-03 09:36:33.275  INFO 80288 --- [o-auto-1-exec-1] o.s.amqp.rabbit.core.RabbitAdmin         : Auto-declaring a non-durable, auto-delete, or exclusive Queue (input.anonymous.5HvFyAvXQSCgkP2qqe6mIg) durable:false, auto-delete:true, exclusive:true. It will be redeclared if the broker stops and is restarted while the connection factory is alive, but all messages will be lost.
```

What about if something goes wrong with the consumer and the message is lost? If something can go wrong, it will. The producer should not wait indefinitely.

Do I recommend this kind of integrations to handle user requests? NO. I'm a big fan of [SAGA pattern](https://microservices.io/patterns/data/saga.html) where services/consumers/producers should be totally independently each other. In this guide, we coupled the producer with the consumer and we open the door for many things that can go wrong.
