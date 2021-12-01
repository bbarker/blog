

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
