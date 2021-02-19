---
layout: post
title: Java 11 New Features
date: 2019-10-02
tags: [ Java ]
---

After looking into [the new features in Java 10](https://sgitario.github.io/java-10-new-features/), I wanted to continue my series of Java new features with [Java 11](https://openjdk.java.net/projects/jdk/11/).

## New String Methods

| New Method | Example |
| ------------- | ------------- |
| isBlank() | "".isBlank() == true // " ".isBlank() == true // "something".isBlank() == false |
| lines() | "Hello\nWorld".lines().collect(Collectors.toList()) == ["Hello", "World"] |
| repeat(times) | "a".repeat(4) == "aaaa" |
| stripLeading() | "    str".stripLeading() == "str" |
| stripTrailing() | "str    ".stripLeading() == "str" |
| strip() | "    str    ".stripLeading() == "str" |

## New File Methods

| New Method | Example |
| ------------- | ------------- |
| Files.writeString(path, str) | Files.writeString(Path.of("my.txt"), "Hello World") |
| Files.readString(path) | Files.readString(Path.of("my.txt")) == "Hello World |
| Files.isSameFile(path, path) | Files.isSameFile(Path.of("my.txt"), Path.of("my.txt")) == true |

## Pattern Methods

- asMatchPredicate()

```java
var str = Pattern.compile("myPattern").asMatchPredicate();
str.test("myPattern"); // returns true
str.test("aba"); // returns false
```

## [JEP 181: Nest-Based Access Control](https://openjdk.java.net/jeps/181)

This feature is about to avoid having nested classes like:

```java
public class Main {
 
    public void myPublic() {
    }
 
    private void myPrivate() {
    }
 
    class Nested {
 
        public void nestedPublic() {
            myPrivate();
        }
    }
}
```

Before Java 11, we could do something like above. However, via Reflection it threw an illegal access exception. After Java 11, we cannot design classes like above.

## [JEP 315: Improve Aarch64 Intrinsics](https://openjdk.java.net/jeps/315)

Specialized CPU architecture-specific code patterns improve the performance of user applications and benchmarks (on AArch64 processors). This includes (but is not limited to) such typical operations as String::compareTo, String::indexOf, StringCoding::hasNegatives, Arrays::equals, StringUTF16::compress, StringLatin1::inflate, and various checksum calculations.

## [JEP 318: Epsilon: A No-Op Garbage Collector](https://openjdk.java.net/jeps/318)

Develop a GC that handles memory allocation but does not implement any actual memory reclamation mechanism. Once the available Java heap is exhausted, the JVM will shut down.

The target usages are in performance testing, memory pressure testing and last drop latency improvements. Sometimes our applications behave with poor performance because of this memory reclamation mechanism. Now, situations like this can be easily spotted and we can increase the memory allocation at startup.

## [JEP 320: Remove the Java EE and CORBA Modules](https://openjdk.java.net/jeps/320)

Remove the Java EE and CORBA modules from the Java SE Platform and the JDK. These modules were deprecated in Java SE 9 with the declared intent to remove them in a future release. The to-be-removed modules include java.xml.ws, java.xml.ws.annotation, jdk.xml.ws, java.xml.bind, jdk.xml.bind. The to-be-removed tools include wsgen, wsimport, schemagen, xjc, and servertool.

## [JEP 321: HTTP Client (Standard)](https://openjdk.java.net/jeps/321)

The incubated HTTP Client API in Java 9 and 10 is standardized under the module java.net.http. This API supports both HTTP 1.1 and 2 and WebSocket through main types HttpClient, HttpRequest, HttpResponse and WebSocket and both sync and async operations.

We already introduced this API in [the Java 9 new features post](https://sgitario.github.io/java-9-new-features/).

## [JEP 323: Local-Variable Syntax for Lambda Parameters](https://openjdk.java.net/jeps/323)

We can allow to type our variables in the lambda expressions. This is something that was in place in Java 8, but was removed in Java 10.

```java
BiFunction<Integer, Integer, Integer> sum = (var s1, var s2) -> s1 + s2;
System.out.println(sum.apply(1, 2));
```

Note, we can use the var in the lambda expression. But what did we win with this? The benefit is that we can use annotations like @NotNull or @Nullable. Another bit to take into account is that we can either define all the types or none (using var).

## [JEP 330: Launch Single-File Source-Code Programs](https://openjdk.java.net/jeps/330)

Now, we can launch our Java programs directly from our sources:

```bash
java Program.java
```

Where Program.java can be something like:

```java
public class Program {
	public static void main(String args[]) {
		System.out.println("Hello 330: Launch Single-File Source-Code Programs");
	}
}
```

## [JEP 328: Flight Recorder (JFR)](https://openjdk.java.net/jeps/328)

Let's embrace a mecanism to produce/consume data as events in-build in Java to gather diagnostics and profiling data. Let's see an example with a producer that registry messages and a consumer that listen these messages:

```java
@Label("Number of messages")
@Description("Track the number of messages sent")
public class FlightRecorderProducerMain extends Event {
	@Label("message")
	String message;

	public static void main(String[] args) throws Exception {
		for (int i = 0; i < 10; i++) {
			final FlightRecorderProducerMain j = new FlightRecorderProducerMain();
			j.message = String.valueOf(i);
			j.commit();
		}
	}
}
```

This is our producer and this is how we run the application:

```bash
java -XX:StartFlightRecording=filename=recording.jfr FlightRecorderProducerMain.java
```

Then, our consumer:

```java
public class FlightRecorderConsumerMain {

	public static void main(String[] args) throws Exception {
		Path p = Paths.get("recording.jfr");
		for (RecordedEvent e : RecordingFile.readAllEvents(p)) {
			final List<ValueDescriptor> lvd = e.getFields();
			for (ValueDescriptor vd : lvd) {
				System.out.println(vd.getLabel() + "=" + e.getValue(vd.getName()));
			}
		}

	}
}
```

By running our consumer, we'll see our messages:

```java
java FlightRecorderConsumerMain.java
```

There is A LOT more about this feature, so if you want to know more, please go to [the Bealdung post](https://www.baeldung.com/java-flight-recorder-monitoring) that as usual provides really helpful information.

## [JEP 332: Transport Layer Security (TLS) 1.3](https://openjdk.java.net/jeps/332)

TLS 1.3 is now supported, however without 0-RTT data, Post-handshake authentication, and Signed certificate timestamps (SCT) (RFC 6962) support.

## [JEP 333: ZGC: A Scalable Low-Latency Garbage Collector (Experimental)](https://openjdk.java.net/jeps/333)

The Z Garbage Collector, also known as ZGC, is a scalable low-latency garbage collector.

- GC pause times should not exceed 10ms
- Handle heaps ranging from relatively small (a few hundreds of megabytes) to very large (many terabytes) in size
- No more than 15% application throughput reduction compared to using G1
- Lay a foundation for future GC features and optimizations leveraging colored pointers and load barriers
- **Initially supported platform: Linux/x64**

