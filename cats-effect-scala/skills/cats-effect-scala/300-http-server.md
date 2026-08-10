# HTTP Server with http4s

[http4s](https://http4s.org) models a service as a function
`Request[F] => F[Response[F]]`. Everything else — routing, middleware,
composition — is ordinary function composition.

## Dependencies

```sbt
val http4sVersion = "0.23.36"

libraryDependencies ++= Seq(
  "org.http4s" %% "http4s-ember-server" % http4sVersion,
  "org.http4s" %% "http4s-dsl"          % http4sVersion,
  "org.http4s" %% "http4s-circe"        % http4sVersion,
)
```

> **Note:** `0.23.x` is the stable series. `1.0.0-Mxx` is a milestone line —
> capable, but its API still shifts. Use `0.23.x` unless you have a specific
> reason not to.

## The core types

| Type | Meaning |
| --- | --- |
| `HttpRoutes[F]` | `Request => OptionT[F, Response]` — may not match |
| `HttpApp[F]` | `Request => F[Response]` — total, always responds |
| `Http[F, G]` | the general form both specialise |

Routes compose with `<+>` (first match wins) and become an `HttpApp` with
`.orNotFound`.

## Defining routes

```scala
import cats.effect.IO
import org.http4s.*
import org.http4s.dsl.io.*

val helloRoutes: HttpRoutes[IO] = HttpRoutes.of[IO]:
  case GET -> Root / "hello" / name =>
    Ok(s"Hello, $name.")

  case GET -> Root / "users" / UUIDVar(id) =>
    userService.find(UserId(id)).flatMap:
      case Some(user) => Ok(user)
      case None       => NotFound()

  case req @ POST -> Root / "users" =>
    for
      body     <- req.as[CreateUser]
      created  <- userService.create(body)
      response <- Created(created)
    yield response

  case DELETE -> Root / "users" / UUIDVar(id) =>
    userService.delete(UserId(id)) *> NoContent()
```

The DSL's extractors are plain pattern matches:

* `Root` — the start of the path; `/` separates segments.
* `UUIDVar`, `IntVar`, `LongVar` — typed path parameters.
* `req @ POST -> ...` — bind the request when you need the body or headers.
* `-> Root / "files" / path` with `path: Path` — match the remainder.

Query parameters use matchers:

```scala
object PageParam    extends OptionalQueryParamDecoderMatcher[Int]("page")
object SizeParam    extends OptionalQueryParamDecoderMatcher[Int]("size")
object RequiredSort extends QueryParamDecoderMatcher[String]("sort")

val listRoutes: HttpRoutes[IO] = HttpRoutes.of[IO]:
  case GET -> Root / "users" :? PageParam(page) +& SizeParam(size) =>
    userService.list(page.getOrElse(0), size.getOrElse(20)).flatMap(Ok(_))
```

For a validated matcher that reports a decode failure instead of not matching,
use `ValidatingQueryParamDecoderMatcher`.

## Composing and mounting

```scala
import cats.syntax.all.*
import org.http4s.implicits.*
import org.http4s.server.Router

val apiRoutes: HttpRoutes[IO] = userRoutes <+> orderRoutes
val httpApp: HttpApp[IO] =
  Router(
    "/"        -> healthRoutes,
    "/api/v1"  -> apiRoutes,
  ).orNotFound
```

> **Important:** `<+>` tries each route set in order and takes the first that
> matches. A route that matches a path but rejects the *method* still consumes
> the match, so order matters — put more specific routes first.

## Starting the server

```scala
import com.comcast.ip4s.{Host, Port, ipv4, port}
import org.http4s.ember.server.EmberServerBuilder
import org.http4s.server.Server

def serverResource(
    host: Host,
    port: Port,
    httpApp: HttpApp[IO],
): Resource[IO, Server] =
  EmberServerBuilder
    .default[IO]
    .withHost(host)
    .withPort(port)
    .withHttpApp(httpApp)
    .withShutdownTimeout(30.seconds)
    .build
```

Run it from `IOApp`:

```scala
object Main extends IOApp.Simple:
  val run: IO[Unit] =
    serverResource(ipv4"0.0.0.0", port"8080", httpApp).useForever
```

Because the server is a `Resource` and `IOApp` cancels on SIGTERM, shutdown is
graceful for free: Ember stops accepting connections, waits up to
`withShutdownTimeout` for in-flight requests, then releases.

> **Warning:** binding to `127.0.0.1` (the `EmberServerBuilder` default host is
> loopback) makes the service unreachable from outside a container. Set
> `ipv4"0.0.0.0"` explicitly for anything containerised.

## Middleware

Middleware is a function `HttpRoutes[F] => HttpRoutes[F]`:

```scala
import org.http4s.server.middleware.*

val app: HttpApp[IO] =
  ErrorHandling.Recover.total(          // 500 instead of a dropped connection
    ErrorAction.log(                    // log the failure
      CORS.policy.withAllowOriginAll(
        Logger.httpApp(logHeaders = true, logBody = false)(
          Router("/api" -> apiRoutes).orNotFound
        )
      ),
      messageFailureLogAction = (t, msg) => logger.warn(t)(msg),
      serviceErrorLogAction   = (t, msg) => logger.error(t)(msg),
    )
  )
```

Commonly used: `Logger`, `CORS`, `GZip`, `Timeout`, `RequestId`,
`ErrorHandling`, `ErrorAction`, `HSTS`, `AutoSlash`, `CSRF`.

> **Important:** never log request bodies in production (`logBody = true`) —
> they routinely contain credentials and personal data.

## Authentication

`AuthMiddleware` turns an `HttpRoutes` into an `AuthedRoutes` carrying the
authenticated principal:

```scala
import cats.data.{Kleisli, OptionT}
import org.http4s.server.AuthMiddleware

val authUser: Kleisli[OptionT[IO, *], Request[IO], User] =
  Kleisli { req =>
    for
      header <- OptionT.fromOption[IO](req.headers.get[Authorization])
      token  <- OptionT.fromOption[IO](bearerToken(header))
      user   <- OptionT(tokenService.validate(token))
    yield user
  }

val onFailure: AuthedRoutes[String, IO] =
  Kleisli(req => OptionT.liftF(Forbidden(req.context)))

val middleware = AuthMiddleware(authUser, onFailure)

val securedRoutes: AuthedRoutes[User, IO] = AuthedRoutes.of[User, IO]:
  case GET -> Root / "me" as user => Ok(user)

val routes: HttpRoutes[IO] = middleware(securedRoutes)
```

## Streaming responses

A `Response` body is an `fs2.Stream[F, Byte]`, so streaming is the default
rather than a special case:

```scala
case GET -> Root / "export" =>
  val rows: Stream[IO, String] = userService.streamAll.map(_.asJson.noSpaces + "\n")
  Ok(rows.through(fs2.text.utf8.encode))

// Server-sent events:
case GET -> Root / "events" =>
  Ok(eventStream.map(e => ServerSentEvent(data = Some(e.asJson.noSpaces))))
```

The connection stays open and memory stays flat regardless of result size.

## Testing routes without a server

`HttpRoutes` is a function — call it directly:

```scala
val response: IO[Response[IO]] =
  routes.orNotFound.run(Request[IO](Method.GET, uri"/hello/world"))
```

See the testing chapter for assertions on status and body.
