# JSON Request and Response Bodies

Bodies are converted by two typeclasses shared across the HTTP server and
client:

* **`BodyEncoder[A]`** — used by `Response.ok`, `Response.created`, and request
  bodies. Produces the body string and sets `Content-Type`.
* **`BodyDecoder[A]`** — used by `Request.as[A]`, `HttpResponse.as[A]`, and
  `HttpError.as[E]`. Failures are raised as `DecodingError`.

Built-in instances exist for `String`, `Int`, `Long`, `Double`, and `Boolean`.
JSON comes from one of the two integration modules below; neither is pulled in
by the server or client on its own.

## Dependencies

Pick one JSON backend:

```sbt
// circe
"io.yaes" %% "yaes-http-circe"
"io.circe" %% "circe-generic" % "<latest>"   // only for automatic derivation

// or jsoniter-scala
"io.yaes" %% "yaes-http-jsoniter"
"com.github.plokhotnyuk.jsoniter-scala" %% "jsoniter-scala-macros" % "<latest>" % Provided
```

## circe

Importing `io.yaes.http.circe.given` brings both instances into scope; each is
gated on a single circe constraint, so a type needs only the direction it
actually uses:

```scala
given circeBodyEncoder[A](using Encoder[A]): BodyEncoder[A]
given circeBodyDecoder[A](using Decoder[A]): BodyDecoder[A]
```

```scala
import io.yaes.*
import io.yaes.Log.given
import io.yaes.http.server.*
import io.yaes.http.core.DecodingError
import io.yaes.http.circe.given
import io.circe.{Encoder, Decoder}

case class User(name: String, age: Int) derives Encoder.AsObject, Decoder

val server = YaesServer.route(
  GET(p"/users" / param[Int]("id")) { (req, _, _) =>
    Response.ok(User("Alice", 30))        // encoded to JSON automatically
  },
  POST(p"/users") { req =>
    Raise.fold {
      val user = req.as[User]             // decoded from JSON automatically
      Response.created(user)
    } { case error: DecodingError =>
      Response.badRequest(error.message)
    }
  }
)
```

The encoder writes compact JSON (`asJson.noSpaces`) and sets
`Content-Type: application/json`.

## jsoniter-scala

Same shape, gated on a single `JsonValueCodec[A]`:

```scala
given jsoniterBodyEncoder[A](using JsonValueCodec[A]): BodyEncoder[A]
given jsoniterBodyDecoder[A](using JsonValueCodec[A]): BodyDecoder[A]
```

```scala
import io.yaes.http.jsoniter.given
import com.github.plokhotnyuk.jsoniter_scala.core.*
import com.github.plokhotnyuk.jsoniter_scala.macros.*

case class User(name: String, age: Int)
given JsonValueCodec[User] = JsonCodecMaker.make
```

Codecs are derived at compile time by `JsonCodecMaker.make`, which handles a
whole type in one pass — nested case classes, collections, and simple opaque
types included.

## Choosing between them: error granularity

The two backends do **not** report failures the same way, and it shows up in
what your 400 responses can say.

**circe distinguishes syntax from schema.** A `ParsingFailure` becomes
`DecodingError.ParseError`; a `DecodingFailure` becomes
`DecodingError.ValidationErrors`, carrying a `NonEmptyList[String]` of every
accumulated reason. A client sending well-formed JSON with three bad fields can
be told about all three.

**jsoniter collapses both.** Invalid syntax and a missing or wrong field alike
surface as `JsonReaderException`, mapped to `DecodingError.ParseError`. There is
no `ValidationErrors` mapping, so the response is one message about the first
problem.

> **Important:** pick circe when clients need actionable per-field validation
> errors, and jsoniter when throughput matters more than the shape of your error
> payload. Retrofitting the distinction later means changing every handler that
> pattern-matches on `DecodingError`.

The handler shapes differ accordingly — circe raises a single `DecodingError`,
jsoniter raises a `List[DecodingError]`:

```scala
// circe
Raise.fold { Response.created(req.as[User]) } {
  case error: DecodingError => Response.badRequest(error.message)
}

// jsoniter
Raise.fold { Response.created(req.as[User]) } {
  case errors: List[DecodingError] => Response.badRequest(errors.map(_.message).mkString(", "))
}
```

## Hand-rolled codecs

Neither module is required — any JSON library works by implementing the two
typeclasses directly:

```scala
import io.yaes.{Raise, raises}
import io.circe.syntax.*
import io.circe.parser.decode

case class User(id: Int, name: String)

given BodyEncoder[User] with
  def contentType: String = "application/json"
  def encode(user: User): String = user.asJson.noSpaces

given BodyDecoder[User] with
  def decode(body: String): User raises List[DecodingError] =
    decode[User](body).fold(
      error => Raise.raise(List(DecodingError.ParseError(error.getMessage))),
      user => user
    )
```

This is also the escape hatch for a wire format that isn't JSON at all — set
`contentType` accordingly and the same `Response.ok(value)` / `req.as[A]` calls
work unchanged.

> **Warning:** `Response.ok(value)` picks its `Content-Type` from the encoder.
> If you serialise a body yourself and pass the resulting `String`, you get the
> built-in `String` encoder and `text/plain`. Override it explicitly with
> `Response.ok(rawJson, extraHeaders = Map(Headers.ContentType -> "application/json"))`
> — `extraHeaders` wins on collision. See [HTTP Server](400-http-server.md).
