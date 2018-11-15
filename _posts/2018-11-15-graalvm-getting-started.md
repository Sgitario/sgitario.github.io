---
layout: post
title: GraalVM Getting Started
date: 2018-11-15
tags: [ Java, GraalVM ]
---

"The world is polyglot". This is why GraalVM is here. There are many languages around. Many of these languages have a strong community behind. There is no a perfect language, but all of them has strongs and weaks. So, what about if we could run any language on the same machine without dealing to install anything else? This is [GraalVM](https://www.graalvm.org/)!

GraalVM is a Java Virtual Machine that supports many other languages. It works as an interpreter for Java, Groovy, Scala, JS, Ruby, Python, C, C++ ... Therefore, we can run polyglot programs - programs written in multiple languages in only one environment. 

So... Is GraalVM just a Java Virtual Machine that support more languages? **No!** The goal of GraalVM is not only to support more languages, but also to improve the performance of native based runtimes... including Java Virtual Machine. Then, let's see how good is the performance in Java with a very simple example.

# Installation

Follow the instructions from [GraalVM documentation](https://www.graalvm.org/docs/getting-started/):

1. Download the image [here](https://github.com/oracle/graal/releases)
2. Extract the content into a path
3. Include the binaries into the environment variables

If everything went ok, we should see:

```
> java -version                                                                    
openjdk version "1.8.0_192"
OpenJDK Runtime Environment (build 1.8.0_192-20181024123616.buildslave.jdk8u-src-tar--b12)
GraalVM 1.0.0-rc9 (build 25.192-b12-jvmci-0.49, mixed mode)
```

# Benchmark

As we already said, our focus is to measure how good is GraalVM when running the same Java application. The first thing we need is a benchmark. We're going to use [the JMH tool](https://openjdk.java.net/projects/code-tools/jmh/) from OpenJDK for writing benchmarks.

First, create a Java 8 Maven project with [the JMH dependency](https://mvnrepository.com/artifact/org.openjdk.jmh) in the *pom.xml*:

```xml	
<properties>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
</properties>

<dependencies>
    <dependency>
        <groupId>org.openjdk.jmh</groupId>
        <artifactId>jmh-core</artifactId>
        <version>1.19</version>
    </dependency>
    <dependency>
        <groupId>org.openjdk.jmh</groupId>
        <artifactId>jmh-generator-annprocess</artifactId>
        <version>1.19</version>
    </dependency>
</dependencies>
```

We'll use the *shade* plugin to create the manifest of our standalone benchmark application:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-shade-plugin</artifactId>
            <version>2.2</version>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>shade</goal>
                    </goals>
                    <configuration>
                        <finalName>benchmarks</finalName>
                        <transformers>
                            <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                <mainClass>org.openjdk.jmh.Main</mainClass>
                            </transformer>
                        </transformers>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

Finally, let's write a simple test using streams in Java that transform the data several times and then output a result:

```java
package benchmarks;

import java.util.Arrays;
import org.openjdk.jmh.annotations.Benchmark;
import org.openjdk.jmh.annotations.Scope;
import org.openjdk.jmh.annotations.State;

@State(Scope.Benchmark)
public class Streams {
	
  private double[] values = new double[2000000];
  
  @Benchmark
  public double mapReduce() {
    return Arrays.stream(values)
      .map(x -> x + 1)
      .map(x -> x * 2)
      .map(x -> x + 5)
      .reduce(0, Double::sum);
  }
}
```

To run this benchmark:

```
> mvn package
> java -jar target/benchmarks.jar mapReduce  -f1 -wi 4 -i4
```

Where:
- f1: Only one fork
- wi 4: 4 warmup iterations
- i4: 4 iterations

# Results

This is the output running the benchmark in JDK 11:

```
# JMH version: 1.19
# VM version: JDK 11.0.1, VM 11.0.1+13-LTS
# VM invoker: /Library/Java/JavaVirtualMachines/jdk-11.0.1.jdk/Contents/Home/bin/java
# VM options: <none>
# Warmup: 4 iterations, 1 s each
# Measurement: 4 iterations, 1 s each
# Timeout: 10 min per iteration
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Throughput, ops/time
# Benchmark: benchmarks.Streams.mapReduce

# Run progress: 0.00% complete, ETA 00:00:08
# Fork: 1 of 1
# Warmup Iteration   1: 33.821 ops/s
# Warmup Iteration   2: 36.528 ops/s
# Warmup Iteration   3: 36.776 ops/s
# Warmup Iteration   4: 36.554 ops/s
Iteration   1: 37.442 ops/s
Iteration   2: 37.369 ops/s
Iteration   3: 37.498 ops/s
Iteration   4: 37.517 ops/s


Result "benchmarks.Streams.mapReduce":
  37.456 ±(99.9%) 0.431 ops/s [Average]
  (min, avg, max) = (37.369, 37.456, 37.517), stdev = 0.067
  CI (99.9%): [37.026, 37.887] (assumes normal distribution)


# Run complete. Total time: 00:00:08

Benchmark           Mode  Cnt   Score   Error  Units
Streams.mapReduce  thrpt    4  37.456 ± 0.431  ops/s
```

And in GraalVM:

```
# JMH version: 1.19
# VM version: JDK 1.8.0_192, VM 25.192-b12-jvmci-0.49
# VM invoker: /Users/josecarvajal/Development/Apps/graalvm-ce-1.0.0-rc9/Contents/Home/jre/bin/java
# VM options: <none>
# Warmup: 4 iterations, 1 s each
# Measurement: 4 iterations, 1 s each
# Timeout: 10 min per iteration
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Throughput, ops/time
# Benchmark: benchmarks.Streams.mapReduce

WARNING: Not a HotSpot compiler command compatible VM ("GraalVM 1.0.0-rc9-1.8.0_192"), compilerHints are disabled.
# Run progress: 0.00% complete, ETA 00:00:08
# Fork: 1 of 1
# Warmup Iteration   1: 5.501 ops/s
# Warmup Iteration   2: 29.954 ops/s
# Warmup Iteration   3: 59.042 ops/s
# Warmup Iteration   4: 59.700 ops/s
Iteration   1: 60.943 ops/s
Iteration   2: 60.185 ops/s
Iteration   3: 61.112 ops/s
Iteration   4: 60.701 ops/s


Result "benchmarks.Streams.mapReduce":
  60.735 ±(99.9%) 2.609 ops/s [Average]
  (min, avg, max) = (60.185, 60.735, 61.112), stdev = 0.404
  CI (99.9%): [58.126, 63.344] (assumes normal distribution)


# Run complete. Total time: 00:00:08

Benchmark           Mode  Cnt   Score   Error  Units
Streams.mapReduce  thrpt    4  60.735 ± 2.609  ops/s
```

There is a **HUGE** difference! The same benchmark in GraalVM scored **60.735** operations per second and in latest JDK 11 **37.456**.