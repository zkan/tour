---
title: The Go Programming Language turns two
date: 2011-11-10
by:
- Andrew Gerrand
tags:
- appengine
- community
- gopher
summary: Happy 2nd birthday, Go!
---


Two years ago a small team at Google went public with their fledgling project -
the Go Programming Language.
They presented a language spec, two compilers,
a modest standard library, some novel tools,
and plenty of accurate (albeit succinct) documentation.
They watched with excitement as programmers around the world began to play with Go.
The team continued to iterate and improve on what they had built,
and were gradually joined by dozens - and then hundreds - of programmers
from the open source community.
The Go Authors went on to produce lots of libraries,
new tools, and reams of [documentation](https://golang.org/doc/docs.html).
They celebrated a successful year in the public eye with a [blog post](https://blog.golang.org/2010/11/go-one-year-ago-today.html)
last November that concluded "Go is certainly ready for production use,
but there is still room for improvement.
Our focus for the immediate future is making Go programs faster and more
efficient in the context of high performance systems."

Today is the second anniversary of Go's release,
and Go is faster and more stable than ever.
Careful tuning of Go's code generators, concurrency primitives,
garbage collector, and core libraries have increased the performance of Go programs,
and native support for [profiling](https://blog.golang.org/2011/06/profiling-go-programs.html)
and [debugging](http://blog.golang.org/2011/10/debugging-go-programs-with-gnu-debugger.html)
makes it easier to detect and remove performance issues in user code.
Go is also now easier to learn with [A Tour of Go](http://tour.golang.org/),
an interactive tutorial you can take from the comfort of your web browser.

This year we introduced the experimental [Go runtime](http://code.google.com/appengine/docs/go/)
for Google's App Engine platform,
and we have been steadily increasing the Go runtime's support for App Engine's APIs.
Just this week we released [version 1.6.0](http://code.google.com/appengine/downloads.html)
of the Go App Engine SDK,
which includes support for [backends](http://code.google.com/appengine/docs/go/backends/overview.html)
(long-running processes),
finer control over datastore indexes, and various other improvements.
Today, the Go runtime is near feature parity with - and is a viable alternative
to - the Python and Java runtimes.
In fact, we now serve [golang.org](https://golang.org/) by running a version
of [godoc](https://golang.org/cmd/godoc/) on the App Engine service.

While 2010 was a year of discovery and experimentation,
2011 was a year of fine tuning and planning for the future.
This year we issued several "[release](https://golang.org/doc/devel/release.html)"
versions of Go that were more reliable and better supported than weekly snapshots.
We also introduced [gofix](https://golang.org/cmd/gofix/) to take the
pain out of migrating to newer releases.
Furthermore, last month we announced a [plan for Go version 1](https://blog.golang.org/2011/10/preview-of-go-version-1.html) -
a release that will be supported for years to come.
Work toward Go 1 is already underway and you can observe our progress by
the latest weekly snapshot at [weekly.golang.org](http://weekly.golang.org/pkg/).

The plan is to launch Go 1 in early 2012.
We hope to bring the Go App Engine runtime out of "experimental" status at the same time.

But that's not all. 2011 was an exciting year for the gopher, too.
He has manifested himself as a plush toy (a highly prized gift at Google
I/O and other Go talks) and in vinyl form (received by every attendee at
OSCON and now available at the [Google Store](http://www.googlestore.com/Fun/Go+Gopher+Figurine.axd)).

{{image "2years/2years-gophers.jpg"}}

And, most surprisingly, at Halloween he made an appearance with his gopher girlfriend!

{{image "2years/2years-costume.jpg"}}

Photograph by [Chris Nokleberg](https://plus.google.com/106640494112897458359/posts).
