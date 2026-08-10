# Error Handling

`IO` carries a `Throwable` error channel. That channel is for *unexpected*
failures. Expected, domain-level failures belong in the **return type**, where
the compiler forces the caller to deal with them.

## Two kinds of failure

```scala
// Expected: "this user might not exist" is a normal outcome.
def findUser(id: UserId): IO[Either[UserError, User]]

// Unexpected: the database is unreachable. Nothing local can fix it.
def query(sql: String): IO[ResultSet]   // may fail with SQLException
```

> **Important:** do not use `IO.raiseError` for control flow across module
> boundaries. A caller cannot see from `IO[User]` which errors are expected, so
> they either catch everything or nothing.

Define domain errors as a sealed hierarchy:

```scala
enum UserError:
  case NotFound(id: UserId)
  case EmailTaken(email: Email)
  case Invalid(reasons: NonEmptyList[String])
```

## The MonadError combinators

```scala
import cats.effect.IO
import cats.syntax.all.*

// Move the error into the value:
val attempted: IO[Either[Throwable, A]] = action.attempt

// ...and back out:
val rethrown: IO[A] = attempted.rethrow

// Only the errors you can classify:
val narrowed: IO[Either[TimeoutException, A]] = action.attemptNarrow[TimeoutException]

// Recover with a pure value:
val recovered: IO[A] = action.handleError(_ => fallbackValue)

// Recover with another effect:
val recoveredF: IO[A] = action.handleErrorWith(e => logError(e) *> fetchFallback)

// Recover only some errors — others propagate:
val partial: IO[A] = action.recoverWith:
  case _: TimeoutException => fetchFromCache

// Transform the error:
val remapped: IO[A] = action.adaptError:
  case e: SQLException => RepositoryError("query failed", e)

// Raise one:
val failed: IO[Nothing] = IO.raiseError(new IllegalStateException("bad"))

// Conditionally raise:
val checked: IO[Unit] = IO.raiseUnless(count > 0)(EmptyBatch)
val guarded: IO[Unit] = IO.raiseWhen(count > 100)(BatchTooLarge)
```

> **Warning:** `handleErrorWith` catches cancellation-related failures too if
> you are not careful. It does **not** catch `Outcome.Canceled` — cancellation
> is not an error — but a broad `case _ =>` will swallow genuine bugs like
> `NullPointerException`. Match on the error types you actually expect.

## Converting between the channels

```scala
// Either in the value -> IO error channel:
val io: IO[User] = IO.fromEither(eitherUser)          // Either[Throwable, User]
val io2: IO[User] = eitherUserError.leftMap(toThrowable).liftTo[IO]

// Option -> error:
val io3: IO[User] = IO.fromOption(maybeUser)(UserNotFound)

// Try / Future:
val io4: IO[A] = IO.fromTry(t)
val io5: IO[A] = IO.fromFuture(IO(makeFuture()))

// IO error channel -> Either in the value:
val e: IO[Either[Throwable, User]] = loadUser.attempt
```

## EitherT — composing typed errors

When a whole call chain shares one error type, `EitherT` removes the nested
pattern matching:

```scala
import cats.data.EitherT

def register(req: SignupRequest): IO[Either[UserError, User]] =
  (for
    email <- EitherT.fromEither[IO](Email.parse(req.email))
    _     <- EitherT(checkNotTaken(email))
    user  <- EitherT.liftF(createUser(email, req.name))
    _     <- EitherT.liftF(sendWelcomeEmail(user))
  yield user).value
```

Helpers: `EitherT.liftF` (an `IO[A]` that cannot fail), `EitherT.fromEither`,
`EitherT.fromOption`, `EitherT.cond`, `.leftMap`, `.leftSemiflatMap`.

> **Important:** keep `EitherT` inside the implementation and return a plain
> `IO[Either[E, A]]` from public methods. Monad transformers in a public API
> force every caller into the same stack.

## Accumulating errors with Validated

`Either` short-circuits on the first failure. Form validation should report all
of them:

```scala
import cats.data.{Validated, ValidatedNel}
import cats.syntax.all.*

def validate(req: SignupRequest): ValidatedNel[String, ValidUser] =
  (
    validateEmail(req.email),
    validateAge(req.age),
    validatePassword(req.password),
  ).mapN(ValidUser.apply)

// At the boundary, convert back:
val result: IO[Either[UserError, ValidUser]] =
  IO.pure(validate(req).toEither.leftMap(UserError.Invalid.apply))
```

`Validated` is applicative-only — it deliberately has no `Monad`, because
accumulation and sequencing are incompatible. Use `mapN` / `parMapN`, not
`flatMap`.

## Cleanup on failure

```scala
// Runs on error only; the error still propagates:
action.onError { case e => logger.error(e)("action failed") }

// Runs on every outcome:
action.guarantee(releaseSomething)

// Branch on the outcome:
action.guaranteeCase {
  case Outcome.Succeeded(_) => metrics.success
  case Outcome.Errored(e)   => metrics.failure *> logger.error(e)("failed")
  case Outcome.Canceled()   => metrics.cancelled
}
```

These are cancellation-safe. Plain `flatMap` after an `attempt` is not — if the
fiber is cancelled between the two, the cleanup never runs.

## Retries

Do not hand-roll retry loops. `cats-retry` composes policies:

```sbt
libraryDependencies += "com.github.cb372" %% "cats-retry" % "4.0.0"
```

```scala
import retry.*
import scala.concurrent.duration.*

val policy: RetryPolicy[IO, Any] =
  RetryPolicies.limitRetries[IO](5) join RetryPolicies.exponentialBackoff[IO](100.millis)

val resilient: IO[Response] =
  retryingOnErrors(
    policy = policy,
    errorHandler = ResultHandler.retryOnSomeErrors[IO, Throwable, Response] {
      case _: TimeoutException     => IO.pure(true)
      case _: ConnectionException  => IO.pure(true)
      case _                       => IO.pure(false)   // don't retry logic errors
    },
  )(callService)
```

> **Warning:** only retry *transient* failures. Retrying a `400 Bad Request` or
> a constraint violation multiplies load without any chance of success. Always
> combine a retry policy with a cap and a timeout.

A minimal hand-rolled version, when a dependency is not warranted:

```scala
def retry[A](io: IO[A], attempts: Int, delay: FiniteDuration): IO[A] =
  io.handleErrorWith { e =>
    if attempts <= 1 then IO.raiseError(e)
    else IO.sleep(delay) >> retry(io, attempts - 1, delay * 2)
  }
```

## Error handling at the HTTP boundary

Translate domain errors to responses in one place, so handlers stay clean:

```scala
import org.http4s.dsl.io.*

def toResponse(error: UserError): IO[Response[IO]] = error match
  case UserError.NotFound(id)     => NotFound(ErrorBody(s"no user $id"))
  case UserError.EmailTaken(e)    => Conflict(ErrorBody(s"email $e in use"))
  case UserError.Invalid(reasons) => BadRequest(ErrorBody(reasons.toList.mkString("; ")))

val routes = HttpRoutes.of[IO]:
  case GET -> Root / "users" / UUIDVar(id) =>
    service.find(UserId(id)).flatMap(_.fold(toResponse, Ok(_)))
```

Unexpected `Throwable`s should become a 500 with a logged stack trace and no
internal detail in the body — http4s's `ErrorHandling` and `ErrorAction`
middleware do this centrally.
