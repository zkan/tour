---
title: Go 1.14 is released
date: 2020-02-25
by:
- Alex Rakoczy
summary: Go 1.14 adds production-ready module support, faster defers, better goroutine preemption, and more.
---


Today the Go team is very happy to announce the release of Go 1.14. You can get it from the [download page](https://golang.org/dl).

Some of the highlights include:

  - Module support in the `go` command is now ready for production use. We encourage all users to [migrate to `go` modules for dependency management](https://golang.org/doc/go1.14#introduction).
  - [Embedding interfaces with overlapping method sets](https://golang.org/doc/go1.14#language)
  - [Improved defer performance](https://golang.org/doc/go1.14#runtime)
  - [Goroutines are asynchronously preemptible](https://golang.org/doc/go1.14#runtime)
  - [The page allocator is more efficient](https://golang.org/doc/go1.14#runtime)
  - [Internal timers are more efficient](https://golang.org/doc/go1.14#runtime)

For the complete list of changes and more information about the improvements above, see the [**Go 1.14 release notes**](https://golang.org/doc/go1.14).

We want to thank everyone who contributed to this release by writing code, filing bugs, providing feedback, and/or testing the beta and release candidate.
Your contributions and diligence helped to ensure that Go 1.14 is as stable as possible.
That said, if you notice any problems, please [file an issue](https://golang.org/issue/new).

We hope you enjoy the new release!
