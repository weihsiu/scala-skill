# Side-Effecting Services

yaes ships first-class effects for the runtime services every app touches:
`Sync`, `Output`, `Input`, `Random`, `Clock`, `System`, `Log`. Each is a
capability with a handler — production handlers do the obvious thing; test
handlers substitute deterministic implementations.

## Dependencies

```sbt
"in.rcard.yaes" %% "yaes-core"
// Optional: SLF4J backend for Log
// "in.rcard.yaes" %% "yaes-slf4j"
```

## `Sync` — tracking side effects

`Sync` is a capability that flags "this expression has a side effect."
Production code uses it to mark IO functions; the handler runs the work on a
virtual thread.

```scala
import in.rcard.yaes.Sync.*

def saveUser(user: User)(using Sync): Long =
  // any side-effecting body
  throw new RuntimeException("DB timeout")

val async: scala.concurrent.Future[Long] = Sync.run { saveUser(User("J")) }

import scala.concurrent.duration.*
val blocking: Long = Sync.runBlocking(2.seconds) { saveUser(User("J")) }
```

## `Output` and `Input` — console I/O

```scala
import in.rcard.yaes.Output.*
import in.rcard.yaes.Input.*
import in.rcard.yaes.Raise.*
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
import in.rcard.yaes.Random.*

def flipCoin(using Random): Boolean = Random.nextBoolean

val outcome: Boolean = Random.run { flipCoin }

// Available methods:
// Random.nextBoolean / nextInt / nextLong / nextDouble / nextUuid
```

`Random.nextUuid` returns a lowercase RFC 4122 v4 UUID as a `String` — handy
for IDs without pulling in `java.util.UUID`.

## `Clock`

```scala
import in.rcard.yaes.Clock.*

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
import in.rcard.yaes.System.*
import in.rcard.yaes.Raise.*

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
import in.rcard.yaes.Log
import in.rcard.yaes.Log.Level

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
"in.rcard.yaes" %% "yaes-slf4j"
```

```scala
import in.rcard.yaes.Log
import in.rcard.yaes.slf4j.Slf4jLog

Slf4jLog.run {
  val logger = Log.getLogger("OrderService")
  logger.info("placed order via SLF4J")
}
```

## `YaesApp` — one-line composition

For the typical "console app" shape, `YaesApp` installs `Sync`, `Output`,
`Input`, `Random`, `Clock`, and `System` automatically:

```scala
import in.rcard.yaes.*

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
import in.rcard.yaes.Async.*
import in.rcard.yaes.Raise.*
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
