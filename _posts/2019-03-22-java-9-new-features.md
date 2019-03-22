---
layout: post
title: Java 9 New Features
date: 2019-03-22
tags: [ Java, Java 9 ]
---

Sorry for another post about the new features of Java 9. There are already tons of sites talking about this. As usual, I like to create a post as a notebook for me with useful tips when working on something. Moreover, this is how I like to learn about something! No more time to excuses.

# Factory Methods

Oh god! How many times did I want to build an immutable list directly with a single command? 

```java
// In Java 8:
List<Integer> list = Collections.unmodifiableList(Arrays.asList(1, 2, 3)); 

// In Java 9:
List<Integer> list = List.of(1, 2, 3); 
```

Digging in how this feature is implemented, we can find nothing fancy here:

```java
    static <E> List<E> of(E e1) {
        return new ImmutableCollections.List12<>(e1);
    }

    static <E> List<E> of(E e1, E e2) {
        return new ImmutableCollections.List12<>(e1, e2);
    }

    static <E> List<E> of(E e1, E e2, E e3) {
        return new ImmutableCollections.ListN<>(e1, e2, e3);
    }

    // ...
```

However, it's still an improvement. What about maps? This is where this new feature comes trully helpful:

```java
// In Java 8:
Map<String, String> auxMap = new HashMap<>();
auxMap.put("key1", "val1");
auxMap.put("key2", "val2");
Map<String, String> map = Collections.unmodifiableMap(auxMap);

// In Java 9:
Map<String, String> map = Map.of("k1", "val1", "k2", "val2"); // :)
```

Of course, the map factory method does a really well usage of generics to map the key to a type and value to another type. 

# Stream API Improvements

There are four new methods in the Stream API:

## - takeWhile()

The takeWhile() method returns all the elements until they match Predicate condition, then the process is stopped. 

```java
Stream<Integer> stream = Stream.of(1,2,3,4,5);
stream.takeWhile(x -> x < 4).forEach(System.out::println);
1
2
3
```

It behaves differently for Ordered and Unordered Streams. **So, pay attention if the collection is not sorted:**

```java
Stream<Integer> stream = Stream.of(1,5,3,4,2);
stream.takeWhile(x -> x < 4).forEach(System.out::println);
1
```

## - dropWhile()

The dropWhile() method returns all the elements after they match Predicate condition, then the process is stopped. 

```java
Stream<Integer> stream = Stream.of(1,2,3,4,5);
stream.dropWhile(x -> x < 3).forEach(System.out::println);
3
4
5
```

It behaves differently for Ordered and Unordered Streams. **So, pay attention if the collection is not sorted:**

```java
Stream<Integer> stream = Stream.of(1,5,3,4,2);
stream.dropWhile(x -> x < 3).forEach(System.out::println);
5
3
4
2
```

## iterate()

This intends to replace the *for* loops:

```java
IntStream.iterate(0, x -> x < 3, x -> x++).forEach(System.out::println);
0
1
2
```

## ofNullable()

This method now returns a sequancial Stream:

```java
Stream<Integer> s = Stream.ofNullable(1);
s.forEach(System.out::println);
1
```

# Reactive Streams

