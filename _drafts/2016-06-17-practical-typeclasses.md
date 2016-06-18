---
layout: post
title: "Practical Typeclasses"
description: ""
category: 
tags: []
---
{% include JB/setup %}
# Practical typeclasses
### Inspired by "Parametricity, Types are Documentation - Tony Morris"
https://www.youtube.com/watch?v=BtEEZa_Q8Vw
---
# Motivation

### What does this do

```scala
case class Context(now:DateTime){
  def isRecent(email:Email):Boolean = ???
}
```
---
# Motivation

### What does this do

```scala
case class Context(now:DateTime){
  def isRecent(email:Email):Boolean = ???
}
```

Who knows?

```scala
  def isRecent(email:Email):Boolean = email.sentDate.isBefore(now)
```
or
```scala
  def isRecent(email:Email):Boolean = email.receivedDate.isBefore(now)
```
or
```scala
  def isRecent(email:Email):Boolean = email.from.broker.isDefined
```
???
tough to know which of the fields in `Email` is being used

---
# Motivation

### Alternatively

```scala
case class Context(now:DateTime){
  def isRecent(timestamp:DateTime):Boolean = timestamp.isBefore(now)
}
```
This works, but removes semantics, in some occasions that could be a problem

---
# Motivation

### Bring the semantics back

```scala
case class Context(now:DateTime){
  def isRecent[T:Timestamp](t:T):Boolean = ???
}
```
---
# Motivation

### Bring the semantics back

```scala
case class Context(now:DateTime){
  def isRecent[T:Timestamp](t:T):Boolean = t.timestamp.isBefore(now)
}```

- `T` is some type, nothing more
- `Timestamp` is a typeclass
  - First recomendation: Typeclasses should do _one_ thing

---

# How to do that?

```scala
 trait Timestamp[T]{
  def timestamp(t:T):DateTime
 }
```

---

# How to do that?

```scala
 trait Timestamp[T]{
  def timestamp(t:T):DateTime
 }
```

```scala
 object Timestamp{
  def apply[T:Timestamp]:Timestamp[T] = implicitly[Timestamp[T]]

  implicit object email extends Timestamp[Email] {
    def timestamp(t:Email):DateTime = t.receivedDate
  }

  implicit object timeEntity extends Timestamp[TimeEntity] {
    def timestamp(t:TimeEntity):DateTime = t.timestamp
  }

 }
```
---

# How to do that?

```scala
 trait Timestamp[T]{
  def timestamp(t:T):DateTime
 }
```

```scala
 object Timestamp{
  def apply[T:Timestamp]:Timestamp[T] = implicitly[Timestamp[T]]

  implicit object email extends Timestamp[Email] {
    def timestamp(t:Email):DateTime = t.receivedDate
  }

  implicit object timeEntity extends Timestamp[TimeEntity] {
    def timestamp(t:TimeEntity):DateTime = t.timestamp
  }

  implicit class Syntax[T](t:T){
    def timestamp(implicit ts:Timestamp[T]) = ts.timestamp(t)
  }

 }
```

---
# How to do that?

```scala
import Timestamp.Syntax
case class Context(now:DateTime){
  def isRecent[T:Timestamp](t:T):Boolean = t.timestamp.isBefore(now)
}```

---

# Bonus

### What happens with a sealed trait

```scala
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

```
