---
layout: post
title: Java Application Performance and Memory Management
date: 2019-04-25
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

# JVM Tuning - Flags

## Spring pool tuning
- -XX:+PrintStringTableStatistics
- -XX:StringTableSize=120121 (a prime number)