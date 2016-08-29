---
layout: post
title: "Comments on Simplicity"
description: ""
category:
tags: []
---
{% include JB/setup %}

Simplicity, as we all know, is important. You hear it all the time. But what is it? It seems to be related with small functions. All of the readable and simple code out there has small functions.

I think small functions are a consequence, not a way, of achieving simplicity in code.

Consider this lazy example:

~~~
import org.joda.time.LocalDate
// this is how a date range is represented in the database
case class DateRange(start: LocalDate, end: LocalDate)
~~~

These are some of the operations we want for it:

~~~
def dayExists(dateRange:DateRange, d:LocalDate):Boolean  // checks if certain day exists in the range
def intersect(dr1:DateRange, dr2:DateRange):Option[DateRange] // given two ranges, return the intersection, or nothing
def diff(dr1:DateRange, dr2:DateRange):List[DateRange] // given two ranges, return the ranges that are not shared

~~~

If we where to attempt the implementation of those methods, we would have some messiness initially, but a better abstraction would emerge:

~~~
def days(dateRange:DateRange):List[LocalDate] // returns all the days in a DateRange
def dateRange(days:List[LocalDate]):List[DateRange] // returns a list of DateRange's where the days are adjacent

~~~

We would have the obvious realization that a range of dates is a list of dates, and that `LocalDate` is the simplest abstraction of our problem. `DateRange` and `List[LocalDate]` are two isomorphic ways to store `LocalDate`s -- i.e. we can "move" from one to another. (I really hope this is not a "draw the owl" case)

All the desired operations are instantly supported by `List`, along with many others. Also `List[A]` has already a very clear api, and most readers of the code will be familiarized with it, whereas `DateRange` would require to someone to *memorize* yet another set of operations:


~~~
days(range1).exists(day) // dayExists
days(range1).intesect(days(range2)) // intersect
days(range1).diff(range2) // diff
~~~


Where to go from here? Do we still write the implementations of the api but using `days`? Well it depends, right now we have a low level api that allows to express anything we want, as long it can be expressed as a `List[LocalDate]`. We might want another layer on top of it, but why over think it? I would go with `List[LocalDate]` until is obvious that certain operations are too common and can be grouped.

Having `LocalDate` as the main abstraction means that if _performance becomes an issue_, we could use `DateRange` or anything else that helps with it. But as it is, it makes it easy to understand that this api is about days/`LocalDate`. Whereas in the initial api the main abstraction was `DateRange` which is a very "opinionated" data structure.

Unfortunately we usually work with conflicting requirements, usually described in a very imperative manner -- i.e. a flow chart. That, with some tight time constraints, makes us to go ahead with whatever abstraction was initially obvious, and squeeze the solution out of it. The code then will not represent a solution of the problem, but rather it'll be some kind of "mechanical accident" that happens to produce the solution you are looking for. We would have gone with `DateRange` and its api. It would have worked, but our code would be hard to read, it'll be very hard to know *how* is it solving the problem.

There is no obvious ways to simplify your code, but you could use some things I do to improve the odds:

1 -  Try to make your methods as context free as possible. In the example above we would be using `List` to do operations about dates, it can't be more agnostic than that.

2 - I try to reduce the amount of "data preparation" that a method has to do:

Instead of:

~~~
def foo(i:Int, b:String):Int = {
    val bInt = b.toInt
    i + bInt
}
~~~

I rather do:

~~~ 
def foo(i:Int, b:Int):Int = i + b
~~~

3 - If I have a method that takes a container, e.g.


~~~
def foo(ls:List[Int]):List[String]
~~~

I would explore the possibility of make it about the contained elements:

~~~
def foo(i:Int):String
~~~

### Conclusions


The code is the most reliable documentation you will ever have about the algorithms used on the project. If it's hard to follow, if the solution was found "by accident", and left like that; it will make it very hard to know _why_ the system is doing certain thing in certain context, i.e. will be hard to debug. It will also be very hard to extend or modify for the same reason. It's then when you start hearing the whispers in your head -- and in slack -- "refactoring, refactoring, refactoring". 
