# HTTP Client

`yaes-http-client` wraps Java's `java.net.http.HttpClient` in yaes effects. The
client's lifetime is managed by `Resource`, transport failures are raised as
`ConnectionError` via `Raise`, and HTTP status failures are raised separately as
`HttpError` — only when you decode.

## Dependencies

```sbt
"io.yaes" %% "yaes-http-client"
```

## Creating a client

`YaesClient.make` registers the underlying Java client for cleanup, so it must
run inside `Resource.run`:

```scala
import io.yaes.*
import io.yaes.http.client.*
import scala.concurrent.duration.*

Sync.runBlocking(30.seconds) {
  Raise.run[ConnectionError] {
    Resource.run {
      val client = YaesClient.make()

      Raise.run[Uri.InvalidUri] {
        val response = client.send(HttpRequest.get(Uri("https://example.com/api")))
        println(s"${response.status}: ${response.body}")
      }
    }
  }
}
```

Configure it with `YaesClientConfig`:

```scala
val client = YaesClient.make(YaesClientConfig(
  connectTimeout  = Some(5.seconds),
  followRedirects = RedirectPolicy.Never,
  httpVersion     = HttpVersion.Http2
))
```

Create the client **once** and reuse it — see [Resource — Lifecycle
Management](120-resource-effect.md) for where the acquisition belongs.

## Building requests

```scala
val get     = HttpRequest.get(uri)
val head    = HttpRequest.head(uri)
val delete  = HttpRequest.delete(uri)
val options = HttpRequest.options(uri)

// Bodies require a BodyEncoder in scope — see chapter 420
val post  = HttpRequest.post(uri, """{"name": "Alice"}""")
val put   = HttpRequest.put(uri, """{"name": "Bob"}""")
val patch = HttpRequest.patch(uri, """{"name": "Charlie"}""")
```

Requests are immutable; the fluent methods return new values:

```scala
val request = HttpRequest.get(uri)
  .header("Authorization", "Bearer my-token")
  .queryParam("page", "1")
  .queryParam("limit", "10")
  .timeout(30.seconds)
```

`header` adds or **replaces** (keys are lowercased), `queryParam` appends and
allows duplicate keys, and `timeout` sets the per-request timeout — a non-finite
or non-positive duration clears it rather than failing.

## URIs and path parameters

`Uri` is an opaque type validated at construction, so building one raises
`Uri.InvalidUri`:

```scala
Raise.run {
  val valid   = Uri("https://example.com/api")
  val invalid = Uri("not a valid uri :::")   // raises InvalidUri
}
```

For paths built from values, prefer the `uri"…"` interpolator. Literal parts are
validated **at compile time** and interpolated values are URL-encoded via
`PathParamStringifier`, so no `Raise` is needed at runtime:

```scala
val userId: Int     = 42
val orderId: String = "ord-99"

HttpRequest.get(uri"https://api.example.com/users/$userId/orders/$orderId")
```

Built-in instances cover `String`, `Int`, `Long`, `Boolean`, `Double`, and
`UUID`. Supply your own for domain types:

```scala
case class ItemId(value: Int)
given PathParamStringifier[ItemId] with
  def encode(v: ItemId): String = s"item-${v.value}"

HttpRequest.get(uri"https://api.example.com/items/$id")  // → /items/item-5
```

> **Important:** a missing `PathParamStringifier` is a compile error — the
> interpolator never falls back to `.toString`. That is what stops an opaque id
> or a case class from silently landing in a URL as `ItemId(5)`.

To append segments to an existing `Uri`, use `/`. It encodes through the same
typeclass and preserves query strings and fragments:

```scala
val uri = Uri("https://api.example.com") / "users" / 42 / "orders"

val base = Uri("https://api.example.com/users?active=true")
val one  = base / 42     // → https://api.example.com/users/42?active=true
```

## Sending and reading responses

```scala
val response: HttpResponse = client.send(request)

response.status                  // Int
response.body                    // String
response.header("content-type")  // Option[String], case-insensitive
```

> **Important:** `send` returns the response whatever the status — it raises
> only `ConnectionError`, never `HttpError`. A 500 is a perfectly good return
> value here. Status is enforced at decode time by `response.as[A]`, so code
> that calls `send` and reads `.body` without decoding will happily treat an
> error page as success.

```scala
Raise.run[HttpError | DecodingError] {
  val user = response.as[User]
}
```

## The two error layers

**`ConnectionError`** — raised by `send`, meaning the request never completed:
`ConnectionRefused(host, port)`, `ConnectTimeout(host)`, `RequestTimeout(url)`,
`Unexpected(cause)`.

**`HttpError`** — raised by `response.as[A]` for any non-2xx status. The
variants map to familiar codes (`BadRequest` 400, `Unauthorized` 401,
`Forbidden` 403, `NotFound` 404, `MethodNotAllowed` 405, `Conflict` 409, `Gone`
410, `UnprocessableEntity` 422, `TooManyRequests` 429, `InternalServerError`
500, `BadGateway` 502, `ServiceUnavailable` 503, `GatewayTimeout` 504), with
`OtherClientError(status, body)` and `OtherServerError(status, body)` catching
the rest.

Match on the `ClientHttpError` / `ServerHttpError` marker traits to treat the
categories differently — a 4xx is usually a bug in your request, a 5xx is
usually worth retrying:

```scala
Raise.either[HttpError | DecodingError, String](response.as[String]) match
  case Left(e: ClientHttpError)   => log.warn(s"client error ${e.status}: ${e.body}")
  case Left(e: ServerHttpError)   => retry()
  case Left(error: DecodingError) => log.error(s"decode failed: ${error.message}")
  case Right(value)               => use(value)
```

Keeping the layers separate is the point: a `ConnectionError` means retrying may
help, while a `NotFound` means it never will. See [Side-Effecting
Services](200-side-effecting-services.md) for wiring that distinction into
`Retry` and `CircuitBreaker` — and type both at the full error union.

### Typed error bodies

APIs that return a structured payload alongside a non-2xx status can be decoded
with `err.as[E]`, using the same `BodyDecoder` machinery as the success path:

```scala
case class ApiError(field: String, message: String)

Raise.fold {
  response.as[User]
} {
  case err: HttpError             => Raise.either[DecodingError, ApiError | User](err.as[ApiError])
  case error: DecodingError       => Left(error)
} {
  user => Right(user)
}
```

`err.as[E]` raises `DecodingError` on failure, exactly like `response.as[A]`.
