---
title: Compiler Directives
last_modified_at: 2023-09-10T23:26:02+10:00
excerpt: "A look at all of Go's _pragmas_ (except build tags). How **//go:debug** directive enhances Go's backward compatibility. Plus recent changes to the language that affect some directives."
toc: true
toc_sticky: true
categories: [language,directives]
tags: [pragma,directive]
header:
  overlay_image: "/assets/images/directive.jpg"
  overlay_filter: 0.4
  caption: "Image: [Noun Project](https://thenounproject.com/)"
permalink: /blog/directive.html
---
The first Sydney Go Meetup I attended (2018) had a great talk by Dave Cheney on **compiler directives**, or **pragmas** as he called them (see [Go's hidden #pragmas](https://dave.cheney.net/2018/01/08/gos-hidden-pragmas)).

A lot has happened to Go since then, in particular, there is a new `//go:debug` directive, and the way **build tags** work has changed.  I'll also look at a few other things such as a subtle difference to the behaviour of `//go:nosplit` due to changes in the Go runtime.

# Background

<details markdown="1">
<summary>General</summary>
<br/>
**Language Implementations**

Unlike Go, most programming languages have different (sometimes many) _implementations_.  That is compilers for the same language are created by different individuals or organizations.  Most implementations allow some sort of control (usually command line options or environment variables) of the compilation process, especially for code generation (optimization, target processor(s), etc).

Even early on, many implementors found this was not enough.  They wanted more fine-grained control.  But since they had little or no control over the language specification the obvious way to add more control of the compilation process in the source code is to add special character sequences inside comments.  That way the code would still compile with other implementations that did not understand them.

**Standard Directives**

Some compiler directives are supported by most or all implementations, either as standard features or de facto standards.  Standard directives could often be more simply handled with attributes or some other part of the language (and often are).  I guess they are added in this way so as not to tarnish the purity of the language itself.

As a general rule, directives are localised and do not affect the entire program.  Typically, they only take effect from the current location in the source file until the end of file, or until they are reverted/changed by a following directive.  Sometimes (as in many Go directives) they only affect whatever immediately follows.

**Fortran**

Fortran's directives appear as a comment line, most commonly of the form `!DIR$...`.

**Pascal**

The first compiler directives I encountered were in Pascal.  They were hidden in braces (which enclose comments in Pascal) and started with $ - for example `{$FATAL}`. Other implementations used different conventions.

**C**

There are probably more compilers in existence for C than any other language.  (I worked on one myself.)  Consequently, there are many ways that directives are handled.

In the first C compilers, the _preprocessor_ was not even considered to be part of the language.  (Nowadays the so-called preprocessor directives are a fundamental part of the language.)  So `#pragma` was added as a "preprocessor directive" to allow implementations to have an explicit escape mechanism.  Over time many C `#pragma`s (like `#pragma pack`, `#pragma once` etc.), though not part of the standard language, were implemented by most compilers as a _de facto_ standard.

The C standard has another escape mechanism - identifiers starting with two underscores.  Implementations can even add new keywords to their version of the language by prefixing them with "__".

**Java**

Java allows directives, and you can even create custom ones.  It does not get used much, that I am aware of, probably because it is so confusing.

**C#**

C# has a "preprocessor" (modelled after the one in C) that allows many directives such as conditional compilation and code generation options.  These are all prefixed with the `#` character.

As I mentioned in [C# Overflow Checking using Checked](https://www.codeproject.com/Articles/7776/Arithmetic-Overflow-Checking-using-checked-uncheck) the design of C# can be confusing.  The C# `checked` keyword should probably have been a compiler directive as it only affects code generation within the source file, but it looks more like a statement/function.

---
</details>
<!--****************************-->
<details markdown="1">
<summary>Go</summary>
<br/>

**Directives in Comments**

Go use the traditional method of hiding them in comments.  I'm not sure why.  There aren't many implementations of Go, so directives could have been handled as part of the language, which would have avoided a few problems (esp. with `//+build`).  I suspect they are done this way for one or more of these reasons:

* to be like other languages (unlikely given Go's contrarian approach)
* to discourage their use, as they were mainly added for use by the standard library
* to avoid corrupting the purity of the language
* so as not to burden anticipated alternative implementations (which can ignore them)
* so they can be removed in a later version of Go (unlikely to happen I think)

**import "unsafe"**

The import of the `unsafe` "package" is really just a compiler directive.  There is no actual package of that name.  It just flags to the compiler to allow certain "unsafe" features.  (There are probably more unsafe features than you think -- which I might cover in a future post.)

---
</details>
<!--****************************-->
<br/>

# What Are Go's Compiler Directives?

Compiler directives are a sort of "escape hatch" allowing you to control the output of the compiler without actually using the language itself.  They are for the sort of things you use a compiler command line option or environment variable for, but with more fine-grained control.

<!-- Actually, the distinction between compiler directives used in a computer language, and the language per se, is a bit vague and varies between languages.  Traditionally, directives are "hidden" inside comments but there are other escape mechanisms and some languages do something similar using "attributes" and other features of the language itself.
-->

In Go compiler directives are hidden within comments, like a lot of other languages.

## Syntax

Most directives begin with the characters at the start of a new line "//go:" and are followed by the directive name, and possibly some extra parameters.  There should be **no spaces** until the end of the directive name.

The general format is:

```go
//go:directive [params]
```

Most directives must appear on a line just before the declaration they apply to.  If a directive is in the wrong place it may simply be ignored.

**Warning** You have to be precise with the syntax and placement of directives.  If you get it wrong then you will not get any indication; the compiler will just ignore it as a comment.
{: .notice--danger}

## Examples

Using the new `//go:debug` directive as an example, the compiler, and even go vet, will not warn you about any of the following lines:

```go
// *** INCORRECT - these will be ignored ***
// go:debug panicnil=1
//go: debug panicnil=1
  //go:debug panicnil=1
//go:debig panicnil=1
```

But this will work:

```go
// *** OK (if before package declaration) ***
//go:debug panicnil=1
package main
...
}
```

Fortunately, if you use an unknown setting (like `panicnull` below), the compiler _will_ tell you:

```go
// *** Unknown setting (panicnull) -> build error ***
//go:debug panicnull=1
```

Finally, even a seemingly  _correct_ `//go:debug` directive will be ignored if it is not placed at the top of a source file.  It only works if it's just before the `package main` declaration.

# Conditional Compilation

Go has a simple, and surprisingly effective, method of conditional compilation.  This was originally handled with the `//+build` directive but is now done with the `//go:build` directive.

This is a big topic, so I have decided to defer talking about it till my next post.  Stay tuned.

# //go:debug

I have always found that each Go release does an amazing job of backward compatibility.  This is for many reasons, not just Go's [Compatibility Promise](https://go.dev/doc/go1compat), but also due to only building from source, extensive testing, etc.

Unfortunately, changes sometimes need to be made to Go that break backward compatibility.  This is not done lightly.  It's usually to address some bug, or vulnerability.  These changes can break existing code that depends on the old behaviour.

Luckily, Go has (for many years) allowed you to get the old behaviour for at least 2 years even when building with the latest Go release.  This was done by adding a setting to the GODEBUG environment variable - see [Go Backwards Compatibility and GODEBUG](https://go.dev/doc/godebug).

More recently, even more control was added by Russ Cox (see [Backward Compatibility, Go 1.21, and Go 2](https://go.dev/blog/compat)) including the `//go:debug` directive.

## GODEBUG Environment Variable

Go has always had certain environment variables that are used to control how your program runs.  Usually these control some aspect of the runtime system (eg `GOGC`, `GOMAXPROCS`, etc).  `GODEBUG` is another that originally triggered debug information by way of individual settings such as `gctrace` (trace info. on garbage collections), `schedtrace` (goroutine scheduling), etc.

Since GODEBUG just consisted of a list of key=value pairs it quickly gathered settings to control the runtime and parts of the standard library, including ways to retain old behaviour, when a change that broke backward compatibility was deemed essential.

For example, there was a fix to avoid a possible security vulnerability in Go 1.15 which broke a lot of production software.  To enable the old behaviour in Go 1.15 (and the next few releases) you could use the `x509ignoreCN=0` setting.  This allowed software to continue to work, allowing more time to address the issue properly.

```shell
export GODEBUG=x509ignoreCN=0
```

It is hard to find a full list of GODEBUG settings, especially as they change between releases.  Russ Cox's proposal (see below) gives quite a few, though it doesn't mention the `panicnil` setting introduced in Go 1.21.

## Using //go:debug

Changes have recently been made to Go to allow more control of GODEBUG settings - see Russ Cox's [Proposal: Extended backwards compatibility for Go](https://go.googlesource.com/proposal/+/master/design/56986-godebug.md).  This was implemented in Go 1.21.

The effective value for any particular GODEBUG setting is determined by this order:

1. Go compiler release
2. Go version as specified in go.work or go.mod
3. `//go:debug` directive
4. Value in GODEBUG environment variable

Using the `panicnil` setting as an example: If you built with Go 1.21 then, by default, the `panicnil` setting would be 1. However, if the go.mod file contained the line:

```
go 1.20
```

then `panicnil` would be set to 0.  This, in turn, could be overridden with the `//go:debug` directive like this:

```go
//go:debug panicnil=1
package main
...
```

Note that you can verify the `//go:debug` directives that were used when a program was built using `go list` like this:

```shell
{% raw %}
$ go list -f '{{.DefaultGODEBUG}}'
{% endraw %}
panicnil=1
```

Finally, you can override any setting using the GODEBUG environment variable. How to set an environment variable depends on your operating system but this may work for you:

```shell
export GODEBUG=panicnil=1
# or if GODEBUG already has a value
export GODEBUG=$GODEBUG,panicnil=1
```

## When to use //go:debug

You need to use this directive when all the following conditions (for the program being built) are met:
* the code depends on behaviour that has changed in the new release of Go
* the code can't be fixed (yet) to be compatible with the new release
* the new release is required for some other reason
* you can't rely on GODEBUG always being correctly set in production

Note that by "the code" I do not necessarily mean your code.  More than likely it is due to a package you are using that has not been updated.  You need to get the package owner to fix the package.  (You could fork the package and make the fix yourself, but I would not recommend that approach unless the package is not being maintained.)

## How to use //go:debug

Remember that `//go:debug` settings apply to the whole program.  You can't use different settings per source file or package.

**Important:** To have any effect a `//go:debug` setting **must** appear before the `package main` declaration, or it will be **ignored**.  It can be added to any .go file of the main package, but if you have multiple instances of the same setting (perhaps in different .go files) you will get a build error, **even if they use the same value**.
{: .notice--warning}

To check that your `//go:debug` directives were effective use `go list`.

```shell
{% raw %}
$ go list -f '{{.DefaultGODEBUG}}'
{% endraw %}
```

# //go:noescape

This directive signals that any parameters passed to the function do not "escape" the function, so do not need to be placed on the heap.

It can _only_ be used before a forward declaration of a function. If the full function is available (not simply a forward declaration) then the compiler can determine for itself if any parameters escape.

## Escape Analysis

Escape analysis is one of the under-appreciated gems of Go.  It saves you having think about whether your variables need to be on the stack or the heap.

You can skip the following details if you understand escape analysis, or are not interested. On the other hand you might check out [Escape Analysis](https://www.ardanlabs.com/blog/2017/05/language-mechanics-on-escape-analysis.html) if you are really interested.

<details markdown="1">
<summary>Escape Analysis Details</summary>
<br/>
Every function has a bit of memory for local variables at the top of the stack.  This is called the function's stack frame.  It's better if variables are local (stored on the stack frame) rather than placed on the garbage-collected heap because the variable's:
* memory does not need to be allocated - it's just part of the stack frame for the function
* memory does not need to be deallocated - it is automatically freed when the function returns
* not involved in garbage collection, so does not add to GC load
* more likely to be in cache memory

For this reason, the Go compiler does its utmost to make sure a variable is stored on the stack and does not "escape to the heap".  The part of the compiler that is used to determine this is called the **escape analysis** phase.

There are many ways variables can escape but one way is if you pass a reference to the variable to a function and the function retains that reference somehow. Here is a simple example:

```go
package main

func main() {
	i := 42
	escape(&i)
}

var q *int

func escape(p *int) {
	q = p
}
```
In this code, the `escape()` function saves the address of `i` when it is called from `main()`.  So `i` cannot be stored on `main`'s stack frame in case it is accessed after `main` has returned (ignoring the fact that `main` is a special function that ends the program when it returns).

You can use the `-gcflags -m` build option to see the results of escape analysis.  For example:

```shell
$ go build -gcflags -m
...
.\main.go:4:2: moved to heap: i        
.\main.go:10:14: leaking param: p    
```
This output from the **escape analysis** phase of the compiler shows that, because the parameter `p` "leaks" from the `escape()` function, the variable `i` is stored on the heap.

Of course, most functions don't do anything as silly as save an address to a global, but if the `escape` function above was written in assembly (or C) then the escape analysis, without _further information_, would have to assume the worst.  The `//go:noescape` directive supplies that _further information_.

---
</details>
<br/>
There are a lot of low-level functions in the Go standard library that take "reference" types as parameters and that are, of necessity, written in assembly.  These use the `//go:noescape` directive to tell the Go compiler that their parameters do not escape.

## When to use //go:noescape

You only need this directive if you are writing a low-level function in assembly (or maybe C).  If you do use it then you better ensure that the function's parameters do **not** escape.

# //go:uintptrescapes

This directive is, in a way, the antithesis of `//go:noescape`.  The Go compiler assumes that pointers (and other "reference" types) escape to the heap if they are passed to a function which is not written in Go.  `//go:noescape` tells the compiler that the parameters do not need to be on the heap.

In contrast, the `uintptr` type is **not** a "reference" type so the Go compiler assumes any parameters of that type do not escape to the heap.  `//go:uintptrescapes` tells the compiler that any `uintptr` parameters should be placed on the heap.

## When to use //go:uintptrescapes

Use this directive if you call a function written in assembly (or C) that takes `uintptr` parameters and the values are somehow used after the function returns (so need to be on the heap).

# //go:nosplit

You are probably aware that every goroutine has a stack.  The stack starts off small, but can grow virtually without limit.  The way it grows is that at the start of every function there is a bit of code (called the preamble) that checks if the stack needs to be expanded (ie, if the function's required memory, or "stack frame", would cause the current stack size to be exceeded).

The `//go:nosplit` turns off this preamble. But it is clever enough to do so safely.  Read on to find out how!

## Stack Resizing and the Red Zone

Originally in Go, when the stack needed to be expanded a new block of stack was added (in a sort of linked list).  That is, the stack, which was just one block, was "split" in two.  Due to different issues the way the Go stack grows was changed.  Now, a new bigger block is allocated on the heap and the old stack is copied into it.  So **nosplit** is a bit of a misnomer since the stack is never _split_.
{: .notice--info}

Althugh stacks will grow, when necessry, the runtime always keeps a little bit of empty space above the top of the stack (or below the bottom, in architectures where the stack grows downwards).  This is called the **red zone**.  The size of the red zone is about 700 bytes, but can vary between releases and for other reasons.

The size of the red zone is fixed at compile time.
{: .notice--warning}

The red zone is needed for a few reasons: to allow the runtime a bit of space to handle interrupts.  More relevant is that it can also be used by Go functions (typically low-level standard library functions).  If a function is preceded by the `//go:nosplit` directive it does not get a preamble, which means the goroutine's stack will never be expanded when that function is called.  Of course, there are some restrictions on the size of the function's stack frame (memory used by non-escaping local variables).

In the best case the stack frame size can be up to the size of the red zone, but if the function calls, or is called by other function(s) that also use `//go:nosplit` then the allowed frame size is commensurately reduced.

By analysing the call trees of all functions that use the `//go:nosplit` directive the compiler can determine if the red zone would be exceeded at compile time.  If you add the `//go:nosplit` directive to a function which would cause the red zone to be exceeded the compiler will give you an error.

<details markdown="1">
<summary>Goroutine Preemption</summary>
<br/>

A slight detour is required here because the function preamble has been (in earlier release of Go) involved in go-routine scheduling.

If there are more goroutines in a running Go program than there are (unblocked) threads allocated to the programs (as determined by GOMAXPROCS) then the Go runtime has to schedule the goroutines onto the available threads.

Up until Go 1.14 this scheduling was "co-operative" and only done at certain places in the code, one of which was in the function preamble.  In this system a goroutine that avoided (deliberately or by accident) any of the "co-operative" scheduling points could hog a thread, which could have nasty consequences for the runtime, often completely freezing the program (halting all goroutines!) when the runtime is trying to start a garbage collection.

Using the `//go:nosplit` directive allowed a goroutine to avoid the function call "co-operative" scheduling point.

Due to some amazing work of the Go Authors, goroutines are preemptively scheduled (since Go 1.14?).  There is no longer any way a goroutine can do these nasty things to the runtime.

---
</details>
<br/>

The `//go:nosplit` directive is used quite a bit by low-level standard library functions.  Apart from its performance advantage, this directive is essential for some runtime functions that deal with memory allocation.  If these functions needed to expand the stack they would end up calling themselves whic might lead to infinite recursion.

This should **not** be a consideration for any functions you write.

## When to use //go:nosplit

The only advantage to using `//go:nosplit` would be to eliminate the preamble making the functions slightly smaller and faster.  However, the benefit would be negligible, except for small functions that are called a lot, but these would more than likely be automatically inlined.  So it would only be useful for an often-called function that is not inlined for some reason.

In the past the directive was also used to prevent the goroutine from being "descheduled" on entry to the function (see **Goroutine Preemption** above).  This no longer works since goroutines are preemptively scheduled.
{: .notice--warning}

# //go:norace

This directive specifies that race detection is not to be applied to the function following it.

## The Race Detector

This directive has no effect unless the code is using the race detector.

Note: if you are not making use of the **Race Detector** then you probably should be.
{: .notice--warning}

<details markdown="1">
<summary>Data Races</summary>
<br/>
In lots of ways Go avoids all sorts of problems.  One way is that it is usually hard to introduce undefined behaviour (unlike the myriad of ways you can do it in C/C++ :).  But there is one easy way in Go, just by using the `go` keyword.  This code is perfectly safe:

```go
func main() {
    for i := 0; i < 10; i++ {
        fmt.Println(i)
    }
}
```

Just adding `go` introduces a data race.  The behaviour of this code is undefined.

```go
func main() {
    for i := 0; i < 10; i++ {
        go fmt.Println(i)
    }
}
```

---
</details>
<details markdown="1">
<summary>Race Detector</summary>
<br/>

Concurrent code in Go is simpler and far less prone to data races than other languages when done properly (see [Share Memory By Communicating](https://go.dev/blog/codelab-share)), but accidents still happen.

The creators of the Go language realised this was an issue, so they gave us the race detector.

If you use goroutines, then you should be using the race detector. Remember too, that you may even be using goroutines without knowing it - for example, any handlers fired up from `http.ListenAndServe()` run in a separate goroutine.

To enable race detection just build your code with the `-race` command line option.  Generally, this is only done for test builds but, if possible, I recommend running it in production.  Of course, you will need a lot of spare capacity, as it can add make your code run an order of magnitude slower, or more.

Why run it in production?  The race detector will only detect potential races in code paths that are executed (see [Data Race Detector](https://go.dev/doc/articles/race_detector)).  The best way to detect data races that might occur in production is to run it there.

One way to cope with the extra burden might be to only run the slower version at quiet times, as long as it still represents typical usage.  If you are running multiple instances behind a load-balancer you could permanently run one instance with race detection turned on, leaving it off for the others.

---
</details>
<br/>

## When to use //go:norace

Should you use it?  The first thing to note is that this directive has no effect unless you have built the code with race-detection 
enabled (using the `-race` command line option).

Common advice is not to use `//go:norace` since the race detector never produces false positives.  My opinion is that it can be useful if you are using race detection **in production**.  Turning it off for a function that is executed many times in an inner loop would give a large boost to performance.

That said, I would **never** use this directive on a function _unless there is absolutely no chance of a data race_ (even allowing for possible future code changes).  For example, it would be safe for a "pure" function -- i.e. does not have side effects.

# //go:noinline

This directive signals that the following function should **not** be inlined.  Inlining is a very important optimization feature of the compiler.

## Optimization

Optimization is where the compiler reorganizes the initial "draft" of the code that it generated to be more efficient in some way.  All compilers do some sort of basic optimizations.  Over the years the Go compiler has made incremental improvements to optimization including the recent PGO added in Go 1.21 - see [Profile Guided Optimization](https://go.dev/doc/pgo).

One of the most important optimizations is inlining.  Not only does it save function call overhead of the inlined function, but it enables and assists other types of optimizations.

Here is an explanation if you are interested.

<details markdown="1">
<summary>Inlining</summary>
<br/>
Inlining is the process of using the code of a function call "in-line" within the calling function.  This can have benefits and drawbacks.

**Function Call Overhead**

With any function call there is overhead in pushing parameters (or loading them into registers), saving/restoring the stack frame, jumping and returning, etc.  Each function in Go (as discussed above in [//go:nosplit](#go-nosplit)) also needs a "preamble" to check such things like if there is enough stack space.

All these things are small but can add up, particularly for a function that is called a lot.

Here is an example, to clarify my explanation:

```go
func fma(a, b, c int) int {	return add(a*b, c) }

func add(m, n int) int { return m + n }
```

The `add` function will probably be inlined.  The compiler will generate code as if the outer function was written like this:

```go
func fma(a, b, c int) int {	return a*b + c }

```

**Pros and Cons**

Inlining also has other benefits such improved cache use since the inlined function's local variables are now in the same stack frame as the calling function's.  However, probably the biggest benefit is that it enables many local code optimizations since the inlined code effectively becomes part of the calling function's code.

But you don't always want to inline functions.  When the code for a function is inlined it adds more code at the call site - if the inlined function is called in lots of places then this can greatly increase the total size of the code generated.  (Though for very small functions it could do the opposite if the inlined code is less than the code for parameter marshalling, function call, etc.)

Whether a function should be inlined is extremely complicated.  For example, it may be a good idea **not** to inline a function at some calls sites if it is on a rarely executed code path and/or it has a large stack frame (local variables) which would add to the callers stack requirements (possibly causing unnecessary stack growth).

The rules on how functions are inlined in Go are often tweaked between releases, but as a rule of thumb if a function is small and called a lot then it is a good candidate; but **not** if it's big and called from many different places.

Finally, I'll just mention how (Go 1.21's) PGO benefits from code inlining.  The profiling performed as the first step of PGO allows the compiler to better decide what functions should be inlined.

---
</details>
<br/>

There are several compiler flags that control inlining.  When building a Go program these are specified using the `-gcflags` (Go compiler flags) option.  Use the `-l` option to disable inlining or use the `-N` option to disable all optimizations (not just inlining).  To ask the compiler to do more inlining using `-l -l` and even `-l -l -l`.  You can use the `-m` option to check all optimizations including inlining.

Here is an example of building using maximum inlining and displaying the effect:

```shell
$ go build -gcflags "-l -l -l -m"
.\main.go:16:6: can inline a
.\main.go:44:6: can inline main
.\main.go:47:12: inlining call to add
.\main.go:149:12: inlining call to errors.New
....
```

## When to use //go:noinline

`//go:noinline` is most commonly used with benchmarking.  For example, I recently wanted to compare the performance of different implementations of the same facility.  One of the implementations used recursion (so was not able to be inlined), but I wanted to test my other implementation with and _without_ inlining just to understand where the time was being spent.

In production, you might want to turn off inlining if you have a relatively large function called from many different places, and you suspect it is bloating the size of your code.  The first thing is to check whether (and how often) it is being inlined using the `-gcflags -m` command line option.  For example, this checks if and where `add()` is being inlined:

```shell
$ go build -gcflags -m 2>&1 | grep "inlining call to add"
.\main.go:47:12: inlining call to add
.\main.go:47:20: inlining call to add
...
```

Since `add` is inlined you would next check the size of the executable file with and without inlining of the function.

You might also want to selectively control where a function is inlined.  Say you have a function that is called in many places but only in one place (eg. innermost loop)) is performance critical.  The only way to (currently) do this is to have two variations.

```go
// addInlined is only used in performance critical code
func addInlined(m, n int) int { return m + n }

//go:noinline
func add(m, n int) int { return m + n }
```

# //go:linkname

This directive is more of a "linker" option than a compiler option.  It creates an object-file symbol for a function or non-local variable.  This can be used to create an alias for an exported (capitalised) function or variable or allow an unexported function or variable to be exported (using a different name).  The new name can include the package name - so it allows the function/variable to appear to be part of another package.


Unlike the directives above it need not appear directly above the affected function or variable in the source code, but it would be confusing if you put it elsewhere.  To use it **you must import "unsafe"** as it can cause major problems if used incorrectly.

For example, the following code creates the symbol "g.h" that the linker will use to link to the function `f()`.

```go
...
import "unsafe"
...
//go:linkname f g.h
func f() {
...
```

Then to call this function from the `g` package you must add a forward declaration for the function `h()` so that the compiler knows how to call it.

```go
package g

import "unsafe"

//go:linkname h
func h()
```

Note that this use of the `//go:linkname` directive (with one parameter instead of two) is just so the compiler accepts the forward declaration, otherwise it will complain that `h()` does not have a function body.  (There are other ways to allow forward declarations such as including assembly source files in the package.)

**Warning:** The forward declaration must exactly match the original variable/function, otherwise horrible things will happen!  Functions (such as `f`/`h` above) must match exactly in terms of parameters and return values.
{: .notice--danger}

## When to use //go:linkname

I can't imagine a good use for this directive as it circumvents Go's (limited) information hiding facilities.

# gccgo directives

The only "other" implementation of Go (apart from TinyGo) is gccgo which is part of the GNU compiler suite.  I believe that it's not used much anymore, especially as it does not (yet?) support generics which appeared in the "default" Go compiler more than a year ago.

Of the above directives, gccgo only supports `//go:noescape`, `//go:nosplit`, `//go:noinline`, as well as `//inline` (below).  However, it has a couple of its own directives:

**//extern** sets the externally visible name of a function and must immediately precede a forward function declaration.

This is usually used to allow a C or assembly function to be invoked from Go with a different name.  For example, the following forward declaration allows the UNIX system call `open()` to be invoked from Go as `c_open()`:

```go
//extern open
func c_open(name *byte, mode int, perm int) int
```

Note that `//go:linkname` can be used for the same purpose.

**//go:compile**

This is similar to `//go:linkname` but renames the object-file symbol rather than creating an alias.  This example, makes the function `F` externally visible as `f`.

```go
//go:compile F f
func F() {
...
```

# Code Generation

## // Code generated (DO NOT EDIT)

If you write software that generates Go code you should indicate this with a line at the top of the source file like this:

// Code generated ... DO NOT EDIT.

with a description (in place of ...) of what generated it, and perhaps it's version, date, etc.

This indicates to a Go-aware editor/IDE that the user should be prevented from (or warned about) editing the file, since the next time the code is generated any changes will be clobbered.

**When to use**

This directive should be used if you are generating Go code that may be overwritten.

## //line

The `//line` directive is also intended for use by generated code.  In particular, it's used when .go files are generated as the "target" from a different "source" file.  The info, in the directive can be used by the compiler, or anything that processes the .go file, eg:

* displaying the location of a syntax error during compilation
* highlighting the current location when stepping through code in a debugger

In other words, it allows a mapping of locations in the .go file (the location that the `//line` directive appears) to locations in the original source file (from the file name, line and column number given in the directive).  

Here is a real example from `main.go` which was transpiled from `main.go2` using the go2go transpiler (used for experimental syntax before generics were finalised and added to Go).  The directive indicates that the package declaration occurred at line 1 of `main.go2`.

```
// Code generated by go2go; DO NOT EDIT.

//line main.go2:1
package main
...
```

See [doc.go at or after lines 160-200](https://github.com/golang/go/blob/master/src/cmd/compile/doc.go#L200) for details.

**When to use**

This directive should be used if you are generating Go code from another (text) file.  The `//line` directive can be regularly inserted into the Go code to indicate the corresponding place in the original source file.

**Warning:** Unlike other directives there _must_ be a space immediately after `//line` - don't use a colon.  You do need a colon (:) between the file name and line number.  You also need a colon (preceded by a space) before the line number, if no file name is specified.
{: .notice--danger}

## //go:generate

This directive is also used for code generation but, in this case, to actually run code generation.  Note that it's **not** used by the compiler or `go build` tool, only by `go generate` tool, though that _is_ typically run at the start of the build process.

My impression (probably wrong) is that the `go generate` tool was added in response to criticism of missing features of the language, such as generics, preprocessor, and enums.

See [Using go generate to reduce boilerplate code](https://blog.logrocket.com/using-go-generate-reduce-boilerplate-code/) for a great article on how to use it.

**When to use**

As the Go language has developed, especially with the addition of generics, the need for the `go generate` tool has decreased.  However, it is still very useful for generating Go code.  I will continue to use the `//go:generate` directive to invoke the `stringer` tool (at least until Go adds enumerated types to the language :).

# Conclusion

Compiler directives are interesting though you may never need to use them.  Those that I find the most useful are `//go:debug` and `//go:build`. The recently added capabilities of `//go:debug` add to Go's amazing compatibility features.  Unfortunately, I did not have enough time to cover conditional compilation (`//go:build`) but I will do very soon in my next post.

I have tried to show uses for some of the other directives but most of the time you won't need them. If you do use them it is important to understand their persnickety syntax.  If you get it wrong you probably won't get an error message, so you need to verify that it had an effect (eg use the `-gcflags -m` build flags to verify that `//go:noinline` had an effect).
