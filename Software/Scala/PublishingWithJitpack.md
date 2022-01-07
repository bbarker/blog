# Publishing JVM artifacts quickly with JitPack
## Git packed with JitPack

Recently, while discussing my non-idyllic past experiences of publishing
JVM artifacts, a [colleague](https://twitter.com/cal_fern) introduced me
to [Jitpack](https://jitpack.io/), which integrates nicely
with Git repositories hosted on  GitHub or BitBucket, making the publishing process
much simpler.

While this article focuses on building Scala artifacts with
SBT, I'd encourage anyone publishing JVM artifacts to
give JitPack a try; at the time of this writing, JitPack currently
can build Gradle, Maven, SBT, and Leiningen projects. We'll be
looking at SBT, which is a Scala-oriented build tool, but it
also supports other languages.

## Immutability

One argument against just using git repository services, such as GitHub, for distributing
artifacts or code for production builds is that no guarantees for the preservation
of artifacts exist; a version could be updated, a branch removed,
or an entire repository deleted.

A famous example of this in recent history occurred with NPM, which is an actual package
repository rather than a code repository service. In this case, a (perhaps rightfully)
disgruntled developer removed all of his packages from NPM,
[wreaking havoc](https://www.theregister.com/2016/03/23/npm_left_pad_chaos/).
Whether or not we agree with the developer's motivations, most of us can probably
agree that we don't want to be subject to the consequences of such actions.

JitPack [takes an approach](https://jitpack.io/docs/#immutable-artifacts)
allowing for a mutable grace period of 7 days, after which the artifacts for
that version become immutable. For me, this hits a sweet spot, and I feel
like it is in line with the general vibe of JitPack. Still, they recommend
publishing new artifacts most of the time, and since it is so easy to do so
with JitPack, why not!

## Scala Versions in the artifact Path

If you are publishing Scala artifacts, the Scala version needs to be correctly integrated
into the artifact name. For instance, for Scala 2.13.X, we would want an artifact that
looks like `zio-diffx_2.13-0.0.4.jar`, not `zio-diffx-0.0.4.jar`. This will also show up
in the URL path, e.g.
[https://jitpack.io/io/github/bbarker/zio-diffx_2.13/0.0.4/](https://jitpack.io/io/github/bbarker/zio-diffx_2.13/0.0.4/).

In order to achieve this with JitPack, we need to instruct JitPack that we're
publishing with SBT. We do this by creating `jitpack.yml` in the root of our repository,
with contents like:

```yaml
install:
  - sbt -v +publishM2
```

I like to keep the `-v` (for "verbose") to make sure settings are
being correctly applied if something goes wrong.
It isn't clear to me what the process would be if publishing Scala
artifacts with other build tools - if you try it, let me know!

## SBT Memory issues

Despite the [Jitpack SBT docs](https://jitpack.io/docs/BUILDING/#sbt-projects),
implying otherwise, using `env` in `jitpack.yml` seems to be outdated.
At least, it didn't work for me (the resulting `build.log`):

```
Build starting...
Error parsing yml config file
Error reading Map: env
```

The good news is that there are other ways to go about this; I opted for
putting my configuration into an `.sbtopts` file residing in my repository's
root directory. I noticed when running `sbt -v`, which displays various
JVM options, that they were all reset when using `.sbtopts` even when
they weren't specified in `.sbtopts`. So I copied these defaults and then
made the changes I wanted, in this case, just adding an option for
`-XX:MaxMetaspaceSize`. Altogether, this resulted in the following content for my
`.sbtopts`:

```
-J-Xms1024m
-J-Xmx1024m
-J-Xss4M
-J-XX:ReservedCodeCacheSize=128m
-J-XX:MaxMetaspaceSize=512m
```

In another project that depends on zio-diffx, I add the dependency to SBT in
the usual way, in this case (it is a test dependency, otherwise you would omit `% Test`):

```scala
"io.github.bbarker" %% "zio-diffx" % "0.0.4" % Test
```

Make sure jitpack is in your resolvers, e.g.:

```scala
  resolvers ++= Seq(
    "jitpack.io" at "https://jitpack.io/",
  ),
```

Then, after running `reload` and `test:compile` in `sbt`, jitpack will kick off the artifact build.
This takes some additional time while the dependency is being built,
but this only happens once per version (once, it seemed I needed to go to https://jitpack.io,
lookup my package and desired version, and click on "Get it" in order to kick things off,
but that could be a coincidence of timing).
Now when I visit `https://jitpack.io/io/github/bbarker/zio-diffx_2.13/0.0.4/` I can see
the results of the build:

```
build.log
zio-diffx_2.13-0.0.4-javadoc.jar
zio-diffx_2.13-0.0.4-javadoc.jar.md5
zio-diffx_2.13-0.0.4-javadoc.jar.sha1
zio-diffx_2.13-0.0.4-sources.jar
zio-diffx_2.13-0.0.4-sources.jar.md5
zio-diffx_2.13-0.0.4-sources.jar.sha1
zio-diffx_2.13-0.0.4.jar
zio-diffx_2.13-0.0.4.jar.md5
zio-diffx_2.13-0.0.4.jar.sha1
zio-diffx_2.13-0.0.4.pom
zio-diffx_2.13-0.0.4.pom.md5
zio-diffx_2.13-0.0.4.pom.sha1
```

Success!


