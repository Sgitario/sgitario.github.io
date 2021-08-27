---
layout: post
title: Java 12 New Features
date: 2021-08-27
tags: [ Java ]
---

After looking into [the new features in Java 11](https://sgitario.github.io/java-11-new-features/), let's continue with my series of Java new features with [Java 12](https://openjdk.java.net/projects/jdk/12/).

## New String Methods

| New Method | Example |
| ------------- | ------------- |
| indent() | "Hello".indent(4) -> "    Hello", "    Hello".indent(-4) -> "Hello" |
| transform() | "Hello".transform(str -> str + "!") -> "Hello!" |

## New File Methods

| New Method | Example |
| ------------- | ------------- |
| Files.mismatch(path, path2) | The method is used to compare two files and find the position of the first mismatched byte in their contents. |

## News in Stream API

- Collectors.teeing

```java
double mean = Stream.of(1, 2, 3, 4, 5)
      .collect(Collectors.teeing( // the new collector teeing
		  Collectors.summingDouble(i -> i), // sum
		  Collectors.counting(), // count
		  (sum, count) -> sum / count)); // final value: mean
assertEquals(3.0, mean);
```

## [JEP 325: Switch Expressions (Preview)](https://openjdk.java.net/jeps/325)

| This is on preview, so to use it, you need to supply the `--enable-preview` when running Java.

I really like this change and as far I see, it's the kind of changes that improves the user experience of Java. Before of this, we had to write the `switch` statements like:

```java
switch (day) {
    case MONDAY:
    case FRIDAY:
    case SUNDAY:
        System.out.println(6);
        break;
    case TUESDAY:
        System.out.println(7);
        break;
    case THURSDAY:
    case SATURDAY:
        System.out.println(8);
        break;
    case WEDNESDAY:
        System.out.println(9);
        break;
}
```

Now, we can do simply:

```java
switch (day) {
    case MONDAY, FRIDAY, SUNDAY -> System.out.println(6);
    case TUESDAY                -> System.out.println(7);
    case THURSDAY, SATURDAY     -> System.out.println(8);
    case WEDNESDAY              -> System.out.println(9);
}
```

## [JEP 230: Microbenchmark Suite](https://openjdk.java.net/jeps/230)

This is only meant for developers that are interesed in collaborating on JDK source code.
With these changes, there is a test suite that helps to spot regression issues in terms of performance and allows developers to create new microbenchmark tests.

What framework is used for the microbenchmark suite? [JMH](https://github.com/openjdk/jmh)
How can we run these tests? Information [here](https://github.com/openjdk/jdk/blob/master/doc/testing.md#microbenchmarks)
Where can we find these tests? In [the github repository](https://github.com/openjdk/jdk), at `test/micro`, for example, [the Integers test suite](https://github.com/openjdk/jdk/blob/master/test/micro/org/openjdk/bench/java/lang/Integers.java) looks like:

```java
import org.openjdk.jmh.annotations.Benchmark;
import org.openjdk.jmh.annotations.BenchmarkMode;
import org.openjdk.jmh.annotations.Fork;
import org.openjdk.jmh.annotations.Measurement;
import org.openjdk.jmh.annotations.Mode;
import org.openjdk.jmh.annotations.OutputTimeUnit;
import org.openjdk.jmh.annotations.Param;
import org.openjdk.jmh.annotations.Scope;
import org.openjdk.jmh.annotations.Setup;
import org.openjdk.jmh.annotations.State;
import org.openjdk.jmh.annotations.Warmup;
import org.openjdk.jmh.infra.Blackhole;

import java.util.Random;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@State(Scope.Thread)
@Warmup(iterations = 10, time = 500, timeUnit = TimeUnit.MILLISECONDS)
@Measurement(iterations = 5, time = 1000, timeUnit = TimeUnit.MILLISECONDS)
@Fork(3)
public class Integers {

    @Param("500")
    private int size;

    private String[] strings;

    @Setup
    public void setup() {
        Random r = new Random(0);
        strings = new String[size];
        for (int i = 0; i < size; i++) {
            strings[i] = "" + (r.nextInt(10000) - (5000));
        }
    }

    @Benchmark
    public void parseInt(Blackhole bh) {
        for (String s : strings) {
            bh.consume(Integer.parseInt(s));
        }
    }

	// ...
}
```

Where [Blackhole](http://javadox.com/org.openjdk.jmh/jmh-core/1.6.3/org/openjdk/jmh/infra/Blackhole.html) is to save from the dead-code elimination of the computations resulting in the given values.


## [JEP 189: Shenandoah: A Low-Pause-Time Garbage Collector](https://openjdk.java.net/jeps/189)

From the JEP information:

"Add a new garbage collection (GC) algorithm named Shenandoah which reduces GC pause times by doing evacuation work concurrently with the running Java threads. Pause times with Shenandoah are independent of heap size, meaning you will have the same consistent pause times whether your heap is 200 MB or 200 GB.

This is not the one GC to rule them all. There are other garbage collection algorithms which prioritize throughput or memory footprint over responsiveness. Shenandoah is an appropriate algorithm for applications which value responsiveness and predictable short pauses. The goal is not to fix all JVM pause issues. Pause times due to reasons other than GC like Time To Safe Point (TTSP) issues or monitor inflation are outside the scope of this JEP."

## Some G1 garbage collector enhancements

- [JEP 344: Abortable Mixed Collections for G1](https://openjdk.java.net/jeps/344) - Make G1 mixed collections abortable if they might exceed the pause target.
- [JEP 346: Promptly Return Unused Committed Memory from G1](https://openjdk.java.net/jeps/346) - Enhance the G1 garbage collector to automatically return Java heap memory to the operating system when idle.
