---
title: Command PATH security in Go
date: 2021-01-19
by:
- Russ Cox
summary: How to decide if your programs are vulnerable to PATH problems, and what to do about it.
---


Today’s [Go security release](https://golang.org/s/go-security-release-jan-2021)
fixes an issue involving PATH lookups in untrusted directories
that can lead to remote execution during the `go` `get` command.
We expect people to have questions about what exactly this means
and whether they might have issues in their own programs.
This post details the bug, the fixes we have applied,
how to decide whether your own programs are vulnerable to similar problems,
and what you can do if they are.

## Go command & remote execution

One of the design goals for the `go` command is that most commands – including
`go` `build`, `go` `doc`, `go` `get`, `go` `install`, and `go` `list` – do not run
arbitrary code downloaded from the internet.
There are a few obvious exceptions:
clearly `go` `run`, `go` `test`, and `go` `generate` _do_ run arbitrary code – that's their job.
But the others must not, for a variety of reasons including reproducible builds and security.
So when `go` `get` can be tricked into executing arbitrary code, we consider that a security bug.

If `go` `get` must not run arbitrary code, then unfortunately that means
all the programs it invokes, such as compilers and version control systems, are also inside the security perimeter.
For example, we've had issues in the past in which clever use of obscure compiler features
or remote execution bugs in version control systems became remote execution bugs in Go.
(On that note, Go 1.16 aims to improve the situation by introducing a GOVCS setting
that allows configuration of exactly which version control systems are allowed and when.)

Today's bug, however, was entirely our fault, not a bug or obscure feature of `gcc` or `git`.
The bug involves how Go and other programs find other executables,
so we need to spend a little time looking at that before we can get to the details.

## Commands and PATHs and Go

All operating systems have a concept of an executable path
(`$PATH` on Unix, `%PATH%` on Windows; for simplicity, we'll just use the term PATH),
which is a list of directories.
When you type a command into a shell prompt,
the shell looks in each of the listed directories,
in turn, for an executable with the name you typed.
It runs the first one it finds, or it prints a message like “command not found.”

On Unix, this idea first appeared in Seventh Edition Unix's Bourne shell (1979). The manual explained:

> The shell parameter `$PATH` defines the search path for the directory containing the command.
> Each alternative directory name is separated by a colon (`:`).
> The default path is `:/bin:/usr/bin`.
> If the command name contains a / then the search path is not used.
> Otherwise, each directory in the path is searched for an executable file.

Note the default: the current directory (denoted here by an empty string,
but let's call it “dot”)
is listed ahead of `/bin` and `/usr/bin`.
MS-DOS and then Windows chose to hard-code that behavior:
on those systems, dot is always searched first,
automatically, before considering any directories listed in `%PATH%`.

As Grampp and Morris pointed out in their
classic paper “[UNIX Operating System Security](https://people.engr.ncsu.edu/gjin2/Classes/246/Spring2019/Security.pdf)” (1984),
placing dot ahead of system directories in the PATH
means that if you `cd` into a directory and run `ls`,
you might get a malicious copy from that directory
instead of the system utility.
And if you can trick a system administrator to run `ls` in your home directory
while logged in as `root`, then you can run any code you want.
Because of this problem and others like it,
essentially all modern Unix distributions set a new user's default PATH
to exclude dot.
But Windows systems continue to search dot first, no matter what PATH says.

For example, when you type the command

	go version

on a typically-configured Unix,
the shell runs a `go` executable from a system directory in your PATH.
But when you type that command on Windows,
`cmd.exe` checks dot first.
If `.\go.exe` (or `.\go.bat` or many other choices) exists,
`cmd.exe` runs that executable, not one from your PATH.

For Go, PATH searches are handled by [`exec.LookPath`](https://pkg.go.dev/os/exec#LookPath),
called automatically by
[`exec.Command`](https://pkg.go.dev/os/exec#Command).
And to fit well into the host system, Go's `exec.LookPath`
implements the Unix rules on Unix and the Windows rules on Windows.
For example, this command

	out, err := exec.Command("go", "version").CombinedOutput()

behaves the same as typing `go` `version` into the operating system shell.
On Windows, it runs `.\go.exe` when that exists.

(It is worth noting that Windows PowerShell changed this behavior,
dropping the implicit search of dot, but `cmd.exe` and the
Windows C library [`SearchPath function`](https://docs.microsoft.com/en-us/windows/win32/api/processenv/nf-processenv-searchpatha)
continue to behave as they always have.
Go continues to match `cmd.exe`.)

## The Bug

When `go` `get` downloads and builds a package that contains
`import` `"C"`, it runs a program called `cgo` to prepare the Go
equivalent of the relevant C code.
The `go` command runs `cgo` in the directory containing the package sources.
Once `cgo` has generated its Go output files,
the `go` command itself invokes the Go compiler
on the generated Go files
and the host C compiler (`gcc` or `clang`)
to build any C sources included with the package.
All this works well.
But where does the `go` command find the host C compiler?
It looks in the PATH, of course. Luckily, while it runs the C compiler
in the package source directory, it does the PATH lookup
from the original directory where the `go` command was invoked:

	cmd := exec.Command("gcc", "file.c")
	cmd.Dir = "badpkg"
	cmd.Run()

So even if `badpkg\gcc.exe` exists on a Windows system,
this code snippet will not find it.
The lookup that happens in `exec.Command` does not know
about the `badpkg` directory.

The `go` command uses similar code to invoke `cgo`,
and in that case there's not even a path lookup,
because `cgo` always comes from GOROOT:

	cmd := exec.Command(GOROOT+"/pkg/tool/"+GOOS_GOARCH+"/cgo", "file.go")
	cmd.Dir = "badpkg"
	cmd.Run()

This is even safer than the previous snippet:
there's no chance of running any bad `cgo.exe` that may exist.

But it turns out that cgo itself also invokes the host C compiler,
on some temporary files it creates, meaning it executes this code itself:

	// running in cgo in badpkg dir
	cmd := exec.Command("gcc", "tmpfile.c")
	cmd.Run()

Now, because cgo itself is running in `badpkg`,
not in the directory where the `go` command was run,
it will run `badpkg\gcc.exe` if that file exists,
instead of finding the system `gcc`.

So an attacker can create a malicious package that uses cgo and
includes a `gcc.exe`, and then any Windows user
that runs `go` `get` to download and build the attacker's package
will run the attacker-supplied `gcc.exe` in preference to any
`gcc` in the system path.

Unix systems avoid the problem first because dot is typically not
in the PATH and second because module unpacking does not
set execute bits on the files it writes.
But Unix users who have dot ahead of system directories
in their PATH and are using GOPATH mode would be as susceptible
as Windows users.
(If that describes you, today is a good day to remove dot from your path
and to start using Go modules.)

(Thanks to [RyotaK](https://twitter.com/ryotkak) for [reporting this issue](https://golang.org/security) to us.)

## The Fixes

It's obviously unacceptable for the `go` `get` command to download
and run a malicious `gcc.exe`.
But what's the actual mistake that allows that?
And then what's the fix?

One possible answer is that the mistake is that `cgo` does the search for the host C compiler
in the untrusted source directory instead of in the directory where the `go` command
was invoked.
If that's the mistake,
then the fix is to change the `go` command to pass `cgo` the full path to the
host C compiler, so that `cgo` need not do a PATH lookup in
to the untrusted directory.

Another possible answer is that the mistake is to look in dot
during PATH lookups, whether happens automatically on Windows
or because of an explicit PATH entry on a Unix system.
A user may want to look in dot to find a command they typed
in a console or shell window,
but it's unlikely they also want to look there to find a subprocess of a subprocess
of a typed command.
If that's the mistake,
then the fix is to change the `cgo` command not to look in dot during a PATH lookup.

We decided both were mistakes, so we applied both fixes.
The `go` command now passes the full host C compiler path to `cgo`.
On top of that, `cgo`, `go`, and every other command in the Go distribution
now use a variant of the `os/exec` package that reports an error if it would
have previously used an executable from dot.
The packages `go/build` and `go/import` use the same policy for
their invocation of the `go` command and other tools.
This should shut the door on any similar security problems that may be lurking.

Out of an abundance of caution, we also made a similar fix in
commands like `goimports` and `gopls`,
as well as the libraries
`golang.org/x/tools/go/analysis`
and
`golang.org/x/tools/go/packages`,
which invoke the `go` command as a subprocess.
If you run these programs in untrusted directories –
for example, if you `git` `checkout` untrusted repositories
and `cd` into them and then run programs like these,
and you use Windows or use Unix with dot in your PATH –
then you should update your copies of these commands too.
If the only untrusted directories on your computer
are the ones in the module cache managed by `go` `get`,
then you only need the new Go release.

After updating to the new Go release, you can update to the latest `gopls` by using:

	GO111MODULE=on \
	go get golang.org/x/tools/gopls@v0.6.4

and you can update to the latest `goimports` or other tools by using:

	GO111MODULE=on \
	go get golang.org/x/tools/cmd/goimports@v0.1.0

You can update programs that depend on `golang.org/x/tools/go/packages`,
even before their authors do,
by adding an explicit upgrade of the dependency during `go` `get`:

	GO111MODULE=on \
	go get example.com/cmd/thecmd golang.org/x/tools@v0.1.0

For programs that use `go/build`, it is sufficient for you to recompile them
using the updated Go release.

Again, you only need to update these other programs if you
are a Windows user or a Unix user with dot in the PATH
_and_ you run these programs in source directories you do not trust
that may contain malicious programs.

## Are your own programs affected?

If you use `exec.LookPath` or `exec.Command` in your own programs,
you only need to be concerned if you (or your users) run your program
in a directory with untrusted contents.
If so, then a subprocess could be started using an executable
from dot instead of from a system directory.
(Again, using an executable from dot happens always on Windows
and only with uncommon PATH settings on Unix.)

If you are concerned, then we've published the more restricted variant
of `os/exec` as [`golang.org/x/sys/execabs`](https://pkg.go.dev/golang.org/x/sys/execabs).
You can use it in your program by simply replacing

	import "os/exec"

with

	import exec "golang.org/x/sys/execabs"

and recompiling.

## Securing os/exec by default

We have been discussing on
[golang.org/issue/38736](https://golang.org/issue/38736)
whether the Windows behavior of always preferring the current directory
in PATH lookups (during `exec.Command` and `exec.LookPath`)
should be changed.
The argument in favor of the change is that it closes the kinds of
security problems discussed in this blog post.
A supporting argument is that although the Windows `SearchPath` API
and `cmd.exe` still always search the current directory,
PowerShell, the successor to `cmd.exe`, does not,
an apparent recognition that the original behavior was a mistake.
The argument against the change is that it could break existing Windows
programs that intend to find programs in the current directory.
We don’t know how many such programs exist,
but they would get unexplained failures if the PATH lookups
started skipping the current directory entirely.

The approach we have taken in `golang.org/x/sys/execabs` may
be a reasonable middle ground.
It finds the result of the old PATH lookup and then returns a
clear error rather than use a result from the current directory.
The error returned from `exec.Command("prog")` when `prog.exe` exists looks like:

	prog resolves to executable in current directory (.\prog.exe)

For programs that do change behavior, this error should make very clear what has happened.
Programs that intend to run a program from the current directory can use
`exec.Command("./prog")` instead (that syntax works on all systems, even Windows).

We have filed this idea as a new proposal, [golang.org/issue/43724](https://golang.org/issue/43724).
