# Testing

Tests return `IO` values; the framework runs them. No `unsafeRunSync`, no
`Await.result`, no arbitrary sleeps.

## Dependencies

```sbt
libraryDependencies ++= Seq(
  "org.typelevel" %% "munit-cats-effect"   % "2.2.0" % Test,
  "org.typelevel" %% "cats-effect-testkit" % "3.7.0" % Test,
)
```

> **Note:** the artifact is `munit-cats-effect` (2.x). The older
> `munit-cats-effect-3` (1.x) name still appears in some documentation.

## Writing tests

```scala
import cats.effect.IO
import munit.CatsEffectSuite

class UserServiceSuite extends CatsEffectSuite:

  test("returns the user when it exists"):
    for
      repo <- UserRepo.inMemory
      _    <- repo.save(ada)
      got  <- repo.get(ada.id)
    yield assertEquals(got, Some(ada))

  // assertion helpers on IO directly:
  test("counts rows"):
    repo.count.assertEquals(0L)

  test("fails on a duplicate email"):
    service.register(duplicate).intercept[EmailTaken]
```

Useful helpers: `assertIO`, `assertIO_`, `assertIOBoolean`, `interceptIO`,
`intercept[E]`, and `.assertEquals` / `.assert` as extension methods on `IO`.

Combine multiple assertions with `*>` or a `for`-comprehension — a test is one
`IO`, so assertions must be *sequenced*, not merely constructed:

```scala
// WRONG — the first assertion is built and discarded
test("both"):
  service.a.assertEquals(1)
  service.b.assertEquals(2)      // only this one runs

// RIGHT
test("both"):
  service.a.assertEquals(1) *> service.b.assertEquals(2)
```

`-Wvalue-discard` catches exactly this mistake at compile time.

## Fixtures for resources

`ResourceSuiteLocalFixture` acquires once for the whole suite:

```scala
class RepoSuite extends CatsEffectSuite:

  private val transactor = ResourceSuiteLocalFixture(
    "transactor",
    Database.transactor(testConfig),
  )

  override def munitFixtures = List(transactor)

  test("inserts a user"):
    val xa = transactor()
    UserRepo.insert(ada).transact(xa).assertEquals(1)
```

Use `ResourceFunFixture` when each test needs its own fresh copy.

## Deterministic time with TestControl

`TestControl` replaces the runtime clock with a mock, so a test of a one-hour
retry policy finishes in microseconds and never flakes.

```scala
import cats.effect.testkit.TestControl
import scala.concurrent.duration.*

test("retries three times before succeeding"):
  var attempts = 0
  val action = IO:
    attempts += 1
    if attempts < 3 then throw TransientError else "ok"

  TestControl.executeEmbed(retry(action, 5, 1.minute)).assertEquals("ok")
```

`executeEmbed` runs the program to completion with time advanced automatically.
For assertions on intermediate states, use `execute`:

```scala
test("emits a heartbeat every 30 seconds"):
  TestControl.execute(heartbeatLoop(events)).flatMap { control =>
    for
      _ <- control.tick                       // run until the first sleep
      _ <- events.get.assertEquals(1)
      _ <- control.advanceAndTick(30.seconds)
      _ <- events.get.assertEquals(2)
      _ <- control.advanceAndTick(30.seconds)
      _ <- events.get.assertEquals(3)
    yield ()
  }
```

Key methods: `tick` (run until every fiber sleeps), `nextInterval`, `advance`,
`advanceAndTick`, `results`, `isDeadlocked`.

> **Warning:** advance time only *after* a `tick` has let the fibers reach
> their `IO.sleep`. Advancing first means the sleep is registered against the
> new time and the test hangs.

> **Important:** `TestControl` runs single-threaded, so it will not surface
> genuine race conditions. Use it for time-dependent logic; use real
> concurrency tests for race conditions.

## Test doubles

A `Ref`-backed implementation of the trait beats a mocking framework — it is
type-checked, has no reflection, and is inspectable:

```scala
object FakeEmailer:
  def create: IO[(Emailer, IO[List[Email]])] =
    Ref[IO].of(List.empty[Email]).map { sent =>
      val emailer = new Emailer:
        def send(email: Email): IO[Unit] = sent.update(_ :+ email)
      (emailer, sent.get)
    }

test("sends a welcome email on signup"):
  for
    (emailer, sentEmails) <- FakeEmailer.create
    service                = UserService(repo, emailer)
    _                     <- service.register(signup)
    sent                  <- sentEmails
  yield assertEquals(sent.map(_.to), List(signup.email))
```

To test failure paths, make the fake fail:

```scala
val failingRepo = new UserRepo:
  def save(u: User): IO[Unit] = IO.raiseError(DatabaseUnavailable)
```

## Testing http4s routes

Routes are functions, so no server is needed:

```scala
import org.http4s.*
import org.http4s.implicits.*
import org.http4s.circe.CirceEntityCodec.given

class UserRoutesSuite extends CatsEffectSuite:

  private def run(request: Request[IO]): IO[Response[IO]] =
    UserRoutes(stubService).orNotFound.run(request)

  test("GET /users/:id returns 200 and the user"):
    for
      response <- run(Request[IO](Method.GET, uri"/users" / ada.id.toString))
      _        <- IO(assertEquals(response.status, Status.Ok))
      body     <- response.as[User]
    yield assertEquals(body, ada)

  test("GET /users/:id returns 404 when missing"):
    run(Request[IO](Method.GET, uri"/users" / UUID.randomUUID().toString))
      .map(_.status)
      .assertEquals(Status.NotFound)
```

Test the client side by building a `Client` from routes — no sockets:

```scala
val client: Client[IO] = Client.fromHttpApp(stubApi.orNotFound)
```

## Property-based testing

`scalacheck-effect` adds `forAllF`, which works with effectful properties:

```sbt
libraryDependencies += "org.typelevel" %% "scalacheck-effect-munit" % "2.0.0-M2" % Test
```

```scala
import munit.{CatsEffectSuite, ScalaCheckEffectSuite}
import org.scalacheck.effect.PropF

class RoundTripSuite extends CatsEffectSuite with ScalaCheckEffectSuite:

  test("save then get round-trips"):
    PropF.forAllF { (user: User) =>
      for
        repo <- UserRepo.inMemory
        _    <- repo.save(user)
        got  <- repo.get(user.id)
      yield assertEquals(got, Some(user))
    }
```

## Integration tests with Testcontainers

```sbt
libraryDependencies += "com.dimafeng" %% "testcontainers-scala-postgresql" % "0.43.0" % Test
```

Wrap the container in a `Resource` so it is torn down with the rest of the
suite:

```scala
def postgres: Resource[IO, PostgreSQLContainer] =
  Resource.make(
    IO.blocking(PostgreSQLContainer().configure(_.start()))
  )(c => IO.blocking(c.stop()))
```

Combine it with `ResourceSuiteLocalFixture` so one container serves the whole
suite rather than starting per test.

## What not to do

* Do not call `unsafeRunSync()` in tests — it defeats `TestControl`, hides
  cancellation bugs, and blocks a thread.
* Do not use `IO.sleep` to "wait for" a concurrent effect. Use `Deferred` to
  signal, or `TestControl` to control time.
* Do not assert on wall-clock durations. Under CI load they flake; under
  `TestControl` they are exact.
