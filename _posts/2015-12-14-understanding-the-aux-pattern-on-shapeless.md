---
layout: post
title: "Understanding the Aux pattern on shapeless"
category: Functional Programming
tags: [scala, shapeless]
---
{% include JB/setup %}

This is an exploration of shapeless, by trying to solve an actual use case that
I happen to have in my company.

There is an API that needs to return a data structure back to the caller,
this data structure can be modeled as a case class:
~~~
case class Time(millis: Long)
case class Location(place: String)
sealed trait Value
case class ValueLocation(detail: String)
case class ValueTime(after: Long)

case class Response(
  preferedTimes: List[Time],
  preferedLocations: List[Location],
  commonValues: List[Value]
  )

val data: List[Data] = List(
    TimesFromHome(
      times = List(
        Time(1L),
        Time(2l)
      ),
      value = ValueTime(100L)
    ),
    TimesFromHome(
      times = List(
        Time(5L)
        ),
      value = ValueTime(50L)
    ),
    TimesAndLocationsFromOutside(
      times = List(
        Time(1L),
        Time(2l)
        ),
      location = Location("some place"),
      values = List(
        ValueTime(100L),
        ValueTime(50L),
        ValueLocation("deep inside")
      )
    ),
    LocationsAheadOfTime(
      locations = List(
        Location("some other place"),
        Location("yet another")
      ),
      value = ValueLocation("deep inside")
    )
  )

~~~

All its fields have different types. This isn't by design,
it happened to be the case. And that's what allows the solution I'm gonna try.

So the "problem" is that different subsets of `preferedTimes`, `preferedLocations`,
and `commonValues` are created by different parts of the system. So I could have:

~~~


sealed trait Data
case class TimesFromHome(times: List[Time],value: Value) extends Data
case class TimesAndLocationsFromOutside(times: List[Time], location:Location, values: List[Value]) extends Data
case class LocationsAheadOfTime(locations:List[Location], value: Value ) extends Data
  .
  .
  .
and so on

~~~

In order to create the `Response` object I'd like to do something like this:

~~~

Response(
  times = data.flatMap( d => times.filter( d ) ),
  locations = data.flatMap( d => locations.filter( d ) ),
  values = values.flatMap( d => values.filter( d ) )
  )

~~~

Where `times.filter`, `locations.filter`, and `values.filter` are like
"queries" that extract the desired types from a hierarchy of sealed traits/ case classes.

In order to gain enough abstraction to make the type filters, I need to think
in terms of products and coproducts.

A product can be understood as a union of values or types, e.g. :

~~~
val t:(Int, String, Double) = (1, "a", 3D)
case class Foo(i:Int, s: String, d: Double)
~~~

and coproduct is something that can only be either of certain types or values:

~~~
sealed trait Foo
case object A extends Foo
case object B extends Foo
~~~

Shapeless abstracts both concepts by having the types `Coproduct` and `HList`. `HList`
is the equivalent to product, not sure why the name, I guess `Product` was already taken.

Lets try some operations with an `HList`.

~~~
@ import shapeless._
import shapeless._
@ val ls = 1 :: "a" :: 3D :: HNil
ls: Int :: String :: Double :: HNil = ::(1, ::("a", ::(3.0, HNil)))
@ ls.filter[Int]
res2: Int :: HNil = ::(1, HNil)
@ ls.filter[Int].to[List]
res3: List[Int] = List(1)
~~~

You'll notice that `filter` on the `HList`. It is filtering at the type level.
It doesn't care about the values them selves, it can only filter by type.

Most of the logic that you code in shapeless is for the compiler,
not runtime.[[1]](https://gitter.im/milessabin/shapeless?at=5665d5ef835961e946e1be6d)

We can do similar operations with a coproduct.

~~~
@ import shapeless._
import shapeless._
@ type F = Int :+: String :+: Double :+: CNil
Defined type alias F
@ f.filter[Int]
res11: Option[shapeless.:+:[Int,shapeless.CNil]] = Some(1)
@ f.filter[Int].map(_.unify)
res12: Option[Int] = Some(1)
~~~

Shapeless provides a mechanism for "elevating" scala types to products and
coproducts abstractions, by using `Generic`.

For a product:

~~~
@ import shapeless._
import shapeless._
@ case class F(i: Int, s:String, d: Double)
defined class F
@ Generic[F].to(F(1, "a", 3D))
res2: Int :: String :: Double :: HNil = ::(1, ::("a", ::(3.0, HNil)))
~~~

For coproduct:

~~~
@ sealed trait F
defined trait F
@ case class A() extends F
defined class A
@ case class B() extends F
defined class B
@ Generic[F].to(A())
res6: A :+: B :+: CNil = Inl(A())
~~~

My solution for the "queries" starts with a trait that can filter by a type `A`

~~~

trait Field[A]{
    def filter:List[A] = ???
}
~~~

So I can make an instance for an specific type this way:

~~~
object ints extends Field[Int]
object strings extends Field[String]
~~~
