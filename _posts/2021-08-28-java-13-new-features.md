---
layout: post
title: Java 13 New Features
date: 2021-08-28
tags: [ Java ]
---

After looking into [the new features in Java 12](https://sgitario.github.io/java-12-new-features/), let's continue with my series of Java new features with [Java 13](https://openjdk.java.net/projects/jdk/13/).

## [JEP 354: Switch Expressions (Second Preview)](https://openjdk.java.net/jeps/354)

| This is on preview, so to use it, you need to supply the `--enable-preview` when running Java.

Another good enhancement on Switch statements that can be now used as expressions! Before of this, in order to return some values, we had to write the `switch` like:

```java
int numLetters;
switch (day) {
    case MONDAY:
    case FRIDAY:
    case SUNDAY:
        numLetters = 6;
        break;
    case TUESDAY:
        numLetters = 7;
        break;
    case THURSDAY:
    case SATURDAY:
        numLetters = 8;
        break;
    case WEDNESDAY:
        numLetters = 9;
        break;
    default:
        throw new IllegalStateException("Wat: " + day);
}
```

Now, we can do simply:

```java
int numLetters = switch (day) {
    case MONDAY, FRIDAY, SUNDAY -> 6;
    case TUESDAY                -> 7;
    case THURSDAY, SATURDAY     -> 8;
    case WEDNESDAY              -> 9;
};
```

What if the method to return values require more than one line? Then we need to use the `yield` statement:

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

## [JEP 355: Text Blocks (Preview)](https://openjdk.java.net/jeps/355)

| This is on preview, so to use it, you need to supply the `--enable-preview` when running Java.

This is another really useful enhancement in terms of user experience. In order to write a multi line String with special characters, we had to do:

```java
String text = "{\r\n" + "\"name\" : \"Foo\"" + "}";
```

Now, we can write text blocks using `"""`:

```java
String text = """
{
    "name" : "Foo"
}
""";
```

Which is more easier to read and less error prone.

## [JEP 351: ZGC: Uncommit Unused Memory (Experimental)](https://openjdk.java.net/jeps/351)

Enhance ZGC to return unused heap memory to the operating system.
Additionally, ZGC now has a maximum supported heap size of 16TB. Earlier, 4TB was the limit.
