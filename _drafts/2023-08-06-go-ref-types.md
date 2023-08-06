---
title: "Go's \"Reference\" Types"
excerpt: "It's important to understand the behaviour of maps, slices, etc"
toc: true
toc_sticky: true
categories: [blog go]
tags: [types]
header:
  overlay_image: "/assets/images/pointer.jpg"
  overlay_filter: 0.5
  caption: "Image: [Noun Project](https://thenounproject.com/)"
permalink: /blog/go-ref-types.html
---

At the recent [June Sydney Go Meetup](https://www.meetup.com/golang-syd/events/294169772/) there was a small debate on reference types in Go, in particular, whether a *<span style="color: brown;">function</span>* is a ref. type.  See the [Functions section](#functions) below, where I show that a `func` (closure) *is* a ref. type.

Then there is the whole debate on **whether Go even has reference types**.  I don't really want to get into that as the discussion seems to generate more <span style="color: red;">heat</span> than light.

The important thing is to understand how reference types (or whatever you want to call them) work.

(But for all the pedants that insist that Go does **not** have reference variables I show in [Captured Variables](#captured-variables) below
that even using the strictest definition Go _does_ have them.)

# Background

<details markdown="1">
<summary>General</summary>
<br/>
The idea of reference types (as opposed to value types) goes back to Fortran.  Though, now I think about it, assembly languages have addressing modes that are "reference types" -- where a value (register) is used as a memory address (reference/pointer).  This contrasts with "immediate addressing", where there is no memory address just a value.

**Fortran**

In Fortran, when you use a variable as a function (subroutine) parameter the address of the variable is placed on the stack.  When the function makes use of the parameter it is actually working with the original variable (but with a different name), and dereferencing it through the address.

This way of passing parameters came to be called "pass-by-reference".  It was error-prone, resulting in bugs - for example, if a parameter was assumed to be just an "input" parameter but was (deliberately or inadvertently) modified, it could have bewildering consequences.

**Algol**

When Algol appeared a few years later they avoided this problem by making parameters "pass-by-value" (by default).  Algol (and Pascal, etc) optionally allows pass-by-reference in case you wanted to return a value, or for efficiency if you wanted to pass a large object.

**C**

C is derived from Algol - so uses "value" types.  It does **not** have pass by reference, but you acheive the same effect by taking the address of a variable (using the `&` operator).

In this way, C is much more like assembly, a pointer is an address -- you must explicitly dereference it (using the * operator).  This makes things more obvious than the reference types of other languages. (Possibly a drawback is that it is easy to create bugs by dereferencing through NULL or even uninitialized pointers.)

Note that types in C have some "optimizations" (because passing large objects on the stack is inefficient).  First, whenever you use an array in C you get a pointer to the first element*.  Also, originally in C you could **not** pass a struct to a function - only a pointer to it - though this restriction was later removed.

**Java**

Java introduced (or popularised) the idea of a reference _type_ (as opposed to just passing by reference).  In fact, just about all variables are references (addresses on the heap) in Java.

A purist might claim that *use* of a reference type must be indistinguishable from the use of a value type.  In Java, the so-called reference types can take the value `null` - making them more akin to pointers.

**C++**

C++ is (effectively) a superset of C, so it has pointers.  However, to facilitate other features that were added to C++ it also added reference types.  A reference type is just an alias to an existing variable.  Internally it's a pointer, but you can't have a null reference, and you don't need to use the * operator to use the value.

</details>
<!--****************************-->
<details markdown="1">
<summary>Go</summary>
<br/>
Go, like C, is said to be value based. You can use **pointers** for "explicit" references.

But Go has the complication that there are a few types that have "hidden" pointers.  For example, internally maps are just pointers, and slices contain pointers, but it is easy to forget this as you don't need to explicitly dereference the pointer.

For this reason emphasizing that these types are reference types can be useful as a reminder to take care.  Unfortunately, there are 

**Definitions**

I have created a list of definitions I have found in various places.  (Some of these I had to insinuate from various discussions.)

0. Go does not have reference types
1. Any Go type that has a hidden pointer
2. Any type that can be assigned `nil`
3. Any type that can be created using `make`
4. Slice, map, channel, function, pointer
5. Slice, map, channel, interface, function
6. Slice, map, channel, interface
7. Slice, map, channel, function
8. Any above + structs, arrays, or interfaces that contain them
9. Anything that needs "deep" copy/compare

Note that when I talk about a function I mean the `func` type, which is officially called a **closure**.

**What's the best definition?**

We can discount definition 1, as the `string` type has a hidden pointer.  However, as they're immutable (you can't modify the string through the pointer) `string` behaviour is effectively indistinguishable from other primitive types (like int).

Similarly, definition 2 does not work as interfaces are not reference types.  Like strings, interface values are immutable (unless they contain a reference type - see definition 8).

Definition 3 is also wrong as, pointers and `func` types _are_ reference types.

If you insist I would go with definition 4, as they satisfy my steps for determining if a type is a reference type.

**My Steps**

The problem we are trying to highlight is that you can (shallow) copy a variable then inadvertently modify the original when you only think you are modifying the copy.

So I will use these steps to determine if a type is a reference type in Go.

1. Declare variables, `a` and `b`, of the type
2. Create/modify `a` in some way
3. Assign `a` to `b`
4. Modify `b` in a different way
5. Check if there is a discernible difference in `a`

</details>
<!--****************************-->
<br/>

# What are they?

The  **Background->Go** section above goes into detail on how I define "reference" variables.

In brief, it is any type of variable that, when copied, changes to the copy affect the original.  Indeed, it is this behaviour that has lead to many bugs.

**The important thing is to understand how they work, not what you call them.**
{: .notice--warning}

For example, I recently discovered a bug in my own code which caused a data race.  I had passed a map to a different go-routine, through a channel, rather than cloning it first.

So let's look at how different "reference" types work.  I'll also look at arrays (and interfaces), even though they ae not reference types.  Arrays, in particular, are often expected not to be value types (probably because of how they work in C).

## Slices

The following code demonstrates that slices **are** reference types. 

```go
	var a, b []int
	a = []int{1, 2, 3, 4}
	b = a
	b[1] = 1
	log.Println(a) // [1 1 3 4]
```
Remember that even though the contents of `a` can be modified using `b`, you can't change the length or capacity of `a` using `b` (or indeed, change the underlying array that `a` points to).  Perhaps you should think of slices as **"partial" reference types**.

<div style="float: right; width: 350px;"><img src="{{ site.url }}{{ site.baseurl }}/assets/images/ref-slice-copy.png"/></div>

Note: This diagram does not show the capacity of the slices.  They both have a capacity the same as the length (4).

## Not Arrays

```go
	a := [4]int{1, 2, 3, 4}
	b := a
	b[1] = 1
	log.Println(a) // [1 2 3 4]
```
<div style="float: right; width: 300px;"><img src="{{ site.url }}{{ site.baseurl }}/assets/images/ref-array-copy.png"/></div>

Arrays, on the other hand, are value types **not** reference types.

When you use an array you get a complete copy of it.
<br/>

## Channels

```go
	a := make(chan int, 2)
	b := a
	b <- 42
	a <- 1
	log.Println(<-a) // 42
```

<div style="float: right; width: 350px;"><img src="{{ site.url }}{{ site.baseurl }}/assets/images/ref-channel-copy.png"/></div>

Although it is not usually a source of bugs, channels **are** reference types.

## Maps

```go
	a := map[int]string{1: "one"}
	b := a
	b[1] = "42"
	log.Println(a) // map[1:42]
```

As you guessed, maps are reference types in much the same ways as channels.  Just about all Gophers encounter this at some point.

## Pointers

Pointers are reference types.

```go
	var a, b *int
	n := 1
	a = &n
	b = a
	*b = 42
	log.Println(*a) // 42
```
<div style="float: right; width: 200px;"><img src="{{ site.url }}{{ site.baseurl }}/assets/images/ref-pointer-copy.png"/></div>

With pointers, you must explicitly dereference the pointed to value (using the `*` operator) - but this is similar to accessing the values of a map or slice (using the indexing `[]` operation).

## Functions

The following code shows that a closure is a reference type since the value `m` is shared between `a` and `b`

```go
type myInt int

func (m *myInt) f() int {
	*m++
	return int(*m)
}

func main() {
	var a, b func() int
	m := myInt(1)
	a = m.f
	b = a
	b()
	log.Println(a()) // 3
}
```
If closures were value types, the code would print `2` not `3`.

## Not Interfaces

Despite claims to the contrary, interfaces are not reference types, otherwise the following code would print 42.

```go
    var a, b interface{}
    a = 1
    b = a
    b = 42
    log.Println(a) // 1
```

## Composite Types

Composite types are types composed of other types (ie, anything apart from the primitive types - numeric types, bool and string). Even composite types that are normally value types, can act like reference types if they contain a reference type.

```go
    type sp struct { p *int }
    n := 1
    a := sp{p: &n}
    b := a
    *b.p = 42
    log.Println(*a.p) // 42
```
Arrays, structs and interfaces if they _contain_ a reference, are also reference types, according to my definition.

# Does Go even have Reference Types?

As pointed out at [There Are No Reference Types in Go](https://www.tapirgames.com/blog/golang-has-no-reference-values) the concept of reference type does not appear in the spec. (since 2013).

Even Dave Cheney says "Go does not have reference variables" at [There is no pass-by-reference in Go](https://dave.cheney.net/2017/04/29/there-is-no-pass-by-reference-in-go).

My opinion, without getting into pendantic semantics is that the idea (as discussed above), is useful, at the very least as a reminder to take care when copying maps.

However, even with the strictest definition of a reference, Go does indeed have them.

BTW the strictest definition is, as in C++, that a reference is just an alias to an existing variable. Note that even reference types in Java fail this definition, since they can be null.

## Captured Variables

Here is an example of Go code with a reference variable.

```go
    i := 1
    func() {
        log.Println(i)
    }()
```

Here (on the 1st line) we create an integer variable `i`.  Then we use `i` (on the 3rd line), but this is not the same variable, even though it has the same name and _references_ the same value.

The `func` (starting on the 2nd line) is a closure that captures `i` by taking its address.  Any use of `i` in the function is by reference to the original `i`. 

Here's a complete example, which shows that `i` within the closure continues to exist after `ff()` returns and the original `i` is no longer in scope.

```go
func ff() func() {
  i := 1
  return func() {
    log.Println(i)
  }
}

func main() {
  f := ff()
  f()  // 1
}
```
Note that because a reference to 'i' is preserved within the closure, the original `i` is not stored on the stack.  The compiler's **escape analysis** reveals that it is needs to be placed on the heap.

# Deep Operations

While we are on the subject, I just want to explain what is meant by "deep" operations such as **deep copy** and **deep compare**.

When you copy or compare pointers in Go (or C) you are just using the pointer _values_. That is, you are only copying/comparing the memory address (the pointer's value), not the values pointed to.

This is called _shallow_ copying/comparing.  To use the values you must "dereference" the pointer, for a _deep_ operation.

For example, if two pointers point to different variables, they will **not** be equal, even if the variables pointed to have the same value.

```go
    n, m := 1, 1
    p, q := &n, &m
    log.Println(p == q)   // false (shallow compare)
    log.Println(*p == *q) // true (deep compare)
```

Because Go is generally a "value-based" language, when you copy (by assignment or passing as a parameter) or compare (using `==` or `!=`) the compiler always uses "shallow" operations.

In order to perform deep operations on "reference" types you generally need to code it "by hand" or call a function.

## Maps

You need to manually provide deep operations on maps, such as using a loop to copy elements.

Note that Go 1.21 (released soon) provides generic helper functions in the `maps` package: `maps.Copy` and `maps.Clone` to copy a map, and `maps.Equal`, etc to compare maps of the same type.

## Slices

The built-in `copy` function allows you to copy the _contents_ of slices, but this will not change the length of the destination slice.

To get an exact copy of a slice you must create a new slice (with the same length and capacity) then copy over all the elements using a `for ... range` loop.

Deep comparison of slices are done manually, though the standard library does provide `bytes.Equal` for comparing `byte` slices.

Note: like for maps, Go 1.21 will provide generic helpers: `slices.Clone`, `slices.Equal`, etc.

## Pointers

As we saw above, use the `*` operator for deep(er) operations.

## Channel, Function

It's not possible (or generally useful) to perform deep operations on these types.

## Composite Types

Deep operations on these types are only necessary if they _contain_ "reference" types.  In this case you need to manually code deep operations (also see `reflect.DeepEqual` below).

## Standard Library Functions

As mentioned there is a `bytes.Equal` function for comparing values of type `[]byte`.  It returns true if the slices have the same length and contents but ignores capacity.

The standard library also provides `reflect.DeepEqual` which usually works well for performing a deep comparison.  Since it uses reflection it may not be as efficient as a coded comparison, or a generic one.  It can also give strange results for unusual types.

With the advent of generics, the Go authors have created generic functions to perform deeper operations on maps and slices.  Note that the _elements_ themselves are "shallow" copied/compared - ie there is no recursive "depth" as with `reflect.DeepEqual`.

See the `Copy`, `Clone`, `Equal`, `Compare`, etc functions in [maps](https://pkg.go.dev/golang.org/x/exp/maps) and [slices](https://pkg.go.dev/golang.org/x/exp/slices) packages.  These generic `maps` and `slices` packages will be added to the Go standard library in Go 1.21.

# Comparability and Map Keys

The behaviour of "reference" types has lead to other behaviours of the Go language that can seem strange until you understand the reasons.

For example, the rules for comparability (use of `==` and `!=` operators) can seem inconsistent.  Why can you compare channels but not maps?

## Map Keys

From what I can gather, a lot of Go's inconsistent rules come down to the problem of making sure that you can't break maps.

Maps rely on their key values comparing consistently.  For example, to get back an element you added to a map the key value you supply must always compare equal to the key value you used to add the element.

If a key value changes (such that it no longer compares the same to other key values) then it can result in very strange behaviour.  (I've encountered this problem with maps, and their ilk, in C++.)

Map keys are the main reason for the comparability rules of Go.

```go
    // Invalid map keys - *** NOT VALID Go ***
    var a map[[]int]string        // slices are not comparable
    var b map[func()]sring        // funcs are not comparable
    var c map[map[int]bool]string // maps are not comparable
    var d map[chan int]string     // OK (chans are comparable)
```

## Comparability

If you are not familiar with the comparability rules of Go then in brief: there are three types that may **not** be compared: maps, slices and functions.  Moreover, structs, arrays, and interfaces that contain them, are also not comparable.

Ostensibly this is because they are "reference" types. But then why are pointers and channels comparable?

I think it's because it would be confusing if comparing slices and maps did not compare their "contents".  Comparing pointers and channels is less common or less likely to cause confusion.

I was going to explore this in depth, but I'm getting side-tracked.  It would be wordy to explain it thoroughly and this post is long enough - maybe later.

# Conclusion

I hope this was a useful explanation of how types containing internal pointers work.  This is what many people mean when they talk about "reference" types in Go.

It is especially important to understand:
 * slices always have an underlying array (or are nil)
 * a slice's underlying array may be used elsewhere
 * when you assign a map (or pass it as a parameter) you are not getting a copy of all the elements.
