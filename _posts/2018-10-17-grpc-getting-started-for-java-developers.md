---
layout: post
title: gRPC for Java Developpers Getting Started
date: 2018-10-17
tags: [ GRPC, REST, Spring Boot ]
---

# Introduction
[gRPC](https://grpc.io) is a high performance RPC framework that works over HTTP/2. It's designed to "refactor" the REST api to fix some of its pitfalls and bring very many interesting features. 

From my point of view, these are the major features:
- Simple Definition Languages: [Proto3](https://developers.google.com/protocol-buffers/docs/proto3)
- Bi-Direccional Streaming between clients and servers
- Auto-Generated Client Libraries in more than 10 languages

But it's not everything perfect:
- Bye bye, JSON. The client libraries will abstract you of dealing with binary streams, but I'll miss my very simple curl commands. 
- Not ready to be used by web browsers yet.

# Getting Started

Let's start. We'll use Maven for this project. Let's begin with the [gRPC Maven dependencies](https://mvnrepository.com/artifact/io.grpc):

```xml
<dependencies>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-netty-shaded</artifactId>
        <version>1.15.1</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-protobuf</artifactId>
        <version>1.15.1</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-stub</artifactId>
        <version>1.15.1</version>
    </dependency>
</dependencies>
```

And finally let's add the protobuf plugin:

```xml
<build>
    <extensions>
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>1.5.0.Final</version>
        </extension>
    </extensions>
    <plugins>
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.5.1</version>
            <configuration>
                <protocArtifact>com.google.protobuf:protoc:${protoc.version}:exe:${os.detected.classifier}</protocArtifact>
                <pluginId>grpc-java</pluginId>
                <pluginArtifact>io.grpc:protoc-gen-grpc-java:${grpc.version}:exe:${os.detected.classifier}</pluginArtifact>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <goal>compile-custom</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

The [extension](https://github.com/trustin/os-maven-plugin) is used to generate various useful platform-dependent properties in the build. The [protobuf plugin](https://www.xolstice.org/protobuf-maven-plugin/) will create the Java sources from the .proto files.

# Lottery Service

We'll build a very simple lottery service that allows anybody enters and sees the list of players in live.

## Service Definition

This is the service definition written in [Proto3](https://developers.google.com/protocol-buffers/docs/proto3). This language is pretty basic but good enough to see directly what our service is intended to do. 

```java
syntax = "proto3";

option java_package = "com.sgitario.grpc.lottery";

message Empty {
}

message Player {
    string name = 1;
}

message ListPlayers {
    repeated Player players = 1;
}

service LotteryService {
    // Sign Up in the lottery
    rpc enter(Player) returns (Empty);

    // See the players joining to the lottery in live
    rpc seePlayers(Empty) returns (stream Player);

    // See the list of total players
    rpc listPlayers(Empty) returns (ListPlayers);

    // Pick a winner and end all the streaming connections
    rpc pickWinner(Empty) returns (Player);
}
```

The number next to each field in the message is the position within the final binary object. We should not modify this once our services are in production.

We set the java package as the option in "java_package".

## Generating sources

Once the definition is done, we need to generate the sources by doing:

```bash
> mvn clean verify
```

And all the sources will be in the package "com.sgitario.grpc.lottery". The important class is com.sgitario.grpc.lottery.LotteryServiceGrpc.LotteryServiceImplBase. This is the {package}.{service-name}Grpc.{service-name}ImplBase. 

## Service Implementation

All we need to extend is the LotteryServiceImplBase class:

```java
public class LotteryService extends LotteryServiceImplBase {
    // ...
}
```

Then, let's start with the services in our definition. I guess all of these services are self-explanatory.

- Stream players

```java
    private static Set<StreamObserver<Player>> observers = ConcurrentHashMap.newKeySet();

    @Override
    public void seePlayers(Empty request, StreamObserver<Player> responseObserver) {
        players.forEach(responseObserver::onNext);
        observers.add(responseObserver);
    }
```

This service will join the input stream in a list of observers.

- Enter

```java
    private final Set<Player> players = new HashSet<>();

    @Override
    public void enter(Player request, StreamObserver<Empty> responseObserver) {
        LOG.info("New player: " + request.getName());
        players.add(request);
        observers.forEach(o -> o.onNext(request));
        responseObserver.onNext(Empty.getDefaultInstance());
        responseObserver.onCompleted();
	}
```

This service will add the new player into a list and it will notify the observers streams.

- pickWinner

```java
    @Override
    public void pickWinner(Empty request, StreamObserver<Player> responseObserver) {
        observers.forEach(o -> o.onCompleted());
        observers.clear();

        Random rnd = new Random(new Date().getTime());
        int winner = rnd.nextInt(players.size());
        Iterator<Player> it = players.iterator();
        Stream.iterate(0, i -> i + 1).limit(winner).forEach(i -> it.next());
        players.clear();
        responseObserver.onNext(it.next());
        responseObserver.onCompleted();
	}
```

This service will close the observers streams and will pick a random winner. 

- listPlayers

```java
    @Override
    public void listPlayers(Empty request, StreamObserver<ListPlayers> responseObserver) {
        responseObserver.onNext(ListPlayers.newBuilder().addAllPlayers(players).build());
        responseObserver.onCompleted();
	}
```

I was corious about how to have a list if a message and how sources will look like. This is the example. I only had to add the *repeated* keyboard in the definition and I have a list in my message object and the builder. Easy peasy.

### Start The Server

This is just to run a java application and there are a lot of ways to do this. As a Spring adopter, I recommend to use [Spring Boot ecosystem](https://github.com/LogNet/grpc-spring-boot-starter). For this proof of concept, the easiest way is:

```java
import io.grpc.Server;
import io.grpc.ServerBuilder;

public class LotteryServer {
    public static final int PORT = 9090;

    public static void main(String[] args) throws InterruptedException, IOException {
        Server server = ServerBuilder.forPort(PORT).addService(new LotteryService()).build();
        server.start();
        System.out.println("Lottery server at localhost:9090");
        server.awaitTermination();
    }
}
```

This is a new class and run it as a java application. 

## Client Side

We have already our server up and running. But we miss the client side. gRPC also generates some stubs that eases the work a lot:

```java
public class LotteryClient {

    private final ManagedChannel channel;

    public LotteryClient() {
        channel = ManagedChannelBuilder.forAddress("localhost", LotteryServer.PORT).usePlaintext().build();
    }

    // ...
}
```

We only need to use the ManagedChannelBuilder builder from gRPC and that's it. Again, this is the easiest approach.

gRPC provides a blocking and async approach to invoke the methods:

```java
    public LotteryServiceStub async() {
        return LotteryServiceGrpc.newStub(channel);
    }

    public LotteryServiceBlockingStub blocking() {
        return LotteryServiceGrpc.newBlockingStub(channel);
    }
```

And finally all we need is to start using the services accodingly:

- Stream Players

```java
LotteryClient client = new LotteryClient();
client.async().seePlayers(Empty.getDefaultInstance(), new StreamObserver<Player>() {

    @Override
    public void onCompleted() {
        System.out.println("Lottery is completed");
    }

    @Override
    public void onError(Throwable t) {
        System.out.println("Error: " + t);
    }

    @Override
    public void onNext(Player value) {
        System.out.println("New player: " + value.getName());

    }
});
```

This client implementation needs to use the async mode to listen for new players.

- enter

```java
LotteryClient client = new LotteryClient();
String clientName = UUID.randomUUID().toString();
client.blocking().enter(Player.newBuilder().setName(clientName).build());
```

In the other hand, this will use the blocking mode to just enter a new player to the lottery.

# Conclusion

gRPC is very straightforward to use once you understand all the concepts around. Also, there are a very huge community behind giving support and ideas and work. 

One of my main concerns is about how to troubleshoot issues in production. So far, I'm used to work with curl commands and see the responses. But again, thanks to the community, there are some workarounds to, as an example, [return JSON responses or to work as an usual REST API](https://github.com/grpc-ecosystem/grpc-gateway).

See the sources in [my github project](https://github.com/Sgitario/grpc-getting-started).