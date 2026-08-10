# HTTP Client with http4s

The http4s `Client[F]` is the mirror image of the server: a function from a
`Request` to a `Resource[F, Response]`.

## Dependencies

```sbt
libraryDependencies += "org.http4s" %% "http4s-ember-client" % "0.23.36"
```

## Acquiring a client

The client owns a connection pool, so it is a `Resource` and MUST be created
once for the application — not per request.

```scala
import cats.effect.{IO, Resource}
import org.http4s.client.Client
import org.http4s.ember.client.EmberClientBuilder
import scala.concurrent.duration.*

val clientResource: Resource[IO, Client[IO]] =
  EmberClientBuilder
    .default[IO]
    .withTimeout(10.seconds)
    .withIdleConnectionTime(60.seconds)
    .withMaxTotal(100)
    .build
```

> **Warning:** creating a client per request (`EmberClientBuilder...build.use`
> inside a handler) opens a new pool every time. Under load this exhausts file
> descriptors. Build it once in the composition root and pass it down.

## Making requests

```scala
import org.http4s.*
import org.http4s.client.dsl.io.*
import org.http4s.Method.*
import org.http4s.circe.CirceEntityCodec.given

// expect — decodes the body, raises on non-2xx:
val user: IO[User] = client.expect[User](uri"https://api.example.com/users/1")

// With a full request:
val created: IO[User] =
  client.expect[User](
    Request[IO](POST, uri"https://api.example.com/users")
      .withEntity(CreateUser("ada"))
      .putHeaders(Authorization(Credentials.Token(AuthScheme.Bearer, token)))
  )

// expectOr — map the error response yourself:
val fetched: IO[User] =
  client.expectOr[User](req) { failed =>
    failed.as[ApiError].map(e => ServiceError(failed.status, e.message))
  }

// run — full control over the response, including status:
val outcome: IO[Either[ApiError, User]] =
  client.run(req).use { response =>
    response.status match
      case Status.Ok       => response.as[User].map(Right(_))
      case Status.NotFound => IO.pure(Left(ApiError.NotFound))
      case status          => response.as[String].flatMap(b =>
                                IO.raiseError(UnexpectedStatus(status, b)))
  }
```

| Method | Returns | Non-2xx |
| --- | --- | --- |
| `expect[A]` | `F[A]` | raises `UnexpectedStatus` |
| `expectOption[A]` | `F[Option[A]]` | `None` on 404, raises otherwise |
| `expectOr[A]` | `F[A]` | your handler decides the error |
| `run` | `Resource[F, Response[F]]` | you inspect the status |
| `stream` | `Stream[F, Response[F]]` | streaming responses |

> **Important:** the response body is only readable inside the `Resource` /
> `use` block. Returning `response` itself, or a lazily-decoded value, gives
> you a closed connection. Decode inside the block.

## URIs and query parameters

```scala
import org.http4s.implicits.*     // the uri"" interpolator

val base = uri"https://api.example.com/users"
val withQuery = base
  .withQueryParam("page", 2)
  .withQueryParam("size", 50)
  .withOptionQueryParam("sort", maybeSort)

// Dynamic path segments — use / rather than string interpolation, so
// segments are encoded correctly:
val userUri = uri"https://api.example.com" / "users" / userId.toString
```

> **Warning:** never build a URI by interpolating unescaped user input into a
> string. `Uri.unsafeFromString(s"…/$name")` breaks on a name containing `/` or
> `?`. The `/` operator encodes each segment.

## Client middleware

Middleware wraps a `Client` the same way it wraps routes:

```scala
import org.http4s.client.middleware.*

val instrumented: Client[IO] =
  Logger(logHeaders = true, logBody = false)(
    Retry(RetryPolicy(RetryPolicy.exponentialBackoff(maxWait = 5.seconds, maxRetry = 3)))(
      FollowRedirect(maxRedirects = 3)(
        client
      )
    )
  )
```

Available: `Retry`, `FollowRedirect`, `Logger`, `RequestLogger`,
`ResponseLogger`, `GZip`, `Metrics`, `Caching`.

`RetryPolicy.defaultRetriable` retries idempotent methods on 5xx and connection
failures only — the right default. Retrying a `POST` needs an idempotency key
on the server side.

## Wrapping a client in a service

Keep http4s types out of your domain. A thin adapter makes the dependency
testable:

```scala
trait PaymentGateway:
  def charge(request: ChargeRequest): IO[Either[ChargeError, Receipt]]

object PaymentGateway:
  def http(client: Client[IO], baseUri: Uri, apiKey: String): PaymentGateway =
    new PaymentGateway:
      def charge(request: ChargeRequest): IO[Either[ChargeError, Receipt]] =
        val req = Request[IO](POST, baseUri / "charges")
          .withEntity(request)
          .putHeaders(Header.Raw(ci"X-Api-Key", apiKey))

        client.run(req).use { response =>
          response.status match
            case Status.Ok                 => response.as[Receipt].map(Right(_))
            case Status.PaymentRequired    => response.as[ApiError].map(e => Left(ChargeError.Declined(e.message)))
            case Status.TooManyRequests    => IO.pure(Left(ChargeError.RateLimited))
            case s                         => IO.raiseError(UnexpectedStatus(s, req.uri))
        }.timeout(10.seconds)
```

Tests substitute a stub `PaymentGateway`; no HTTP mocking needed. When you do
want to exercise the HTTP layer itself, build a client from routes with no
socket involved:

```scala
import org.http4s.client.Client

val stubClient: Client[IO] = Client.fromHttpApp(stubRoutes.orNotFound)
```

## Streaming request and response bodies

```scala
// Streaming upload — constant memory:
val upload: IO[Unit] =
  client.expect[Unit](
    Request[IO](PUT, uri"https://example.com/upload")
      .withEntity(fs2.io.file.Files[IO].readAll(path))
  )

// Streaming download, line by line:
val lines: Stream[IO, String] =
  client.stream(Request[IO](GET, uri"https://example.com/big.csv"))
    .flatMap(_.body.through(fs2.text.utf8.decode).through(fs2.text.lines))
```
