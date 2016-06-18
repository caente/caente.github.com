---
layout: post
title: "Practical Typeclasses"
description: ""
category: Scala
tags: [scala, type, typeclass]
---
{% include JB/setup %}
# Motivation


When programming, we usually need to write a method that has a very strong domain semantics, for instance:

~~~
case class Context(now:DateTime){
  def isRecent(email:Email):Boolean = ???
}
~~~

This method is responding to a "question": is this `Email` recent? But that `Email` has many fields, which one is going to be used here?

~~~
  def isRecent(email:Email):Boolean = email.sentDate.isBefore(now)
~~~

or

~~~
  def isRecent(email:Email):Boolean = email.receivedDate.isBefore(now)
~~~

or even

~~~
  def isRecent(email:Email):Boolean = email.from.broker.isDefined
~~~

The only way to know for sure is to look into the source code. This example is trivial, but is not hard to imagine how difficult to understand can be a code where these huge objects are passed around.

### Alternatively

A very common approach, that I actually recommend over what's next, is to strip the method of semantis:

~~~
case class Context(now:DateTime){
  def isRecent(timestamp:DateTime):Boolean = timestamp.isBefore(now)
}
~~~

This works, but in some occasions is not as desirable. 

# Bring the semantics back

For those cases where we do want to preserve the semantics of the arguments, we can use typeclasses.

~~~
case class Context(now:DateTime){
  def isRecent[T:Timestamp](t:T):Boolean = t.timestamp.isBefore(now)
}
~~~

- `T` is some type, nothing more
- `Timestamp` is a typeclass -- I strongly recommend that typeclasses do _one_ thing

With a typeclass we can define the expected property of the argument. Even without being very familiar with context bounds -- `T:Timestamp` is the same as adding `(implicit ts:Timestamp[T])` -- it's kind of obvious that the only thing the method "knows" about `T` is that it has a `Timestamp`.


### The practical part

If you are interested in doing the above, you will quickly notice that there is no way `T` will have a *member* called `timestamp`, because, well, it's just a generic type. It has stuff like `toString` and `equals`, because java, but that's about it.

So let's make the typeclass "pretty".

This is how we "declare" the typeclass, just a trait, and a method. The parameter `T` is what ever will implement this typeclass.

~~~
 trait Timestamp[T]{
  def timestamp(t:T):DateTime
 }
~~~

We put the implementations in the companion object.

~~~
 object Timestamp{
  def apply[T:Timestamp]:Timestamp[T] = implicitly[Timestamp[T]] // for easy "invocation" of the instances

  implicit object email extends Timestamp[Email] {
    def timestamp(t:Email):DateTime = t.receivedDate
  }

  implicit object timeEntity extends Timestamp[TimeEntity] {
    def timestamp(t:TimeEntity):DateTime = t.timestamp
  }
 }
~~~

Now we have the instances we want for timestamp, i.e. the data types that have the "property" `Timestamp`. As for adding the `timestamp` "member", we can add `Syntax` implicit class.

~~~
 object Timestamp{
 .
 .
 .
  implicit class Syntax[T](t:T){
    def timestamp(implicit ts:Timestamp[T]) = ts.timestamp(t)
  }

 }
~~~

The implicit class wraps _any_ type, and will throw a compile error if the method `timestamp` is invoked on a datatype with that has no instance of `Timestamp`

~~~
import Timestamp.Syntax
case class Context(now:DateTime){
  def isRecent[T:Timestamp](t:T):Boolean = t.timestamp.isBefore(now)
}
~~~

# Bonus

### What happens with a sealed trait

If you are interested in how to make instances of sealed traits, well you can go the shapeless way. First, you need to create the instances for each of the "children" of the sealed trait, and include this on the companion object `Timestamp`.

~~~

import shapeless._

implicit object cnil extends Timestamp[CNil] {
        def timestamp( t: CNil ) = throw new RuntimeException( "Hell NO!" )
}

implicit def coproduct[H, C <: Coproduct]( 
              implicit 
              tcHead: Timestamp[H], 
              tcTtail: Timestamp[C] 
          ): Timestamp[H :+: C] = new Timestamp[H :+: C] {
            def timestamp( t: H :+: C ) = t match {
              case Inl( head ) => tcHead.timestamp( head )
              case Inr( tail ) => tcTtail.timestamp( tail )
            }
          }

~~~

There is _a lot_ going on there, enough for another post, but enough is to say, if you want the compiler to help you to document and make your code safer, shapelss will be a great too for that.

#### References

- [Parametricity, Types are Documentation - Tony Morris](https://www.youtube.com/watch?v=BtEEZa_Q8Vw)
- [Theorems for free - Philip Walder](https://www.mpi-sws.org/~dreyer/tor/papers/wadler.pdf)
- [Scrap Your Type Class Boilerplate - Aakash N S](http://aakashns.github.io/better-type-class.html)
