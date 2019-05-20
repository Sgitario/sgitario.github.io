---
layout: post
title: Java 10 New Features
date: 2019-05-20
tags: [ Java, Java 10 ]
---

After looking into [the new features in Java 9](https://sgitario.github.io/java-9-new-features/), I wanted to continue my series of Java new features with Java 10. 

# Local-Variable Type Inference

Let's recap a bit about this topic. Earlier than Java 7, we had to declare the types as:

```java
Map<String, String> map = new HashMap<String, String>();
```

Then, Java 7 brought [the Diamond Operator](https://www.javaworld.com/article/2074080/core-java-jdk-7-the-diamond-operator.html) where we don't need to provide the right type:

```java
Map<String, String> map = new HashMap<>();
```

What if we don't need to know about the left side type? Embrace [the local-variable type inference](http://openjdk.java.net/jeps/286). This feature is native in Javascript and has been available in C# from a long while ago ...

```java
var map = new HashMap<String, String>();
```

It might look like a very unnecessary, however when working with Streams, sometimes it's a bit frustrating to infer the type your self:

```java
var list = stream.collect(Collectors.toList());
```

# APIs for Creating Unmodifiable Collections

List.copyOf, Set.copyOf and Map.copyOf methods have been added in the Collections API in order to make unmodifiable copies:

```java
var origin = new ArrayList<Integer>();
origin.add(1);
origin.add(2);
var copy = List.copyOf(origin);
copy.add(3); // unsopported operation exception
```

Also, the Collectors API have been updated to create unmodifiable collections:

```java
var list = stream.collect(Collectors.toUnmodifiableList());
```

# Parallel Full GC for G1

We already talked about the new default garbace collector since Java 9: G1GC. In Java 10, this garbage collector has been refactored to improve the worst-case latencies. More in [here](http://openjdk.java.net/jeps/307).

# Minor Changes
- New Optional.orElseThrow() Method
- Removal of Deprecated Methods
- Removal of Runtime.getLocalizedInputStream and getLocalizedOutputStream Methods
- Removal of javah 