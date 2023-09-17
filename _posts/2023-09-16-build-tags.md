---
title: "Build Tags"
last_modified_at: 2023-09-17T11:46:02+10:00
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

Unlike other languages, Go only gives you very coarse control of what is built -- i.e. you can only include/exclude at the file level, rather than individual lines of code.  This may seem crude, but it actually makes it far easier to understand what code is being built.  Plus other features of Go (eg empty functions essentially disappear, due to inlining), make it very effective.

# Background

<details markdown="1">
<summary>General</summary>
<br/>

**C**

Conditional compilation has always been important in C as it allows you to save memory, by not including unneeded code.  It also means you don't have the performance cost of testing a flag at run-time to decide what to do.  In modern hardware these considerations are less important.

A common use in C, is to create separate debug and release versions.  The debug version has all sorts of assertions and checks that the code is not straying into the realms of undefined behaviour (as C programs are wont to do).  The release versions eschews these checks in the name of performance.

When C's approach to conditional compilation was invented it was done on a line-by-line basis using the pre-processor.  One reason it was done this way is that C compilers (for at least a decade) did not have the benefit of inline functions.

Even in simple cases, the intermingling of conditional compilation with the actual code can make it difficult to understand.  More complex using multiple nested conditions often **lead to code that's unfathomable**.

---
</details>
<!--****************************-->
<details markdown="1">
<summary>Go</summary>
<br/>

After using C for decades, I initially found Go's approach inflexible.  It's true that `//go:build` is a simple mechanism.  It does not even involve the compiler, just the `build` tool.

But once I started using it, and understood its nuances (discussed below) I found it quite effective and, more importantly, it makes the whole process easier to understand.  All the "conditions" are clearly documented at the top of their respective .go file.

Moreover, by requiring use of a separate file, it makes for smaller source files (always a good thing), and even assists **separation of concerns** (the fashionable name for decoupling).

**New Syntax**

