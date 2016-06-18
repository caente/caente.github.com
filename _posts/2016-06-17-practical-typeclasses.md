---
layout: post
title: "Practical Typeclasses - WIP"
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

The only way to know for sure is to look into the source code. This example is trivial, but when most of your methods receive big objects like `Email`, it's hard to know what their responsibilities are, since there will be occasions when the name will not be very clear.

### Alternatively

There are two approaches that I actually recommend over typeclasses for most situations.

#### No semantics

We don't care about the "meaning" of the  `DateTime` that we will pass to `isRecent`.

~~~
case class Context(now:DateTime){
  def isRecent(timestamp:DateTime):Boolean = timestamp.isBefore(now)
}
~~~

This works, but it removes domain constraints from our program.

#### Wrapper class

We create a case class to wrap the `DateTime`, and use it for `receivedDate` instead of `DateTime`

~~~
case class Timestamp(t:DateTime)
case class Email(
                .
                .
                .
                receivedDate: Timestamp
                )
ase class Context(now:DateTime){
  def isRecent(timestamp:Timestamp):Boolean = timestamp.t.isBefore(now)
}
~~~

That would preserve the semantics, and it adds some "documentation". The problem is that it's just a wrapper, and at the call site is possible to do something like `isRecent(Timestamp(someDate))`, which kind of defeats the purpose. If at the call site you don't care  about the right `DateTime`, then why would you care in `isRecent`?

# Typeclasses for semantics with safety

The typeclass will "wrap" `Email`, and the call site will be something like:

~~~
context.isRecent(email)
~~~

Which is rather convenient, and it looks exactly like the first version. The implementation is as follows:

~~~
case class Context(now:DateTime){
  def isRecent[T:Timestamp](t:T):Boolean = t.timestamp.isBefore(now)
}
~~~

- There is no reference to `Email`
- `T` is some type, nothing more
- `Timestamp` is a typeclass -- I strongly recommend that typeclasses do _one_ thing

With a typeclass we can define the expected property of the argument. 

___
>>>>>>>`T:Timestamp` is the same as adding `(implicit ts:Timestamp[T])` 

___

Even without being very familiar with context bounds, it's kind of obvious that the only thing the method "knows" about `T` is that it has a `Timestamp`.


### How to write a typeclass

If you are interested in doing the above, you will quickly notice that there is no way `T` will have a *member* called `timestamp`, because, well, it's just a generic type. It has stuff like `toString` and `equals`, because java, but that's about it.

This is how we "declare" the typeclass, just a trait, and a method. The parameter `T` is what ever will be "wrapped" by `Timestamp`.

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
~~~

Now we have the instances we want for timestamp, i.e. the data types that have the "property" `Timestamp`. As for adding the `timestamp` "member", we can add a `Syntax` implicit class.

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

The implicit class "wraps" _any_ type, and will throw a compile error if the method `timestamp` is invoked on a datatype with no instance of `Timestamp`.

~~~
import Timestamp.Syntax
case class Context(now:DateTime){
  def isRecent[T:Timestamp](t:T):Boolean = t.timestamp.isBefore(now)
}
~~~

# Practicality

For the case when `isRecent` just receives a `DateTime`, the call site seems very elquent:

~~~
isRecent(email.receivedDate)
~~~

In this case however:

~~~
isRecent(email)
~~~

We cannot know at first glance what property of `Email` is being used. So in a way, it can be considered less simple or less readable. But, if someone would want to know what `isRecent` does, they wouldn't need to look into the source code, but rather, only the signature, and they would need to find an instance of `Timestamp` for the datatype.

So the trade off seems to be: Lose readability at the call site, but make the methods easier to understand and learn. And also add more domain constraints to your code, making it closer to be correct.


# Bonus

### What happens with a sealed trait

If you are interested in how to make instances of sealed traits, the most straightforward method is with shapeless. First, you need to create the instances for each of the "children" of the sealed trait, and include this on the companion object `Timestamp`.

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

There is _a lot_ going on there, enough for another post. Shapeless is a very indispensable tool to write this kind of code in scala, aka safer and more correct code.
Since you are basically _proving_ constraints of your domain to the compiler.

#### References

- [Parametricity, Types are Documentation - Tony Morris](https://www.youtube.com/watch?v=BtEEZa_Q8Vw)
- [Theorems for free - Philip Walder](https://www.mpi-sws.org/~dreyer/tor/papers/wadler.pdf)
- [Scrap Your Type Class Boilerplate - Aakash N S](http://aakashns.github.io/better-type-class.html)
