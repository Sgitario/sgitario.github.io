---
layout: post
title: Java Application Performance and Memory Management
date: 2019-08-14
tags: [ java ]
---

Performance considerations:
- Memory constraints
- Application speed

XX means advance configuration
: + or - enable or disable
feature in capital letters

-XX:+PrintCompilation: See compilation details.

The higher lever of compilation, the most optimised will be your code.

-XX:+UnlockDiagnosticVMOptions -XX:+LogCompilation: See compilation details in file with more details.

-XX:+PrintCodeCache: See the size of the code cache.
	Change the code cache:
	-XX:InitialCodeCacheSize
	-XX:ReservedCodeCacheSize
	-XX:CodeCacheExpansionSize

-XX:CompileThreshold: num of threads to use for native compilation
-XX:CompileThreshold: num of times that a method needs to be invoked before the code is cached.

## 32 bit vs 64 bit

If app needs more than 4GB and using many long or double types, use 64 bit.

What is the -client flag? More flags: -server or -d64

## Useful commands

- jps

get all running java comands and [ID]s.

- jinfo -flag [A FLAG] [ID]

get the flag value of a running java process.

# Memory

There are three sections:
- Stack (many of them): efficient data structure First-In-Last-Out managed by JVM. This is for local primitives.
- Heap for complex data types. 
- Method spaces

## Memory Leaks

The standard definition of a memory leak is a scenario that occurs when objects are no longer being used by the application, but the Garbage Collector is unable to remove them from working memory – because they’re still being referenced. As a result, the application consumes more and more resources – which eventually leads to a fatal OutOfMemoryError.

- Soft leaks: when an object remains referenced when no longer needed.

## Generate Heap Dump

```bash
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=somepath
```

# JVM Tuning - Flags

## String pool tuning
- -XX:+PrintStringTableStatistics
- -XX:StringTableSize=120121 (a prime number)

# Garbage Collection

- Mark and Sweep algorithm: Stop the world!
- Generational Garbage Collection: Most objects don't live for long, if an object survives, it's likely to live forever. Young and old generation. 
