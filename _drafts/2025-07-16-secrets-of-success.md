---
title: "Secrets of Go's Success"
last_modified_at: 2025-06-16T12:28:00+10:00
excerpt: "TODO"
toc: true
toc_sticky: true
categories: [language]
tags: [build,performance,escape analysis]
header:
  overlay_image: "/assets/images/todo.jpg"
  overlay_filter: 0.4
permalink: /blog/secrets.html
---

# Go's Success

<font size="+6">❝</font><font size="+3">Go has not achieved the recognition it deserves</font><font size="+6">❞</font><br/>

Go has had greater success than anyone anticipated, but I believe Go has still not achieved the recognition that it deserves.  Why?  I think there are a few reasons:

* the simplicity of learning Go is overstated, so beginners who try it can be frustrated
* Go often takes a "contrarian" approach (usually successfully), but this can cause misunderstanding
* Go deliberately lacks many "essential" features because they lead to code that is less understandable
* Go has some big advantages that are not recognised or appreciated

It's this last point that I will look at today.

<font size="+6">❝</font><font size="+3">Go has big advantages that are not recognised</font><font size="+6">❞</font><br/>

## Known Reasons

We all know lots of reasons that make Go a great language.  My favourite is the way Go handles concurrency (using goroutines and channels).  Of course, there are other well-known things like performance, simplicity (writing/maintaining), performance, ease of creating tests, fantastic standard library and (of course) the community.

## Wrong Reasons

There are also a few reasons that are overstated, even incorrect.

For example, it's often said that Go is a simple language.  It is not that simple to learn some of the vagaries


- perf, simple, type safety

## The Secrets

- things left out: OO  ++maintainability
- escape analysis
- perf
- build/upgrade
- gopls
- wholisitic approach


## Background

<!--****************************-->
<details markdown="1">
<summary>General</summary>
<br/>

**C**

**C#**

- easier build process than C/C++

- class/struct example

---
</details>
<!--****************************-->
<details markdown="1">
<summary>Go</summary>
<br/>

Here I will just quickly look at a few things which are often touted as great things about Go, but are not as good as they are cracked up to be.

**Simplicity**

**Performance**

**Type Safety**

---
</details>
<!--****************************-->
<br/>

## Simplicity

### Simple Tests

- well known by some

### Simple Maintenance

- left out of the language

### Simple Upgrades

XXX BIGGIE XXX

### GOPLS

## Contrarian Features

try different thing and if it works keep it
- enums - almost work but not enough type safety

### Escape Analysis

### Time

## Performance

### Development Speed

### Cross Compiling

## Build Speed

## Conclusion

- upgrades
  - ease of code evolution
