---
title: "CPU Profiling and Flame Graphs"
last_modified_at: 2023-12-26T15:26:02+10:00
excerpt: "<br/><br/><br/><br/>Easily track down performance problems with flame graphs."
toc: true
toc_sticky: true
categories: [profiling,flame graphs]
tags: [profile,pprof,flame,graphviz,benchmarks,trace]
header:
  overlay_image: "/assets/images/flames.jpg"
  overlay_filter: 0.5
permalink: /blog/flame-graphs.html
---

Merry Christmas-time and happy new year!  I've been a bit busy the last couple of months with [GopherconAU](https://gophercon.com.au/) (which was a _great_ success), etc.  I've got a large list of blog topics to do, but one thing that drew a lot of interest at GopherCon was a lightning talk on **flame graphs**.

It seems that a lot of Gophers have trouble understanding and using Go's performance analysis tools including flame graphs.  This is a shame as they are very easy to use once you understand all the options and know a few shortcuts.

So I'm going to explain the options and demonstrate **the simplest way to generate useful flame graphs**.  If you just want to quickly find out how to generate a flame graph then see the [Summary](#summary) at the end.  OTOH if you want _more_ details, plus links to full explanations, see the [Background](#background) section next.

Part of the confusion is that Go comes with several different tools for analysing runtime performance including benchmarks, sample-based profiling (**pprof tool**), the execution tracer (**trace tool**), etc.  Moreover, there are also several different types of _profiling_.  If you are unclear, I have clarified all this in the Background->Go section below.

Today, I'm just looking at profiles - specifically, CPU profiles - how to generate, view and understand them using flame graphs.

# Background

<details markdown="1">
<summary>General</summary>
<br/>
**Pprof**

