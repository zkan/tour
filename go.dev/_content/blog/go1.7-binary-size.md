---
title: Smaller Go 1.7 binaries
date: 2016-08-18
by:
- David Crawshaw
summary: Go 1.7 includes some binary size reductions important for small devices.
---

## Introduction

Go was designed for writing servers.
That is how it is most widely used today, and as a result a lot of
work on the runtime and compiler is focused on issues that matter to
servers: latency, ease of deployment, precise garbage collection,
fast startup time, performance.

As Go gets used for a wider variety of programs, there are new issues that must be considered.
One of these is binary size.
It has been on the radar for a long time
(issue [\#6853](https://golang.org/issue/6853) was filed over two
years ago), but the growing interest in using Go for
deploying binaries on smaller devices — such as the Raspberry Pi or
mobile devices — means it received some attention for the Go 1.7
release.

## Work done in Go 1.7

Three significant changes in Go 1.7 affect binary size.

The first is the new SSA backend that was enabled for AMD64 in this release.
While the primary motivation for SSA was improved performance, the
better generated code is also smaller.
The SSA backend shrinks Go binaries by ~5%.
We expect larger gains for the more RISC-like architectures
like ARM and MIPS when those backends have been converted to SSA in Go 1.8.

The second change is method pruning.
Until 1.6, all methods on all used types were kept, even if some of
the methods were never called.
This is because they might be called through an interface, or called
dynamically using the reflect package.
Now the compiler discards any unexported methods that do not match an
interface.
Similarly the linker can discard other exported methods, those that are only
accessible through reflection, if the corresponding
[reflection features](https://golang.org/pkg/reflect/#Value.Call)
are not used anywhere in the program.
That change shrinks binaries by 5–20%.

The third change is a more compact format for run-time type
information used by the reflect package.
The encoding format was originally designed to make the decoder in
the runtime and reflect packages as simple as possible. By making this
code a bit harder to read we can compress the format without affecting
the run-time performance of Go programs.
The new format shrinks Go binaries by a further 5–15%.
Libraries built for Android and archives built for iOS shrink further
as the new format contains fewer pointers, each of which requires
dynamic relocations in position independent code.

In addition, there were many small improvements such as improved
interface data layout, better static data layout, and simplified
dependencies. For example, the HTTP client no longer links in the entire HTTP
server.
The full list of changes can be found in issue
[\#6853](https://golang.org/issue/6853).

## Results

Typical programs, ranging from tiny toys to large production programs,
are about 30% smaller when built with Go 1.7.

The canonical hello world program goes from 2.3MB to 1.6MB:

	package main

	import "fmt"

	func main() {
		fmt.Println("Hello, World!")
	}

When compiled without debugging information the statically
linked binary is now under a megabyte.

{{image "go1.7-binary-size/graph.png"}}

A large production program used for testing this cycle, `jujud`, went from 94MB
to 67MB.

Position-independent binaries are 50% smaller.

In a position-independent executable (PIE), a pointer in a read-only
data section requires a dynamic relocation.
Because the new format for type information replaces pointers by
section offsets, it saves 28 bytes per pointer.

Position-independent executables with debugging information removed
are particularly important to mobile developers, as this is the kind
of program shipped to phones.
Big downloads make for a poor user experience, so the reduction here
is good news.

## Future Work

Several changes to the run-time type information were too late for the
Go 1.7 freeze, but will hopefully make it into 1.8, further shrinking
programs, especially position-independent ones.

These changes are all conservative, reducing binary size without increasing
build time, startup time, overall execution time, or memory usage.
We could take more radical steps to reduce binary size: the
[upx](http://upx.sourceforge.net/) tool for compressing executables
shrinks binaries by another 50% at the cost of increased startup time
and potentially increased memory use.
For extremely small systems (the kind that might live on a keychain)
we could build a version of Go without reflection, though it is
unclear whether such a restricted language would be sufficiently
useful.
For some algorithms in the runtime we could use slower but more
compact implementations when every kilobyte counts.
All of these call for more research in later development cycles.

To the many contributors who helped make Go 1.7 binaries smaller,
thank you!
