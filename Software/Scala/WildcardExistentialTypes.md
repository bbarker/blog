# Existential Types and Variance in Scala

If you'd like to follow along with code examples, most examples are in Scala 3
and [on Scastie](https://scastie.scala-lang.org/bbarker/sXhlCN0rQS6uj72Wds8HTQ/10).
I will provide an example for Scala 2 and Java near the end of the post. That said,
almost everything here should apply to Scala 2 and Scala 3, but there is an interesting
difference between Scala and Java.

## A Note on Variance

This blog post is not an introduction to variance, but several [good references](#Variance) are available,
and I provide some explanation of variance as it pertains to our examples.

John von Neumann [is quoted](https://en.wikiquote.org/wiki/John_von_Neumann) as saying:

> ... in mathematics you don't understand things. You just get used to them.

I feel like this applies particularly well to variance. Scala making variance more
explicit than any other language (that I'm aware of) forces us to think of these issues
when we are designing and using libraries at compile time rather than runtime.

## Background

Recently, I came across a wildcard type in a library we use ([Tapir](https://tapir.softwaremill.com/)).
Until then, I had viewed wildcard types in Scala as something that could be avoided. Could I
bypass them here? As it turns out, I could not. In what follows, I hope to show why existential types
can be necessary in Scala, and how they relate to parametric types (also known as
higher-kinded types, or HKTs) and `Any`.

## Quick Introduction

Wildcard types are the simplest form of existentially quantified types in Scala 2, and in Scala 3,
they are the [only form](https://docs.scala-lang.org/scala3/reference/dropped-features/existential-types.html)
of existential types. Also note, the syntax for wildcards [has changed](https://docs.scala-lang.org/scala3/reference/changed-features/wildcards.html
) from `_` in Scala 2 to `?` in Scala 3.

In Scala, in their simplest form, wilcards look similar in usage to `Any`, but using `?`:

```scala
val anythings1: List[Any] = List("foo", 3.14, new java.lang.Object)
val anythings2: List[?]   = List("foo", 3.14, new java.lang.Object)
```

However, unlike `Any`, `?` acts as a type variable; we can constrain it:

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
point, but if you are curious, I wrote up a
[brief description](https://gist.github.com/bbarker/082082c24e1c307d8bc4e58f7717d921).

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
val someStuffEx:  List[SomeClass[?]]   = List(SomeClass("foo"), SomeClass(1))

def funPoly[A](ss: List[SomeClass[ A ]]): Unit = ()
def     funAny(ss: List[SomeClass[Any]]): Unit = ()
def      funEx(ss: List[SomeClass[ ? ]]): Unit = ()
```

We've managed to define lists of a higher-kinded type using both `Any` and `?`;
the lists have identical values. We've also defined three not-so-interesting
functions that each return `Unit`, and in no way do they depend on the types
(or even values) passed in: the only difference is how each function
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
`SomeClass[A]`. In Scala, `?` does not automatically widen to a common supertype;
each "`?`" in the list may be a different type, and as we'll see, even if they are the
same, Scala has no way of knowing this (without invoking reflection, at least).

In the second case, `funAny` doesn't work for similar reasons - each value of type "`?`"
(again, "`?`" is not a real type) may be of distinct but unknown types,
which Scala can't assume to be `Any`, and we'll see
why a bit later. Finally, we can be relieved that `funEx` will do what we want, as we are
explicit with using a wildcard type.

### Example 2

Now let's look at a similar example where issues arise when constructing our
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
val outsFromInsEx:  List[Outer[?]]   = List(in1, in2).map(Outer.apply)
```

To see what is going wrong in `outsFromInsAny`, we can look at a simpler problem:

```scala
val outer1: Outer[Any] = Outer.apply(in1)
// ❌ Found:    (Playground.in1 : Playground.Inner[String])
//    Required: Playground.Inner[Any]
```

This problem would not arise if *either* or both of `Outer` or `Inner` had
been declared with a covariant type parameter (`+A`) (try it), e.g.:

```scala
case class Inner[+A](a: A)
// Or:
case class Outer[+A](a: Inner[A])
```

If we have `Inner[+A]`, then `Inner[String] <: Inner[Any]`, for example. Then `Outer[Any]`,
being invariant, requires an `Inner[Any]` (or since `Inner` is covariant, its parameter
can be any subtype of `Any`, which is anything at all).
If `Inner` were itself invariant, subtypes
would not be acceptable, thus the problem when both `Outer` and `Inner` are invariant.

Checking the other alternative, if we have `Outer` covariant (`Outer[+A]`) and `Inner`
invariant (`Inner[A]`), then if we construct an `Inner[String]`, and then construct an
`Outer[Any]` using this inner value, it is perfectly fine, since `String <: Any`.

At this point, you may think back to example 1, and wonder why we could construct
a list of values that are invariant in their type parameter:

```scala
val someStuffAny: List[SomeClass[Any]] = List(SomeClass("foo"), SomeClass(1))
```

This is due to type inference: since we ask for a `List[SomeClass[Any]]` and
we weren't explicit with the type of the values, Scala was able to infer each
inner value (`"foo"` and `1`) to be of type `Any`. If we explicitly ascribe
more precise types for each value, then we run into the same kind of error:

```scala
val someStuffAny2: List[SomeClass[Any]] =
  List(SomeClass("foo"): SomeClass[String], SomeClass(1): SomeClass[Int])
// ❌ Found:    Playground.SomeClass[Int]
//    Required: Playground.SomeClass[Any]
// ❌ Found:    Playground.SomeClass[String]
//    Required: Playground.SomeClass[Any]
```

This implies that what we have is a type inference problem.
If we explicity type the list values, then we can create our list of `Any`s:

```scala
val in1any: Inner[Any] = Inner("foo")
val in2any: Inner[Any] = Inner(3)

val outsFromInsAny2: List[Outer[Any]] = List(in1any, in2any).map(Outer.apply)
```

In this case, though, it would have just been easier to use the wildcard type,
and certainly, we don't always have the luxury of re-ascribing types to values
of invariant types. Imagine if the values were from a third-party library we don't
control. We can emulate such behavior here:

```scala
val in1any2: Inner[Any] = (Inner("foo"): Inner[String])

// ❌ Found:    Playground.Inner[String]
//    Required: Playground.Inner[Any]
```

With that exercise in variance out of the way (and in my experience, it takes a lot of
variance exercise to become comfortable with the idea), let's return to what might
be a niggling question: why was the following OK even when both `Outer` and `Inner`
are invariant?

```scala
val outsFromInsEx: List[Outer[?]] = List(in1, in2).map(Outer.apply)
```

The wildcard allows us to effectively ignore these variance issues, as `?` isn't a
concrete type. In the following example, we'll learn how type-safety can be
affected when ignoring variance through the use of wildcards.

### Example 3

To see what some of the safety implications are, we need to try to do *something*
with values passed to a function that can accept values of any type.
Our safe options should be limited to just returning a
value of an unknown type, so we'll endeavor to try something a little more interesting.
First, we'll create a new class and a few example values:

```scala
case class ScalaBean[A](initA: A):
  private var theA: A = initA
  def getTheA: A = theA
  def setTheA(a: A): Unit = {
    this.theA = a
  }

val bean1: ScalaBean[Int] = ScalaBean(1)
val bean2: ScalaBean[String] = ScalaBean("Foo")
val bean3: ScalaBean[Int] = ScalaBean(2)

// val beanList1: List[ScalaBean[Any]] = List(bean1, bean2, bean3)
val beanList2: List[ScalaBean[?]] = List(bean1, bean2, bean3)
val beanList3: List[ScalaBean[Int]] = List(bean1, bean3)
```

`ScalaBean` is certainly not a great example of good Scala code
or even good Java code, but it serves our purpose here. Again,
we don't attempt to construct a `List[ScalaBean[Any]]` for similar
reasons as from example 2: `ScalaBean[A]` is invariant in `A`, so
we can't do anything like:

```scala
val anyBean: ScalaBean[Any] = bean1
```

Since we do know a fairly specific type for `beanList3`, we can do something with
its members:

```scala
println(beanList3.map(_.getTheA))
beanList3.foreach{bean =>
  bean.setTheA(bean.getTheA+1)
}
println(beanList3.map(_.getTheA))
```

This yields output showing the incrementation worked as expected:

```
List(1, 2)
List(2, 3)
```

If we were instead dealing with a `List[ScalaBean[Any]]` or a `List[ScalaBean[?]]` it
would be clear that we couldn't even try to do something like this since `+` isn't
defined on `Any` or on unknown types. But we could still try to set values. Let's
try making such a function:

```scala
def polySetAllBeansTo[A](beanList: List[ScalaBean[A]], initBean: ScalaBean[A]): Unit =
  beanList.foreach{bean =>
    bean.setTheA(initBean.getTheA)
  }

polySetAllBeansTo(beanList3, bean1)
// everything is now set to bean1's value

println(beanList2.map(_.getTheA))
polySetAllBeansTo(beanList2, bean1)
// ❌ Found:    (Playground.beanList2 : List[Playground.ScalaBean[?]])
//    Required: List[Playground.ScalaBean[Int]]
println(beanList2.map(_.getTheA))
```

Scala catches this sleight of hand and doesn't allow us to do anything potentially
nasty here. But what if we use wildcard types in our function instead of polymorphic types?
Scala won't even let us try it (good!):

```scala
def exSetAllBeansTo(beanList: List[ScalaBean[?]], initBean: ScalaBean[?]): Unit =
  beanList.foreach{bean =>
    bean.setTheA(initBean.getTheA)
    // ❌ Found:    initBean.A
    //    Required: bean.A
  }
```

OK, but let's try and be slightly more clever and use an element contained in the list:

```scala
def exSetAllBeansToFirst(beanList: List[ScalaBean[?]]): Unit =
  val firstBean = beanList.head // bad: don't use head if the list may be empty
  beanList.foreach{bean =>
    bean.setTheA(firstBean.getTheA)
    // ❌ Found:    firstBean.A
    //    Required: bean.A
  }
```

We can't do that either. This example has the same result
[in Scala 2](https://scastie.scala-lang.org/bbarker/EU5MnKcoTauG2KSD1UQ6nA/13), though in
Java there is one difference that relates to the fact that Java (unlike Scala) does
not support declaration-site variance (i.e., you can't specify `+A` or `-A` in your type
definitions):

```java
ArrayList<JavaBean<Object>> beanList1 = new ArrayList(Arrays.asList(bean1, bean2, bean3));

public <A> void polySetAllBeansToFirst(List<JavaBean<A>> beanList) {
  beanList.forEach((bean) -> bean.setTheA(beanList.get(0).getTheA()));
}

polySetAllBeansToFirst(beanList1);
// ^ We can't do this in Scala, because we can't even construct beanList1
```

In this case, there is nothing unsafe in itself and may even feel more convenient than in
Scala. However, there are other cases where the lack of declaration-site variance can
[cause issues](https://medium.com/javarevisited/variance-in-java-and-scala-63af925d21dc).
The complete example in Java is available interactively [on JDoodle](jdoodle.com/a/4aHU)
(also as a [gist backup](https://gist.github.com/bbarker/c2cfcf176c1fcdcffbb68c001fe44fa2)).

There are probably some interesting cases not covered here, particularly related to contravariance,
which may be the subject of a future blog post.

## Conclusions

We've seen that wildcard (existential types) allow us to create collections of elements of
various types, even when we can't employ the `Any` type due to nested invariance
constraints or previously ascribed types (such as from third-party libraries).
When using wildcards, Scala and Java prevent us from accessing values that would have the wildcard
type in their signature, at least directly (see [Example 3](#example-3)). This makes sense,
as we can't talk about a type in any meaningful way when we don't know what it is.
We also saw that since Java doesn't employ declaration-site variance, we can directly create a list of
`Foo<Object>` values without worrying about the variance constraints, but the lack of
declaration-site variance
[can cause runtime errors in Java](https://stackoverflow.com/questions/28570877/java-covariant-array-bad)
if the users don't *properly* employ use-site variance; the burden of variance in Scala
is on the API designer, but in Java, it is on the user of the API. Proper variance declarations
allow us to be more precise, and if properly used, wildcards can usually be avoided.
On the other hand, wildcards allow us to ignore variance, but at the cost of not being able
to refer to values of the wildcard types in any way.

Hopefully, the examples here proved helpful - I encourage anyone working with Scala or other
languages where variance comes up to keep reading and experimenting.
Feel free to comment here if you have questions or suggestions, or you can
[find me on Twitter](https://twitter.com/b_barker) where you can ask me anything.

## References

### Variance

1. [A Complete Guide to Variance in Java and Scala](https://medium.com/javarevisited/variance-in-java-and-scala-63af925d21dc);
  also touches on existential types.
2. [About Variance](https://contramap.dev/posts/2020-02-12-variance/)
3. [Zionomicon](https://www.zionomicon.com/), Appendix: Mastering Variance.
4. [Understanding Covariance and Contravariance](https://blog.kaizen-solutions.io/2020/variance-in-scala/)
5. [SO question inspiring this post](https://stackoverflow.com/questions/70193216/why-cant-a-polymorphic-function-accept-wildcard-existential-types-in-scala),
  where the key insight was pointed out to me by Matthias Berndt.

### Wildcards and Existential Types

1. [Wildcard Arguments in Types](https://docs.scala-lang.org/scala3/reference/changed-features/wildcards.html) - Scala 3 Language Reference.
2. [Dropped: Existential Types](https://docs.scala-lang.org/scala3/reference/dropped-features/existential-types.html) - Scala 3 Language Reference.
3. [scala - Any vs underscore in generics](https://stackoverflow.com/a/15204140/3096687)
