---
title: Two recent Go talks
date: 2013-01-02
by:
- Andrew Gerrand
tags:
- talk
- video
- ethos
summary: "Two Go talks: “Go: A Simple Programming Environment” and “Go: Code That Grows With Grace”."
---

## Introduction

Late last year I wrote a couple of Go talks and presented them at [Strange Loop](http://thestrangeloop.com/),
[Øredev](http://oredev.com), and various other venues.
The talks are designed to give insight into the practice of Go programming,
each describing the construction of a real program and demonstrating the
power and depth of the Go language and its libraries and tools.

The following videos are, in my opinion, the best recordings of these talks.

## Go: a simple programming environment

Go is a general-purpose language that bridges the gap between efficient
statically typed languages and productive dynamic language.
But it’s not just the language that makes Go special – Go has broad
and consistent standard libraries and powerful but simple tools.

This talk gives an introduction to Go, followed by a tour of some real programs
that demonstrate the power,
scope, and simplicity of the Go programming environment.

{{video "https://player.vimeo.com/video/53221558?badge=0" 500 281}}

See the [slide deck](https://talks.golang.org/2012/simple.slide) (use the left and right arrows to navigate).

## Go: code that grows with grace

One of Go's key design goals is code adaptability;
that it should be easy to take a simple design and build upon it in a clean and natural way.
In this talk I describe a simple "chat roulette" server that matches pairs
of incoming TCP connections,
and then use Go's concurrency mechanisms,
interfaces, and standard library to extend it with a web interface and other features.
While the function of the program changes dramatically,
Go's flexibility preserves the original design as it grows.

{{video "https://player.vimeo.com/video/53221560?badge=0" 500 281}}

See the [slide deck](https://talks.golang.org/2012/chat.slide) (use the left and right arrows to navigate).
