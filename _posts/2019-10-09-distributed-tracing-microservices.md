---
layout: post
title: Distributed tracing in Microservices architectures
date: 2019-10-9
tags: [ Design Patterns, Microservices, Jaeger ]
---

Observability is the ability to:

* Which services did the request go
* What did every service do when processing the request
* If the request was slow, where were the bottlenecks
* If the request failed, where did the errors happen
* How different was the execution from the normal system behavior (structural vs performance)
* What was on the critical path of the request
* Who should be paged

Source: [here](https://www.youtube.com/watch?v=EW9GjQNcyzI).
For Distributed Tracing, use [Jaeger](https://www.jaegertracing.io/docs/1.8/).
