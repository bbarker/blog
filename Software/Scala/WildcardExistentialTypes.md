# Existential Types and Variance in Scala

# TODO
1. Considering looking into contravariance examples
2. Test code (at least some) in Scala 2 to see if results hold there
3. Try Java? Similar reasons for trying Scala 2

## Background

Recently, I came across a wildcard type in a library we use ([Tapir](https://tapir.softwaremill.com/)).
Until then, I had viewed wildcard types in Scala as something that could always be avoided. Could I
bypass them here? As it turns out, I could not. In what follows, I hope to show why existential types
can be necessary in Scala, and how they relate to parametric types (also known as
higher-kinded types, or HKTs) and `Any`.

## Quick Introduction

Wildcard types are the simplest form of existentially quantified types in Scala 2, and in Scala 3,
they are the [only form](https://docs.scala-lang.org/scala3/reference/dropped-features/existential-types.html)
of existential types. Also note, the syntax for wildcards [has changed](https://docs.scala-lang.org/scala3/reference/changed-features/wildcards.html
) from `_` in Scala 2 to `?` in Scala 3.

In Scala, in the simplest form, wilcards look similar in usage to `Any`, but using `_`:

```scala
val anythings1: List[Any] = List("foo", 3.14, new java.lang.Object)
val anythings2: List[_]   = List("foo", 3.14, new java.lang.Object)
```

However, unlike `Any`, `_` acts like a type variable; we can constrain it:

```scala
trait At
trait Bt extends At
trait Ct extends Bt
trait Dt extends Ct
trait Et extends Dt

val bcd: List[? <: Bt] = List(new Bt {}, new Ct {}, new Dt {})
val acd: List[? <: Bt] = List(new At {}, new Ct {}, new Dt {})
// ❌ Found:    Object with Playground.At {...}
//    Required: Playground.Bt
val abd1: List[? >: Ct] = List(new At {}, new Bt {}, new Dt {})
val abd2: List[? >: Ct <: At] = List(new At {}, new Bt {}, new Dt {})
// in abd and abd2, ? would be equivalent to At
val what1: List[? >: Ct] = List(new At {}, new Bt {}, 1)
// in what1, ? would likely be equivalent to Any
val what2: List[? >: Ct <: At] = List(new At {}, new Bt {}, 1)
// ❌ Found:    (1 : Int)
//    Required: Playground.At

```

## The Problem

While refactoring some akka-http code to use Tapir, I had gotten to the point where I was
nearly ready to wrap up my first collection of routes and endpoints and needed to wire them
together. I don't want to discuss this here as it is somewhat beside the
point, but if you are curious, I wrote up a [brief description](https://gist.github.com/bbarker/082082c24e1c307d8bc4e58f7717d921).

The main issue was that the function I'd written to merge routes needed to take
endpoints with different input and output types and compose them into a single
route. At first, I thought the Scala type might automatically widen to a type
large enough to accommodate these types (such as `Any`),
but that didn't appear to be the case.

Let's look at a simple example:

### Example 1

```scala
case class SomeClass[A](a: A)

val someStuffAny: List[SomeClass[Any]] = List(SomeClass("foo"), SomeClass(1))

val someStuffEx: List[SomeClass[_]] = List(SomeClass("foo"), SomeClass(1))

def funPoly[A](ss: List[SomeClass[ A ]]): Unit = ()
def     funAny(ss: List[SomeClass[Any]]): Unit = ()
def      funEx(ss: List[SomeClass[ _ ]]): Unit = ()
```

We've managed to define lists of a higher-kinded type using both `Any` and `_`;
the lists have identical values. We've also defined three not-so-interesting
functions that each return `Unit`, and in no way do they depend on the types
(or even values) being passed in: the only difference is how each function
approaches dealing with the type parameter of `SomeClass`. Let's try to use them:

```scala
// These don't work, but why?
val u1 = funPoly(someStuffEx)
// ❌ Found:    (Playground.someStuffEx : List[Playground.SomeClass[?]])
//    Required: List[Playground.SomeClass[A]]
//    where:    A is a type variable
val u2 = funAny(someStuffEx)
// ❌ Found:    (Playground.someStuffEx : List[Playground.SomeClass[?]])
//    Required: List[Playground.SomeClass[Any]]

// This works fine
val u3 =  funEx(someStuffEx)
```

In the first case, `funPoly` doesn't work, apparently because we required a list of
`SomeClass[A]`. In Scala, `_` does not automatically widen to a common supertype;
each "`_`" in the list may be a different type, and as we'll see, even if they are the
same, Scala has no way of knowing this (without invoking reflection, at least).

In the second case, `funAny` doesn't work for similar reasons - each value of type "`_`"
may be distinct, but unknown types, which Scala can't assume to be `Any`, and we'll see
why a bit later. Finally, we can be relieved that `funEx` will do what we want, as we are
being explicit with using a wildcard type.

### Example 2

Now let's look at a similar example where issues arise when actually constructing our
lists.

```scala
case class Inner[A](a: A)
case class Outer[A](a: Inner[A])

val in1 = Inner("foo")
val in2 = Inner(3)

// Doesn't work as we'd hoped
val outsFromInsAny: List[Outer[Any]] = List(in1, in2).map(Outer.apply)
// ❌ Found:    Playground.Inner[?1.CAP]
//    Required: Playground.Inner[Any]
//    where:    ?1 is an unknown value of type scala.runtime.TypeBox[String & Int, String | Int]
val outsFromInsEx:  List[Outer[_]]   = List(in1, in2).map(Outer.apply)
```

To see what is going wrong in `outsFromInsAny`, we can look at a simpler problem:

```scala
val outer1: Outer[Any]= Outer.apply(in1)
// ❌ Found:    (Playground.in1 : Playground.Inner[String])
//    Required: Playground.Inner[Any]
```

This problem would not arise if *either* or both of `Outer` or `Inner` were declared
with a covariant type parameter (`+A`) (try it), e.g.:

```scala
case class Inner[+A](a: A)
// Or:
case class Outer[+A](a: Inner[A])
```

If just one of them is declared as invariant, (i.e. `A` instead of `+A`); if we have
`Inner[+A]`, then `Inner[String] <: Inner[Any]`, for example. Then `Outer[Any]`,
being invariant, requires an `Inner[Any]` (or since `Inner` is covarinant), any subtype
of `Any`, which of course is anything at all. If `Inner` were itself invariant, subtypes
would not be acceptable, thus the problem when both `Outer` and `Inner` are invariant.

Checking the other alternative, if we have `Outer` covariant (`Outer[+A]`) and `Inner`
invariant (`Inner[A]`), then if we construct an `Inner[String]`, then construct an
`Outer[Any]` using this inner value, it is perfectly fine, since `String <: Any`.

With that exercise in variance out of the way (and in my experience, it takes a lot of
variance exercise to really get comfortable with the idea), let's return to what might
be a niggling question: why was the following OK even when both `Outer` and `Inner`
are invariant?

```scala
val outsFromInsEx:  List[Outer[_]]   = List(in1, in2).map(Outer.apply)
```

The wildcard allows us to efectively ignore these variance issues, as they aren't tracked.
In the next example, we'll look to see if there are any type-safety implications from
ignoring variance when using wildcards.



## References

### Variance

1. [Zionomicon](https://www.zionomicon.com/), Appendix: Mastering Variance
2. [Understanding Covariance and Contravariance](https://blog.kaizen-solutions.io/2020/variance-in-scala/)
