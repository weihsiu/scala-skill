# Raise — Error Handling in yaes

`Raise[E]` is the yaes effect for typed, recoverable errors. A function with
`Raise[E]` in scope may signal an error of type `E`. A handler converts the
effect into a plain `Either`, `Option`, nullable, or other shape.

## Dependencies

```sbt
"in.rcard.yaes" %% "yaes-core"
// Optional: cats-based polymorphic accumulation
// "in.rcard.yaes" %% "yaes-cats"
```

## Signalling errors

```scala
import in.rcard.yaes.Raise.*

case class DivisionByZero()

def divide(a: Int, b: Int): Int raises DivisionByZero =
  if b == 0 then Raise.raise(DivisionByZero())
  else a / b
```

`raises DivisionByZero` is infix for `(using Raise[DivisionByZero])`. Use infix
when there is a single error type — it makes signatures readable.

`Raise.ensure` collapses the common "check or raise" pattern:

```scala
def positive(n: Int): Int raises String =
  Raise.ensure(n > 0)("must be positive")
  n
```

## Catching exceptions

To bridge from an exception-throwing API into the `Raise` effect:

```scala
import java.io.IOException

def readFileSize(path: String): Long raises IOException =
  Raise.catching[IOException] {
    java.nio.file.Files.size(java.nio.file.Path.of(path))
  }
```

`Raise.catching[Ex]` runs the block and converts any thrown `Ex` into a raised
`Ex`. Other exceptions are not caught.

## Handlers

A `Raise[E]` is introduced by a handler that decides the result shape:

```scala
val asEither: Either[DivisionByZero, Int] = Raise.either { divide(10, 0) }
val asOption: Option[Int]                 = Raise.option { divide(10, 0) }
val asNullable: Int | Null                = Raise.nullable { divide(10, 0) }
val asUnion: DivisionByZero | Int         = Raise.run { divide(10, 0) }
```

Pick the shape that matches your downstream code:

* `Raise.either` — when the error needs to flow up several layers
* `Raise.option` — when the caller does not care *why* it failed
* `Raise.nullable` — for interop with code that uses `null` (yes, sometimes)
* `Raise.run` — when you want the raw union type, no boxing

> **Important:** every handler ends the `Raise[E]` capability. If you need to
> raise the same error type both inside and outside the handler, raise it
> *after* unwrapping the handler's result.

## Mapping one error into another

When a service-layer error needs to be reinterpreted as a domain error:

```scala
sealed trait DbError
case object ConnectionTimeout  extends DbError
case object RecordNotFound     extends DbError

sealed trait ServiceError
case class OperationFailed(msg: String) extends ServiceError
case class ValidationFailed(msg: String) extends ServiceError

given MapError[DbError, ServiceError] = MapError {
  case ConnectionTimeout => OperationFailed("Database unavailable")
  case RecordNotFound    => ValidationFailed("User not found")
}

def serviceLayer()(using Raise[ServiceError]): User =
  databaseLayer()  // databaseLayer raises DbError; MapError lifts it
```

When `MapError[A, B]` is in scope, calls inside that need `Raise[A]` are
satisfied by an ambient `Raise[B]`.

## Accumulating errors

By default, `Raise[E]` short-circuits on the first error. To collect multiple
errors before bailing out, wrap the section in `Raise.accumulate` and use the
`accumulating { … }` helper for each candidate value:

```scala
val validated = Raise.either {
  Raise.accumulate {
    val name = accumulating { validateName("") }
    val age  = accumulating { validateAge(-1) }
    (name, age)
  }
}
// Left(List("Name cannot be empty", "Age cannot be negative"))
```

> **Important:** every error needs to flow through `accumulating { … }` — a
> bare `Raise.raise` inside `accumulate` still short-circuits.

For accumulation into a non-empty list (with `yaes-cats`):

```scala
import in.rcard.yaes.RaiseNel  // alias for Raise[NonEmptyList[E]]
import cats.data.NonEmptyList

val result: Either[NonEmptyList[String], (Int, Int)] = Raise.either {
  Raise.accumulate[NonEmptyList, String, (Int, Int)] {
    val a = accumulating { validateA() }
    val b = accumulating { validateB() }
    (a, b)
  }
}
```

## Error tracing

To attach a stack trace when an error is raised (useful for diagnostics
without forcing exceptions into the type signature):

```scala
given TraceWith[String] = trace =>
  println(s"Error: ${trace.original}")
  trace.printStackTrace()

val result = Raise.either {
  traced {
    riskyOperation(-5)
  }
}
```

The `TraceWith` handler runs the moment an error is raised inside a `traced`
block — the original error type is preserved.

## Pitfalls

> **Warning:** `Raise[E]` is a capability, not an exception. Throwing an
> exception is **not** caught by `Raise.either`. Use `Raise.raise` (or
> `Raise.catching`) to signal recoverable errors.

> **Warning:** do not handle `Raise` deep inside business logic just to
> "clean up the signature." Handlers are interpretation boundaries — putting
> them in the wrong place erases the type-level error tracking you adopted yaes
> for.
