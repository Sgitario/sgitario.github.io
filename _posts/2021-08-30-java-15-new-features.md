---
layout: post
title: Java 15 New Features
date: 2021-08-30
tags: [ Java ]
---

After looking into [the new features in Java 14](https://sgitario.github.io/java-14-new-features/), let's continue with my series of Java new features with [Java 15](https://openjdk.java.net/projects/jdk/15/).

## [JEP 360: Sealed Classes (Preview)](https://openjdk.java.net/jeps/360)

| This is on preview, so to use it, you need to supply the `--enable-preview` when running Java.

A sealed class or interface can be extended or implemented only by those classes and interfaces permitted to do so:

```java
public abstract sealed class Shape permits Circle, Rectangle, Square {

    public int getCenter(Shape shape) {
        if (shape instanceof Circle) {
            return ... ((Circle)shape).center() ...
        } else if (shape instanceof Rectangle) {
            return ... ((Rectangle)shape).length() ...
        } else if (shape instanceof Square) {
            return ... ((Square)shape).side() ...
        }
    }
}
```

Previously, without having declared the `permits`, we had to implement the `getCenter` method like:

```java
int getCenter(Shape shape) {
    if (shape instanceof Circle) {
        return ... ((Circle)shape).center() ...
    } else if (shape instanceof Rectangle) {
        return ... ((Rectangle)shape).length() ...
    } else if (shape instanceof Square) {
        return ... ((Square)shape).side() ...
    }

    throw new RuntimeException("Unsupported!");
}
```

With sealed, we don't need to throw any runtime exception because we're only allowing the three types of shapes.

Another important point is that every permitted subclass must define a modifier: 
- sealed: to declare that the class is an implementation.
- final: it's like a strong sealed where we can't reimplement the class.
- non-sealed: to say that the current class is still open to extension

More information about these changes in [Baeldung](https://www.baeldung.com/java-sealed-classes-interfaces).

## [JEP 384: Records (Second Preview)](https://openjdk.java.net/jeps/384)

| This is on preview, so to use it, you need to supply the `--enable-preview` when running Java.

This feature was entered in Java 14 as preview, and it's improved here by allowing to extend the generated class, for example to redefine the default constructor:

```java
public record Person(String name, int age) { 
    Person {
        if (age < 18) {
            throw new IllegalArgumentException("Too young!");
        }
    }
};
```

## [JEP 375: Pattern Matching for instanceof (Second Preview)](https://openjdk.java.net/jeps/375)

| This is on preview, so to use it, you need to supply the `--enable-preview` when running Java.

This feature was entered in Java 14 as preview, and it's improved here by allowing to use the variable in the combined expression:

```java
if (person instanceof Person person && person.getAge() > 23) {
    //...
}
```

## [JEP 378: Text Blocks](https://openjdk.java.net/jeps/378)

This feature was entered in Java 13 and 14 as preview and it's now officially supported. 
As a summary, we can write multiline text as:

```java
String text = """
{
    "name" : "Foo", \
    "surname" : "Buu" \s
}
""";
```

## [JEP 371: Hidden Classes](https://openjdk.java.net/jeps/371)

Create classes that are intended to be used internally by frameworks via reflection:

```java
import java.lang.invoke.MethodHandles;
import java.lang.reflect.Constructor;

import static java.lang.invoke.MethodHandles.Lookup.ClassOption.NESTMATE;

public class MyHiddenClass {
    public static void main(String[] args) throws Throwable {
        MethodHandles.Lookup lookup = MethodHandles.lookup();
        ClassWriter cw = GenerateClass.getClassWriter(MyHiddenClass.class);
        byte[] bytes = cw.toByteArray();

        Class<?> c = lookup.defineHiddenClass(bytes, true, NESTMATE).lookupClass();
        // ...
    }
}
```

## Change Garbage Collector Implementations from Experimental to Production

- [JEP 377: ZGC: A Scalable Low-Latency Garbage Collector](https://openjdk.java.net/jeps/377)
- [JEP 379: Shenandoah: A Low-Pause-Time Garbage Collector](https://openjdk.java.net/jeps/379)
