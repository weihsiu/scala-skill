# HTTP Server

`yaes-http-server` is a type-safe HTTP/1.1 server built on `java.net.ServerSocket`
with one virtual thread per request. Each request is handled in its own fiber via
`Async.fork`, so structured concurrency, cancellation, and graceful shutdown all
work the way they do everywhere else in yaes.

## Dependencies

```sbt
"io.yaes" %% "yaes-http-server"
```

> **Required:** Java 24+ and Scala 3.7.4+. The routing DSL relies on named
> tuples and context functions.

## Starting a server

The server needs `Sync`, `Log`, and `Shutdown` in scope. Handlers are declared
inline and `run` blocks until shutdown is initiated:

```scala
import io.yaes.*
import io.yaes.Log.given
import io.yaes.http.server.*
import scala.concurrent.duration.*
import scala.concurrent.ExecutionContext.Implicits.global

Sync.runBlocking(Duration.Inf) {
  Shutdown.run {
    Log.run() {
      val server = YaesServer.route(
        GET(p"/hello") { req => Response.ok("Hello, World!") },
        POST(p"/echo") { req => Response.ok(req.body) }
      )
      server.run(port = 8080)
    }
  }
}.get
```

`Routes(...)` builds the same route list as a standalone value when you want to
assemble it separately from the server.

## Routing DSL

Paths are written with the `p` interpolator. Typed parameters are declared once
with `param[T]("name")` and composed into the path with `/`:

```scala
val userId = param[Int]("userId")
val postId = param[Long]("postId")

val routes = Routes(
  GET(p"/health") { req => Response.ok("OK") },

  GET(p"/users" / userId) { (req, path, _) =>
    Response.ok(s"User ${path.userId}")
  },

  GET(p"/users" / userId / "posts" / postId) { (req, path, _) =>
    Response.ok(s"User ${path.userId}, Post ${path.postId}")
  }
)
```

Query parameters use `?` for the first and `&` for each subsequent one.
Optionality rides in the element type — `queryParam[Option[Int]]("page")`, not a
separate combinator:

```scala
GET(p"/search" ? queryParam[String]("q") & queryParam[Int]("limit")) {
  (req, _, query) => Response.ok(s"${query.q}, limit ${query.limit}")
}

GET(p"/search" ? queryParam[Option[Int]]("page")) { (req, _, query) =>
  Response.ok(s"Page ${query.page.getOrElse(1)}")
}
```

Both kinds compose in one route, and the handler receives them separately:

```scala
GET(p"/users" / userId ? queryParam[String]("include")) { (req, path, query) =>
  Response.ok(s"User ${path.userId} with ${query.include}")
}
```

### Handler shapes and named tuples

A route with no parameters takes a one-argument handler, `{ req => … }`. A route
with any parameter takes `(request, path, query)`, where `path` and `query` are
**named tuples**: read fields by the name given to `param` / `queryParam`
(`path.userId`), so access is order-independent and self-documenting. Ignore the
tuple you don't need with `_`.

> **Warning:** path and query parameters were encoded as bespoke HList-like
> structures before yaes 0.23.0, with handlers destructuring positionally. The
> named-tuple rewrite is a breaking change — examples written against 0.22.x do
> not compile, and neither do handlers that pattern-match positionally.

Built-in parameter types are `String`, `Int`, and `Long`; supply a
`given PathParamParser[T]` for anything else. Paths and query values are
URL-decoded automatically (`/users/john%20doe` arrives as `"john doe"`).

Methods available: `GET`, `POST`, `PUT`, `DELETE`, `PATCH`.

### Matching order

Exact routes (no parameters) are matched first via an O(1) map lookup, then
parameterized routes **in definition order**. The first match wins; anything
unmatched returns 404. Order your parameterized routes from most to least
specific — `p"/users" / userId` declared before `p"/users" / "me"` will swallow
`/users/me` whenever the parameter type accepts it.

## Request and response

```scala
case class Request(
    method: Method,                        // Method.GET, Method.POST, …
    path: String,                          // URL-decoded
    headers: Map[String, String],          // names lowercased
    body: String,
    queryString: Map[String, List[String]] // URL-decoded, raw
)

req.header("Content-Type")  // Option[String], case-insensitive
req.queryString.get("q")    // Option[List[String]]
```

Header names are stored lowercase for HTTP/1.1 compliance, so `header` lookups
are case-insensitive regardless of how the client sent them.

Responses come from factory methods, each accepting an optional
`extraHeaders: Map[String, String]`:

```scala
Response.ok(body)                       // 200
Response.created(body)                  // 201
Response.accepted(body)                 // 202
Response.noContent()                    // 204
Response.badRequest(message)            // 400
Response.notFound(message)              // 404
Response.internalServerError(message)   // 500
Response.serviceUnavailable(message)    // 503
Response.withStatus(status, value)      // anything else

Response.created(user, extraHeaders = Map("location" -> s"/users/${user.id}"))
Response.withStatus(301, "", extraHeaders = Map("location" -> "/new-path"))
```

Header names in `extraHeaders` are lowercased for you. Methods that encode a
body set `content-type` from the encoder, and `extraHeaders` wins on collision —
which is how you override it for a pre-serialised body. See [JSON Request and
Response Bodies](420-json-bodies.md) for typed bodies.

## Configuration

```scala
server.run(port = 8080)
server.run(port = 8080, deadline = Deadline.after(10.seconds))

server.run(ServerConfig(
  port          = 8080,
  deadline      = Deadline.after(30.seconds),
  maxHeaderSize = 16384,    // 16 KB
  maxBodySize   = 1048576   // 1 MB
))
```

## Graceful shutdown

`Shutdown` registers JVM hooks for SIGTERM, SIGINT, and normal JVM shutdown, so
this works whether the process is killed or stops on its own. To keep doing work
while the server runs, fork it:

```scala
Sync.runBlocking(Duration.Inf) {
  Shutdown.run {
    Log.run() {
      Shutdown.onShutdown {
        println("Cleaning up resources…")
      }

      val server = YaesServer.route(
        GET(p"/work") { req =>
          Async.delay(5.seconds)
          Response.ok("Done")
        }
      )

      val serverFiber = Async.forkNamed("server") { server.run(port = 8080) }
      // … other work …
      Shutdown.initiateShutdown()
      serverFiber.join()
    }
  }
}.get
```

Once shutdown begins, the server stops accepting connections, in-flight requests
keep running until the deadline, and **new requests get 503 Service
Unavailable** rather than being dropped. After the deadline the remaining
requests are interrupted, then shutdown hooks run. See [Async — Structured
Concurrency](130-async-structured-concurrency.md) for `Shutdown` and deadlines
in general.

## Pitfalls

> **Important:** this is a deliberately small HTTP/1.1 implementation. There is
> no keep-alive (one request per connection), no chunked transfer encoding, no
> request streaming, no WebSocket upgrade, and **no TLS** — terminate HTTPS at a
> reverse proxy. Bodies are fully buffered in memory, so bound them with
> `maxBodySize` rather than assuming a streaming server's limits.

> **Warning:** `param[T]` and `queryParam[T]` names are what the handler reads
> fields by. Renaming a parameter silently changes the named-tuple field, so the
> handler stops compiling — which is the point, but it means the string is part
> of your API surface, not a label.
