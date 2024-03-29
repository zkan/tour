---
title: Using Subtests and Sub-benchmarks
date: 2016-10-03
by:
- Marcel van Lohuizen
tags:
- testing
- hierarchy
- table-driven
- subtests
- sub-benchmarks
summary: How to use Go 1.7's new subtests and sub-benchmarks.
---

## Introduction

In Go 1.7, the `testing` package introduces a Run method on the
[`T`](https://golang.org/pkg/testing/#T.Run) and
[`B`](https://golang.org/pkg/testing/#B.Run) types
that allows for the creation of subtests and sub-benchmarks.
The introduction of subtests and sub-benchmarks enables better handling of
failures, fine-grained control of which tests to run from the command line,
control of parallelism, and often results in simpler and more maintainable code.

## Table-driven tests basics

Before digging into the details, let's first discuss a common
way of writing tests in Go.
A series of related checks can be implemented by looping over a slice of test
cases:

	func TestTime(t *testing.T) {
		testCases := []struct {
			gmt  string
			loc  string
			want string
		}{
			{"12:31", "Europe/Zuri", "13:31"},     // incorrect location name
			{"12:31", "America/New_York", "7:31"}, // should be 07:31
			{"08:08", "Australia/Sydney", "18:08"},
		}
		for _, tc := range testCases {
			loc, err := time.LoadLocation(tc.loc)
			if err != nil {
				t.Fatalf("could not load location %q", tc.loc)
			}
			gmt, _ := time.Parse("15:04", tc.gmt)
			if got := gmt.In(loc).Format("15:04"); got != tc.want {
				t.Errorf("In(%s, %s) = %s; want %s", tc.gmt, tc.loc, got, tc.want)
			}
		}
	}

This approach, commonly referred to as table-driven tests, reduces the amount
of repetitive code compared to repeating the same code for each test
and makes it straightforward to add more test cases.

## Table-driven benchmarks

Before Go 1.7 it was not possible to use the same table-driven approach for
benchmarks.
A benchmark tests the performance of an entire function, so iterating over
benchmarks would just measure all of them as a single benchmark.

A common workaround was to define separate top-level benchmarks
that each call a common function with different parameters.
For instance, before 1.7 the `strconv` package's benchmarks for `AppendFloat`
looked something like this:

{{raw `
	func benchmarkAppendFloat(b *testing.B, f float64, fmt byte, prec, bitSize int) {
		dst := make([]byte, 30)
		b.ResetTimer() // Overkill here, but for illustrative purposes.
		for i := 0; i < b.N; i++ {
			AppendFloat(dst[:0], f, fmt, prec, bitSize)
		}
	}

	func BenchmarkAppendFloatDecimal(b *testing.B) { benchmarkAppendFloat(b, 33909, 'g', -1, 64) }
	func BenchmarkAppendFloat(b *testing.B)        { benchmarkAppendFloat(b, 339.7784, 'g', -1, 64) }
	func BenchmarkAppendFloatExp(b *testing.B)     { benchmarkAppendFloat(b, -5.09e75, 'g', -1, 64) }
	func BenchmarkAppendFloatNegExp(b *testing.B)  { benchmarkAppendFloat(b, -5.11e-95, 'g', -1, 64) }
	func BenchmarkAppendFloatBig(b *testing.B)     { benchmarkAppendFloat(b, 123456789123456789123456789, 'g', -1, 64) }
	...
`}}

Using the `Run` method available in Go 1.7, the same set of benchmarks is now
expressed as a single top-level benchmark:

{{raw `
	func BenchmarkAppendFloat(b *testing.B) {
		benchmarks := []struct{
			name    string
			float   float64
			fmt     byte
			prec    int
			bitSize int
		}{
			{"Decimal", 33909, 'g', -1, 64},
			{"Float", 339.7784, 'g', -1, 64},
			{"Exp", -5.09e75, 'g', -1, 64},
			{"NegExp", -5.11e-95, 'g', -1, 64},
			{"Big", 123456789123456789123456789, 'g', -1, 64},
			...
		}
		dst := make([]byte, 30)
		for _, bm := range benchmarks {
			b.Run(bm.name, func(b *testing.B) {
				for i := 0; i < b.N; i++ {
					AppendFloat(dst[:0], bm.float, bm.fmt, bm.prec, bm.bitSize)
				}
			})
		}
	}
`}}

Each invocation of the `Run` method creates a separate benchmark.
An enclosing benchmark function that calls a `Run` method is only run once and
is not measured.

The new code has more lines of code, but is more maintainable, more readable,
and consistent with the table-driven approach commonly used for testing.
Moreover, common setup code is now shared between runs while eliminating the
need to reset the timer.

## Table-driven tests using subtests

Go 1.7 also introduces a `Run` method for creating subtests.
This test is a rewritten version of our earlier example using subtests:

	func TestTime(t *testing.T) {
		testCases := []struct {
			gmt  string
			loc  string
			want string
		}{
			{"12:31", "Europe/Zuri", "13:31"},
			{"12:31", "America/New_York", "7:31"},
			{"08:08", "Australia/Sydney", "18:08"},
		}
		for _, tc := range testCases {
			t.Run(fmt.Sprintf("%s in %s", tc.gmt, tc.loc), func(t *testing.T) {
				loc, err := time.LoadLocation(tc.loc)
				if err != nil {
					t.Fatal("could not load location")
				}
				gmt, _ := time.Parse("15:04", tc.gmt)
				if got := gmt.In(loc).Format("15:04"); got != tc.want {
					t.Errorf("got %s; want %s", got, tc.want)
				}
			})
		}
	}

The first thing to note is the difference in output from the two implementations.
The original implementation prints:

	--- FAIL: TestTime (0.00s)
		time_test.go:62: could not load location "Europe/Zuri"

Even though there are two errors, execution of the test halts on the call to
`Fatalf` and the second test never runs.

The implementation using `Run` prints both:

	--- FAIL: TestTime (0.00s)
	    --- FAIL: TestTime/12:31_in_Europe/Zuri (0.00s)
	    	time_test.go:84: could not load location
	    --- FAIL: TestTime/12:31_in_America/New_York (0.00s)
	    	time_test.go:88: got 07:31; want 7:31

`Fatal` and its siblings causes a subtest to be skipped but not its parent or
subsequent subtests.

Another thing to note is the shorter error messages in the new implementation.
Since the subtest name uniquely identifies the subtest there is no need to
identify the test again within the error messages.

There are several other benefits to using subtests or sub-benchmarks,
as clarified by the following sections.

## Running specific tests or benchmarks

Both subtests and sub-benchmarks can be singled out on the command line using
the [`-run` or `-bench` flag](https://golang.org/cmd/go/#hdr-Description_of_testing_flags).
Both flags take a slash-separated list of regular expressions that match the
corresponding parts of the full name of the subtest or sub-benchmark.

The full name of a subtest or sub-benchmark is a slash-separated list of
its name and the names of all of its parents, starting with the top-level.
The name is the corresponding function name for top-level tests and benchmarks,
and the first argument to `Run` otherwise.
To avoid display and parsing issues, a name is sanitized by replacing spaces
with underscores and escaping non-printable characters.
The same sanitizing is applied to the regular expressions passed to
the `-run` or `-bench` flags.

A few examples:

Run tests that use a timezone in Europe:

	$ go test -run=TestTime/"in Europe"
	--- FAIL: TestTime (0.00s)
	    --- FAIL: TestTime/12:31_in_Europe/Zuri (0.00s)
	    	time_test.go:85: could not load location

Run only tests for times after noon:

	$ go test -run=Time/12:[0-9] -v
	=== RUN   TestTime
	=== RUN   TestTime/12:31_in_Europe/Zuri
	=== RUN   TestTime/12:31_in_America/New_York
	--- FAIL: TestTime (0.00s)
	    --- FAIL: TestTime/12:31_in_Europe/Zuri (0.00s)
	    	time_test.go:85: could not load location
	    --- FAIL: TestTime/12:31_in_America/New_York (0.00s)
	    	time_test.go:89: got 07:31; want 7:31

Perhaps a bit surprising, using `-run=TestTime/New_York` won't match any tests.
This is because the slash present in the location names is treated as
a separator as well.
Instead use:

	$ go test -run=Time//New_York
	--- FAIL: TestTime (0.00s)
	    --- FAIL: TestTime/12:31_in_America/New_York (0.00s)
	    	time_test.go:88: got 07:31; want 7:31

Note the `//` in the string passed to `-run`.
The `/` in time zone name `America/New_York` is handled as if it were
a separator resulting from a subtest.
The first regular expression of the pattern (`TestTime`) matches the top-level
test.
The second regular expression (the empty string) matches anything, in this case
the time and the continent part of the location.
The third regular expression (`New_York`) matches the city part of the location.

Treating slashes in names as separators allows the user to refactor
hierarchies of tests without the need to change the naming.
It also simplifies the escaping rules.
The user should escape slashes in names, for instance by replacing them with
backslashes, if this poses a problem.

A unique sequence number is appended to test names that are not unique.
So one could just pass an empty string to `Run`
if there is no obvious naming scheme for subtests and the subtests
can easily be identified by their sequence number.

## Setup and Tear-down

Subtests and sub-benchmarks can be used to manage common setup and tear-down code:

{{raw `
	func TestFoo(t *testing.T) {
		// <setup code>
		t.Run("A=1", func(t *testing.T) { ... })
		t.Run("A=2", func(t *testing.T) { ... })
		t.Run("B=1", func(t *testing.T) {
			if !test(foo{B:1}) {
				t.Fail()
			}
		})
		// <tear-down code>
	}
`}}

The setup and tear-down code will run if any of the enclosed subtests are run
and will run at most once.
This applies even if any of the subtests calls `Skip`, `Fail`, or `Fatal`.

## Control of Parallelism

Subtests allow fine-grained control over parallelism.
To understand how to use subtests in the way
it is important to understand the semantics of parallel tests.

Each test is associated with a test function.
A test is called a parallel test if its test function calls the Parallel
method on its instance of `testing.T`.
A parallel test never runs concurrently with a sequential test and its execution
is suspended until its calling test function, that of the parent test,
has returned.
The `-parallel` flag defines the maximum number of parallel tests that can run
in parallel.

A test blocks until its test function returns and all of its subtests
have completed.
This means that the parallel tests that are run by a sequential test will
complete before any other consecutive sequential test is run.

This behavior is identical for tests created by `Run` and top-level tests.
In fact, under the hood top-level tests are implemented as subtests of
a hidden master test.

### Run a group of tests in parallel

The above semantics allows for running a group of tests in parallel with
each other but not with other parallel tests:

	func TestGroupedParallel(t *testing.T) {
		for _, tc := range testCases {
			tc := tc // capture range variable
			t.Run(tc.Name, func(t *testing.T) {
				t.Parallel()
				if got := foo(tc.in); got != tc.out {
					t.Errorf("got %v; want %v", got, tc.out)
				}
				...
			})
		}
	}

The outer test will not complete until all parallel tests started by `Run`
have completed.
As a result, no other parallel tests can run in parallel to these parallel tests.

Note that we need to capture the range variable to ensure that `tc` gets bound to
the correct instance.

### Cleaning up after a group of parallel tests

In the previous example we used the semantics to wait on a group of parallel
tests to complete before commencing other tests.
The same technique can be used to clean up after a group of parallel tests
that share common resources:

{{raw `
	func TestTeardownParallel(t *testing.T) {
		// <setup code>
		// This Run will not return until its parallel subtests complete.
		t.Run("group", func(t *testing.T) {
			t.Run("Test1", parallelTest1)
			t.Run("Test2", parallelTest2)
			t.Run("Test3", parallelTest3)
		})
		// <tear-down code>
	}
`}}

The behavior of waiting on a group of parallel tests is identical to that
of the previous example.

## Conclusion

Go 1.7's addition of subtests and sub-benchmarks allows you to write structured
tests and benchmarks in a natural way that blends nicely into the existing
tools.
One way to think about this is that earlier versions of the testing package had
a 1-level hierarchy: the package-level test was structured as a set of
individual tests and benchmarks.
Now that structure has been extended to those individual tests and benchmarks,
recursively.
In fact, in the implementation, the top-level tests and benchmarks are tracked
as if they were subtests and sub-benchmarks of an implicit master test and
benchmark: the treatment really is the same at all levels.

The ability for tests to define this structure enables fine-grained execution of
specific test cases, shared setup and teardown, and better control over test
parallelism.
We are excited to see what other uses people find. Enjoy.
