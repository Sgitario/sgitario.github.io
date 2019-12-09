---
layout: post
title: RSocket Messaging with Spring
date: 2019-12-09
tags: [ Spring, RSocket ]
---

Recently, I found out [RSocket](http://rsocket.io/) messaging framework and what surprise me the most is that it was already being used by many many well known companies like Netflix, Facebook or Pivotal. Let's explore what is RSocket and what problem is addressing.

## Introduction

RSocket is an high performance messaging framework that introduces a **binary** protocol to provide Reactive Streams among applications. It uses websockets over low level connections and send/receive binary data:

![Layers of OSI Model]({{ site.url }}{{ site.baseurl }}/images/rsocket-network-layers-1.png)

Looking at the above picture extracted from [here](https://www.geeksforgeeks.org/layers-of-osi-model/), RSocket works over the Transport Layer (TCP or UDP among others), not **HTTP** which works on the Application Layer. This is a huge difference that we need to take into account, but let's revisit this later on.

What about the implementations? RSocket has many implementations for lot of languages like Java, JS or .NET.

### Interaction Models

RSocket provides multiplex communication with 4 interaction models. This means that we'll have a single connection where we can make multiple requests with.

- Request Response (IN <-> OUT): The typical communication pattern where we send a request and wait until receive the response.
- Request Stream (IN <-> OUT,OUT,OUT, ...): We send a request and asynchronously receive responses one by one.
- Request Channel (IN,IN,IN,... <-> OUT,OUT,OUT,...): Duplex connection which means that the server can send out messages to the requester. The requester can send out multiple requests as well.
- Request and Forget: (IN -): Send a request and don't wait for the server to reply.

### More Features

- Message format

Basically, each request contains the frame (message type); data; and metadata (information about the reouing, security, tracing, etc). However, internally this is a lot of more complicated, so in order to get a better understanding about this, go to [its documentation](http://rsocket.io/docs/Protocol#data-and-metadata) which is very well documented.

- Session resumption

RSocket is able to resume a session after connection failures.

- Backpressure per stream

The requester can specify the amount of data that can handle and won't receive more until it notifies the server that is ready to process more.

- Leasing

This is similar to the backpressure but on the server side only. If the server detects that there is some queuing on the requester side, it will reduce the rate of the requests on it. The requester does not need to have any knowledge to this.

- [Fragmentation and reassembly](https://github.com/rsocket/rsocket/blob/master/Protocol.md#fragmentation-and-reassembly)

We're missing out a few network layers, so what about sending large amound of data? The RSocket supports it as well by fragmenting the request in smaller objects. This is also transparent to us.

## Using RSocket without Spring

In order to understand how RSocket works is a good practice to see how to setup a server/client example from scratch. So, let's avoid using Spring for now and start from the basics. For this, I recommend to follow [this tutorial](https://www.baeldung.com/rsocket) from Baeldung. However, basically this is about to:

### Server

1.- Add the RSocket dependencies in the *pom.xml*:

```xml
<dependency>
	<groupId>io.rsocket</groupId>
	<artifactId>rsocket-transport-netty</artifactId>
	<version>0.12.1</version>
</dependency>
```

We'll be using Netty transport over TCP.

2.- Create the server application:

```java
public class ServerApp {

	public static final int TCP_PORT = 8091;

	private final Disposable server;

	public ServerApp() {
		this.server = RSocketFactory.receive().acceptor(acceptor())
				.transport(TcpServerTransport.create("localhost", TCP_PORT)).start().subscribe();
	}

	private SocketAcceptor acceptor() {
		return (setupPayload, reactiveSocket) -> Mono.just(new AbstractRSocket() {
			@Override
			public Mono<Payload> requestResponse(Payload p) {
				try {
					return Mono.just(p);
				} catch (Exception x) {
					return Mono.error(x);
				}
			}

		});
	}

	public void dispose() {
		this.server.dispose();
	}

	public static void main(String[] args) throws IOException {
		ServerApp server = new ServerApp();
		System.out.println("\nPress any key to exit.\n");
		System.in.read();
		server.dispose();
	}
}
```

As we can see in the *acceptor()* method, we're only implementing the request and response interaction model. If we are interested in implementing them all, have a look into the *AbstractRSocket* class.

After running our application, we're now accepting TCP connections using the 8091 port.

3.- Create the client application:

```java
public class ClientApp {

	public static void main(String[] args) throws IOException {
		RSocket socket = RSocketFactory.connect().transport(TcpClientTransport.create("localhost", ServerApp.TCP_PORT))
				.start().block();

		socket.requestResponse(DefaultPayload.create("Hello")).map(Payload::getDataUtf8).onErrorReturn("error")
				.doOnNext(System.out::println).block();
	}
}
```

When we run the client application, we should see that the server gets the message in the output:

```plain
12:31:32.498 [reactor-tcp-nio-2] DEBUG io.rsocket.FrameLogger - sending -> 
Frame => Stream ID: 1 Type: NEXT_COMPLETE Flags: 0b101100000 Length: 14
Metadata:

Data:
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 48 65 6c 6c 6f                                  |Hello           |
+--------+-------------------------------------------------+----------------+
```

Great, so we have our first running application using the request-response interaction model.

## Using RSocket with Spring

Spring supports [Reactive Streams](https://docs.spring.io/spring-framework/docs/5.0.0.M1/spring-framework-reference/html/web-reactive.html) from Spring Framework 5.0, therefore using RSocket is only about to configure our Spring application properly.

1.- Add the Spring Boot RSocket starter dependency in the *pom.xml*:

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-rsocket</artifactId>
	<version>2.2.2.RELEASE</version>
</dependency>
```

2.- Add the RSocket server configuration using the *application.properties*:

```properties
spring.rsocket.server.port=8091
```

Spring will autoconfigure the server for us. See the full list of parameters [here](https://docs.spring.io/spring-boot/docs/2.2.0.M2/reference/html/appendix.html#rsocket-properties).

3.- Add the controller:

```java
@Controller
public class EchoController {
	@MessageMapping("/requestresponse")
	public Mono<String> service(String request) {
		return Mono.just(request);
	}
}

@SpringBootApplication
public class ServerApp {

	public static void main(String[] args) throws IOException {
		SpringApplication.run(ServerApp.class, args);
	}
}
```

This is purely Spring and implement a request and response message pattern. Note the annotation *@MessageMapping* which allows to write reactive controllers in Spring. 

After running the server application, our RSocket server will start up accordingly.

4.- Let's configure the client to use our Spring RSocket server:

For doing this, we'll create a REST application that internally uses a RSocket client:

```java
@RestController
public class RestEchoController {
	@Autowired
	private RSocketRequester rSocket;

	@GetMapping(value = "/echo/{message}")
	public Publisher<String> echo(@PathVariable("message") String message) {
		return rSocket.route("requestresponse").data(message).retrieveMono(String.class);
	}
}
```

Where the *RSocketRequester.java* is configured as:

```java
@Configuration
public class ClientConfiguration {
	@Bean
	public RSocket rSocket(@Value("${app.rsocket.server.port}") Integer serverPort) {
		return RSocketFactory.connect()
				.mimeType(MimeTypeUtils.APPLICATION_JSON_VALUE, MimeTypeUtils.APPLICATION_JSON_VALUE)
				.frameDecoder(PayloadDecoder.ZERO_COPY).transport(TcpClientTransport.create(serverPort)).start()
				.block();
	}

	@Bean
	RSocketRequester rSocketRequester(RSocket rSocket, RSocketStrategies rSocketStrategies) {
		return RSocketRequester.wrap(rSocket, MimeTypeUtils.APPLICATION_JSON, MimeTypeUtils.APPLICATION_JSON,
				rSocketStrategies);
	}
}
```

And that's all! We have a RSocket server using Spring Webflux and a REST application that internally calls to our RSocket server.

Our application is using the request and response message pattern, let's see how to implement the rest:

- Fire And Forget:

*Server:*

```java
@Controller
public class EchoController {
	@MessageMapping("/fireandforget")
	public Mono<Vood> service(String request) {
		// Do something with the request and then:
		return Mono.empty();
	}
}
```

*Client:*

```java
@RestController
public class RestEchoController {
	@Autowired
	private RSocketRequester rSocket;

	@GetMapping(value = "/echo/{message}")
	public Publisher<Void> echo(@PathVariable("message") String message) {
		return rSocket.route("requestresponse").data(message).send();
	}
}
```

- Request Stream

*Server:*
```java
@Controller
public class EchoController {
	@MessageMapping("/requeststream")
	public Flux<String> service(String request) {
		return repository.getAllByRequest(request);
	}
}
```

*Client:*
```java
@RestController
public class RestEchoController {
	@Autowired
	private RSocketRequester rSocket;

	@GetMapping(value = "/echo/{message}")
	public Publisher<String> echo(@PathVariable("message") String message) {
		return rSocket.route("requestresponse").data(message).retrieveFlux(String.class);
	}
}
```

## Conclusions

We have introduced a bit of RSocket main features and also how easy is to implement the message patterns using Spring.

While writing the post, I asked myself what is different between RSocket and [gRPC](https://sgitario.github.io/grpc-getting-started-for-java-developers/) since both are appearently trying to solve the same problem: allowing duplex connections in and out. However, internally the way of solving this problem is radically different. gRPC is using the new features from HTTP/2 (Application Network) using a Protobuf contract to connect applications whereas RSocket is using the Transport Network. It might sound arbitrary but the implication to use one or another is huge and everything must be taken into account.

Morever, what about Integration Testing and Security? Please, watch out [this video](https://www.youtube.com/watch?v=iSSrZoGtoSE) from Spring to address this issue. At the moment, testing and security need to improve yet but Spring and the rest of community are progressing on it.

Please, checkout the source code from [my Github account](https://github.com/Sgitario/rsocket_tutorial). However, I rather recommend [the Spring Flights RSocket source code](https://github.com/bclozel/spring-flights) which contains integration tests and security.