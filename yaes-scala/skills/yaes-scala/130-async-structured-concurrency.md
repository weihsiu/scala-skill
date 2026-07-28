# Async — Structured Concurrency

The `Async` effect provides **fibers** running on Java 21+ Structured
Concurrency (Virtual Threads underneath). Forks are tied to their enclosing
`Async.run { … }` scope: when the scope exits, all child fibers are joined or
cancelled. No fiber outlives its parent.

## Dependencies

```sbt
"io.yaes" %% "yaes-core"
```

> **Required:** Java 24+. Earlier JDKs do not have the structured-concurrency
> APIs that yaes relies on.

## Forking a fiber

```scala
import io.yaes.Async.*
import io.yaes.Raise.*

def findUser(name: String): User = User(name)

val maybeUser: Either[Cancelled, User] = Raise.either {
  Async.run {
    val fb = Async.fork { findUser("John") }
    fb.value
  }
}
```

`Async.fork` returns a `Fiber[A]`. Retrieve the result with one of:

* `fb.value` — wait for the result; requires `Raise[Cancelled]` because the
  fiber may have been cancelled.
* `fb.join()` — wait for completion, return `Unit` (no `Raise` needed).

## Parallel operations

```scala
// Two unrelated computations in parallel:
val (a, b) = Async.par(compute1(), compute2())

// First to complete wins; the other is cancelled:
val winner = Async.race(slowPath(), fastPath())

// Same as race but returns the loser's fiber too:
val (result, loser) = Async.racePair(work1(), work2())

// Map a list in parallel, preserving order:
val profiles: Seq[UserProfile] = Async.run {
  Async.parTraverse(List(1, 2, 3, 4, 5))(fetchUserProfile)
}
```

## Cancellation

A fiber is cancelled by `fb.cancel()` or by the enclosing scope exiting.

```scala
import java.util.concurrent.ConcurrentLinkedQueue
import scala.concurrent.duration.*

val log = Async.run {
  val q = new ConcurrentLinkedQueue[String]()
  val cancellable = Async.fork {
    Async.delay(2.seconds)
    q.add("cancellable-done")  // never runs
  }
  val canceller = Async.fork {
    Async.delay(500.millis)
    cancellable.cancel()
    q.add("cancelled-it")
  }
  cancellable.join()
  q
}
```

Cancellation is **cooperative at suspension points** — `Async.delay`, channel
operations, and other yaes effects are cancellation-aware. Tight CPU loops
need to poll cooperation explicitly.

## Delaying

```scala
Async.delay(2.seconds)
Async.delay(500.millis)
```

`Async.delay` is a suspension point — it yields the carrier virtual thread.

## Graceful shutdown

For long-running servers, combine `Shutdown` (SIGTERM/SIGINT handling) with
`Async.withGracefulShutdown` and a deadline:

```scala
import io.yaes.*
import io.yaes.Async.{Deadline, ShutdownTimedOut}
import scala.concurrent.duration.*

val outcome: Either[ShutdownTimedOut, Unit] = Shutdown.run {
  Raise.either {
    Async.withGracefulShutdown(Deadline.after(30.seconds)) {
      val server = Async.fork("server") {
        while !Shutdown.isShuttingDown() do
          acceptAndHandleOne()
        cleanup()
      }
      server.join()
    }
  }
}
```

`Shutdown.isShuttingDown()` flips when SIGTERM arrives. The deadline bounds
how long graceful drain may take before `ShutdownTimedOut` is raised.

## Pitfalls

> **Warning:** never use `Async.fork` without an enclosing `Async.run`. The
> fork would have no parent scope to attach to and the compiler will reject
> it. This is the structural-concurrency guarantee.

> **Important:** `fb.value` requires `Raise[Cancelled]`. Decide at the call
> site how cancellation is reported — as a `Left(Cancelled)` from
> `Raise.either`, or by joining without retrieving the value.

> **Warning:** do not capture mutable state across fibers without a
> thread-safe container (`ConcurrentLinkedQueue`, `AtomicReference`, …) or a
> `Channel`. The `State` effect is intentionally **not** thread-safe; it is for
> single-threaded sequential computation.