Pprof started life (more than 2 decades ago) as part of the **gperftools** (Google Performance Tools) suite which were mainly written in C++, though Pprof itself was a Perl script I think.  After being used with the Go project for many years it was rewritten in Go with many improvements.  The original (deprecated) version is still available at [gperftools](https://github.com/gperftools/gperftools).

Pprof is used to analyse profiles consisting of stack traces.  When written to file they usually have a .pprof extension and are stored in a compressed format based on [Protocol Buffers](https://protobuf.dev/).  See [pprof on github](https://github.com/google/pprof/blob/main/README.md).  Profiles can store stack traces related to any useful run-time statistics but are most often used for CPU and memory profiles.

Pprof provides many options and reports but originally visualization was only provided by generation of a **GraphViz** .dot file

Pprof documents and reports use the terms **flat** and **cumulative** to mean the amount of time spent in a function _not_ including (flat) and including (cum) time spent in callees (functions called from the function).  Note that when I mention **functions** I also mean Go **methods**.
{: .notice--info}

**Graphviz**

[GraphViz](http://www.graphviz.org/) provides a simple and effective way of visualizing directed graphs** written in the **.dot text format**.  In its most basic form .dot files are very simple to generate and understand.  (Though the full DOT language is more complicated - see [DOT language](https://www.graphviz.org/doc/info/lang.html))

GraphViz (using .dot files) is traditionally the way that Go CPU profiles are _visualized_.  It displays a directed graph, where nodes represent functions and vertices function calls.  Each vertex pointing to a node represents the callers of the function and vertices exiting the node are the **callees**.  Leaf nodes are functions that do not call any other functions (i.e. they have no callees).

This visual display makes it easy to see where the CPU time is being taken.

<div style="float: right; "><img src="{{ site.url }}{{ site.baseurl }}/assets/images/simple-dot-graph.png"/></div>

For example, here is the graph from the [Example Code](#flame-graph-example) discussed later.  Notice that the functions using the most CPU (`fact()` and `sum()` are displayed larger and in a different colour.

Since the example code starts no goroutines, the interesting stuff occurs under `main()`.  The other functions (smaller and in grey), are associated with the Go runtime or the profiling library.

See [CPU Profile](https://graphviz.org/Gallery/directed/pprof.html) for another example.

** _Graphs_ in this case refers to the specific vertex+edge graphs of [Graph Theory](https://en.wikipedia.org/wiki/Graph_theory).  On the other hand _flame graphs_ use a type of "graph" that is more like a bar chart. 
{: .notice--info}

**Flame Graphs**

Many people prefer flame graphs which were specifically invented to visualize performance profiles (not necessarily for Go).  They were invented about a decade ago by [Brendan Gregg](https://en.wikipedia.org/wiki/Brendan_Gregg).  You won't need the source code, but it is available from [GitHub](https://github.com/brendangregg/FlameGraph).

The horizontal axis shows the (cumulative) time spent in a function while the vertical axis represents the depth of the call stack.  The **tips** of the flames are functions that do not call other functions. 

Flame graphs are supported by the Go **pprof tool** web GUI for several years now.  (I only just discovered this :)
{: .notice--info}

<div style="float: right; "><img src="{{ site.url }}{{ site.baseurl }}/assets/images/simple-flame-graph.png"/></div>

Here is a flame graph (from the same [Example Code](#flame-graph-example)) as shown in the **pprof tool** web GUI.  (A flame graph, like this, where the flames point downwards, is sometimes called an **icicle graph**.)  I explain in detail how to use flame graphs with Go CPU profiles in the body of this article below.

The _horizontal_ axis of a flame graph represents the amount of time spent in each function, but is **not** shown in chronological order.
{: .notice--warning}

---
</details>
<details markdown="1">
<summary>Go</summary>
<br/>
Go comes with many tools that can be used to find and analyse performance (and other runtime) issues.  Here is a brief overview of them all, so you understand how CPU profiling fits into the picture.

<!-- I hope to get around to discussing tracing, benchmarking and the different types of profiling at a later date. -->

**Benchmarks**

Among other things the Go **test tool** allows you to run benchmarks on a function or other small piece of code. See [How to write benchmarks - Dave Cheney](https://dave.cheney.net/2013/06/30/how-to-write-benchmarks-in-go) for details.

Typically, you use benchmarks on small pieces of code that will be executed often.  You may have created a package and are aware of a function or method that you know will be called a lot.  Or you may have identified (using CPU profiling) a bottleneck in a low-level piece of code.

In these situations you can benchmark alternatives to find a faster implementation.

**Warning**: Be careful to benchmark on all target platforms as the relative speed of code may vary significantly between architectures.  (Also take heed of other recommendations in Dave Cheney's blog above.)
{: .notice--warning}

Benchmarks can also be useful for finding performance regressions on critical areas of the code.  That is, you can re-run benchmarks (on the same hardware), after making code changes, to ensure there is no unintended loss of speed.

**Note**: You can use **flame graphs** _with_ your benchmarks, to understand more about where the benchmarked code may be slow.  For this you need to generate a CPU profile using the `-cpuprofile` option of the test tool.
{: .notice--info}

<!--
**Warning**: Don't optimize code if it makes the code more complicated or harder to understand _unless_ you are sure it is causing a performance issue.  You probably should profile the code first (as discussed below).  Most of the time the problematic code will not be where you think.
{: .notice--warning}
-->

**Profiling**

The way Go generates CPU profiles is like most runtime profilers.  It samples what the CPU is doing 100 times/second.  You typically save these samples to file and later inspect them using the **pprof tool** or visualise them, e.g. with flame graphs as explained later.

Go also allows you to generate other types of profiles for other, usually more specialised, purposes.  Since we are looking at performance we just discuss CPU profiles below.  However, other types of profiles, especially heap profiles, can be useful for understanding and fixing some performance issues.

Just for completeness here are the other types of profiles and a short description:

* heap - use to understand how memory is allocated and garbage collected
* threadcreate - you're unlikely to need this
* goroutine - stack traces of all goroutines
* block - useful for synchronisation problems such as deadlocks
* mutex - synchronisation problems involving mutexes

See the Go Blog [Profiling Go Programs - Russ Cox](https://go.dev/blog/pprof) for more details.  Note that (except that it does not cover flame graphs), this is still relevant despite being written in 2011.

**Comparing/Combining Profiles**

TODO

**Execution Tracer**

At a somewhat lower level is the Go **trace tool**.  Unlike profiling, it doesn't sample your code, but records _everything_ that your goroutines are doing.  It allows you to see when and where (which thread) goroutines are active, plus how they interact with each other and the garbage collector.

This is a brilliant tool which AFAIK is unique to Go.  It allows you to understand how your code is working at a very low level.  This is perfect for understanding interactions in highly concurrent software, including performance issues.

**Note**: The execution tracer is not to be confused with tracing packages like [golang.org/x/net/trace](https://pkg.go.dev/golang.org/x/net/trace) (from the Go authors) or [Open Telemetry](https://github.com/open-telemetry/opentelemetry-go) (from the Open Telemetry Project). It's a low-level facility requiring no code changes (except perhaps to invoke it).  In contrast, traces (such as used by Open Telemetry) require you to instrument your code to allow you to follow the flow of data through the application, or even between applications if using distributed traces.
{: .notice--info}

**The runtime package**

TODO

**GODEBUG**

TODO

---
</details>
<!--****************************-->
<br/>

# Generating CPU Profiles

There are several different ways to generate profiles in Go, but you generally want to end up with a .pprof file for later analysis.

To avoid misleading results only run one profile at a time.  When generating a CPU profile do **not** generate a heap profile or run the execution tracer at the same time as this will distort the measurements.
{: .notice--warning}

## The net/http/pprof package

You can directly create a profile using the **runtime/pprof** package (see next section) but a convenient way (especially if your application already includes a web server) is to use the _other_ **pprof** package in the Go standard library: **net/http/pprof**.  Once added to you server you can generate different profiles _at any time_ with a simple HTTP request.  

**Warning**: Many servers use the **runtime/pprof** package to assist diagnosing problems in production, since there is very little performance impact (except while actually generating a profile).  However, there are **security issues** that you should be aware of.  The generated stack traces may expose sensitive data.  Even just generating a profile (using one of the HTTP endpoints mentioned below) could be used for a DOS attack.  See [Your pprof is showing](https://mmcloughlin.com/posts/your-pprof-is-showing) for the issues and how to address them.
{: .notice--danger}

If you are running the default web handler then you don't have to do anything but import the package.  The **net/http/pprof** package's `init()` function hooks into the default web-server.

```go
import (
	_  net/http/pprof
)
```

It registers the following http endpoints (and possibly others):

* http://localhost:6060/debug/pprof/profile
* http://localhost:6060/debug/pprof/heap
* http://localhost:6060/debug/pprof/block
* http://localhost:6060/debug/pprof/mutex
* http://localhost:6060/debug/pprof/trace

If your code does **not** start a web-server then you can create one (e.g. on `localhost:6060`) with this code: `go http.ListenAndServe(":6060", nil))`.  See [Writing Web Applications](https://go.dev/doc/articles/wiki/) for details.
{: .notice--info}

Confusingly, the `debug/pprof/trace` path runs the **execution tracer** _not_ the profiler (see [Background](#background) section above for details on Go's **execution tracer** tool).  It generates a _trace_, **not** a profile.
{: .notice--warning}

Also note that if you are **not** running the default web-server then you will have to register your own handler for `pprof.Profile`.  For example:

```go
    mux.HandleFunc("/debug/pprof/profile", pprof.Profile)
```

Once the application (including the webserver) is running you can use one of the above HTTP endpoints to generate a profile, eg using `curl`.  Since we want a CPU profile we will use `/debug/pprof/profile` then use the **pprof tool**.

```shell
$ curl --output cpu.pprof http://localhost:6060/debug/pprof/profile?seconds=5
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   527  100   527    0     0    104      0  0:00:05  0:00:05 --:--:--   138

$ go tool pprof cpu.pprof
Type: cpu
Time: Dec 24, 2023 at 10:14pm
Duration: 5.01s, Total samples = 10ms (  0.2%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) ...
```

Or you can generate a profile and run the **pprof tool** in one step:

```shell
$ go tool pprof http://localhost:6060/debug/pprof/profile?seconds=5
Fetching profile over HTTP from http://localhost:6060/debug/pprof/profile?seconds=5
Saved profile in C:\Users\Andre\pprof\pprof.samples.cpu.004.pb.gz
Type: cpu
Time: Dec 24, 2023 at 10:19pm (AEDT)
Duration: 5.01s, Total samples = 10ms (  0.2%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) ...
```

See [interactive CLI](#interactive-cli) below, for how to use the **pprof tool** in interactive mode to analyse a CPU profile.

## The runtime/pprof package

To programmatically control exactly what code you are profiling you need to use the "other" pprof package called `runtime/pprof`.  Call `runtime.StartCPUProfile()` to start CPU profiling and `runtime.StopCPUProfile()`.  You need to pass an `io.Writer` (usually just a file open for writing) to `runtime.StartCPUProfile()`.  For example:

```go
	f, err := os.Create("cpu.pprof")
	if err != nil {
		panic(err)
	}
	defer f.Close()

	pprof.StartCPUProfile(f)
	defer pprof.StopCPUProfile()
	// code to profile
```

Of course, _all_ goroutines of the process are profiled until `StopCPUProfile()` is called.  If there are other running goroutines then their code will also be profiled.  If you just want to profile code in one goroutine (ie, just the above `// code to profile`) then you need to suspend other goroutines or avoid creating them.
{: .notice--info}

Once you have created the `cpu.pprof` file you can analyse and visualize it as discussed below.

## Profile package from Dave Cheney

Some time ago Dave Cheney created a **profile** package to make it even simpler to run the Go profiler simply by calling `profile.Start()` to begins profiling and `Stop()` to finish. A .pprof file is created in your "TEMP" directory and the name is logged (written to stderr).

By default, it creates a CPU profile (but there are options for other types of profiles).  All you need is this:

```go
package main

import "github.com/pkg/profile"

func main() {
    defer profile.Start().Stop()
    // code to profile
}
```

You should see something like this when you run it.

```shell
$ go run main.go
2023/12/25 17:12:29 profile: cpu profiling enabled, /temp/profile847634849/cpu.pprof
2023/12/25 17:12:30 profile: cpu profiling disabled, /temp/profile847634849/cpu.pprof

$ go tool pprof /temp/profile847634849/cpu.pprof
```

I can't see how it could be much simpler, but see [Simple Profiling Package](https://dave.cheney.net/2014/10/22/simple-profiling-package-moved-updated) for more info.

It's recommended to use the above **github.com/pkg/profile** package rather than the earlier **github.com/davecheney/profile**, which is deprecated.
{: .notice--warning}

## The test tool

When benchmarking using the Go **test tool** it may not be obvious where the problem is if there is a performance issue.  You could experiment by modifying the benchmark code, but it may be simpler and faster to just run a CPU profile on the benchmark itself.  This is easily achieved using the `-cpuprofile` command line flag.

```shell
$ go test -bench=. -cpuprofile cpu.pprof

$ go tool pprof cpu.pprof
```

## Converting Other Formats to Pprof

TODO

# Inspecting Profiles

There are lots of ways to examine profiles (.pprof files) most of which involve the Go **pprof tool**, such as:

1. use options like `-top` and `-peek` for a text report
2. use `-png`, `-svg`, `-pdf`, etc to visualize as a graph
3. use `-dot`, `-callgrind`, etc - convert to a different format
4. without options to invoke an interactive CLI tool
5. using the `-http` option to invoke an interactive GUI mode in a web browser (incl. flame graphs!)

You can also generate or view a flame graph directly from a .pprof file, as I explain later.

## Text Reports

The **pprof tool** outputs a text report when you use the `-text`, `-top`, `-peek`, etc command line flags.  These have the same information as the corresponding commands in the [interactive CLI](#interactive-cli).

## Graphs 

The `-gif`, `-pdf`,`-png`,`-ps`, `-svg`, etc command line flags of **pprof tool** generate a visual call graph.  These are a static representation of the .dot file format.  Using Graphviz to view a .dot is preferred (see `-dot` option below).

## Other File Formats

The `-dot` command line flag is used to generate a .dot file which can be used to display a call graph with Graphviz.  You have to install GraphViz as it is not part of the Go installation, but it's a useful program to have anyway.  See the section on GraphViz in the [Background](#background) above.

The `-callgrind` flag is used to generate a callgrind file.  This could be useful if you are used to working with callgrind utilities.  See [callgrind](https://valgrind.org/docs/manual/cl-manual.html) for more information.

## Interactive CLI

In interactive (CLI) mode pprof provides a lot of commands, but you will probably only ever use the *top* and the *peek* commands, so I briefly explain those.  Many of the other commands are for generating different profiles or graphs in different file formats, which can be done from the command line anyway (see above).

### Top command

The top command shows the 10 (by default) most expensive functions.  For CPU profiles this means the functions that used the most "flat" CPU time, or time in the function *not* including callees (functions called from it).  If you want to see the top functions by CPU time _including_ callees use the --cum option

Note: time spent in the function is called the _flat_ time, while time in the function plus sub-functions (callees) is the _cumulative_ time.
{: .notice--info}

```shell
$ go tool pprof cpu.pprof
Type: cpu
Time: Dec 24, 2023 at 10:14pm
Duration: 5.01s, Total samples = 10ms (  0.2%)
Entering interactive mode (type "help" for commands, "o" for options)

(pprof) top 4 --cum
Showing nodes accounting for 1230ms, 98.40% of 1250ms total
Showing top 4 nodes out of 37
flat  flat%   sum%        cum   cum%
160ms 12.80% 12.80%     1230ms 98.40%  main.main
0     0%     12.80%     1230ms 98.40%  runtime.main
540ms 43.20% 56.00%      570ms 45.60%  main.fact (inline)
480ms 38.40% 94.40%      490ms 39.20%  main.sum (inline)
```

For cumulative times `main()` will be at the top as everything (except initialisation) runs beneath it.  but we can ee rom this that the functions `fact()` and `sum()` use almost all the time within `main()`.

### Peek command

The peek command shows a function (or functions matching a regular expression) plus their callers and callees.  It shows relative amounts of time spent in each of the callees.  This can be useful to see where a function is spending most of its time.

```shell
$ go tool pprof cpu.pprof
Type: cpu
Time: Dec 24, 2023 at 10:14pm
Duration: 5.01s, Total samples = 10ms (  0.2%)
Entering interactive mode (type "help" for commands, "o" for options)

(pprof) peek sum
Showing nodes accounting for 1.25s, 100% of 1.25s total
----------------------------------------------------------+-------------
flat  flat%   sum%        cum   cum%   calls calls% + context
----------------------------------------------------------+-------------
0.49s   100% |   main.main (inline)
0.48s 38.40% 38.40%      0.49s 39.20%                     | main.sum
0.01s  2.04% |   runtime.asyncPreempt
----------------------------------------------------------+-------------
```

Here we can see that `sum()` is only called from `main()` and only calls `runtime.asyncPreempt()`.  It uses about 40% of all the CPU used in the profile.

### Web command

The web command opens a visual graph in your web browser.  This is just a simple graph and is **not** the same as (and not as useful as) the [Interactive Web UI](#interactive-web) discussed next.

## Interactive web

If you specify the "-http <address>" option to the pprof tool then an interactive web page is created.  Once you go to the address in your web browser you will see many useful options in the drop-down menus including:

* View/Top - top CPU functions - like [Top Command](#top-command)
* View/Peek - like [CLI Peek Command](#peek-command)
* View/Graph - displays a call graph
* View/Flame Graph - displays a **flame graph**
* View/Source - source code listing with times
* View/Disassemble - assembly code listing

```shell
$ go tool pprof -http :6060 cpu.pprof
Serving web UI on http://localhost:6060
```

Many people (including me until recently) are unaware that the web GUI of the Go **pprof tool** has supported **flame graphs** for several years!
{: .notice--info}

<div><img src="{{ site.url }}{{ site.baseurl }}/assets/images/pprof-flame-graph.png"/></div>
<br/>

## Flame Graphs

There are many different ways to view a flame graph once you have a CPU profile, but the simplest way is just to drag your .pprof file onto http://flamegraph.com/ (thanks Gary for finding this one :) or https://www.speedscope.app/.

For completeness here are the ways you can use flame graphs with Go:

* When I first used flame graphs I had to run a Perl script to create a flame graph from a .pprof file
  * the script is downloadable from [github.com/brendangregg/FlameGraph](https://github.com/brendangregg/FlameGraph/blob/master/stackcollapse-go.pl)
  * this requires you to have Perl installed
* Uber provides [Go Torch](https://github.com/uber/go-torch) to generate profiles and view a flame graph
  * using the containerised version is simplest but requires docker to be installed
  * see [Proiling Go Applications with Flame Graphs](https://brendanjryan.com/2018/02/28/profiling-go-applications.html) for details
* Use the flame graph(s) provided by interactive web UI of the **pprof tool** as described [above](#interactive-web).
* Upload the file to http://flamegraph.com/
* Drop the .pprof file on https://www.speedscope.app

Flame graphs are typically drawn from the bottom up.  When drawn from the top down they are often called **icicle graphs**.
{: .notice--info}

# Flame Graph Example

To demonstrate I wrote a test program with a `fact()` function that multiplies all the numbers, up to a value, together (ie, calculates the factorial), and a`sum()` function that adds all the numbers up to a value.  I call the functions millions of times in a loop.

## Go Code

**Note**: I had to increase the `count` constant much more than I expected to get a measureable result.  With smaller values the resulting profile showed nothing of interest.  If you have a fast CPU then you may need to increase `count` even more.
{: .notice--info}

```go
package main

import (
	"fmt"
	"github.com/pkg/profile"
)

func main() {
    const count = 1e8
	defer profile.Start().Stop()

	var i, s, f int
	for i = 0; i < count; i++ {
		s = sum(20)
	}
	fmt.Println(s)
	for i = 0; i < count; i++ {
		f = fact(20)
	}
	fmt.Println(f)
}

func sum(n int) int {
	r := 0
	for i := 0; i <= n; i++ {
		r += i
	}
	return r
}

func fact(n int) int {
	r := 1
	for i := 2; i <= n; i++ {
		r *= i
	}
	return r
}
```

## Generating the Flame Graph

To view the flame graph, run the program and then open the resulting profile in the **pprof tool** web GUI (at http://localhost:6060 in this example).

```shell
$ go run main.go
2023/12/31 12:26:15 profile: cpu profiling enabled, /temp/profile3084922120/cpu.pprof
210
2432902008176640000
2023/12/31 12:26:31 profile: cpu profiling disabled, /temp/profile3084922120/cpu.pprof

$ go tool pprof -http :6060 main.pprof
Serving web UI on http://localhost:6060
```

<div style="float: right;"><img src="{{ site.url }}{{ site.baseurl }}/assets/images/simple-flame-graph.png"/></div>

Then select **Flame Graph** from the View drop-down menu.

## Execution Times

Notice the block for `main.main` in the flame graph.  This represents the **cumulative** time spent in `main()` so it is to be expected that it extends nearly the whole width (apart from a small part of Go runtime code and `profile.init()`).  However, most of the cumulative time for `main()` [95%] is spent in calls to `fact()` [45%] and `sum()` [45%].

The **flat** time for `main()` [5%] is that small area to the right of the `main.sum` block where there is nothing "above" the `main.main` block.  This CPU usage is simply due to  loop overhead.

Note that none of the profile samples caught the execution of `fmt.Println()`, as it is only called twice, and executes quickly.
{: .notice--info}

## Optimizing the Code

Of course, the above `sum()` function could be made a lot faster.

```go
func sum(n int) int {
	return (n * (n - 1)) / 2
}
```

<div style="float: right;"><img src="{{ site.url }}{{ site.baseurl }}/assets/images/opt-flame-graph.png"/></div>

With this change the flame graph does not even show that the `sum()` function is called.  It is so fast that none of the samples captured its execution.  Now `fact()` consumes about 90% of the time, since the program runs almost twice as quickly as it did before.

# Understanding Flame Graphs

Flame graphs make it easy to see the relationship between callers and callees and compare their flat and cumulative times.  For example, many performance issues are easily seen as "plateaus" - flat areas at the top of the graph with no block (called functions) directly above.  This indicates that a function is using a lot of CPU time, perhaps in a loop, which (if unexpected) could be an opportunity for optimization.

Remember that the horizontal axis represents the amount of (CPU) time spent in a function (and children) but it is not in chronological order.  Any particular block represents _all_ calls of a function with the same call stack.  These calls could have occurred at different times.  [Brenda Gregg's video](https://www.youtube.com/watch?v=VMpTU15rIZY&ab_channel=GOTOConferences) explains this in detail.

The **block colours** in flame graphs usually don't mean anything, except the same function in different call stacks will have the same colour.  Sometimes, blocks have very different colours to indicate something significant such as different languages or execution environments.
{: .notice--info}

# Summary

There are different ways to use flame graphs to understand performance issues.  Here I will just summarise the simplest and most effective way(s) I have found to do this.

## Generate a CPU profile

The first step is to generate a Go CPU profile in a .pprof file.  There are two different scenarios where I would use the **runtime/pprof** package of the **net/http/pprof** package.  (You can also profile tests and benchmarks, but I have never found the need.)

There are also ways to generate and view flame graphs of CPU profiles _in one step_.  I prefer not to do this.  It is better to have the CPU profile in a file for reproduceability and later reference.
{: .notice--info :}

### runtime/pprof

Most of the time you know you have a performance problem and have a rough idea of where it is happening.  In this case it is a simple matter of wrapping the problematic code in calls to `pprof.Start()` and `pprof.Stop()` to generate a profile and write it to disk.

In a highly concurrent program you may have a performance issue in just one goroutine and all the other, unrelated code will muddy the results.  (Though, in my experience most concurrent software has goroutines that are fairly homogenous.)  In this case you may need to find a way to suspend them or avoid creating lots of goroutines.  Setting GOMAXPROCS to 1 may or may not help.

For simplicity, I just use the **github.com/pkg/profile** package as explained above in [Profile package from Dave Cheney](#profile-package-from-dave-cheney).

### net/http/pprof

The other scenario that is often encountered is a continuously running production server where occasionally something triggers the whole server to slow down - for example causing large latencies in requests.  It is usually not immediately, or not at all, obvious what data is triggering the problem.

Many such backends are/have a web-server (or can easily have one added).  Thence, it is just a matter of importing the **net/http/pprof** package.  When the issue occurs in production you can generate a CPU profile with a simple HTTP request.  Note that the package typically has negligible impact on server performance unless you request a profile (in which case you probably alredu have a perfoamnce issue)

See the above [The net/http/pprof Package](#the-net-http-pprof-package) section for details, especially the **security warning**.

## Create a flame graph

Once you have a CPU profile (in a .pprof file) you can use the Web GUI facility in the Go **pprof tool**, of you can simply drag the file onto a flame graph site in your browser.

### Go Pprof Tool Web GUI

Once you have a profile (e.g. in a file `cpu.pprof`) you can open it in the **pprof tool** web GUI like this:

```shell
$ go tool pprof -http :  cpu.pprof
Serving web UI on http://localhost:53947
```
then open the flame graph from the **VIEW** drop down menu. 

### Flame Graph Web Sites

A simple way to view a flame graph is to open https://www.speedscope.app/ or https://flamegraph.com/ in you web browser then darg your `cpu.pprof` file to it.

## Identify Performance Problems

The final step is to use the **flame graph** to find the problem(s).  (Note that sometimes a flame graph is called an **icicle graph** if it grows down instead of up.)

Look for blocks in the flame graph that are unexpectedly wide.  This indicates that the function is being called a lot OR it is taking a long time to run.  However, if the function has other blocks above (or below it for an **icicle graph**) then the problem may be in a **callee** (a function called by the function).

The Go pprof tool designates times as **flat** and **cumulative**.  Other language and flame graph tools may have other name such as **self** and **total**.  Self (flat) is the time spent in a function; total (cumulative) is the time spent in the function and the functions it calls.  The same function may appear in different call stacks with different self and total times.
{: .notice--info :}

As a general rule, time is taken in the top layer of the flame graph (bottom in an icicle graph). That is, the leaf nodes of the call tree (functions that don't call other functions).  Also, some block may have callees (blocks above) but these callees represent a small proportion of the function execution time - ie the function's "flat" time is a large proportion of its "cumulative" time.  So look for large plateaus.  If there are no such areas (ie, the graph has lots of small sharp tips) then you may not be able to find any performance improvements, or there are a myriad or problems.

Most flame graph viewers allow you to see more information about a block such as the flat and cumulative times and the number of samples that hit that function.  You may also be able to click a blok to zoom in on it.

The number of samples that a block has determines the reliability of the figures.  If it only has a handful of samples then the results will no be useful - you may need to generate a better profile, e.g. for a longer time period.
{: .notice--info :}

See [Understanding Flame Graphs](#understanding-flame-graphs) above for more tips.
