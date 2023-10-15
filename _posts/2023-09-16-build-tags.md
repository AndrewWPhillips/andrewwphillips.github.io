---
title: "Build Tags"
last_modified_at: 2023-09-23T11:00:59+10:00
excerpt: "The other \"directive\" that I didn't cover last week is `//go:build`.  This time I look at how and why you would use build tags and the new syntax."
toc: true
toc_sticky: true
categories: [language,directives,build,tags]
tags: [pragma,directive,build tags]
header:
  overlay_image: "/assets/images/build-tags.jpg"
  overlay_filter: 0.3
  caption: "Image: [Noun Project](https://thenounproject.com/)"
permalink: /blog/build-tags.html
---

Build constraints or _tags_ allow you to control which source files are used to create your program.  This is how Go achieves what is called **conditional compilation** in other languages.

Conditional compilation is less important than it has been in the past due to advances in hardware.  Nowadays, you might use a run-time flag to handle what may have been traditionally handled at compile-time.  I explain this fully in **Background** below.  However, conditional compilation still has its uses, such as for targeting different operating systems.

Unlike other languages, Go only gives you very coarse control of what is built -- i.e. you can only include/exclude at the file level, rather than individual lines of code.  This may seem crude, but it actually makes it far easier to understand what code is being built.  Plus other features of Go (e.g. empty functions essentially disappear, due to inlining), make it very effective.

# Background

<details markdown="1">
<summary>General</summary>
<br/>

**C**

Conditional compilation has always been important in C as it allows you to save memory, by not including unneeded code.  It also means you don't have the performance cost of testing a flag at run-time to decide what to do.  In modern hardware these considerations are less important.

A common use in C, is to create separate debug and release versions.  The debug version has all sorts of assertions and checks that the code is not straying into the realms of undefined behaviour (as C programs are wont to do).  The release versions eschews these checks in the name of performance.

When C's approach to conditional compilation was invented it was done on a line-by-line basis using the pre-processor.  Perhaps one reason it was done this way, rather than the approach Go uses, is that C compilers (for at least a decade) did not have the benefit of inline functions.  More likely, is that conditional compilation on a file-by-file, instead of line-by-line, basis was seen as too inflexible.

Even in simple cases, the intermingling of conditional compilation with the actual code can make it difficult to understand.  More complex cases using multiple nested conditions etc., can **lead to code that's unfathomable**.

---
</details>
<!--****************************-->
<details markdown="1">
<summary>Go</summary>
<br/>

After using C for decades, I initially found Go's approach inflexible.  It's true that `//go:build` is a simple mechanism.  It does not even involve the compiler, just the `build` tool.

Once I started using it, and understood its nuances (discussed below) I found it quite effective and, more importantly, it makes the whole process easier to understand.  All the "conditions" are clearly documented at the top of their respective .go file.

Moreover, by requiring use of a separate file, it makes for smaller source files (always a good thing), and even assists **separation of concerns** (the fashionable name for decoupling).

**New Syntax**

