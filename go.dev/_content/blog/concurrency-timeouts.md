---
title: "Go Concurrency Patterns: Timing out, moving on"
date: 2010-09-23
by:
- Andrew Gerrand
tags:
- concurrency
- technical
summary: How to implement timeouts using Go's concurrency support.
---


Concurrent programming has its own idioms.
A good example is timeouts. Although Go's channels do not support them directly,
they are easy to implement.
Say we want to receive from the channel `ch`,
but want to wait at most one second for the value to arrive.
We would start by creating a signalling channel and launching a goroutine
that sleeps before sending on the channel:

{{raw `
	timeout := make(chan bool, 1)
	go func() {
	    time.Sleep(1 * time.Second)
	    timeout <- true
	}()
`}}

We can then use a `select` statement to receive from either `ch` or `timeout`.
If nothing arrives on `ch` after one second,
the timeout case is selected and the attempt to read from ch is abandoned.

{{raw `
	select {
	case <-ch:
	    // a read from ch has occurred
	case <-timeout:
	    // the read from ch has timed out
	}
`}}

The `timeout` channel is buffered with space for 1 value,
allowing the timeout goroutine to send to the channel and then exit.
The goroutine doesn't know (or care) whether the value is received.
This means the goroutine won't hang around forever if the `ch` receive happens
before the timeout is reached.
The `timeout` channel will eventually be deallocated by the garbage collector.

(In this example we used `time.Sleep` to demonstrate the mechanics of goroutines and channels.
In real programs you should use ` [time.After](https://golang.org/pkg/time/#After)`,
a function that returns a channel and sends on that channel after the specified duration.)

Let's look at another variation of this pattern.
In this example we have a program that reads from multiple replicated databases simultaneously.
The program needs only one of the answers,
and it should accept the answer that arrives first.

The function `Query` takes a slice of database connections and a `query` string.
It queries each of the databases in parallel and returns the first response it receives:

{{raw `
	func Query(conns []Conn, query string) Result {
	    ch := make(chan Result)
	    for _, conn := range conns {
	        go func(c Conn) {
	            select {
	            case ch <- c.DoQuery(query):
	            default:
	            }
	        }(conn)
	    }
	    return <-ch
	}
`}}

In this example, the closure does a non-blocking send,
which it achieves by using the send operation in `select` statement with a `default` case.
If the send cannot go through immediately the default case will be selected.
Making the send non-blocking guarantees that none of the goroutines launched
in the loop will hang around.
However, if the result arrives before the main function has made it to the receive,
the send could fail since no one is ready.

This problem is a textbook example of what is known as a [race condition](https://en.wikipedia.org/wiki/Race_condition),
but the fix is trivial.
We just make sure to buffer the channel `ch` (by adding the buffer length
as the second argument to [make](https://golang.org/pkg/builtin/#make)),
guaranteeing that the first send has a place to put the value.
This ensures the send will always succeed,
and the first value to arrive will be retrieved regardless of the order of execution.

These two examples demonstrate the simplicity with which Go can express complex interactions between goroutines.
