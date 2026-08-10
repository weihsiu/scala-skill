# New Project Setup

A `build.sbt` and module layout for a Cats Effect service on the Typelevel
stack.

## build.sbt

```sbt
ThisBuild / scalaVersion := "3.3.6"
ThisBuild / organization := "com.example"

val catsEffectVersion = "3.7.0"
val http4sVersion     = "0.23.36"
val circeVersion      = "0.14.16"
val fs2Version        = "3.13.0"
val doobieVersion     = "1.0.0-RC13"
val log4catsVersion   = "2.8.0"
val munitCEVersion    = "2.2.0"

lazy val root = (project in file("."))
  .settings(
    name := "my-service",
    libraryDependencies ++= Seq(
      "org.typelevel" %% "cats-effect"         % catsEffectVersion,
      "org.http4s"    %% "http4s-ember-server" % http4sVersion,
      "org.http4s"    %% "http4s-ember-client" % http4sVersion,
      "org.http4s"    %% "http4s-dsl"          % http4sVersion,
      "org.http4s"    %% "http4s-circe"        % http4sVersion,
      "io.circe"      %% "circe-core"          % circeVersion,
      "io.circe"      %% "circe-generic"       % circeVersion,
      "co.fs2"        %% "fs2-core"            % fs2Version,
      "org.typelevel" %% "doobie-core"         % doobieVersion,
      "org.typelevel" %% "doobie-hikari"       % doobieVersion,
      "org.typelevel" %% "doobie-postgres"     % doobieVersion,
      "org.typelevel" %% "log4cats-slf4j"      % log4catsVersion,
      "ch.qos.logback" % "logback-classic"     % "1.5.18" % Runtime,
      // Test
      "org.typelevel" %% "munit-cats-effect"   % munitCEVersion    % Test,
      "org.typelevel" %% "cats-effect-testkit" % catsEffectVersion % Test,
    ),
    scalacOptions ++= Seq(
      "-Wvalue-discard",     // catch discarded IO values
      "-Wunused:all",
      "-deprecation",
      "-feature",
      "-source:future",
    ),
  )
```

> **Important:** `-Wvalue-discard` is the Scala 3 equivalent of Scala 2's
> `-Wnonunit-statement`. Without it, an `IO` you forget to sequence is silently
> discarded and never runs — the single most common Cats Effect bug. Turn it
> on from day one.

> **Note:** doobie moved from groupId `org.tpolecat` to `org.typelevel` in
> `1.0.0-RC13`, and its packages moved from `doobie.*` to
> `org.typelevel.doobie.*`. Older tutorials still show the `org.tpolecat`
> coordinates.

For a scaffolded starting point:

```sh
sbt new typelevel/ce3.g8       # Cats Effect only
sbt new http4s/http4s.g8       # full http4s service
```

## `IO` or `F[_]`?

Two conventions, both idiomatic:

```scala
// Concrete IO — simpler, better inference, fine for applications:
class UserService(repo: UserRepo):
  def find(id: UserId): IO[Option[User]] = repo.find(id)

// Tagless final — abstract over the effect, for libraries and testability:
trait UserRepo[F[_]]:
  def find(id: UserId): F[Option[User]]

class UserService[F[_]: Monad](repo: UserRepo[F]):
  def find(id: UserId): F[Option[User]] = repo.find(id)
```

Guidance:

* **Application code** — use concrete `IO`. It is simpler to read, gives better
  type inference and error messages, and you can still test with fakes.
* **Library code you publish** — abstract over `F[_]`, constrained to the
  weakest typeclass that works (`Functor` < `Applicative` < `Monad` <
  `Concurrent` < `Temporal` < `Async`). The constraint documents exactly what
  the code needs.

> **Warning:** do not reflexively make everything tagless-final. `F[_]: Async`
> everywhere buys nothing over `IO` and costs readability on every signature.
> Abstract when a weaker constraint is genuinely meaningful.

## Module layout

```
src/main/scala/com/example/
├── Main.scala              ← IOApp, thin
├── config/
│   └── AppConfig.scala     ← case classes, loaded once
├── domain/                 ← pure model + business rules, no IO
│   ├── User.scala
│   └── UserError.scala
├── repo/
│   └── UserRepo.scala      ← doobie / persistence
├── service/
│   └── UserService.scala   ← orchestration
└── http/
    ├── UserRoutes.scala    ← HttpRoutes
    └── Server.scala        ← EmberServerBuilder
```

Keep `domain/` free of `IO` — pure functions and data are trivially testable
and stay reusable.

## The composition root

Build the entire application as one `Resource`, then run it. Everything
acquired is released in reverse order on shutdown or cancellation.

```scala
import cats.effect.{IO, IOApp, Resource}
import org.http4s.ember.client.EmberClientBuilder
import org.typelevel.log4cats.Logger
import org.typelevel.log4cats.slf4j.Slf4jLogger

object Main extends IOApp.Simple:

  private def app(using Logger[IO]): Resource[IO, Unit] =
    for
      config     <- Resource.eval(AppConfig.load)
      xa         <- Database.transactor(config.db)
      httpClient <- EmberClientBuilder.default[IO].build
      userRepo    = UserRepo(xa)
      userService = UserService(userRepo, httpClient)
      _          <- Server.resource(config.http, UserRoutes(userService))
    yield ()

  val run: IO[Unit] =
    Slf4jLogger.create[IO].flatMap { logger =>
      given Logger[IO] = logger
      app.useForever
    }
```

This is constructor injection — no DI framework needed. The `for`-comprehension
over `Resource` *is* the wiring, and the compiler checks it.

> **Important:** `useForever` (or `.use(_ => IO.never)`) keeps the resources
> alive for the process lifetime. `IOApp` cancels the fiber on SIGTERM, which
> unwinds the `Resource` stack and shuts everything down cleanly.

## Configuration

Load configuration once, into case classes, at startup. `ciris` composes
cleanly with Cats Effect:

```sbt
libraryDependencies += "is.cir" %% "ciris" % "3.10.0"
```

```scala
import cats.effect.IO
import cats.syntax.all.*
import ciris.*

final case class HttpConfig(host: Host, port: Port)
final case class DbConfig(url: String, user: String, password: Secret[String])
final case class AppConfig(http: HttpConfig, db: DbConfig)

object AppConfig:
  def load: IO[AppConfig] =
    (
      (
        env("HTTP_HOST").as[Host].default(host"0.0.0.0"),
        env("HTTP_PORT").as[Port].default(port"8080"),
      ).parMapN(HttpConfig.apply),
      (
        env("DB_URL").as[String],
        env("DB_USER").as[String],
        env("DB_PASSWORD").secret,
      ).parMapN(DbConfig.apply),
    ).parMapN(AppConfig.apply).load[IO]
```

`parMapN` accumulates *all* configuration errors, so a misconfigured deployment
reports every missing variable at once instead of one per restart.
