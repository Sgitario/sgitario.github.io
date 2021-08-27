---
layout: post
title: Java 14 New Features
date: 2021-08-29
tags: [ Java ]
---

After looking into [the new features in Java 13](https://sgitario.github.io/java-13-new-features/), let's continue with my series of Java new features with [Java 14](https://openjdk.java.net/projects/jdk/14/).

## [JEP 361: Switch Expressions](https://openjdk.java.net/jeps/361)

The changes around the Switch expressions from Java 12 and Java 13 are no longer preview! In summary, we can now do something like:

```java
int numLetters = switch (day) {
    case MONDAY, FRIDAY, SUNDAY -> 6;
    case TUESDAY                -> 7;
    case THURSDAY, SATURDAY     -> 8;
    case WEDNESDAY              -> 9;
    default      -> {
        int result = foo(day); // some function
        yield result;
    }
};
```

## [JEP 368: Text Blocks (Second Preview)](https://openjdk.java.net/jeps/368)

| This is on preview, so to use it, you need to supply the `--enable-preview` when running Java.

To extend the text blocks capabilities from Java 13, text blocks now have two new escape sequences:

- `\`: to indicate the end of the line, so that a new line character is not introduced
- `\s`: to indicate a single space (can be used in traditional strings)

Example:

```java
String text = """
{
    "name" : "Foo", \
    "surname" : "Buu" \s
}
""";
```

Additional methods:
- String::stripIndent(): used to strip away incidental white space from the text block content
- String::translateEscapes(): used to translate escape sequences
- String::formatted(Object... args): simplify value substitution in the text block

## [JEP 305: Pattern Matching for instanceof (Preview)](https://openjdk.java.net/jeps/305)

| This is on preview, so to use it, you need to supply the `--enable-preview` when running Java.

Another super enhancement in regards of user experience! The usage of the `instanceof` statement. Before, we had to do something like:

```java
if (obj instanceof String) {
    String s = (String) obj;
    // use s
}
```

Now, we can simply do:

```java
if (obj instanceof String s) {
    // can use s here
} else {
    // can't use s here
}
```

## [JEP 359: Records (Preview)](https://openjdk.java.net/jeps/359)

| This is on preview, so to use it, you need to supply the `--enable-preview` when running Java.

Are you bored of having to define the fields, getters and setters every time for a POJO class? Something like:

```java
public class Person {
    private String name;
    private int age;

    public Person(String name, int age) { 
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }
}
```

In Java 14, there is a new `record` statement that will ease the above by simply doing:

```java
public record Person(String name, int age) { };
```

A really nice addition based on Kotlin!

## [JEP 358: Helpful NullPointerExceptions](https://openjdk.java.net/jeps/358)

Having this code in production:

```java
a.b.c.i = 99;
```

When throwing a null pointer exception and we would see something like:

```
Exception in thread "main" java.lang.NullPointerException
    at Prog.main(Prog.java:5)
```

But... what is the field that is null? `a` or `b` or `c`? Now, the JVM will include more helpful information to help in these situations. The new message will look like as:

```
Exception in thread "main" java.lang.NullPointerException: 
        Cannot assign field "i" because "b" is null
    at Prog.main(Prog.java:5)
```

## [JEP 349: JFR Event Streaming](https://openjdk.java.net/jeps/349)

Previously, to get the JFR events, we had to dump the full content of the jfr file and read it. Now, we can read the events asynchronously, simply doing:

```java
try (var rs = new RecordingStream()) {
  rs.enable("jdk.CPULoad").withPeriod(Duration.ofSeconds(1));
  rs.enable("jdk.JavaMonitorEnter").withThreshold(Duration.ofMillis(10));
  rs.onEvent("jdk.CPULoad", event -> {
    System.out.println(event.getFloat("machineTotal"));
  });
  rs.onEvent("jdk.JavaMonitorEnter", event -> {
    System.out.println(event.getClass("monitorClass"));
  });
  rs.start();
}
```

## [JEP 370: Foreign-Memory Access API (Incubator)](https://openjdk.java.net/jeps/370)

Introduce an API to allow Java programs to safely and efficiently access foreign memory outside of the Java heap.

This feature is really well described in [Baeldung](https://www.baeldung.com/java-foreign-memory-access).

## [JEP 345: NUMA-Aware Memory Allocation for G1](https://openjdk.java.net/jeps/345)

Improve G1 performance on large machines by implementing NUMA-aware memory allocation.

## ZGC garbage collector is now supported on [Windows](https://openjdk.java.net/jeps/365) and [macOS](https://openjdk.java.net/jeps/364)

The Z Garbage Collector, a scalable, low-latency garbage collector, was first introduced in Java 11 as an experimental feature. Initially, the only supported platform was Linux/x64, and now is supported on Windows and macOS.