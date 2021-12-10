## TL;DR

Here's a [tiny library](https://github.com/bbarker/zio-diffx)
for zio-test and diffx integration. Check it out
if you use one and want to use the other as well. However,
it is small enough you may just want to drop the code into your
project.

### Dependency Coordinates

Make sure jitpack is in your resolvers, e.g.:

```scala
  resolvers ++= Seq(
    "jitpack.io" at "https://jitpack.io/",
  ),
```

The general suggestion is to use the most recent release listed on the
GitHub releases page for the project (not necessarily what is shown below):

```scala
"io.github.bbarker" %% "zio-diffx" % "0.0.5" % Test
```

## Background

[At work](https://www.linkedin.com/company/caesars-digital/mycompany/)
we use the [diffx](https://github.com/softwaremill/diffx)
family of libraries for displaying mismatching data structures.
Diffx offers several testing library integrations, including for
ScalaTest, which is what we predominantly use. However, we have
started using zio-test on a limited basis; zio-test can
more easily work with ZIO effects (or other effects converted to ZIO effects),
and tests themselves can be effects, which makes the testing UX much better
for functional effect code, generally.

As our services become
more heavily managed by the ZIO effect system and runtime, we don't want to rely solely
on ScalaTest, though we'll certainly be using it
for quite some time as well due to our pre-existing extenstive test suite.


## Show Me the Code

For ZIO 1 (which is all I've written this for so far), there are the effectful `*M`
(e.g. `testM`, `assertM`)
and the pure variants (`test`, `assert`); this is consolidated in ZIO 2 for
a more streamlined experience.

The effectful variant:

```scala
import com.softwaremill.diffx.{ConsoleColorConfig, Diff}
import zio.UIO
import zio.test.AssertionM.Render.*
import zio.test.*

trait DiffxAssertionsM {

  val matchesToAssertionNameM = "matchesTo"

  /**
   * Just like isTrue from zio-test, but with a different name parameter
   */
  private def assertTrueM[A](actual: UIO[A]): AssertionM[Boolean] =
    AssertionM.assertionM(matchesToAssertionNameM)(param(actual))(identity(UIO(_)))

  def matchesToM[A: Diff](expected: A)(implicit
    c: ConsoleColorConfig
  ): AssertionM[A] =
    AssertionM.assertionDirect(matchesToAssertionNameM)(param(expected)) { actual =>
      val result = Diff.compare(expected, actual)
      assertTrueM(UIO(actual)).label(result.show()).runM(result.isIdentical)
    }
}
```

And the pure variant:

```scala
import com.softwaremill.diffx.{ConsoleColorConfig, Diff}
import zio.test.Assertion.Render.*
import zio.test.*

trait DiffxAssertions {

  val matchesToAssertionName = "matchesTo"

  /**
   * Just like isTrue from zio-test, but with a different name parameter
   */
  private def assertTrue[A](actual: A): Assertion[Boolean] =
    Assertion.assertion(matchesToAssertionName)(param(actual))(identity(_))

  def matchesTo[A: Diff](expected: A)(implicit c: ConsoleColorConfig): Assertion[A] =
    Assertion.assertionDirect(matchesToAssertionName)(param(expected)) { actual =>
      val result = Diff.compare(expected, actual)
      assertTrue(actual).label(result.show())(result.isIdentical)
    }
}
```

The two traits have members with different names so they can be mixed into the same class
or trait in test code, as desired.

For example usage, if it isn't clear, feel free to take a look at the tests (`*Spec.scala`)
in the zio-diffx repository.

## ZIO 2

*I'll update this section when there is a ZIO 2 release*

In ZIO 2, custom assertions will be possible, so we won't have to
chain on an existing assertion. Currently, the effect of this is
that we get two error messages when there is a mismatch: one
from the boolean test, and one from the label that
displays the mismatched data structures:

![zio-diffx mismatch example](/Software/Scala/ZioDiffx/ZIO1diffx.png)


## Scala 3

Currently, there are no diffx-core artifacts published for Scala 3,
but there have been some PRs merged that add support for Scala 3,
so we can expect to see it at some point.

## Conclusion

ZIO-test and diffx are great, use them! If you beat me to adding Scala 3 or ZIO 2
support in zio-diffx, don't hesitate to put in a PR (or put in a PR for any cool features
I haven't thought of).

Feel free to comment here if you have questions or suggestions, or you can
[find me on Twitter](https://twitter.com/b_barker) where you can ask me anything.