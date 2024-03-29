---
title: Introducing HTTP Tracing
date: 2016-10-04
by:
- Jaana Burcu Dogan
tags:
- http
- technical
summary: How to use Go 1.7's HTTP tracing to understand your client requests.
---

## Introduction

In Go 1.7 we introduced HTTP tracing, a facility to gather fine-grained
information throughout the lifecycle of an HTTP client request.
Support for HTTP tracing is provided by the [`net/http/httptrace`](https://golang.org/pkg/net/http/httptrace/)
package. The collected information can be used for debugging latency issues,
service monitoring, writing adaptive systems, and more.

## HTTP events

The `httptrace` package provides a number of hooks to gather information
during an HTTP round trip about a variety of events. These events include:

  - Connection creation
  - Connection reuse
  - DNS lookups
  - Writing the request to the wire
  - Reading the response

## Tracing events

You can enable HTTP tracing by putting an
[`*httptrace.ClientTrace`](https://golang.org/pkg/net/http/httptrace/#ClientTrace)
containing hook functions into a request's [`context.Context`](https://golang.org/pkg/context/#Context).
Various [`http.RoundTripper`](https://golang.org/pkg/net/http/#RoundTripper)
implementations report the internal events by
looking for context's `*httptrace.ClientTrace` and calling the relevant hook functions.

The tracing is scoped to the request's context and users should
put a `*httptrace.ClientTrace` to the request context before they start a request.

{{code "http-tracing/trace.go" `/START/` `/END/`}}

During a round trip, `http.DefaultTransport` will invoke each hook
as an event happens. The program above will print the DNS
information as soon as the DNS lookup is complete. It will similarly print
connection information when a connection is established to the request's host.

## Tracing with http.Client

The tracing mechanism is designed to trace the events in the lifecycle
of a single `http.Transport.RoundTrip`. However, a client may
make multiple round trips to complete an HTTP request. For example, in the case
of a URL redirection, the registered hooks will be called as many times as the
client follows HTTP redirects, making multiple requests.
Users are responsible for recognizing such events at the `http.Client` level.
The program below identifies the current request by using an
`http.RoundTripper` wrapper.

{{code "http-tracing/client.go"}}

The program will follow the redirect of google.com to www.google.com and will output:

	Connection reused for https://google.com? false
	Connection reused for https://www.google.com/? false

The Transport in the `net/http` package supports tracing of both HTTP/1
and HTTP/2 requests.

If you are an author of a custom `http.RoundTripper` implementation,
you can support tracing by checking the request context for an
`*httptest.ClientTrace` and invoking the relevant hooks as the events occur.

## Conclusion

HTTP tracing is a valuable addition to Go for those who are interested
in debugging HTTP request latency and writing tools for network debugging
for outbound traffic.
By enabling this new facility, we hope to see HTTP debugging, benchmarking
and visualization tools from the community — such as
[httpstat](https://github.com/davecheney/httpstat).
