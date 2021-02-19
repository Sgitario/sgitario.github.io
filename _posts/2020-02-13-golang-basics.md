---
layout: post
title: Golang Basics
date: 2020-02-13
tags: [ Golang ]
---

This is a very basic post about some golang main concepts. 

[Golang](https://golang.org/) is a open source programming language, designed by Google. The core concepts around golang are:

- Developer Experience: golang aims for simplicity structure and syntax, based mainly on functions. It gets rid of dynamic types or automatic conversions among types, in order to gain performance and intuitive compiler errors (errors if a variable is not used as an example).
- Types: integers, string, boolean, pointers, complex... that's it!
- Concurrency: it supports concurrency using coroutines and channels. 
- Cross-platform: the runtime program can be executed in all OS.
- Garbage Collector
- Strong naming conventions: Exportable functions are capital letters (when we're writing libraries) and Internal functions lowercase.

## First example

*main.go:*
```go
package main

import "fmt" // printf, ... format utils

/**
* functions: func
**/ 
func main() {
	fmt.Println("Hello world")
}

```

Run:

```bash
go run main.go
```

## Eclipse integration

- Install golang extension
- Setup golang preferences

Preferences> Go

Directory: /usr
Eclipse GOPATH: :/home/josecarvajalhilario/go

- Create go project
- Move the above file to src/main (where main is the package name we declared in the go file)

## Build and run the binaries

```bash
go fmt main.go -- format the code
go build main.go -- build the binaries
./main -- run the binaries
```

## Functions

```go
func foo() string {
    return "word"
}

func main() { // if we don't specify a type, then it's void.
    fmt.Println(foo())
}
```

Moreover, functions can return more than one value:

```go
func foo() string, string {
    return "hello", "world"
}
```

## Variables

Every variable starts with *var*

```go
var Glob string // global variable

func main() { 
    var s string // local variable
    s = foo() // ... or directly: var s = foo()
    // ...  or s := foo()
}
```

## Time package

```go
import "time"
func foo() string {
	time.Sleep(time.Second) // sleep one second
	return "faa"
}
```

## Threads

```go
import "fmt"
import "time"

func f1(s string) {
    time.Sleep(time.Second)
    fmt.Println("async: " + s)
}

func main() {
    go f1("1") // 
    go f1("2") // these two calls are done in parallel
    time.Sleep(3 * time.Second)
	fmt.Println("done")
}
```

- channels: to share data across threads

```go
package main

import "fmt" // printf, ... format utils

func main() {
	c := make(chan string)
	go fromChannels(c)
	c <- "One" // send the data to the channel
	c <- "Two" 
    c <- "Three" 
    close(c) // close the channel: won't be more data
	fmt.Println("main function")
}

func fromChannels(c chan string) {
	for {
		local, ok := <- c // wait for the data in the channel
		if !ok { // check whether the channel is closed
			return
		}
		fmt.Println("Received from channel: " + local)
	}
}
```

Or just:

```go
func fromChannels(c chan string) {
	for local := range(c) { // wait for the data in the channel
		fmt.Println("Received from channel: " + local)
	}
	
	fmt.Println("Channel closed")
}
```