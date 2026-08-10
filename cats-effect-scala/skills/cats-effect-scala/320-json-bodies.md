# JSON Bodies with circe

[circe](https://circe.github.io/circe/) provides `Encoder[A]` / `Decoder[A]`;
`http4s-circe` lifts those into `EntityEncoder` / `EntityDecoder` so they work
as request and response bodies.

## Dependencies

```sbt
val circeVersion = "0.14.16"

libraryDependencies ++= Seq(
  "org.http4s" %% "http4s-circe"    % "0.23.36",
  "io.circe"   %% "circe-core"      % circeVersion,
  "io.circe"   %% "circe-generic"   % circeVersion,
  "io.circe"   %% "circe-literal"   % circeVersion % Test,
)
```

## Deriving codecs in Scala 3

```scala
import io.circe.{Decoder, Encoder}
import io.circe.generic.semiauto.{deriveDecoder, deriveEncoder}

final case class User(id: UUID, name: String, email: String)

object User:
  given Encoder[User] = deriveEncoder
  given Decoder[User] = deriveDecoder
```

Scala 3 also supports `derives` on the class itself:

```scala
import io.circe.Codec

final case class User(id: UUID, name: String, email: String) derives Codec.AsObject
```

> **Important:** prefer semi-automatic derivation (`deriveEncoder` /
> `derives`) over `import io.circe.generic.auto.*`. Automatic derivation
> re-derives the codec at every use site, which inflates compile times
> dramatically on large models and silently derives codecs you did not intend.

Enums and sealed traits:

```scala
enum Status derives Codec.AsObject:
  case Active
  case Suspended(reason: String)

// Simple enums as plain strings:
enum Role:
  case Admin, Member

object Role:
  given Encoder[Role] = Encoder.encodeString.contramap(_.toString)
  given Decoder[Role] = Decoder.decodeString.emap: s =>
    Role.values.find(_.toString == s).toRight(s"invalid role: $s")
```

Newtypes and value classes need explicit codecs — derive them from the
underlying type:

```scala
opaque type UserId = UUID

object UserId:
  def apply(uuid: UUID): UserId = uuid
  extension (id: UserId) def value: UUID = id
  given Encoder[UserId] = Encoder[UUID].contramap(_.value)
  given Decoder[UserId] = Decoder[UUID].map(UserId.apply)
```

## Wiring into http4s

The simplest approach imports codecs for every type that has a circe instance:

```scala
import org.http4s.circe.CirceEntityCodec.given   // both directions

val routes = HttpRoutes.of[IO]:
  case req @ POST -> Root / "users" =>
    for
      create   <- req.as[CreateUser]      // uses Decoder[CreateUser]
      user     <- service.create(create)
      response <- Created(user)           // uses Encoder[User]
    yield response
```

`CirceEntityDecoder.given` and `CirceEntityEncoder.given` import one direction
each. For explicit control, build the entity codec per type:

```scala
import org.http4s.circe.{jsonOf, jsonEncoderOf}

given EntityDecoder[IO, CreateUser] = jsonOf[IO, CreateUser]
given EntityEncoder[IO, User]       = jsonEncoderOf[IO, User]
```

> **Note:** explicit `jsonOf` instances keep the set of types accepted as
> request bodies small and visible, which is a modest security benefit — an
> endpoint can only decode what you declared.

## Handling decode failures

`req.as[A]` raises `MalformedMessageBodyFailure` on invalid JSON. Unhandled,
http4s turns it into a 400 with a terse message. To control the response,
decode explicitly:

```scala
case req @ POST -> Root / "users" =>
  req.attemptAs[CreateUser].value.flatMap:
    case Right(create) =>
      service.create(create).flatMap(Created(_))
    case Left(failure) =>
      BadRequest(ErrorBody("invalid request body", failure.message))
```

For field-level errors, decode to `Json` first and use `Decoder.accumulating`,
which reports every failure rather than stopping at the first:

```scala
import io.circe.Json

case req @ POST -> Root / "users" =>
  req.as[Json].flatMap { json =>
    Decoder[CreateUser].decodeAccumulating(json.hcursor).fold(
      errors  => BadRequest(ErrorBody.fromDecodingFailures(errors)),
      create  => service.create(create).flatMap(Created(_)),
    )
  }
```

> **Warning:** never echo the raw parse failure back to an untrusted caller
> verbatim. circe's messages include the JSON path and can leak internal field
> names. Map them to a stable, curated error shape.

## Shaping the JSON

```scala
import io.circe.syntax.*

// Drop nulls from the output:
given Encoder[User] = deriveEncoder[User].mapJson(_.deepDropNullValues)

// Rename fields / add computed ones:
given Encoder[User] = Encoder.instance: user =>
  Json.obj(
    "id"           := user.id,
    "display_name" := user.name,
    "created_at"   := user.createdAt.toString,
  )

// snake_case everywhere, via configured derivation:
import io.circe.derivation.{Configuration, ConfiguredCodec}

given Configuration = Configuration.default.withSnakeCaseMemberNames

final case class UserProfile(userId: UUID, displayName: String)
  derives ConfiguredCodec       // serialises as user_id / display_name
```

> **Note:** on Scala 3 configured derivation lives in `io.circe.derivation`
> within `circe-generic`. The separate `circe-generic-extras` artifact and its
> `@ConfiguredJsonCodec` annotation are a Scala 2 story — do not reach for them
> in a Scala 3 project.

## Literal JSON in tests

`circe-literal` gives compile-time-checked JSON, which reads far better than
escaped strings:

```scala
import io.circe.literal.*

val expected = json"""
  { "id": $id, "name": "Ada", "email": "ada@example.com" }
"""

test("returns the user"):
  routes.orNotFound
    .run(Request[IO](GET, uri"/users" / id.toString))
    .flatMap(_.as[Json])
    .assertEquals(expected)
```

## jsoniter — when circe is not fast enough

For very high throughput, `jsoniter-scala` generates codecs at compile time and
is substantially faster, at the cost of macro-heavy derivation and less
flexible shaping:

```sbt
libraryDependencies ++= Seq(
  "com.github.plokhotnyuk.jsoniter-scala" %% "jsoniter-scala-core"   % "2.39.1",
  "com.github.plokhotnyuk.jsoniter-scala" %% "jsoniter-scala-macros" % "2.39.1" % Provided,
  "org.http4s" %% "http4s-jsoniter" % "0.23.36",
)
```

Start with circe. Move to jsoniter only when profiling shows JSON codec time is
actually the bottleneck.
