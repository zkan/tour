---
title: Building StatHat with Go
date: 2011-12-19
by:
- Patrick Crosby
tags:
- guest
summary: How StatHat uses Go, and why they chose it.
---

## Introduction

My name is Patrick Crosby and I'm the founder of a company called Numerotron.
We recently released [StatHat](http://www.stathat.com).
This post is about why we chose to develop StatHat in [Go](https://golang.org),
including details about how we are using Go.

[StatHat](http://www.stathat.com) is a tool to track statistics and events in your code.
Everyone from HTML designers to backend engineers can use StatHat easily,
as it supports sending stats from HTML, JavaScript,
Go, and twelve other languages.

You send your numbers to StatHat; it generates beautiful,
fully-embeddable graphs of your data.
StatHat will alert you when specified triggers occur,
send you daily email reports, and much more.
So instead of spending time writing tracking or reporting tools for your application,
you can concentrate on the code.
While you do the real work, StatHat remains intensely vigilant,
like an eagle in its mountaintop nest, or a babysitter on meth.

Here's an example of a StatHat graph of the temperature in NYC, Chicago, and San Francisco:

{{image "stathat/weather.png"}}

## Architecture Overview

StatHat consists of two main services:  incoming statistic/event API calls
and the web application for viewing and analyzing stats.
We wanted to keep these as separate as possible to isolate the data collection
from the data interaction.
We did this for many reasons, but one major reason is that we anticipate
handling a ton of automated incoming API HTTP requests and would thus have
different optimization strategies for the API service than a web application
interacting with humans.

{{image "stathat/architecture.png"}}

The web application service is multi-tiered.
The web server processes all requests and sends them to an interactor layer.
For simple tasks, the interactor will handle generating any necessary data.
For complex tasks, the interactor relies on multiple application servers
to handle tasks like generating graphs or analyzing data sets.
After the interactor is finished, the web server sends the result to a presenter.
The presenter responds to the HTTP request with either HTML or JSON.
We can horizontally scale the web, API, application servers,
and databases as the demand for services grows and changes over time.
There is no single point of failure as each application server has multiple copies running.
The interactor layer allows us to have different interfaces to the system:
http, command line, automated tests, mobile API.
StatHat uses MySQL for data storage.

## Choosing Go

When we designed StatHat, we had the following check list for our development tools:

  - same programming language for backend and frontend systems

  - good, fast HTML templating system

  - fast start-up, recompilation, testing for lots of tinkering

  - lots of connections on one machine

  - language tools for handling application-level concurrency

  - good performance

  - robust RPC layer to talk between tiers

  - lots of libraries

  - open source

We evaluated many popular and not-so-popular web technologies and ended up choosing to develop it in Go.

When Go was released in November 2009, I immediately installed it and loved
the fast compilation times,
goroutines, channels, garbage collection,
and all the packages that were available.
I was especially pleased with how few lines of code my applications were using.
I soon experimented with making a web app called [Langalot](http://langalot.com/)
that concurrently searched through five foreign language dictionaries as
you typed in a query.
It was blazingly fast.  I put it online and it's been running since February, 2010.

The following sections detail how Go meets StatHat's requirements and our experience using Go to solve our problems.

## Runtime

We use the standard Go [http package](https://golang.org/pkg/http/) for
our API and web app servers.
All requests first go through Nginx and any non-file requests are proxied
to the Go-powered http servers.
The backend servers are all written in Go and use the [rpc package](https://golang.org/pkg/rpc/)
to communicate with the frontend.

## Templating

We built a template system using the standard [template package](https://golang.org/pkg/template/).
Our system adds layouts, some common formatting functions,
and the ability to recompile templates on-the-fly during development.
We are very pleased with the performance and functionality of the Go templates.

## Tinkering

In a previous job, I worked on a video game called Throne of Darkness that was written in C++.
We had a few header files that, when modified,
required a full rebuild of the entire system, 20-30 minutes long.
If anyone ever changed `Character.h`, he would be subject to the wrath of
every other programmer.
Besides this suffering, it also slowed down development time significantly.

Since then, I've always tried to choose technologies that allowed fast, frequent tinkering.
With Go, compilation time is a non-issue.
We can recompile the entire system in seconds, not minutes.
The development web server starts instantly,
tests complete in a few seconds.
As mentioned previously, templates are recompiled as they change.
The result is that the StatHat system is very easy to work with,
and the compiler is not a bottleneck.

## RPC

Since StatHat is a multi-tiered system, we wanted an RPC layer so that all
communication was standard.
With Go, we are using the [rpc package](https://golang.org/pkg/rpc/) and
the [gob package](https://golang.org/pkg/gob/) for encoding Go objects.
In Go, the RPC server just takes any Go object and registers its exported methods.
There is no need for an intermediary interface description language.
We've found it very easy to use and many of our core application servers
are under 300 lines of code.

## Libraries

We don't want to spend time rewriting libraries for things like SSL,
database drivers, JSON/XML parsers.
Although Go is a young language, it has a lot of system packages and a growing
number of user-contributed packages.
With only a few exceptions, we have found Go packages for everything we have needed.

## Open source

In our experience, it has been invaluable to work with open source tools.
If something is going awry, it is immensely helpful to be able to examine
the source through every layer and not have any black boxes.
Having the code for the language, web server,
packages, and tools allows us to understand how every piece of the system works.
Everything in Go is open source.  In the Go codebase,
we frequently read the tests as they often give great examples of how to
use packages and language features.

## Performance

People rely on StatHat for up to the minute analysis of their data and we
need the system to be as responsive as possible.
In our tests, Go's performance blew away most of the competition.
We tested it against Rails, Sinatra, OpenResty, and Node.
StatHat has always monitored itself by tracking all kinds of performance
metrics about requests,
the duration of certain tasks, the amount of memory in use.
Because of this, we were able to easily evaluate different technologies.
We've also taken advantage of the benchmark performance testing features
of the Go testing package.

## Application-Level Concurrency

In a former life, I was the CTO at OkCupid.
My experience there using OKWS taught me the importance of async programming,
especially when it comes to dynamic web applications.
There is no reason you should ever do something like this synchronously:
load a user from the database, then find their stats,
then find their alerts.
These should all be done concurrently, yet surprisingly,
many popular frameworks have no async support.
Go supports this at the language level without any callback spaghetti.
StatHat uses goroutines extensively to run multiple functions concurrently
and channels for sharing data between goroutines.

## Hosting and Deployment

StatHat runs on Amazon's EC2 servers.  Our servers are divided into several types:

  - API

  - Web

  - Application servers

  - Database

There are at least two of each type of server,
and they are in different zones for high availability.
Adding a new server to the mix takes just a couple of minutes.

To deploy, we first build the entire system into a time-stamped directory.
Our packaging script builds the Go applications,
compresses the CSS and JS files, and copies all the scripts and configuration files.
This directory is then distributed to all the servers,
so they all have an identical distribution.
A script on each server queries its EC2 tags and determines what it is responsible
for running and starts/stops/restarts any services.
We frequently only deploy to a subset of the servers.

## More

For more information on StatHat, please visit [stathat.com](http://www.stathat.com).
We are releasing some of the Go code we've written.
Go to [www.stathat.com/src](http://www.stathat.com/src) for all of the
open source StatHat projects.

To learn more about Go, visit [golang.org](https://golang.org/).
