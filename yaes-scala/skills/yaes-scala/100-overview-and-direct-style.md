# yaes Overview and Direct Style

[yaes (λÆS)](https://github.com/yaes-io/yaes) is an experimental Scala 3
**algebraic effect system** that uses context parameters and context functions
to track effects in types, then handles them with explicit handler functions.
You write effectful code in plain direct style — no `Future`, no `IO`, no
`for`-comprehensions.

## Dependencies

```sbt
libraryDependencies += "io.yaes" %% "yaes-core" % "<latest>"
// Optional add-ons:
// "io.yaes" %% "yaes-data"  // Channel, Flow
// "io.yaes" %% "yaes-cats"  // Cats / Cats Effect interop, RaiseNel
// "io.yaes" %% "yaes-slf4j" // SLF4J backend for Log
// "io.yaes" %% "yaes-http-server" / "-client"   // HTTP stack (chapters 400, 410)
// "io.yaes" %% "yaes-http-circe" / "-jsoniter"  // JSON bodies (chapter 420)
// "io.yaes" %% "yaes-core-test-scalatest" % Test
```

> **Required:** yaes uses Java Virtual Threads and Structured Concurrency.
> Run on Java 24 or newer.

> **Note:** yaes moved from the `in.rcard.yaes` package and groupId to
> `io.yaes` in 0.21.0. Upgrading an existing codebase from 0.20.x or earlier
> does not need a hand edit — the `yaes-migration` module ships a Scalafix
> rule that rewrites imports, fully-qualified references, and comments.

## The mental model

An **effect** is a capability that must be present in scope to call certain
operations. Effects are expressed as `using` parameters (context parameters)
and propagate through context functions:

```scala
import io.yaes.*
import io.yaes.Raise.*

case class NotFound(what: String)

def loadUser(id: Long)(using Raise[NotFound]): User =
  if id <= 0 then Raise.raise(NotFound("user"))
  else User(id, "John")
```

The signature now says: "this function returns a `User` *and* requires the
ability to raise `NotFound`." A caller who has not handled `Raise[NotFound]`
cannot compile.

A **handler** introduces the capability and interprets it:

```scala
val outcome: Either[NotFound, User] = Raise.either {
  loadUser(42)
}
```

Inside `Raise.either { ... }`, `Raise[NotFound]` is in scope. Outside, it is
gone — the result is a plain `Either`.

## Infix syntax

For a single effect, prefer the infix alias:

```scala
def loadUser(id: Long): User raises NotFound = ...
def computeLog: Int writes String = ...
def readMaxRetries: Int reads Config = ...
```

These desugar to `(using Raise[NotFound])`, `(using Writer[String])`, etc.
They read better in API signatures.

## Stacking effects

Effects compose by stacking handlers:

```scala
import scala.concurrent.duration.*

val result: Either[NotFound, User] = Async.run {
  Raise.either {
    val fb = Async.fork { loadUser(42) }
    fb.value
  }
}
```

Order of handlers from outside in matches the order in the type: `Async`
outside, `Raise` inside. Each handler peels one effect off the capability set.

## Compared to monadic effect systems

| Concern | Monadic (Cats Effect, ZIO) | yaes |
| --- | --- | --- |
| Effect tracking | `IO[A]`, `ZIO[R, E, A]` wrappers | `using` clause; result type unchanged |
| Composition | `flatMap` / `for`-comprehensions | plain `;` and expressions |
| Error type | Inside the wrapper | `Raise[E]` capability |
| Interpretation | `unsafeRunSync` / `Runtime.run` | Handler function (`Raise.either { … }`) |

## Compared to Ox direct-style

Ox (the SoftwareMill direct-style library) also uses Scala 3 context parameters
and runs on virtual threads. The differences:

* **Effect granularity** — Ox has one main capability (`Ox`, the scope) plus
  helpers; yaes gives each effect its own capability (`Raise`, `Resource`,
  `Async`, `State`, …).
* **Error model** — Ox uses `Either` + `.ok()` short-circuit; yaes uses the
  explicit `Raise[E]` effect.
* **Resource model** — Ox ties resources to `Ox` scopes; yaes has a dedicated
  `Resource` effect with LIFO cleanup.
* **HTTP stack** — Ox is paired with the mature Tapir/sttp ecosystem; yaes ships
  experimental `yaes-http-*` modules.

> **Important:** pick one effect system per project. Do not mix `Ox` and yaes
> in the same module — they have overlapping but incompatible takes on
> structured concurrency.

## A complete tiny program

```scala
import io.yaes.*

object Greeting extends YaesApp:
  override def run: Unit =
    val who = if args.nonEmpty then args(0) else "world"
    Output.printLn(s"Hello, $who!")
    val n = Random.nextInt
    Output.printLn(s"Today's number: $n")
```

`YaesApp` is an entry point that installs `Sync`, `Output`, `Input`,
`Random`, `Clock`, and `System` for you. For finer control, compose handlers
yourself in `main`.