Go 1.17 introduced new syntax for build tags, which made things simpler and avoided some pitfalls. See [Dave Cheney's Blog](https://dave.cheney.net/tag/build-tags) if you need to deal with the old syntax for some reason.

---
</details>
<!--****************************-->
<br/>

# Syntax

Build constraints (not to be confused with _type_ constraints used with generics) are probably the most used form of directive.  See my [previous post](https://andrewwphillips.github.io/blog/directive.html) about being careful with the syntax, as it's easy for your build tag to be treated as a comment and ignored.

## Checking The Result

If you want to make sure that your .go file is included in the build it is easy with `go list`.

```shell
$ go list -f '{{.GoFiles}}'
[main.go]
```

This will reflect the settings of `GOARCH` and `GOOS` (see below) if you are cross-compiling.

## Syntax Changes

**Warning** How build tags are used was changed in Go 1.17 to use the `//go:build` directive.  Unfortunately, most blogs gives examples using the old `//+build` format.
{: .notice--warning}

**Note** Whether you use the old or the new format, `gofmt` will add another line to your code so the file will end up with both. However,you should **get used to the new format** as the old format may disappear and the new format avoids some known pitfalls, has syntax like other directives, and has much better control (see [Build Tag Expressions](#build-tag-expressions) below).

## Build Tag Expressions

One of the nice things about the new build tags syntax is it fully supports **boolean expressions**, including brackets.  The old (`//+build`) tags system supported ! (**not** operator) but **and** and **or** were done using commas and spaces (or separate directives).  Plus there were other pitfalls - see the original [Proposal by Russ Cox](https://go.googlesource.com/proposal/+/master/design/draft-gobuild.md) for a full explanation.

## Predefined Tags

The build system defines may tags to identify the operating system and architecture, to be used for platform specific code, as discussed later.  The list of tags can vary between releases.

The best way to find the current set is to look at the [source code](https://go.dev/src/go/build/syslist.go).  Note that as well as the tags for all the different supported operating systems there is also a general "unix" tag which indicates any Unix-like operating system.

## Setting Tags

It's a simple matter to define your own tags when using the build, run, ect tools, by using the `-tags` command line option.  This example set two tags:

```shell
$ go build -tags postgres,unenhanced ...
```

Finally, here is a (silly) example, that only adds a file to the build if the `unenhanced` tag is **not** set, and we are not building for Windows or Android.

```go
//go:build !unenhanced && !(windows || android)
package example
```

# Why use Build Tags?

First consider **do you really need to use build tags**?  First, consider using a run-time flag instead of conditional compilation.  Of course, you need to be able to configure the flag somehow such as using an environment variable, a command line flag, or as part of a configuration file.<br/><br/>Using a run-time flag simplifies the build and installation process since you don't need alternative versions of the executeable.
{: .notice--info}

Here are a few places where you might/must use it.

1. use a facility only supported on specific platform(s)
2. choose between alternative behaviours
3. only allow certain features, such as paid options
4. use a capability of the latest Go release if available

Next, we'll look at these in detail...

# Targeting OS/ARCH

You may already be familiar with the **GOOS** and **GOARCH** environment variables which allow you to cross-compile to any supported target platform.  There are predefined build tags for every target platform (operating system and architecture) that the compiler supports. These tags are used extensively in the standard library to handle OS-specific things (file handling, etc).

You can see a list of every supported combination using:

```shell
$ go tool dist list
```

There are further predefined build tags.  For example, you can use `unix` if your code targets any Unix or Unix-like operating system such as Linux.

You can combine tags, for example to target one specific operating system and architecture.

```go
//go:build windows && amd64
package main
...
```

Of course, you will need to build separate executeables for different targets if you are cross-compiling for different operating systems and/or architectures.  This is done using the **GOOS** and **GOARCH** environment variables.

## File Name Suffixes

When you have a simple requirement for targeting an OS and/or architecture a better alternative is to use file name suffixes. In this case you just add the name of the tag to the end of the file name (before the .go extension), preceded by an underscore. For example, if a file is only to be included for an executable targeting Linux just use a name like `source_linux.go`.

There is no need to add the `//go:build` directive.  In fact this is faster as the `go build` can simply inspect the file name.  The file does not even need to be opened if it is not required.

There are 3 ways this can be specified:
* specific operating system
* specific architecture
* specific operating system and architecture

For example, if you only want a source file to be used for 386 architecture on Linux then you would use a file name like `eg_linux_386.go`.  Note that you must have specify the operating system first - `eg_386_linux.go` would not work.

Of course, if you need something more complicated you need to use `//go:build`.  For example, the following would add the code to every (valid) combination of OS and architecture - ie, 5 different combinations (since darwin/mips64 is not a valid combination).

```go
//go:build (linux || darwin) && (amd64 || arm64 || mips64)
```

# Alternatives

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
Then to select the desired database code just use the `-tags` command line option to Go's build, etc commands.

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

# Conditionally Using New Go Features

Imagine you have created an open-source package.  A recent enhancement you added requires a new feature of the Go compiler, like generics.  But you have also added other features and so you want users of your package to use the latest/greatest version, without forcing them to upgrade to the new compiler release.  This also is a job for build tags.

Note that this scenario is common in other languages, but less so in Go.  It is vey easy to update to the latest Go release and have everything build as normal.  So, in Go, it is more common to say _"if you want to use the latest version of my package, then you need to upgrade to Go 1.XX"_.
{: .notice--info}

The file `enhancement.go` starts like this:

```go
//go:build !unenhanced
package example

func DoEnhancement() {
	// code that requires the latest Go release
}
```

But the alternative file looks like this:

```go
//go:build unenhanced
package example

func DoEnhancement() {} // empty function - inlined to nothing
```

Now your users can upgrade to the latest release of your package without being forced to upgrade to a new release of Go.  They just need to use the `unenhanced` flag like this:

```sh
$ go build -tags unenhanced ...
```

# Conclusion

I hope this post has shown you how and why you would use build tags.

Looking at other blogs, I now realise there are a couple of things I forgot to mention.

First, you can add build tags to any text file involved in the build process, assuming the source file type allows // style comments and whatever processes the file knows how to interpret the tag.  This is used when compiling .c and .s (assembly) files.

Also, build tags can be, and should be (when appropriate), **used with tests**.  For example, you might have operating system specific features that can only be tested when the tests are run on the operating system that is targeted.