Go 1.17 introduced new syntax for build tags, which made things simpler and avoided some pitfalls. See [Dave Cheney's Blog](https://dave.cheney.net/tag/build-tags) if you need to use the old syntax for some reason (not recommended).

---
</details>
<!--****************************-->
<br/>

# Syntax

Build constraints (not to be confused with _type_ constraints used with generics) are probably the most used form of compiler directive.  See my [previous post](https://andrewwphillips.github.io/blog/directive.html) about being careful with the syntax of directives, as it's easy for your build tag to be treated as a comment and ignored.

If you want to check if your .go file is included in the build it is easy with `go list`.

```shell
{% raw %}
$ go list -f '{{.GoFiles}}'
{% endraw %}
[main.go enhancement.go eg_linux.go]
```

This will reflect the settings of `GOARCH` and `GOOS` (see below) if you are cross-compiling.

## Syntax Changes

**Warning** How build tags are used was changed in Go 1.17 to use the `//go:build` directive.  Unfortunately, most blogs gives examples using the old `//+build` format.
{: .notice--warning}

**Note** Whether you use the old or the new format, `gofmt` will add another line to your code so the file will end up with both. However,you should **get used to the new format** as the old format may disappear and the new format avoids some known pitfalls, has syntax like other directives, and gives better control (see [Build Tag Expressions](#build-tag-expressions) below).

## Build Tag Expressions

One of the nice things about the new build tags syntax is it fully supports **boolean expressions**, including brackets.  The old (`//+build`) tags system supported ! (**not** operator) but **and** and **or** were done using commas and spaces (or separate directives).  Plus there were other pitfalls - see the original [Proposal by Russ Cox](https://go.googlesource.com/proposal/+/master/design/draft-gobuild.md) for a full explanation.

## Predefined Tags

The build system defines many tags to identify the operating system and architecture, to be used for platform specific code, as discussed later.  The list of tags can vary between releases.  See the [source code](https://go.dev/src/go/build/syslist.go).

There are also these predefined tags:
* `unix` is used as a simple alternative to listing all Unix-like operating systems
* `gc` or `gccgo` depending on which compiler you are using (there doesn't seem to be one for TinyGo)
* `cgo` is only defined if CGO is enabled (using `CGO_ENABLED` environment variable)
* tags for all versions up to the compiler release (eg `go1.22`)

## Setting Custom Tags

You can define your own tags when using the build, run, etc. tools, by using the `-tags` command line option.  This example set two tags:

```shell
$ go build -tags postgres,unenhanced ...
```

Finally, here is a (silly) example, that only adds a file to the build if the `unenhanced` tag is **not** set, and we are not building for Windows or Android.

```go
//go:build !unenhanced && !(windows || android)
package example
```

# Why use Build Tags?

**Do you really need to use build tags**?  First, consider using a run-time flag instead of conditional compilation.  You'll need to be able to configure it somehow such as a command line flag, or as part of a configuration file.<br/><br/>Using a run-time flag simplifies the build and installation process since you don't need alternative versions of the executable.
{: .notice--info}

Here are a few places where you might/must use it.

1. use a facility only supported on specific platform(s)
2. choose between alternative implementations
3. optional features, such as paid-only
4. use a capability of the latest Go release if available
5. prevent use of an old Go release

Next, we'll look at these in detail...

# Targeting OS/ARCH

You may already be familiar with the **GOOS** and **GOARCH** environment variables which allow you to cross-compile to any supported target platform.  There are predefined build tags for every target platform (operating system and architecture) that the compiler supports. These tags are used extensively in the standard library to handle OS-specific things (file handling, etc).

You can see a list of every supported combination using:

```shell
$ go tool dist list
```

There are further predefined build tags.  For example, you can use `unix` if your code targets any Unix or Unix-like operating system such as Linux.

These tags are used to tailor code for specific environment(s).  For example, you may have a feature that only works on Linux.

A less common need is to target specific processors (architectures).  Generally, this sort of stuff is handled by the standard library, but, as an example, you might be implementing a new communications protocol and need to allow for the order of bits of an integer (so called endianness) which depends on the architecture.

A common use (as in the Go standard library) is to use alternative implementations to handle differences between operating systems.  In other words, you build separate executables for different targets by cross-compiling for different operating systems and/or architectures by setting the **GOOS** and **GOARCH** environment variables.

You can combine tags, for example to target one specific operating system and architecture like this:

```go
//go:build windows && amd64
package main
...
```

Though this example would be better handled using a `..._windows_amd64.go` file name suffix ...

## File Name Suffixes

When you have a simple requirement for targeting a single OS, architecture or OS/architecture combination, a better alternative is to use file name suffixes. In this case you just add the name of the tag to the end of the file name (before the .go extension), preceded by an underscore. For example, if a file is only to be included for an executable targeting Linux just use a name like `filename_linux.go`.

There is no need to add the `//go:build` directive.  This is faster than using the directive since the `go build` command can simply inspect the file name and does not even need to open the file.

There are 3 ways this can be specified:
* specific operating system
* specific architecture
* specific operating system and architecture

For example, if you only want a source file to be used for 386 architecture on Linux then you would use a file name like `eg_linux_386.go`.  Note that you must have specify the operating system first - `eg_386_linux.go` would not work.

If you need something more complicated you _must_ use `//go:build`.  For example, the following would add the code to every (valid) combination of OS and architecture - ie, 5 different combinations - since darwin/mips64 is (currently) not a valid combination.

```go
//go:build (linux || darwin) && (amd64 || arm64 || mips64)
```

# Alternative Implementations

Imagine you have developed an open-source application that requires a database, but different users have different requirements and preferences of the database used.  You can create one (or more) source file(s) for each alternative implementation using a build tag at the top of the file(s) to indicate which database is used.

```go
//go:build postgres
package myPackage

func dbSave(p *dbRecord) error {
	...
}
```

```go
//go:build cockroach
package myPackage

func dbSave(p *dbRecord) error {
	...
}
```
Then to select the desired database code just use the `-tags` command line option to Go's build, etc. commands.

```shell
$ go build -tags cockroach ...
```

## Missing Tags

What happens if you don't specify any tag?  The compiler will not know which implementation you want.  You will get a build error saying something like "undefined `dbSave`".

Of course, if you specify more than one mutually exclusive tag:

```shell
$ go build -tags postgres,cockroach ...
```

the build will also fail with an error like  "`dbSave` redeclared in this package".

# Optional Features

Another use of build tags, is to enable a feature.  For example, you may have developed some software with advanced features that are only for paid customers.  (Obviously not an open-source project in this case.)

Having a single executable, and a run-time switch, risks that the paid features could be somehow accidentally or intentionally turned on.  Only giving paying customers the enhanced executable offers some protection against this.

In this case you need separate source file(s) that use a build tag.

```go
//go:build paid
package main

func PaidAction() {
	...
}

func PaidResult() (float64, error) {
	...
}

func IsPaid() bool { return true}
```

Then you just build with the `paid` tag to get the fully-featured version.

```sh
$ go build -tags paid ...
```

Of course, when you build without the flag you will get a build error unless you implement dummy versions of any required functions.

```go
//go:build !paid
package main

func PaidAction() {
	// nothing here - inlined into nothing
}

func PaidResult() (float64, error) {
	return 0, errors.New("Not Implemented in free version")
}

func IsPaid() bool { return false }
```

# Condition on New Go Release

Imagine you have created an open-source package.  A recent enhancement that you added requires a new feature of the Go compiler, like generics.  But you have also added other features and so you want users of your package to use the latest/greatest version, without forcing them to upgrade to the new compiler release.  This also is a job for build tags.

Note that this sort of requirement is common in other languages like C, but less so in Go.  It is vey easy to update to the latest Go release and have everything build as normal.  So, in Go, it is more common to say _"if you want to use the latest version of my package, then you need to upgrade to Go 1.18 or later"_.
{: .notice--info}

The following code relies on the compiler setting a build tag for every major release up to the latest that the code is being built for.

In this case we use the tag `go1.18` which is set by the compiler for Go 1.18.0 and every subsequent release. You need to provide two source file, the one with the feature uses the `go1.18` tag:

```go
//go:build go1.18

package example

func DoEnhancement(values []things) {
	// code that requires Go 1.18 or later (eg sort using generics)
}
```

But the alternative file looks like this:

```go
//go:build !go1.18

package example

func DoEnhancement([]things) {} // inlines to nothing (eg leave unsorted)
```

Now users of the `example` package can upgrade to the latest release of the package without being forced to upgrade to a new release of Go.

- use latest to build with old compiler

# Prevent Builds with Old Release

You can use the same mechanism to do the opposite - require that the code is built with a specific Go release (or higher).

There is a great article on this at [Version Constraints and Go](https://medium.com/@theckman/version-constraints-and-go-c9309be15773).  This article discusses how Go 1.9 was transparently fixed to better handle time differences using a monotonic clock.  Compiling the code with a Go release before 1.9 could cause problems for a package that relied on the new behaviour.

The simple solution is to require that the relevant code is not used unless built with Go 1.9 or higher.

```go
//go:build go1.9

package example

func HowLongItTook() time.Duration {
...
```

Now the package won't build with any Go release before 1.9.  You will get a build error saying something like "undefined: HowLongItTook".

## Checking Built Versions

Since Go 1.13 the Go compiler has embedded information into the binary about how it was built.  You can easily get this info from `go version`.  For older programs it's a lot harder - see [Dave Cheney's Blog](https://dave.cheney.net/2017/06/20/how-to-find-out-which-go-version-built-your-binary).

```shell
$ go version example.exe
example.exe: go1.17.10
```

Even cooler is that (since Go 1.18) some version control information is embedded into the binary, including the time of last checkin and whether there were any changes since.  This is obtained from Git, Mercurial, etc. (depending on which version control you are using).

Also included are the values of certain environment variables that may have affected how the program was built.

You can see it using `go version -m` like this:

```shell
$ go version -m example
example.exe: go1.22.4
        path    github.com/andrewwphillips/example
        ...
        build   GOARCH=amd64
        build   GOOS=linux
        ...
        build   vcs=git                                              
        build   vcs.revision=f2683762339a117f48997d2946c3cd4b88ffffff
        build   vcs.time=2023-09-09T12:19:29Z
        build   vcs.modified=true
```

# Conclusion

I hope this post has shown you how and why you would use build tags.

Looking at other blogs on build tags, I now realise there are a couple of things I forgot to mention.

First, you can add build tags to any text file involved in the build process, assuming the type of source file, that you are using, allows // style comments.  Since the `go build` tool handles the tags, whatever software compiles/processes the file knows nothing of the tag (just treating it as a comment).  For example, this is used when compiling .c and .s (assembly) files.

Also, build tags can be, and should be (when appropriate), **used with tests**.  For example, you might have operating system specific features that can only be tested when the tests are run on the operating system that is targeted, in which case your `_test.go` file should include a `//go:build` directive specifying the tag for the requisite operating system(s).
