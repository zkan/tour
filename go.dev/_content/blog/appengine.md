---
title: Go and Google App Engine
date: 2011-05-10
by:
- David Symonds
- Nigel Tao
- Andrew Gerrand
tags:
- appengine
- release
summary: Announcing support for Go in Google App Engine.
---


Google’s App Engine provides a reliable,
scalable, easy way to build and deploy applications for the web.
Over a hundred thousand apps are hosted at appspot.com and custom domains
using the App Engine infrastructure.
Originally written for Python apps, in 2009 the system added a Java runtime.
And today, at Google I/O, we’re thrilled to announce that Go will be next.
It’s marked as an experimental App Engine feature for now,
because it’s early days, but both the App Engine and Go teams are very
excited about this milestone.

By early days, we mean that it’s still rolling out.
As of today, the App Engine SDK for Go is [available for download](http://code.google.com/p/googleappengine/downloads/list),
and we will soon enable deployment of Go apps into the App Engine hosting infrastructure.
Today, through the SDK, you’ll be able to write web apps,
learn about the APIs (and the language, if it’s new to you),
and run your web app locally.
Once full deployment is enabled, it’ll be easy to push your app to Google’s cloud.

One of the cool but less obvious things about this news is that it provides
a very easy way to play with Go.
You don’t even need to have Go installed beforehand because the SDK is
fully self-contained.
Just download the SDK, unzip it, and start coding.
Moreover, the SDK’s “dev app server” means you don’t even need to
run the compiler yourself;
everything is delightfully automatic.

What you’ll find in the SDK is many of the standard App Engine APIs,
custom designed in good Go style, including Datastore,
Blobstore, URL Fetch, Mail, Users, and so on.
More APIs will be added as the environment develops.
The runtime provides the full Go language and almost all the standard libraries,
except for a few things that don’t make sense in the App Engine environment.
For instance, there is no `unsafe` package and the `syscall` package is trimmed.
(The implementation uses an expanded version of the setup in the [Go Playground](https://golang.org/doc/play/)
on [golang.org](https://golang.org/).)

Also, although goroutines and channels are present,
when a Go app runs on App Engine only one thread is run in a given instance.
That is, all goroutines run in a single operating system thread,
so there is no CPU parallelism available for a given client request.
We expect this restriction will be lifted at some point.

Despite these minor restrictions, it’s the real language:
Code is deployed in source form and compiled in the cloud using the 64-bit x86 compiler (6g),
making it the first true compiled language that runs on App Engine.
Go on App Engine makes it possible to deploy efficient,
CPU-intensive web applications.

If you want to know more, read the [documentation](http://code.google.com/appengine/docs/go/)
(start with “[Getting Started](http://code.google.com/appengine/docs/go/gettingstarted/)”).
The libraries and SDK are open source, hosted at [http://code.google.com/p/appengine-go/](http://code.google.com/p/appengine-go/).
We’ve created a new [google-appengine-go](http://groups.google.com/group/google-appengine-go) mailing list;
feel free to contact us there with App Engine-specific questions.
The [issue tracker for App Engine](http://code.google.com/p/googleappengine/issues/list)
is the place for reporting issues related to the new Go SDK.

The Go App Engine SDK is [available](http://code.google.com/p/googleappengine/downloads/list)
for Linux and Mac OS X (10.5 or greater);
we hope a Windows version will also be available soon.

We’d like to offer our thanks for all the help and enthusiasm we received
from Google’s App Engine team in making this happen.
