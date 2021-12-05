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

However, unlike `Any`, `_` acts like a type variable: we can constrain it:

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
together. I don't want to discuss this further in this post as it is somewhat beside the
point, but if you are curious, I wrote up a description [here](https://gist.github.com/bbarker/082082c24e1c307d8bc4e58f7717d921).

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


// These don't work, but why?
val u1 = funPoly(someStuffEx)
val u2 = funAny(someStuffEx)

// This works fine
val u3 =  funEx(someStuffEx)
```


### Example 2

```scala
case class Inner[A](a: A)
case class Outer[A](a: Inner[A])

val in1 = Inner("foo")
val in2 = Inner(3)

// Doesn't work as expected due to invariance of type paramemters A
val outsFromInsAny: List[Outer[Any]] = List(in1, in2).map(Outer)
val outsFromInsEx:  List[Outer[_]]   = List(in1, in2).map(Outer.apply)
```



## References

### Variance

1. [Zionomicon](https://www.zionomicon.com/), Appendix: Mastering Variance
2. [Understanding Covariance and Contravariance](https://blog.kaizen-solutions.io/2020/variance-in-scala/)
