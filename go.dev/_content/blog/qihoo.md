---
title: Qihoo 360 and Go
date: 2015-07-06
by:
- Yang Zhou
summary: How Qihoo 360 uses Go.
---


_This guest blog post was written by Yang Zhou, Software Engineer at Qihoo 360._

[Qihoo 360](http://www.360safe.com/) is a major provider of Internet and
mobile security products and services in China, and operates a major
Android-based mobile distribution platform. At the end of June 2014, Qihoo had
about 500 million monthly active PC Internet users and over 640 million mobile
users. Qihoo also operates one of China’s most popular Internet browsers and PC
search engines.

My team, the Push Service Team, provides fundamental messaging services for
more than 50 products across the company (both PC and mobile), including
thousands of Apps in our open platform.

Our "love affair" with Go dates back to 2012 when we first attempted to provide
push services for one of Qihoo’s products. The initial version was built with
nginx + lua + redis, which failed to satisfy our requirement for real-time
performance due to excessive load. Under these circumstances, the
newly-published Go 1.0.3 release came to our attention. We completed a
prototype in a matter of weeks, largely thanks to the goroutine and channel
features it provided.

Initially, our Go-based system ran on 20 servers, with 20 million real-time
connections in total. The system sent 2 million messages a day. That system now
runs on 400 servers, supporting 200 million+ real-time connections. It now
sends over 10 billion messages daily.

With rapid business expansion and increasing application needs for our push
service, the initial Go system quickly reached its bottleneck: heap size went
up to 69G, with maximum garbage collection (GC) pauses of 3-6 seconds. Worse
still, we had to reboot the system every week to release memory. It wouldn’t be
honest if we didn’t consider relinquishing Go and instead, re-writing the
entire core component with C. However, things didn’t go exactly as we planned,
we ran into trouble migrating the code of Business Logic Layer. As a result, it
was impossible for the only personnel at that time (myself) to maintain the Go
system while ensuring the logic transfer to the C service framework.

Therefore, I made the decision to stay with Go system (probably the wisest one
I had to make), and great headway was made soon enough.

Here are a few tweaks we made and key take-aways:

  - Replace short connections with persistent ones (using a connection pool),
    to reduce creation of buffers and objects during communication.
  - Use Objects and Memory pools appropriately, to reduce the load on the GC.

{{image "qihoo/image00.png"}}

  - Use a Task Pool, a mechanism with a group of long-lived goroutines consuming
    global task or message queues sent by connection goroutines,
    to replace short-lived goroutines.

  - Monitor and control goroutine numbers in the program.
    The lack of control can cause unbearable burden on the GC,
    imposed by surges in goroutines due to uninhibited acceptance of external requests,
    as RPC invocations sent to inner servers may block goroutines recently created.

  - Remember to add [read and write deadlines](https://golang.org/pkg/net/#Conn)
    to connections when under a mobile network;
    otherwise, it may lead to goroutine blockage.
    Apply it properly and with caution when under a LAN network,
    otherwise your RPC communication efficiency will be hurt.

  - Use Pipeline (under Full Duplex feature of TCP) to enhance the communication efficiency of RPC framework.

As a result, we successfully launched three iterations of our architecture,
and two iterations of our RPC framework even with limited human resources.
This can all attributed to the development convenience of Go.
Below you can find the up-to-date system architecture:

{{image "qihoo/image01.png"}}

The continuous improvement journey can be illustrated by a table:

{{image "qihoo/table.png"}}

Also, no temporary release of memory or system reboot is required after these
optimizations.

What’s more exciting is we developed an on-line real-time Visibility Platform
for profiling Go programs. We can now easily access and diagnose the system
status, pinning down any potential risks. Here is a screen shot of the system
in action:

{{image "qihoo/image02.png"}}
{{image "qihoo/image03.png"}}

The great thing about this platform is that we can actually simulate the
connection and behavior of millions of online users, by applying the
Distributed Stress Test Tool (also built using Go), and observe all real-time
visualized data. This allows us to evaluate the effectiveness of any
optimization and preclude problems by identifying system bottlenecks.

Almost every possible system optimization has been practiced so far. And we
look forward to more good news from the GC team so that we could be further
relieved from heavy development work. I guess our experience may also grow
obsolete one day, as Go continues to evolve.

This is why I want to conclude my sharing by extending my sincere appreciation
to the opportunity to attend [Gopher China](http://gopherchina.org/).
It was a gala for us to learn, to share and for offering a window showcasing
Go’s popularity and prosperity in China. Many other teams within Qihoo have
already either got to know Go, or tried to use Go.

I am convinced that many more Chinese Internet firms will join us in
re-creating their system in Go and the Go team's efforts will benefit more
developers and enterprises in the foreseeable future.
