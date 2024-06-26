---
title: The cover story
date: 2013-12-02
by:
- Rob Pike
tags:
- tools
- coverage
- testing
summary: Introducing Go 1.12's code coverage tool.
---

## Introduction

From the beginning of the project, Go was designed with tools in mind.
Those tools include some of the most iconic pieces of Go technology such as
the documentation presentation tool
[godoc](/cmd/godoc),
the code formatting tool
[gofmt](/cmd/gofmt),
and the API rewriter
[gofix](/cmd/fix).
Perhaps most important of all is the
[`go` command](/cmd/go),
the program that automatically installs, builds, and tests Go programs
using nothing more than the source code as the build specification.

The release of Go 1.2 introduces a new tool for test coverage that takes an
unusual approach to the way it generates coverage statistics, an approach
that builds on the technology laid down by godoc and friends.

## Support for tools

First, some background: What does it mean for a
[language to support good tooling](/talks/2012/splash.article#TOC_17.)?
It means that the language makes it easy to write good tools and that its ecosystem
supports the construction of tools of all flavors.

There are a number of properties of Go that make it suitable for tooling.
For starters, Go has a regular syntax that is easy to parse.
The grammar aims to be free of special cases that require complex machinery to analyze.

Where possible, Go uses lexical and syntactic constructs to make semantic properties
easy to understand.
Examples include the use of upper-case letters to define exported names
and the radically simplified scoping rules compared to other languages in the C tradition.

Finally, the standard library comes with production-quality packages to lex and parse Go source code.
They also include, more unusually, a production-quality package to pretty-print Go syntax trees.

These packages in combination form the core of the gofmt tool, but the pretty-printer is worth singling out.
Because it can take an arbitrary Go syntax tree and output standard-format, human-readable, correct
code, it creates the possibility to build tools that transform the parse tree and output modified but
correct and easy-to-read code.

One example is the gofix tool, which automates the
rewriting of code to use new language features or updated libraries.
Gofix let us make fundamental changes to the language and libraries in the
[run-up to Go 1.0](/blog/the-path-to-go-1),
with the confidence that users could just run the tool to update their source to the newest version.

Inside Google, we have used gofix to make sweeping changes in a huge code repository that would be almost
unthinkable in the other languages we use.
There's no need any more to support multiple versions of some API; we can use gofix to update
the entire company in one operation.

It's not just these big tools that these packages enable, of course.
They also make it easy to write more modest programs such as IDE plugins, for instance.
All these items build on each other, making the Go environment
more productive by automating many tasks.

## Test coverage

Test coverage is a term that describes how much of a package's code is exercised by running the package's tests.
If executing the test suite causes 80% of the package's source statements to be run, we say that the test coverage is 80%.

The program that provides test coverage in Go 1.2 is the latest to exploit the tooling support in the Go ecosystem.

The usual way to compute test coverage is to instrument the binary.
For instance, the GNU [gcov](http://gcc.gnu.org/onlinedocs/gcc/Gcov.html) program sets breakpoints at branches
executed by the binary.
As each branch executes, the breakpoint is cleared and the target statements of the branch are marked as 'covered'.

This approach is successful and widely used. An early test coverage tool for Go even worked the same way.
But it has problems.
It is difficult to implement, as analysis of the execution of binaries is challenging.
It also requires a reliable way of tying the execution trace back to the source code, which can also be difficult,
as any user of a source-level debugger can attest.
Problems there include inaccurate debugging information and issues such as in-lined functions complicating
the analysis.
Most important, this approach is very non-portable.
It needs to be done afresh for every architecture, and to some extent for every
operating system since debugging support varies greatly from system to system.

It does work, though, and for instance if you are a user of gccgo, the gcov tool can give you test coverage
information.
However If you're a user of gc, the more commonly used Go compiler suite, until Go 1.2 you were out of luck.

## Test coverage for Go

For the new test coverage tool for Go, we took a different approach that avoids dynamic debugging.
The idea is simple: Rewrite the package's source code before compilation to add instrumentation,
compile and run the modified source, and dump the statistics.
The rewriting is easy to arrange because the `go` command controls the flow
from source to test to execution.

Here's an example. Say we have a simple, one-file package like this:

{{code "cover/pkg.go"}}

and this test:

{{code "cover/pkg_test.go"}}

To get the test coverage for the package,
we run the test with coverage enabled by providing the `-cover` flag to `go` `test`:

	% go test -cover
	PASS
	coverage: 42.9% of statements
	ok  	size	0.026s
	%

Notice that the coverage is 42.9%, which isn't very good.
Before we ask how to raise that number, let's see how that was computed.

When test coverage is enabled, `go` `test` runs the "cover" tool, a separate program included
with the distribution, to rewrite the source code before compilation. Here's what the rewritten
`Size` function looks like:

{{code "cover/pkg.cover" `/func/` `/^}/`}}

Each executable section of the program is annotated with an assignment statement that,
when executed, records that that section ran.
The counter is tied to the original source position of the statements it counts
through a second read-only data structure that is also generated by the cover tool.
When the test run completes, the counters are collected and the percentage is computed
by seeing how many were set.

Although that annotating assignment might look expensive, it compiles to a single "move" instruction.
Its run-time overhead is therefore modest, adding only about 3% when running a typical (more realistic) test.
That makes it reasonable to include test coverage as part of the standard development pipeline.

## Viewing the results

The test coverage for our example was poor.
To discover why, we ask `go` `test` to write a "coverage profile" for us, a file that holds
the collected statistics so we can study them in more detail.
That's easy to do: use the `-coverprofile` flag to specify a file for the output:

	% go test -coverprofile=coverage.out
	PASS
	coverage: 42.9% of statements
	ok  	size	0.030s
	%

(The `-coverprofile` flag automatically sets `-cover` to enable coverage analysis.)
The test runs just as before, but the results are saved in a file.
To study them, we run the test coverage tool ourselves, without `go` `test`.
As a start, we can ask for the coverage to be broken down by function,
although that's not going to illuminate much in this case since there's
only one function:

	% go tool cover -func=coverage.out
	size.go:	Size          42.9%
	total:      (statements)  42.9%
	%

A much more interesting way to see the data is to get an HTML presentation
of the source code decorated with coverage information.
This display is invoked by the `-html` flag:

	$ go tool cover -html=coverage.out

When this command is run, a browser window pops up, showing the covered (green),
uncovered (red), and uninstrumented (grey) source.
Here's a screen dump:

{{image "cover/set.png"}}

With this presentation, it's obvious what's wrong: we neglected to test several
of the cases!
And we can see exactly which ones they are, which makes it easy to
improve our test coverage.

## Heat maps

A big advantage of this source-level approach to test coverage is that it's
easy to instrument the code in different ways.
For instance, we can ask not only whether a statement has been executed,
but how many times.

The `go` `test` command accepts a `-covermode` flag to set the coverage mode
to one of three settings:

  - set:    did each statement run?
  - count:  how many times did each statement run?
  - atomic: like count, but counts precisely in parallel programs

The default is 'set', which we've already seen.
The `atomic` setting is needed only when accurate counts are required
when running parallel algorithms. It uses atomic operations from the
[sync/atomic](/pkg/sync/atomic/) package,
which can be quite expensive.
For most purposes, though, the `count` mode works fine and, like
the default `set` mode, is very cheap.

Let's try counting statement execution for a standard package, the `fmt` formatting package.
We run the test and write out a coverage profile so we can present the information
nicely afterwards.

	% go test -covermode=count -coverprofile=count.out fmt
	ok  	fmt	0.056s	coverage: 91.7% of statements
	%

That's a much better test coverage ratio than for our previous example.
(The coverage ratio is not affected by the coverage mode.)
We can display the function breakdown:

	% go tool cover -func=count.out
	fmt/format.go: init              100.0%
	fmt/format.go: clearflags        100.0%
	fmt/format.go: init              100.0%
	fmt/format.go: computePadding     84.6%
	fmt/format.go: writePadding      100.0%
	fmt/format.go: pad               100.0%
	...
	fmt/scan.go:   advance            96.2%
	fmt/scan.go:   doScanf            96.8%
	total:         (statements)       91.7%

The big payoff happens in the HTML output:

	% go tool cover -html=count.out

Here's what the `pad` function looks like in that presentation:

{{image "cover/count.png"}}

Notice how the intensity of the green changes. Brighter-green
statements have higher execution counts; less saturated greens
represent lower execution counts.
You can even hover the mouse over the statements to see the
actual counts pop up in a tool tip.
At the time of writing, the counts come out like this
(we've moved the counts from the tool tips to beginning-of-line
markers to make them easier to show):

	2933    if !f.widPresent || f.wid == 0 {
	2985        f.buf.Write(b)
	2985        return
	2985    }
	  56    padding, left, right := f.computePadding(len(b))
	  56    if left > 0 {
	  37        f.writePadding(left, padding)
	  37    }
	  56    f.buf.Write(b)
	  56    if right > 0 {
	  13        f.writePadding(right, padding)
	  13    }

That's a lot of information about the execution of the function,
information that might be useful in profiling.

## Basic blocks

You might have noticed that the counts in the previous example
were not what you expected on the lines with closing braces.
That's because, as always, test coverage is an inexact science.

What's going on here is worth explaining, though. We'd like the
coverage annotations to be demarcated by branches in the program,
the way they are when the binary is instrumented in the traditional
method.
It's hard to do that by rewriting the source, though, since
the branches don't appear explicitly in the source.

What the coverage annotation does is instrument blocks, which
are typically bounded by brace brackets.
Getting this right in general is very hard.
A consequence of the algorithm used is that the closing
brace looks like it belongs to the block it closes, while the
opening brace looks like it belongs outside the block.
A more interesting consequence is that in an expression like

	f() && g()

there is no attempt to separately instrument the calls to `f` and `g`, Regardless of
the facts it will always look like they both ran the same
number of times, the number of times `f` ran.

To be fair, even `gcov` has trouble here. That tool gets the
instrumentation right but the presentation is line-based and
can therefore miss some nuances.

## The big picture

That's the story about test coverage in Go 1.2.
A new tool with an interesting implementation enables not only
test coverage statistics, but easy-to-interpret presentations
of them and even the possibility to extract profiling information.

Testing is an important part of software development and test
coverage a simple way to add discipline to your testing strategy.
Go forth, test, and cover.
