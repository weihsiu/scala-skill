# Observability and Logging

## Dependencies

```sbt
libraryDependencies ++= Seq(
  "org.typelevel" %% "log4cats-slf4j"  % "2.8.0",
  "ch.qos.logback" % "logback-classic" % "1.5.18" % Runtime,
  "org.typelevel" %% "otel4s-oteljava" % "1.1.0",
  "io.opentelemetry" % "opentelemetry-exporter-otlp"                % "1.64.0" % Runtime,
  "io.opentelemetry" % "opentelemetry-sdk-extension-autoconfigure"  % "1.64.0" % Runtime,
)
javaOptions += "-Dotel.java.global-autoconfigure.enabled=true"
```

## Logging with log4cats

Logging is a side effect, so it belongs in `F`. `log4cats` wraps SLF4J:

```scala
import cats.effect.IO
import org.typelevel.log4cats.Logger
import org.typelevel.log4cats.slf4j.Slf4jLogger

class UserService(repo: UserRepo)(using Logger[IO]):

  def register(cmd: CreateUser): IO[User] =
    for
      _    <- Logger[IO].info(s"registering ${cmd.email}")
      user <- repo.insert(cmd).onError(e => Logger[IO].error(e)("insert failed"))
      _    <- Logger[IO].info(s"registered ${user.id}")
    yield user
```

Create the logger once, in the composition root, and pass it as a `given`:

```scala
object Main extends IOApp.Simple:
  val run: IO[Unit] =
    Slf4jLogger.create[IO].flatMap { logger =>
      given Logger[IO] = logger
      application.useForever
    }
```

> **Warning:** `Slf4jLogger.getLogger[F]` builds a logger *unsafely* outside
> `F`. It is convenient but bypasses the effect system and can surprise you in
> tests. Prefer `Slf4jLogger.create[F]`, which returns `F[Logger[F]]`.

For effect-polymorphic code, take the logger as a constraint:

```scala
def process[F[_]: Async: Logger](batch: Batch): F[Unit] =
  Logger[F].info(s"processing ${batch.size}") *> doWork(batch)
```

### Structured logging

Flat message strings are hard to query. Attach key/value context instead:

```scala
import org.typelevel.log4cats.StructuredLogger

def handle(request: Request)(using logger: StructuredLogger[IO]): IO[Response] =
  val context = Map(
    "request_id" -> request.id,
    "user_id"    -> request.userId.toString,
    "path"       -> request.uri.path.toString,
  )
  logger.info(context)("handling request") *> process(request)
```

With `logstash-logback-encoder` configured, these become JSON fields your log
aggregator can filter on.

> **Important:** never log secrets, tokens, passwords, full request bodies, or
> personal data. Wrap sensitive configuration in `ciris.Secret`, whose
> `toString` is redacted, so an accidental log line cannot leak it.

## Request context with IOLocal

Threading a request id through every signature is noise. `IOLocal` carries it
implicitly across fibers:

```scala
import cats.effect.IOLocal

final class RequestContext(local: IOLocal[Option[RequestId]]):
  def set(id: RequestId): IO[Unit] = local.set(Some(id))
  def get: IO[Option[RequestId]]   = local.get

// Middleware sets it per request; loggers read it:
def withRequestId(ctx: RequestContext)(routes: HttpRoutes[IO]): HttpRoutes[IO] =
  HttpRoutes { request =>
    val id = request.headers.get[`X-Request-Id`].map(_.value).getOrElse(newId())
    OptionT.liftF(ctx.set(RequestId(id))) *> routes(request)
  }
```

`IOLocal` values are inherited by child fibers, so work spawned from a request
handler keeps the context. http4s also ships a `RequestId` middleware that does
the header handling for you.

## Tracing and metrics with otel4s

[otel4s](https://typelevel.org/otel4s/) is the OpenTelemetry implementation for
Cats Effect. It uses `IOLocal` for context propagation, so spans nest correctly
across fibers without manual threading.

```scala
import cats.effect.{IO, IOApp}
import org.typelevel.otel4s.oteljava.OtelJava
import org.typelevel.otel4s.trace.{Tracer, TracerProvider}
import org.typelevel.otel4s.metrics.MeterProvider

object Main extends IOApp.Simple:
  val run: IO[Unit] =
    OtelJava.autoConfigured[IO]().use { otel =>
      for
        given Tracer[IO] <- otel.tracerProvider.get("my-service")
        meter            <- otel.meterProvider.get("my-service")
        requests         <- meter.counter[Long]("http.requests").create
        _                <- application(requests).useForever
      yield ()
    }
```

Creating spans:

```scala
def register(cmd: CreateUser)(using Tracer[IO]): IO[User] =
  Tracer[IO].span("user.register").surround:
    for
      _    <- Tracer[IO].span("validate").surround(validate(cmd))
      user <- Tracer[IO].span("db.insert").surround(repo.insert(cmd))
    yield user

// Attach attributes:
Tracer[IO]
  .spanBuilder("payment.charge")
  .addAttribute(Attribute("payment.amount", amount.cents))
  .build
  .surround(gateway.charge(request))
```

`surround` starts the span, runs the effect, and ends the span on success,
error, or cancellation.

Configure the exporter through standard OpenTelemetry environment variables —
`OTEL_SERVICE_NAME`, `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_TRACES_SAMPLER` — so
no code change is needed per environment.

> **Note:** `Tracer.noop[IO]` is the right instance for tests and for library
> code whose caller has not opted into tracing. Never make tracing mandatory in
> a signature you expect others to call.

### Metrics

```scala
for
  meter    <- otel.meterProvider.get("my-service")
  counter  <- meter.counter[Long]("orders.placed").create
  hist     <- meter.histogram[Double]("order.value").create
  _        <- counter.inc(Attribute("channel", "web"))
  _        <- hist.record(order.total.toDouble)
yield ()
```

http4s provides `Metrics` middleware for both server and client that records
request counts, durations, and active requests automatically.

## Debugging with fiber dumps

When a program hangs, Cats Effect can dump the state of every live fiber —
the async equivalent of a thread dump. On the JVM, send `SIGQUIT` (or press
`Ctrl-\`) and the runtime prints each fiber with its status and trace.

Make the traces useful by keeping tracing enabled (it is on by default in
development):

```
-Dcats.effect.tracing.mode=full        # richest traces, slowest
-Dcats.effect.tracing.mode=cached      # the default
-Dcats.effect.tracing.mode=none        # fastest, no traces
```

Cats Effect also detects **starvation**: if the compute pool cannot schedule
work promptly it logs a warning. That warning almost always means a blocking
call is running on the compute pool — find it and wrap it in `IO.blocking`.

## Health checks

Expose liveness and readiness separately. Liveness says the process is up;
readiness says its dependencies are reachable.

```scala
val healthRoutes: HttpRoutes[IO] = HttpRoutes.of[IO]:
  case GET -> Root / "health" / "live" =>
    Ok("ok")

  case GET -> Root / "health" / "ready" =>
    (checkDatabase, checkCache)
      .parMapN(_ && _)
      .ifM(Ok("ready"), ServiceUnavailable("not ready"))
```

> **Important:** keep the readiness check cheap and give it a timeout. A check
> that runs an expensive query, or that hangs when the database is slow, turns
> a degraded dependency into a restart loop.
