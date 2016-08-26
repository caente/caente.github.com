---
layout: post
title: "Practical Typeclasses - WIP"
description: ""
category: Scala
tags: [scala, type, typeclass]
---
{% include JB/setup %}

# Intended audience

You are aware of the existence of typeclasses, but you are not sure where or when to use them, and you are mainly writing production code. This post is to provide some guidance for how and why use typeclasses. 

You should be familiar with scala, with traits, companion objects and similar machinery.

Inspired by [Strategic Scala Style: Designing Datatypes](http://www.lihaoyi.com/post/StrategicScalaStyleDesigningDatatypes.html), I'll be using _datatype_ when referring to instances of classes and other data-like types, and _type_ for generic types like `T`.

This piece **does not** provide a nuanced guide to write typeclasses, for that you should read the excellent article **Scrap Your Type Class Boilerplate** linked in the [References](#references) section. The goal is to provide a rationale as to why and how use typeclasses.

# The problem

A Point of Sale system for toy stores(own by the government):

- Each child can only have 2 toys a year
- To enforce the above restriction, the parents receive a yearly book with coupons
- The coupons are not only for toys, but also for clothing, or any factory made item.
- The government is trying a pilot program to decide if the coupons book can be automatized. It's starting with toys.

The users of our system are the employees of the stores. They need to:

1. be assholes

~~~
case class ToyBought(toy:Toy, child:Child, timestamp:DateTime)

case class Toy(_id:Toy.Id, name:String)
object Toy{
  case class Id(repr:String)
}

case class Child(_id:Child.Id, name: String)
object Child{
  case class Id(repr:String)
}

sealed trait ToyOrderResult
case class ToyOrder(toy: Toy, child:Child) extends ToyOrderResult

def previousToys(child:Child):Seq[ToyBought]

def buyToy(child:Child, toy:Toy, toyHistory:Seq[ToyBought]):ToyOrder

def assignToy(child:Child, toyOrder:ToyOrder):ToyBought
~~~



Let's assume we have a class 

~~~
case class Email(
  receivedDate:DateTime,
  body:String,
  sentDate:DateTime,
  from:EmailPerson,
  to: Seq[EmailPerson]
  .
  .
  .
  etc.
)
~~~

If we want to know if an email is recent, we could do this:

~~~
def isRecent(referenceDate:DateTime, email:Email ):Boolean = email.receivedDate.isBefore( referenceDate )
~~~

And that's a kind code that I know exists in many codebases, in most of them.

We don't want that. We don't want to pass the _whole_ `Email` object to a method. The object has way too much information in it, more than `isRecent` needs anyway. Besides, what happens if (when) we want to know if a `Person` is recent? or a `CalendarEvent`? The only thing we need there is a `DateTime`. 

### Solutions with no typeclasses

#### No semantics

If is truly just a `DateTime` we need -- i.e. some data without any domain meaning -- we could, and **we should**, do this:

~~~
def isRecent( referenceDate:DateTime, timestamp:DateTime ):Boolean = timestamp.isBefore( referenceDate )
~~~

I recommend this approach 90% of the time. Write your methods/functions without any assumptions about their arguments. 

#### Wrapper class

If we want to preserve the "domain information" on the method signature, we can create a case class to wrap the `DateTime`, and use it for `receivedDate`.

~~~
case class Timestamp( t:DateTime )

case class Email( receivedDate: Timestamp )

def isRecent( referenceDate:DateTime, timestamp:Timestamp ):Boolean = timestamp.t.isBefore( referenceDate )
~~~

That would certainly preserve the "meaning" of `receivedDate`. It will help to anybody reading the method signature. But then you'll need to change `Email` and all usages of `receivedDate` in your code. Also, if it's just a wrapper, at the call site is possible to do something like `isRecent( someDate, Timestamp( receivedDate ) )`, which defeats the purpose. If at the call site you don't care about what `DateTime` means, then why would you care inside `isRecent`?

### What about inheritance?

Inheritance is probably the de facto solution for most developers not familiar with typeclasses. It allows to do most things you can do with typeclasses although with more entanglement in my opinion.

~~~
trait Timestamp{ def timestamp:DateTime }

case class Email( _id: Id, timestamp:DateTime ) extends Timestamp 

def isRecent(referenceDate:DateTime, t:Timestamp ):Boolean = t.timestamp.isBefore( referenceDate )
~~~

I recommend [this](https://www.reddit.com/r/scala/comments/3bh5g8/what_makes_type_classes_better_than_traits/) interesting thread on reddit about typeclasses vs inheritance.

# Typeclasses for semantics with safety

The main point of using typeclasses is that allows to write methods that only take type parameters. 

This how `isRecent` would be written using a typeclass called `Timestamp`.

~~~
def isRecent[T:Timestamp]( referenceDate:DateTime, t:T ):Boolean = t.timestamp.isBefore(referenceDate)
~~~

Technically `t:T` is the same as `t:Any`, but we won't focus on that. Let's assume that `t:T` has no concrete type, as it should be. With that in mind, from the signature we can learn that:
1 - `t` has _one_ "property" provided by `Timestamp`
2 - `Timestamp` is a typeclass -- Although we would need to visit the implementations of `Timestamp` to know exactly what how is implemented for `Email`, it's easier to have a good name for a property than for business logic.

Is **better** than:

~~~
//No semantics
def isRecent( referenceDate:DateTime, receivedDate:DateTime )
~~~

and

~~~
//Inheritance or wrapper
def isRecent( referenceDate:DateTime, receivedDate:Timestamp ) // where Timestamp is a trait or abstract class or a wrapper class
~~~

Of course there are always caveats. But the first version, the one with type parameters, is saying that there is some type `T`, that has _one_ property: `Timestamp`. It is similar to the wrapper/inheritance version. Signature wise that is true. But the type parameter version is more scalable, for example:

If we need to add functionality, for example, if it's important who sent the email, we could just add another typeclass: 

~~~
def isRecent[T:Timestamp:Speaker]( referenceDate:DateTime, t:T ):Boolean
~~~

It is possible to do this with inheritance (the wrapper just won't cut it):

~~~
def isRecent(referenceDate:DateTime, t:Timestamp with Speaker):Boolean
~~~

But at the cost of _modifiying_ the `Email` class. The same goes with any other data structure that we want to use on this method. This is a problem for two reasons:

1 - We might **not control** the datatypes we are using

2 - As said above, we are creating too much entanglement, this is less obvious, but as the requirements change, any entanglement becomes a bottleneck on development.

~~~
def isRecent[T:Timestamp]( referenceDate:DateTime, t:T ):Boolean = t.timestamp.isBefore( referenceDate)
~~~
This is what we know about the method:

- `T` is some type, nothing more
- `Timestamp` is a typeclass -- I strongly recommend that typeclasses do only _one_ thing

With a typeclass we can define the expected property of the argument. 

___
>`[T:Timestamp]` is the same as adding `( implicit ts:Timestamp[T] )` to the method signature
>
> You also can have several typeclasses stacked up,
`[T:Timestamp:Speaker]` would be equivalent to `( implicit ts:Timestamp[T], sp:Speaker[T] )`

___


Even without being very familiar with context bounds in scala, it's kind of obvious that the only thing we know about `T` is that it has a `Timestamp`, and we know this by _looking at the signature_.


### How to write a typeclass

If you are interested in doing the above, you will quickly notice that there is no way `t` will have a *member* called `timestamp`, since it's just a generic type. It has stuff like `toString` and `equals`, because java, but that's about it. We'll get there, but first let's create the typeclass.

This is how we will "declare" the typeclass, just a trait, and a method. The parameter `T` is what will be "wrapped" by `Timestamp`.

~~~
 trait Timestamp[T]{
  def timestamp( t:T ):DateTime
 }
~~~

We put the implementations in the companion object.

~~~
 object Timestamp{
  def apply[T:Timestamp]:Timestamp[T] = implicitly[Timestamp[T]] // for easy "invocation" of the instances

  implicit object emailTimestamp extends Timestamp[Email] {
    def timestamp( t:Email ):DateTime = t.receivedDate
  }
}
~~~

Now we have the instances we want for timestamp, i.e. the data types that have the "property" `Timestamp`. As for adding the `timestamp` "member", we can add a `Syntax` implicit class.

~~~
 object Timestamp{
 .
 .
 .
  implicit class Syntax[T]( t:T ){
    def timestamp( implicit ts:Timestamp[T] ) = ts.timestamp( t )
  }
 }
~~~

The implicit class wraps _every_ type, and throws a compile error if the method `timestamp` is invoked on a datatype with no instance for `Timestamp`.

~~~
import Timestamp.Syntax
def isRecent[T:Timestamp](referenceDate:DateTime, t:T ):Boolean = t.timestamp.isBefore( referenceDate )
~~~

# Conclusions

For the case when `isRecent` just receives a `DateTime`, the call site is very elquent, we can tell what property of `Email` is being used.

~~~
isRecent( someDate, email.receivedDate )
~~~

In this case however(with the typeclass):

~~~
isRecent( someDate, email )
~~~

We cannot know how `isRecent` is using `Email`. So in a way, it can be considered harder to read. I rather think that the implementation detail is not leaking. On the other hand, the signature is enough to find out that information, the name helps, but is not the only source. 

The tradeoffs of typeclasses seems to be: hide information at the call site, but make the methods easier to understand and learn. And also add more domain constraints to your code at the type level, making it closer to correctness, since you need to know _at compile time_ if the object that you are passing to the method is "allowed".

I strongly recommend to read **Scrap Your Type Class Boilerplate** in the [References](#references) section, for a better understanding of the scala machinery for typeclasses.

# Bonus

### What can we do with a sealed trait

If you are interested in how to make instances of sealed traits, the most straightforward method is with shapeless. First, you need to create the instances for each of the "children" of the sealed trait, and then include this on the companion object `Timestamp`.

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

There is _a lot_ going on there, enough for another post, but if you are interested in this kind of programming you will need shapeless.

# References

- [Parametricity, Types are Documentation - Tony Morris](https://www.youtube.com/watch?v=BtEEZa_Q8Vw)
- [Theorems for free - Philip Walder](https://www.mpi-sws.org/~dreyer/tor/papers/wadler.pdf)
- [Scrap Your Type Class Boilerplate - Aakash N S](http://aakashns.github.io/better-type-class.html)
