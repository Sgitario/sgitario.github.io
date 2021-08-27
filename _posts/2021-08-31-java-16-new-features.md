---
layout: post
title: Java 16 New Features
date: 2021-08-31
tags: [ Java ]
---

After looking into [the new features in Java 15](https://sgitario.github.io/java-15-new-features/), let's continue with my series of Java new features with [Java 16](https://openjdk.java.net/projects/jdk/16/).

## OpenJDK has been migrated to GitHub!

Source code is in https://github.com/openjdk/ now.

## [JEP 394: Pattern Matching for instanceof](https://openjdk.java.net/jeps/394)

This feature was entered in Java 14 and 15 and it's now fully supported.
In summary, we can do things like this:

```java
if (obj instanceof String s) {
    // can use s here
} else {
    // can't use s here
}
```

## [JEP 395: Records](https://openjdk.java.net/jeps/395)

This feature was entered in Java 14 and 15 and it's now fully supported.
In summary, we can write POJO classes simply doing:

```java
public record Person(String name, int age) { 
    Person {
        if (age < 18) {
            throw new IllegalArgumentException("Too young!");
        }
    }
};
```

## [JEP 390: Warnings for Value-Based Classes](https://openjdk.java.net/jeps/390)

Java will output warnings when bad practices are committed. For example, compile-time synchronization warnings depend on static typing, while runtime warnings can respond to synchronization on non-value-based class and interface types, like Object.

```
Double d = 20.0;
synchronized (d) { ... } // javac warning & HotSpot warning
Object o = d;
synchronized (o) { ... } // HotSpot warning
```

## [JEP 397: Sealed Classes (Second Preview)](https://openjdk.java.net/jeps/397)

| This is on preview, so to use it, you need to supply the `--enable-preview` when running Java.

This feature was entered in Java 15 as preview and it has been extended with the following refinements:

- Specify the notion of a contextual keyword, superseding the prior notions of restricted identifier and restricted keyword in the JLS. Introduce the character sequences sealed, non-sealed, and permits as contextual keywords.
- As with anonymous classes and lambda expressions, local classes may not be subclasses of sealed classes when determining the implicitly declared permitted subclasses of a sealed class or sealed interface.
- Enhance narrowing reference conversion to perform stricter checking of cast conversions with respect to sealed type hierarchies.

## Garbage Collector enhancements

- [JEP 376: ZGC: Concurrent Thread-Stack Processing](https://openjdk.java.net/jeps/376) - Move ZGC thread-stack processing from safepoints to a concurrent phase.
- [JEP 387: Elastic Metaspace](https://openjdk.java.net/jeps/387)