---
layout: post
title: "Practical Typeclasses - WIP"
description: ""
category: Scala
tags: [scala, type, typeclass]
---
{% include JB/setup %}

# Intended audience

You are aware of the existence of typeclasses, but you are not sure where or when to use them, and you are mainly writing domain specific code. This post is to provide some guidance for how and why use typeclasses. 

You are also interested in a more generic way of programming in Scala. If you are more comfortable with the OOP aspect of the language and want it to keep it that way, then perhaps this piece won't make a lot of sense to you, nevertheless I would recommend to watch Tony Morris talk linked in the [References](#references) section. I wouldn't recommend to blindly follow those advices, but there are a lot of good ideas there, and this piece is trying to help those interested in following the main one: Use type parameters as much as possible.

You should be familiar with scala, with traits, companion objects and similar machinery.

Inspired by [Strategic Scala Style: Designing Datatypes](http://www.lihaoyi.com/post/StrategicScalaStyleDesigningDatatypes.html), I'll be using _datatype_ when referring to instances of classes and other data-like types, and _type_ for generic types like `T`.

This piece **does not** provide a nuanced guide to write typeclasses, for that you should read the excellent article **Scrap Your Type Class Boilerplate** linked in the [References](#references) section. The goal is to provide a rationale as to why and how use typeclasses.

# Motivation


When programming, we usually need to write a method that has very strong domain semantics, for example:

~~~
case class Context( initialDate:DateTime ){
  def isRecent( email:Email ):Boolean = ???
}
~~~

> The Context class is completely irrelevant four our case,
> but it makes the scenario to look more realistic

----------

This method is responding to a "question": Is this `Email` recent? From looking at the signature we learn nothing about it's inner workings. The name helps, but it's the _only_ source of information. What if we want to know other things about `Email`s, `Person`s, etc; that involve some logic? Having too many of these methods will really hinder the readability, and I dare to say, the simplicity of the code.

### The Problem

We want methods like `isRecent` -- where the semantics of the domain matter -- and we want it to be easy to understand, and safer to use than merely pass a big fat object as an argument.

### Alternatives

Typeclasses permit to add properties to type parameters when passed to functions. -- i.e. `isRecent[T]( t: T )`. Where there is no data structure from which is possible to "hack" a solution within the method body. The goal is to provide the certainty -- as much as is possible in the JVM -- that the method is only using the arguments in the "allowed" way.

If the above is not a priority for you, then there are some approaches that I actually recommend over typeclasses for most situations.

In both cases the usage would be:

```
isRecent( email.receivedDate )
```

#### No semantics

We don't care about the "meaning" of the  `DateTime` we pass to `isRecent`.

~~~
case class Context( initialDate:DateTime ){
  def isRecent( timestamp:DateTime ):Boolean = timestamp.isBefore( initialDate )
}
~~~

This works, but it removes domain constraints from our method.

#### Wrapper class

We can create a case class to wrap the `DateTime`, and use it for `receivedDate`.

~~~
case class Timestamp( t:DateTime )

case class Email( receivedDate: Timestamp )

case class Context( initialDate:DateTime ){
  def isRecent( timestamp:Timestamp ):Boolean = timestamp.t.isBefore( initialDate )
}
~~~

That would certainly preserve the semantics. The problem is that it's just a wrapper, and at the call site is possible to do something like `isRecent(Timestamp(someDate))`, which kind of defeats the purpose. If at the call site you don't care about the `DateTime` semantics, then why would you care inside `isRecent`?

Of course you could make the constructor of `Timestamp` private, or the whole class private within `Email`, but that would add _a lot_ of complexity, once you need to also know if something other than an `Email` `isRecent`. Too much entanglement.

### What about inheritance?

When several datatypes share several properties. For example `Email` and `TimeProposed` need a `timestamp`, but they also could have an `_id`. It's possible to make a trait for `Timestamp` and another for `WithID`, and then this two classes would just implement those traits.

~~~
trait Timestamp{ def timestamp:DateTime }

trait WithID{ def _id:Id }

case class Email( _id: Id, timestamp:DateTime ) extends Timestamp with WithID

case class Context( initialDate:DateTime ){
  def isRecent( t:Timestamp ):Boolean = t.timestamp.isBefore( initialDate )
}
~~~

I'm very skeptical about that solution. It leads to more entanglement and very complex hierarchies. At least that's what I have seen in java codebases. Also you need to be in control of the class that have these properties, and need to keep changing _them_, if new requirements change the semantics. With typeclasses you just add an instance for that datatype. No meaningless hierarchies or changes on the datatype are needed, as we'll see below. But this could be me being paranoid and battle scarred.



# Typeclasses for semantics with safety

The main point, of using typeclasses, is that allows to write methods that only take type parameters -- i.e. `isRecent[T]( t:T )`. The code written in this manner tends to be simpler and more correct, since it is impossible to make assumptions about the arguments.

The typeclass  is also a "wrapper" for `Email`. The usage of `isRecent` with a typeclass will be like this:

~~~
context.isRecent( email )
~~~

Which is rather convenient, and it looks exactly like the first version. The implementation is as follows:


~~~
case class Context( initialDate:DateTime ){
  def isRecent[T:Timestamp]( t:T ):Boolean = t.timestamp.isBefore( initialDate)
}
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
case class Context( initialDate:DateTime ){
  def isRecent[T:Timestamp]( t:T ):Boolean = t.timestamp.isBefore( initialDate )
}
~~~

# Conclusions

For the case when `isRecent` just receives a `DateTime`, the call site is very elquent, we can tell what property of `Email` is being used.

~~~
isRecent( email.receivedDate )
~~~

In this case however(with the typeclass):

~~~
isRecent( email )
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
