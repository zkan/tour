---
title: "JSON-RPC: a tale of interfaces"
date: 2010-04-27
by:
- Andrew Gerrand
tags:
- json
- rpc
- technical
summary: How to use the net/rpc package's interfaces to create a JSON-RPC system.
---


Here we present an example where Go's [interfaces](https://golang.org/doc/effective_go.html#interfaces_and_types)
made it easy to refactor some existing code to make it more flexible and extensible.
Originally, the standard library's [RPC package](https://golang.org/pkg/net/rpc/)
used a custom wire format called [gob](https://golang.org/pkg/encoding/gob/).
For a particular application, we wanted to use [JSON](https://golang.org/pkg/encoding/json/)
as an alternate wire format.

We first defined a pair of interfaces to describe the functionality of the
existing wire format,
one for the client, and one for the server (depicted below).

	type ServerCodec interface {
	 ReadRequestHeader(*Request) error
	 ReadRequestBody(interface{}) error
	 WriteResponse(*Response, interface{}) error
	 Close() error
	}

On the server side, we then changed two internal function signatures to
accept the `ServerCodec` interface instead of our existing `gob.Encoder`. Here's one of them:

	func sendResponse(sending *sync.Mutex, req *Request,
	 reply interface{}, enc *gob.Encoder, errmsg string)

became

	func sendResponse(sending *sync.Mutex, req *Request,
	  reply interface{}, enc ServerCodec, errmsg string)

We then wrote a trivial `gobServerCodec` wrapper to reproduce the original functionality.
From there it is simple to build a `jsonServerCodec`.

After some similar changes to the client side,
this was the full extent of the work we needed to do on the RPC package.
This whole exercise took about 20 minutes!
After tidying up and testing the new code,
the [final changeset](https://github.com/golang/go/commit/dcff89057bc0e0d7cb14cf414f2df6f5fb1a41ec) was submitted.

In an inheritance-oriented language like Java or C++,
the obvious path would be to generalize the RPC class,
and create JsonRPC and GobRPC subclasses.
However, this approach becomes tricky if you want to make a further generalization
orthogonal to that hierarchy.
(For example, if you were to implement an alternate RPC standard).
In our Go package, we took a route that is both conceptually simpler and
requires less code be written or changed.

A vital quality for any codebase is maintainability.
As needs change, it is essential to adapt your code easily and cleanly,
lest it become unwieldy to work with.
We believe Go's lightweight, composition-oriented type system provides a
means of structuring code that scales.
