---
layout: post
title: "Recursion with types"
description: ""
category: shapeless, scala
tags: []
---
{% include JB/setup %}

Shapeless is about providing evidence to the compiler. And the way of doing so, is using **typeclasses and recursion at compile time**. 

There is an operation that you can do on `HList`s, that shapeless calls `select`, and consists in, given an `HList` and some type `A`, we get the value out, providing that the product has the value of that type, e.g.:

~~~
("a" :: 1 :: HNil).select[Int] === 1
("a" :: 1 :: HNil).select[Double] // won't compile
~~~

In this post we'll do something less strict, let's write `find`, which, as `Seq.find` it will return `Some[A]` if the product has a value of type `A`, but `None` otherwise:

~~~
("a" :: 1 :: HNil).find[Int] === Some(1)
("a" :: 1 :: HNil).find[Double] === None

~~~

## Interface
Let's get the syntax out of the way:

~~~
object Find {
  implicit class Ops[L <: HList](l: L) {
      def find[A]:Option[A] = ???
  }
}
~~~

If you are not familiar what this technique, I recommend [this exellent post](http://aakashns.github.io/better-type-class.html) about type classes.

This gives us the above syntax, by importing `Find.Ops`, we can do:

~~~
import Find.Ops
(1 :: HNil).find[Int]
~~~

## The type class itself

As mentioned above, `Find` behavior is: 
- Given a `HList` `L` and some type `A`
- It returns `Some[A]` if there is a value of type `A` in `L`
- It returns `None` otherwise

Let's define a type class that expresses that operation:

~~~
trait Find[L <: HList, A]{
  def find(l:L):Option[A]
}

~~~

This definition of `Find` follows a common pattern in shapeless: Given some generic type, e.g. an `HList`, we get some output, in this case an `A`.

Let's modify `Find.Ops` to have its final version:


~~~
object Find {
  implicit class Ops[L <: HList](l: L) {
      def find[A](implicit f: Find[L, A]):Option[A] = f.find(l)
  }
}
~~~

The final code means that, in order to be able to use `find[A]` on an `HList`, there needs to be an implicit instance of `Find[A]` in scope.

If it's difficult to understand what has happened so far, I strongly recommend to read [the piece I linked before](http://aakashns.github.io/better-type-class.html).

## Implementations

Now we are getting to the "the chicken of the rice with chicken", as they say in my country. The instances of `Find`. This is where recursion comes into play. 

First let's see how can we implement `find` for a normal list:

~~~
def find[A](ls:List[A])(f: A => Boolean):Option[A] = ls match {
        case x :: xs if f(x) => Some(x)
        case _ :: xs => find(xs)(f)
        case Nil => None
    }
~~~

We are going to do the _exact same thing_ for our type level `Find[A]`.

Each instance of `Find[A]` will be the equivalent to one of those `case`s above. Let's start with the case of `Nil`.

~~~
implicit def hnil[A] = new Find[HNil, A] {
    def find(l: HNil) = None
  }
~~~

So, if `L` -- the `HList` in which we are trying to find `A` -- is the empty `HList`, `find` returns `None`.

Now, the instance where it returns `Some[A]`.

~~~
implicit def hconsFound[A, H, T <: HList](implicit ev: H =:= A) = new Find[H :: T, A] {
    def find(l: H :: T) = Some(l.head)
  }
~~~

This is loaded, with new stuff and with pretty much everything that shapeless is about. Look at the type of the `Find` instance. It's `Find[H :: T, A]`. Where `H` is some type, and `T` is some `HList`, so `H :: T` is the equivalent to `x :: xs`, **at the type level**. The type argument of `Find` let's it match with any `HList` that is not `HNil`, but that is not enough, remember that in the `find` on `List` the pattern matching also had the check of the boolean condition. In this case we don't have an arbitrary function, but only an equality comparison. `implicit ev: H =:= A` would be the equivalent as to say `ls.find(_ == j)`, again at the type level.

Sumarizing, this instance is used to implement `Find` if:
1 - `L` is not an `HNil`
2 - there is evidence that `A` and the head of `L` have the same types

If these conditions are met, then this is the instance that will be found on:

~~~
def find[A](implicit f: Find[L, A]):Option[A]
~~~

And therefore it will return `Some[A]` *always*. This is important to remember, it seems obvious to me now, but until recently it was very easy to forget that an `HList` is known *at compile time*. When your code is running, it already "knows" what instance is going to use for `Find`.


Let's now implement the last case:

~~~
implicit def hconsNotFound[A, H, T <: HList](implicit f: Find[T, A]) = new Find[H :: T, A] {
    def find(l: H :: T) = f.find(l.tail)
  }
~~~

As with the runtime example, if `L` is not empty, but the head is not the same type as `A`, we ignore the head, and continue looking. In order to do that, we need to have an instance of `Find` for the rest of the `HList`, for which we expect the `(implicit f: Find[T, A])`. If `T` is an `HNil`, the compiler will find the instance for `HNil`, otherwise will try to use the instance with the type equality, otherwise this one. Until `A` is found or `HNil` is reached.

Here is the companion object with everything together.

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

The actual implementation of this can be found in our library [typeless](https://github.com/xdotai/typeless)


