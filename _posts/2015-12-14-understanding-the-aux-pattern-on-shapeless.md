---
layout: post
title: "Understanding the Aux pattern on shapeless"
category: Functional Programming
tags: [scala, shapeless]
---
{% include JB/setup %}

I have been exploring shapeless for a few weeks now. Although I already I'm using coproducts and poly functions in our codebase, it was still very confusing any time I looked at some shapeless code and found those long lists of implicits and `<Combinators>.Aux`.

I new they were some kind of evidence for the compiler, but I realized that without using them, I wouldn't get they relevance.

So this is my exploration. I wanted a way to extract the values of an specific type in a hierarchy of ADTs.

i.e.:

    object ints extends Field[Int]
    object strings extends Field[String]

    sealed trait F
    case class A( i: Int ) extends F
    case class B( s: String ) extends F
    case class C( i: Int, s: String ) extends F
    sealed trait E extends F
    case class H(i: Int) extends E

    val fs: List[F] = List( A( 1 ), B( "b" ), C( 3, "c" ), H(4) )

    assert( fs.map( f => ints.filter( f ) ).flatten == List( 1, 3, 4 ) )
    assert( fs.map( f => strings.filter( f ) ).flatten == List( "b", "c" ) )


Given any `F`  we could find all the integers or strings that it might contain, within itself or any of its children.

One thing I have realized, is that the concepts of product and coproduct are a cornerstone in the shapeless design.

And it makes sense, a lot of what we do in scala can be describe in terms of products and coproducts.

A product can be understood as a union of values or types, e.g. :


    val t:(Int, String, Double) = null

    case class Foo(i:Int, s: String, d: Double)


and coproduct is something that can only be either of certain types or values:

```
trait Foo
case object A extends Foo
```



That's right any hierarchy of sealed traits it's a coproduct, since any instance can only be one of the descendants of the sealed trait parent.

Shapeless abstract both concepts by creating Coproduct and HList. HList is the equivalent to product, not sure why the name, I guess Product was already taken.

Lets try some operations with HList.


    @ import shapeless._
    import shapeless._
    @ val ls = 1 :: "a" :: 3D :: HNil
    ls: Int :: String :: Double :: HNil = ::(1, ::("a", ::(3.0, HNil)))
    @ ls.filter[Int]
    res2: Int :: HNil = ::(1, HNil)
    @ ls.filter[Int].to[List]
    res3: List[Int] = List(1)


You'll notice that filter on the `HList`. So it is filtering at the type level. It doesn't care about the values them selves, it can only filter by types. Which makes sense, an `HList` is a compile time artifact. It cannot be created at runtime, in the same way you don't create tuples or case classes at runtime.

Most of the logic that you code in shapeless is for the compiler, not runtime.[1]

For a coproduct a similar operation would be:


    @ import shapeless._
    import shapeless._
    @ type F = Int :+: String :+: Double :+: CNil
    Defined type alias F
    @ f.filter[Int]
    res11: Option[shapeless.:+:[Int,shapeless.CNil]] = Some(1)
    @ f.filter[Int].map(_.unify)
    res12: Option[Int] = Some(1)

As you can see in both cases, you end up with a different type. It's very important to keep present that all it's been done here is at the type level.

There are many possible operations for coproducts and hlists(products) on shapeless. But none for sealed traits nor tuples or case classes. For that we use Generic.

For a product:

    @ import shapeless._
    import shapeless._
    @ case class F(i: Int, s:String, d: Double)
    defined class F
    @ Generic[F].to(F(1, "a", 3D))
    res2: Int :: String :: Double :: HNil = ::(1, ::("a", ::(3.0, HNil)))

For coproduct:

    @ sealed trait F
    defined trait F
    @ case class A() extends F
    defined class A
    @ case class B() extends F
    defined class B
    @ Generic[F].to(A())
    res6: A :+: B :+: CNil = Inl(A())

If we have a hierarchy of  cases class and sealed traits, we'll need to "transform" the type of the value into either a product or a coproduct.

This is a simple solution, half way between type classes and inheritance.

    trait Field[A]{

      def filter:List[A] = ???

    }


so to create a "query" for a type we only should need to do:

    object ints extends Field[Int]
    object strings extends Field[String]
