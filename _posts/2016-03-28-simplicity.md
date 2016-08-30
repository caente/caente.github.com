---
layout: post
title: "Comments on Simplicity"
description: ""
category:
tags: []
---
{% include JB/setup %}

Some time ago I was trying to write simple code and failing miserably. I was writing small methods, because that was the obvious thing to do, because all cool projects I've seen had small methods. I used to take a semi-arbitrary chunk of code, give it a name, some arguments, and be happy with it. It usually looked good, but time, i.e. changing requirements, always made those chunks obsolete too quickly. 

Nowadays I think small functions are a consequence, not a way, of achieving simplicity.

### Consider this lazy example

~~~
import org.joda.time.LocalDate
// this is how a date range is represented in the database
case class DateRange(start: LocalDate, end: LocalDate)
~~~

These are some of the operations we want for it:

~~~
def dayExists(dateRange:DateRange, d:LocalDate):Boolean  // checks if certain day exists in the range
def intersectRanges(dr1:DateRange, dr2:DateRange):Option[DateRange] // given two ranges, return the intersection, or nothing
def subtractRages(dr1:DateRange, dr2:DateRange):List[DateRange] // given two ranges, return whatever times are not in the second and are in the first one

~~~

Not going to attempt the implementation of these methods, since is pretty straight forward. I rather focus on an alternative:

~~~
def days(dateRange:DateRange):List[LocalDate] // returns all the days in a DateRange
def dateRange(days:List[LocalDate]):List[DateRange] // returns a list of DateRange's where the days are adjacent

~~~

A range of dates is a list of dates. It seems like `LocalDate` is the simplest abstraction of our problem. `DateRange` and `List[LocalDate]` are two ways of storing `LocalDate`s -- i.e. we can "move" from one to another. (I really hope this is not a "draw the owl" case)

All the desired operations are instantly supported by `List`, along with many others. Also `List[A]` has already a very clear api, and most readers of the code will be familiarized with it, whereas `DateRange` would require to someone to *memorize* yet another set of operations:


~~~
days(range1).exists(day) // dayExists
days(range1).intersectRanges(days(range2)) // intersect
days(range1).diff(range2) // subtractRages
~~~


Where to go from here? Do we still write the implementations of the api but using `days`? Well it depends, right now we have a low level api that allows to express anything we want, as long as it can be expressed as `List[LocalDate]`. We might want another layer on top of it, but why overthink it? I would go with `List[LocalDate]` until is obvious that certain operations are too common and can be grouped.

Having `LocalDate` as the main abstraction means that, if _performance_ becomes an issue, we could use `DateRange` or anything else that helps with it. But as it is, it makes it easy to understand that this api is about days/`LocalDate`. Whereas in the initial api the main abstraction was `DateRange` which is a very "opinionated" data structure, and ultimately a misleading one, since the problem is obviously about groups of days, not range of days.

### Actual recommendations

There are no obvious ways to simplify your code, these are some of the practices I follow to improve the odds:

1 -  I try to make my methods as context free as possible:

~~~
def dayExists(dateRange:DateRange, d:LocalDate):Boolean  
~~~

Is too "aware" of the fact that we are dealing with `DateRange`s, whereas a `List[LocalDate].exists` works as good, without any extra noise.

2 - I try to reduce the amount of "data preparation" that a method has to do:

Instead of:

~~~
def foo(i:Int, b:String):Int = {
    val bInt = b.toInt
    i + bInt
}
~~~

I rather provide the `Int`:

~~~ 
def foo(i:Int, b:Int):Int = i + b
~~~

... and make the transformation to `String` at the call site. If the pattern is repeated enough times(for me if is three times), I would consider make a third method that groups the transformation to `String` and the actual call to `foo`

3 - If I have a method that takes a container, e.g.


~~~
def foo(ls:List[Int]):List[String]

foo(List(1,2,3)) 
~~~

I would explore the possibility of make it about the contained elements:

~~~
def foo(i:Int):String

List(1,2,3).map(foo) // note this is similar to what we do with DateRange -> List[LocalDate]
~~~

---
This practices won't make the code simpler, but that they *help* to find simpler patterns in the code.

### Conclusions

There is nothing inherently wrong with `DateRange`. The point here is that using it as main abstraction will make our code less simple, since it is too high level. 

The code is the most reliable documentation you will ever have about the algorithms used on a project. If it's hard to follow, if the solution was found "by accident", and left like that; it will be very hard to know _why_ the system is doing certain thing in certain context, i.e. it will be hard to debug. It will also be very hard to extend or modify for the same reason. It's then when you start hearing the whispers in your head -- and in slack -- "refactoring, refactoring, refactoring". 

**Aug 2016**

