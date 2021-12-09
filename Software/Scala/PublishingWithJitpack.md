
Recently while complaining to a friend about my past experiences publishing
JVM artifacts, he introduced me to [Jitpack](https://jitpack.io/),
which integrates nicely
with GitHub repositories and in principle makes the publishing process
much closer to what we would like, and is much simpler.


While this article focuses on building Scala artifacts with
SBT, I'd encourage anyone publishing any type of JVM artifacts to
give Jitpack a try.

TODO: supported build systems currently are:

## Immutability

TODO: talk about why it is a good thing (leftmap ref?)

## Scala Versions in the artifact Path

If you are doing

## SBT Memory issues

Despite the [Jitpack SBT docs](https://jitpack.io/docs/BUILDING/#sbt-projects),
implying otherwise, `env`, this seems to be outdated. At least, it didn't work for me (the resulting `build.log`):

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
`-XX:MaxMetaspaceSize`. This resulted in the following content for my
`.sbtopts`:

```
-J-Xms1024m
-J-Xmx1024m
-J-Xss4M
-J-XX:ReservedCodeCacheSize=128m
-J-XX:MaxMetaspaceSize=512m
```

`jitpack.yml` is now just (I like to keep `-v` there to make sure settings are
being correctly applied if something goes wrong):

```
install:
  - sbt -v +publishM2
```


In another project that depends on zio-diffx, I add the dependency to SBT in
the usual way, in this case (it is a test dependency, otherwise you would omit `% Test`):

```scala
"io.github.bbarker" %% "zio-diffx" % "0.0.4" % Test
```


Now after running `reload` and `test:compile` in `sbt`, jitpack will kick off the artifact build. This takes some additional time when the dependency
is being built, but this only happens once per version. Now when I visit `https://jitpack.io/io/github/bbarker/zio-diffx_2.13/0.0.4/` I can see
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


