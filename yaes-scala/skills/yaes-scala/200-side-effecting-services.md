# Side-Effecting Services

yaes ships first-class effects for the runtime services every app touches:
`Sync`, `Output`, `Input`, `Random`, `Clock`, `System`, `Log`. Each is a
capability with a handler — production handlers do the obvious thing; test
handlers substitute deterministic implementations.

## Dependencies

```sbt
"io.yaes" %% "yaes-core"
// Optional: SLF4J backend for Log
// "io.yaes" %% "yaes-slf4j"
```

## `Sync` — tracking side effects

`Sync` is a capability that flags "this expression has a side effect."
Production code uses it to mark IO functions; the handler runs the work on a
virtual thread.

```scala
import io.yaes.Sync.*

def saveUser(user: User)(using Sync): Long =
  // any side-effecting body
  throw new RuntimeException("DB timeout")

val async: scala.concurrent.Future[Long] = Sync.run { saveUser(User("J")) }

import scala.concurrent.duration.*
val blocking: Long = Sync.runBlocking(2.seconds) { saveUser(User("J")) }
```

## `Output` and `Input` — console I/O

```scala
import io.yaes.Output.*
import io.yaes.Input.*
import io.yaes.Raise.*
import java.io.IOException

val program: Output ?=> Unit =
  Output.printLn("Hello, world!")
  Output.printErr("…and an error line")

val readResult: Either[IOException, String] = Raise.either {
  Input.run {
    Input.readLn()
  }
}

Output.run { program }
```

`Input.readLn()` raises `IOException` — wrap it with `Raise.either` (or another
handler) at the boundary.

## `Random`

```scala
import io.yaes.Random.*

def flipCoin(using Random): Boolean = Random.nextBoolean

val outcome: Boolean = Random.run { flipCoin }

// Available methods:
// Random.nextBoolean / nextInt / nextLong / nextDouble / nextUuid
```

`Random.nextUuid` returns a lowercase RFC 4122 v4 UUID as a `String` — handy
for IDs without pulling in `java.util.UUID`.

## `Clock`

```scala
import io.yaes.Clock.*

Output.run {
  Clock.run {
    val now      = Clock.now()           // java.time.Instant
    val mono     = Clock.nowMonotonic()  // Duration, monotonically increasing
    Output.printLn(s"wall: $now, monotonic: $mono")
  }
}
```

For testing or simulation, supply a fixed `java.time.Clock` via `given`:

```scala
given customClock: java.time.Clock =
  java.time.Clock.fixed(java.time.Instant.parse("2026-01-01T00:00:00Z"),
                        java.time.ZoneOffset.UTC)
```

## `System` — env and properties, parsed

`System` reads environment variables and JVM system properties, parsed into
the requested type.

```scala
import io.yaes.System.*
import io.yaes.Raise.*

val port: Option[Int]  = Raise.either { System.run { System.env[Int]("PORT") } }.toOption.flatten
val host: String       = System.run { System.env[String]("HOST", "localhost") }
val srvPort            = System.run { System.property[Int]("server.port") }
val srvHost            = System.run { System.property[String]("server.host", "localhost") }
```

Supported types: `String`, `Int`, `Long`, `Double`, `Boolean`, `Float`,
`Short`, `Byte`, `Char`. Parse failures raise `NumberFormatException` — handle
with `Raise.either`.

## `Log` — structured logging

```scala
import io.yaes.Log
import io.yaes.Log.Level

val program = Log.run(Level.Info) {
  val logger = Log.getLogger("OrderService")
  logger.debug("hidden at Info level")
  logger.info("placed order 42")
  logger.warn("queue at 80% capacity")
}
```

Levels: `TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR`, `FATAL`. The default handler
writes `2026-04-22T19:55:59 - INFO - OrderService - placed order 42` to stdout.

### SLF4J backend

For Logback / Log4j2 in production, swap the handler:

```sbt
"io.yaes" %% "yaes-slf4j"
```

```scala
import io.yaes.Log
import io.yaes.slf4j.Slf4jLog

Slf4jLog.run {
  val logger = Log.getLogger("OrderService")
  logger.info("placed order via SLF4J")
}
```

## `YaesApp` — one-line composition

For the typical "console app" shape, `YaesApp` installs `Sync`, `Output`,
`Input`, `Random`, `Clock`, and `System` automatically:

```scala
import io.yaes.*

object MyApp extends YaesApp:
  override def run: Unit =
    Output.printLn(s"args: ${args.mkString(", ")}")
    Output.printLn(s"now: ${Clock.now}")
    Output.printLn(s"random: ${Random.nextInt}")
```

For finer-grained needs (e.g. choosing `Slf4jLog` instead of default `Log`),
compose handlers yourself in `def main(args: Array[String])`.

## Retry handler

`Retry` composes with `Async` and `Raise` to retry failing operations on a
schedule:

```scala
import io.yaes.Async.*
import io.yaes.Raise.*
import scala.concurrent.duration.*

case class DbError(msg: String)
def findUser(id: Int): String raises DbError =
  Raise.raise(DbError("timeout"))

val result: Either[DbError, String] = Async.run {
  Raise.either {
    Retry[DbError](Schedule.fixed(500.millis).attempts(3)) {
      findUser(42)
    }
  }
}
```