Another important feature coming in Java 9 is the native support of reactive streams. So far, we needed to use [RxJava](https://github.com/ReactiveX/RxJava) or [Akka](https://doc.akka.io/docs/akka/2.5.3/scala/stream/stream-flows-and-basics.html). 

This is the natural evalution of the Java 8 Stream API whereby the streams were tied to the place where you were using it. Now, you can subscribe/publish streams in a decoupled approach:

```java
// Create Publisher
SubmissionPublisher<Person> publisher = new SubmissionPublisher<>();

// Register Subscriber
MySubscriber subs = new MySubscriber();
publisher.subscribe(subs);

publisher.submit(new Person()); // subs is invoked here asynced

// close the Publisher
publisher.close();
```

Please, follows to [this](https://www.journaldev.com/20723/java-9-reactive-streams) good explained article about the topic.

# New HTTP 2 Client

Java 9 finally embraces the [HTTP/2 protocol](https://es.wikipedia.org/wiki/HTTP/2) which supports:
- Multiple requests over a single TCP connection
- Data compression of HTTP headers
- WebSockets: Server Push
- ... and more

Let's see an example of usage:

```java
HttpClient httpClient = HttpClient.newHttpClient(); //Create a HttpClient
HttpRequest request = HttpRequest.newBuilder()
    .uri(new URI("https://www.google.com"))
    .version(HttpClient.Version.HTTP_2)
    .GET()
    .build();
HttpResponse<String> response = httpClient
    .send(request, HttpResponse.BodyHandler.asString()); // sync

CompletableFuture<String> httpResponse = httpClient
    .sendAsync(request, HttpResponse.BodyHandler.asString()); // async
```

More examples in [here](https://www.baeldung.com/java-9-http-client).

# Private Methods in Interfaces

We can now create private methods in Interfaces. I would not say this is a very useful feature in Java 9, moreover I would not say the next example is a good practice. Let's see how default methods (from Java 8) and private methods (from Java 9) can work altogether:

```java
public interface Person {
		
		Eye buildEye();
		
		private int getNumberOfEyes() {
			return 2;
		}
		
		public default List<Eye> buildEyes() {
			return IntStream.range(0, getNumberOfEyes()).mapToObj(i -> buildEye()).collect(Collectors.toList());
		}
	}
	
	public class SpanishPerson implements Person {

		//getNumberOfEyes();  -> not found here as expected
		
		@Override
		public Eye buildEye() {
			
			return new Eye();
		}
		
	}
```

I'm not sure whether this is a good example or not, but I could not find any that suits better.

# Process API Improvements

Java 9 released a new API to manage processes in a OS agnostic way. Let's see an example to start a new Java process:

```java
ProcessBuilder pb = new ProcessBuilder("java");
Process p = pb.start();
System.out.printf("Process ID : %s%n", p.pid());
```

In Windows, we could have started a notepad program like:

```java
ProcessBuilder pb = new ProcessBuilder("notepad.exe");
Process p = pb.start();
System.out.printf("Process ID : %s%n", p.pid());
```

What information can we get from the above process?

```java
ProcessHandle.Info info = p.info();
```

And what about to get information about the current process?

```java
ProcessHandle.Info info = ProcessHandle.current();
```

And about another existing process?

```java
ProcessHandle.allProcesses().filter(aFilter()).findOne();
```

And this is the information we can display or use:

| Info Field  | Description |
| ------------- | ------------- |
| info.command() | the executable pathname |
| info.commandLine() | the command line of the process |
| info.startInstant() | the start time of the process |
| info.totalCpuDuration() | the total cputime accumulated of the process |
| info.arguments() | an array of Strings of the arguments |
| info.user() | the user of the process |

# New Default Garbage Collector: G1GC

The default garbage collector has changed from the [ParallelGC](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html) to the [G1GC](https://www.oracle.com/technetwork/articles/java/g1gc-1984535.html). But what is the different between both?

Basically and as a summary, ParallelGC algorithm does nothing during the application execution in order to not disturb the application. However, when the garbage collector needs to be run (there is no more memory or the memory increased too much: it can be configured), the algorithm needs to loop all the memory and so the application threads are stopped longer ("stop-the-world" key). On the other hand, the G1GC algorithm continously does really quick "collections" or "management" jobs in order to sort the objects in a clever way (in three areas actually), so when the application is running out of memory and the garbage collector needs to play, the application threads are not stopped very long (the garbage collector only runs out the objects of an area). 

# Try With Resources Improvement

In Java 7, we can automatically close any instance that inherits to Closeable in a try-catch:

```java
try (BufferedReader reader = new BufferedReader(...)) {
    // ...
}

// ready will be automatically closed outside of the try.
```

However, this didn't work if the instance was created before the try:

```java
BufferedReader reader = new BufferedReader(...);
try (reader) {
    // ... This only works in Java 9!
}
```

# CompletableFuture API Improvements

The CompletableFuture API has been improved by adding delay and timeout method utilities. Let's see an example:

```java
Executor exe = CompletableFuture.delayedExecutor(50L, TimeUnit.SECONDS);
```

This executor will submit a task after 50 seconds delay.

More about this topic [here](https://www.baeldung.com/java-9-completablefuture).

# Stack Walking API

This API allows us to check the thread callers in a single method:

```java
StackWalker walker = StackWalker.getInstance(StackWalker.Option.RETAIN_CLASS_REFERENCE);
System.out.println(walker.getCallerClass()); // single caller class
walker.walk(...); // loop through the stack frames
```

# Java 9 Module System

Here we are. This new feature is not quite popular in the community. Why would a Java developer need this? My opinion is that we do not. I don't think a Java developer needs to deal with modules. However, this feature is still needed as a low level feature that will help Maven or Gradle dependency management tool to build compacter Java files. 

More about this feature [here](https://www.journaldev.com/13106/java-9-modules).

# Extra

There are much more features like JShell and Strings we didn't talk about. As I said in the introducation, this works for me as a notebook and there are many posts with similar topics. Therefore, I will write about them if I use them in the future.
