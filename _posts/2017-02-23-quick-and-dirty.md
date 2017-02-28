---
layout: post
title: "Quick and dirty, Generic Programming with Shapeless"
description: "Generic Programming with Shapeless"
category: 
tags: []
---
{% include JB/setup %}

### Introduction

In this piece I'll try to explain by example, how I use shapeless to bend the will of Scala :-)

#### Required skills

1 - recursion

2 - implicit classes 

3 - knowing the existence of shapeless and `HList`s

### The Problem

This is an actual problem we had at work, it was related to the data-science pipeline.

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

### Type classes as Proofs

#### Generic

First, we need to find the generic representation of our arguments, it doesn't matter if they are in a tuple or a case class, because what we'll work with is an `HList`, in order to do that, we can use the typeclass `Generic`:

~~~
def applyAll[Args <: Product, HArgs <: HList, FF <: HList, R](args: Args)(fs: FF)(
    implicit
    gen: Generic[Args]
  ) = ???
~~~

The `Generic` trait/typeclass looks like this:

~~~
trait Generic[A]{type Repr}
~~~

It takes some type `A` and it returns the generic representation of that type in `Repr`, e.g.:

~~~
scala> :t Generic[(Int,String)]
shapeless.Generic[(Int, String)]{type Repr = shapeless.::[Int,shapeless.::[String,shapeless.HNil]]}
~~~

That means that when we want to find the generic representation of a tuple `(Int,String)`, what we get as a returned type is `Int :: String :: HNil`, the same can be said of a case class:

~~~
val hl = (1,"a")
Generic[(Int,String)].to(hl) === 1 :: "a" :: HNil

case class Foo(i:Int,s:String)
Generic[Foo].to(Foo(1,"a"))  === 1 :: "a" :: HNil

~~~

Now there is a common "interface" to reason about.

#### Type level recursion

Shapeless is about providing evidence to the compiler. And the way of doing so, is using typeclasses and **recursion**, being recursion the corner stone, with the added weirdness that it happens at _compile time_. So let's spend some time understanding it. For that, let's write a typeclass for finding members, by type, in a case class (this typeclass *will not be used* on the problem, it's just a simpler example) e.g.:

~~~
case class Foo(i:String, d:Int)

Foo("a",1).find[Int] === Some(1)
Foo("a",1).find[Double] === None

~~~

Let's get the syntax out of the way first:

~~~
object Find {
  implicit class Ops[L <: HList](l: L) {
      def find[A](implicit f: Find[L, A]):Option[A] = f.find(l)
  }
}
~~~

In words: Given an heterogenous list `L`, there has to be an way to "find" `A` in `L`. 

Now let's implement `Find` itself. 

Based on `Find.Ops`, we need something like this:

~~~
trait Find[L <: HList, A]{
  def find(l:L):Option[A]
}
~~~

This definition of `Find` follows a common pattern in shapeless: Given some generic type, e.g. an `HList`, we get some output, in this case an `A`.

~~~
object Find {
  implicit class Ops[L <: HList](l: L) {
    def find[A](implicit f: Find[L, A]) = f.find(l)
  }
  implicit def hconsFound[A, H, T <: HList](implicit ev: H =:= A) = new Find[H :: T, A] {
    def find(l: H :: T) = Some(l.head)
  }
  implicit def hconsNotFound[A, H, T <: HList](implicit f: Find[T, A]) = new Find[H :: T, A] {
    def find(l: H :: T) = f.find(l.tail)
  }
  implicit def hnil[A] = new Find[HNil, A] {
    def find(l: HNil) = None
  }
}
~~~






---
Let's try to implement `applyAll`. We have a set of functions, and a set of arguments, we need to, for each function, to "try" to apply the arguments.

In a dynamically typed language, of if all functions and arguments were of the same type, our solution might look like this:

~~~
def applyAll(args)(functions) = 
   functions match {
     case x :: xs => applyFunction(args)(x) ++ applyAll(args)(xs)
     case Nil     => Nil
   }
~~~

Where `applyFunction` is the magic sauce that, if the arguments are present, it returns `Some(result)`, otherwise `None`.

In order to do the same with heterogenous functions and arguments, we need to provide some evidence to the compiler that it can do certain things.
