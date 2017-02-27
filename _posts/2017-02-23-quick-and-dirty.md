---
layout: post
title: "Quick and dirty, Generic Programming with Shapeless"
description: "Generic Programming with Shapeless"
category: 
tags: []
---
{% include JB/setup %}

### The Problem

This is an actual problem we had at work, it was related to the data-science pipeline. If you want to find cool stuff to work on, you should try to help the data scientists, they have the coolest and craziest requirements. The craziness of the requirements might imply design issues, but that would be outside the scope of this piece.

We needed that, given a list of functions, e.g.:

~~~
val functions =
      { (x: String) => x.size } ::
      { (x: String, i: Int) => s"$x + $i" } ::
      HNil
~~~

Apply all the functions that can use the provided arguments, which might vary from call site to call site, but the list of functions will always be the same, or not, e.g.:

~~~
applyAll(1,"hi")(functions) === 2 :: "hi + 1" :: HNil

applyAll("hi")(functions) === 2 :: HNil
~~~


## The Solution

The way we approached the problem was:

1 - A function has an argument set

2 - `applyAll` will receive an argument set

3 - Given a function, if its arguments are a subset of the given arguments, select them and apply them to the function

4 - Accumulate the results

We decided to use _types_ as a way to find the subset of arguments. The tool of choice was obviously shapeless.

#### The shapeless way

When working with shapeless, I've found that there are two aspects that need to be part your mindset: 

1 - You use typeclasses to _prove_ stuff to the compiler

2 - Recursion thinking is mandatory -- I enjoy recursive algorithms 


#### The entry point

With that in mind, let's start by defining `applyAll`:

~~~
def applyAll[FF <: HList, R <: HList](args:???)(functions:FF):R
~~~

So far we know that `functions` most be an `HList`, and that the return type is an `HList`. But what can `args` be?

The usual solution for a variable list of arguments is to do something like:

~~~
@ def foo[A](args:A*):Int = as.size
defined function foo
@ foo(1)
res1: Int = 1
@ foo(1,2)
res2: Int = 2
@ foo(1,2,"a")
res3: Int = 3
~~~

The problem with that is that `args` would need to be of the same type:

~~~
@ def foo[A](as:A*):List[A]= as.toList
defined function foo
@ foo(1,2)
res5: List[Int] = List(1, 2) // all OK
@ foo(1,2,"a")
res6: List[Any] = List(1, 2, a) // types our of the window
~~~

There is another way, kinda weird, but we are on the fringe of Scala at this point:

~~~
@ def foo[A <: Product](args:A) = args
defined function foo
@ foo(1,2)
res8: (Int, Int) = (1, 2) 
@ foo(1,2,"a")
res9: (Int, Int, String) = (1, 2, "a") // all types are here still!
~~~

That's right, tuples are `Product`s, and Scala syntax allows to pass a tuple to a function without the extra parenthesis, I heard this might be deprecated though. That also mean that `foo` can take a case class as argument:

~~~
@ case class Arg(i:Int)
defined class Arg
@ foo(Arg(1))
res11: Arg = Arg(1)
~~~

So the signature of `applyAll` looks like this:

~~~
def applyAll[FF <: HList, R <: HList, Args <: Product](args:Args)(functions:FF):R
~~~

### Proofs

Let's try to implement `applyAll`. We have a set of functions, and a set of arguments, we need to, for each function, "try" to apply the arguments.

In a dynamically typed language, of if all items were of the same type, our solution might look like this:

~~~
def applyAll(args)(functions) = 
   functions match {
     case x :: xs if canApply(x, args) => x(args) :: applyAll(args)(xs)
     case _ :: xs => applyAll(args)(xs)
     case Nil => Nil
   }
~~~