Schedule policies:

```scala
Schedule.fixed(500.millis)
Schedule.exponential(100.millis, factor = 2.0, max = 30.seconds)
Schedule.exponential(100.millis).jitter(0.25)   // requires Random
Schedule.exponential(100.millis).jitter(0.25).attempts(5)
```

### Retrying only transient errors

A second `retryable` parameter decides which errors are worth another attempt.
Errors for which it returns `false` are re-raised immediately, without
consuming an attempt:

```scala
sealed trait AppError
case class ConnectionError(host: String) extends AppError
case class AuthError(msg: String)        extends AppError

val result: Either[AppError, String] = Async.run {
  Raise.either {
    Retry[AppError](
      Schedule.exponential(100.millis).attempts(5),
      retryable = {
        case _: ConnectionError => true   // transient — retry
        case _: AuthError       => false  // permanent — re-raise now
      }
    ) {
      connect()
    }
  }
}
```

The default is `retryable = _ => true`, so existing call sites keep retrying
every error.

> **Important:** when the protected block can raise more than one error type,
> type `Retry` at the **full union** and discriminate with `retryable` — do not
> narrow `E` to the subset you want retried. `Raise[-E]` is contravariant, so a
> block that captures an outer `Raise[E | F]` resolves `F` against that outer
> handler and slips past `Retry`'s internal boundary entirely: those failures
> are never counted, never retried, and surface directly at the outer handler.
> Widening `E` keeps every error flowing through one boundary.

## CircuitBreaker handler

`CircuitBreaker` guards a downstream call that is failing repeatedly, so a dead
dependency fails fast instead of absorbing the timeout of every caller. It is
**not an effect** — it is a stateful orchestrator; the protected block just
runs, succeeds, or fails, and never references `CircuitBreaker` itself.

| State | Behaviour |
| --- | --- |
| **Closed** | Block executes. Consecutive counted failures increment a counter. |
| **Open** | Calls fast-fail via `Raise[CircuitBreaker.Open]`; the block never runs. |
| **Half-Open** | One probe is allowed once `resetTimeout` has elapsed. Success → Closed; failure → Open, timer reset. |

The Open → Half-Open transition is **lazy**: it happens on the next incoming
call after `resetTimeout` elapses, not at a fixed wall-clock moment. So a
circuit with no traffic stays Open indefinitely.

```scala
import io.yaes.*
import scala.concurrent.duration.*

case class DbError(msg: String)
def findUser(id: Int): String raises DbError =
  Raise.raise(DbError("connection timeout"))

given CircuitBreaker[DbError] =
  CircuitBreaker.make(CircuitBreaker.Config.consecutive(3, 5.seconds))

val result: Either[CircuitBreaker.Open, Either[DbError, String]] = Clock.run {
  Raise.either[CircuitBreaker.Open, Either[DbError, String]] {
    Raise.either[DbError, String] {
      CircuitBreaker.protect[DbError] {
        findUser(42)
      }
    }
  }
}
// After 3 consecutive failures the circuit opens; later calls return
// Left(CircuitBreaker.Open(resetAt)) without touching the database.
```

`protect` needs a `Clock` in scope for the timeout, and two error channels: the
domain `Raise[E]` and a `Raise[CircuitBreaker.Open]`. `Open` carries `resetAt`,
the wall-clock instant at which a probe may be attempted — useful for a
`Retry-After` header or a log line.

`Config.consecutive(threshold, resetTimeout)` clamps nonsense input rather than
raising: a threshold `≤ 0` becomes `1`, and a non-positive timeout becomes
`Duration.Zero`.

### Counting only some failures

`failingWhen` restricts which errors trip the circuit. Non-matching errors are
still re-raised via `Raise[E]`, but leave the counter alone — and during a
Half-Open probe they close the circuit:

```scala
sealed trait AppError
case class ConnectionError(host: String) extends AppError
case class AuthError(msg: String)        extends AppError

given CircuitBreaker[AppError] = CircuitBreaker.make(
  CircuitBreaker.Config.consecutive[AppError](3, 5.seconds)
    .failingWhen(_.isInstanceOf[ConnectionError])
)
```

> **Important:** as with `Retry`, type the circuit breaker at the **full union**
> of error types and discriminate with `failingWhen`. Narrowing `E` lets the
> contravariance of `Raise[-E]` route the other errors to an outer handler,
> bypassing the breaker's boundary so those failures never count toward the
> threshold.

## Pitfalls

> **Important:** `Sync.runBlocking` blocks the calling carrier thread. Avoid
> nesting it inside `Async.fork` — let `Async` propagate instead, and unwrap
> with `Async.run` at the boundary.

> **Warning:** `System.env[Int]` raises on parse failure. Decide whether a
> missing `PORT` is "use default" (`env[Int]("PORT", 8080)`) or "fail
> startup" (`Raise.either` and exit).

> **Important:** when running on Java 24+, `Random` and `Clock` handlers
> already exist by default. You only need to override them in tests, not in
> production composition.
