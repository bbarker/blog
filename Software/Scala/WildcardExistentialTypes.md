# Existential Types and Variance in Scala

# TODO
1. Considering looking into contravariance examples
2. Test code (at least some) in Scala 2 to see if results hold there

## Background

Recently, I came across a wildcard type in a library we use ([Tapir](https://tapir.softwaremill.com/)).
Up till then I had largely viewed wildcard types in Scala as something that could be avoided. Could I
avoid them here? As it turns out, I could not. In what follows, I hope to show why existential types
can be useful (and necessary) in Scala, and how they relate to parametric types (also known as higher
kinded types, or HKTs) and `Any`.

Wildcard types are the simplest form of existentially quantified types in Scala 2, and in Scala 3,
they are the [only form](https://docs.scala-lang.org/scala3/reference/dropped-features/existential-types.html)
of existential types.



This works:

```scala
 def apply[Env](
    serverEndpoints: List[ServerEndpoint[_, _, _, AkkaStreams & WebSockets, Future]],
    layer: ULayer[Env],
    serverOptions: Option[AkkaHttpServerOptions],
): Route =  serverOptions
    .fold(ifEmpty = AkkaHttpServerInterpreter())(serverOptions => AkkaHttpServerInterpreter(serverOptions))
    .toRoute(serverEndpoints)
```

The call site looks like:


```scala
type Env = Has[BetBoostTokenService[RIOCB]] & Clock & Blocking & Authorization

def serverEndpoints(layer: ULayer[Env]): List[ServerEndpoint[_, _, _, AkkaStreams & WebSockets, Future]] = List(
  ServerEndpoint(tokensByUniverseAccountDef, (_: MonadError[Future]) => unsafeToFutureHandler(tokensByUniverseAccountHandler, layer)),
  ServerEndpoint(tokenByUniverseAccountDef, (_: MonadError[Future]) => unsafeToFutureHandler(tokenByUniverseAccountHandler, layer)),
  ServerEndpoint(
    refundTokenByUniverseAccountDef,
    (_: MonadError[Future]) => unsafeToFutureHandler(refundTokenByUniverseAccountHandler, layer),
  ),
)

def routes(layer: ULayer[Has[BetBoostTokenService[RIOCB]] & Clock & Blocking & Authorization]): Route = ZioAkkaHttpRoute.applyTest(
  serverEndpoints(layer),
  layer,
  None,
)
```


However, if we use a polymorphic type:

```scala
def applyTest[Input, Error, Response, Env]
```

Or even polymorphic types with bounds:

```scala
def applyTest[Input >: Nothing <: Any, Error  >: Nothing <: Any, Response >: Nothing <: Any, Env]
```

We will still get an error at the call site:

```
Omitted.scala:54:20: type mismatch;
[error]   List[
[error]     ServerEndpoint[
[error]       _$1|Any
[error]       ,
[error]       _$2|Any
[error]       ,
[error]       _$3|Any
[error]       ,
[error]       AkkaStreams with WebSockets
[error]       ,
[error]       Future
[error]     ]
[error]   ]
[error]     serverEndpoints(layer),
```
