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

`Async.forkNamed("label") { … }` is the same fork with a name attached, which
surfaces in stack traces and thread dumps — worth it for long-lived fibers.

> **Warning:** `forkNamed` was called `fork(name)(block)` up to yaes 0.22.0.
> The rename landed in 0.23.0 and the one-argument `Async.fork` no longer
> accepts a name, so old call sites fail to compile.

## Parallel operations

```scala
// Two unrelated computations in parallel:
val (a, b) = Async.par(compute1(), compute2())

// First to SETTLE wins — success or failure; the other is cancelled:
val winner = Async.race(slowPath(), fastPath())

// First to SUCCEED wins; a fast failure does not beat a slow success:
val best = Async.raceSuccess(replicaA(), replicaB())

// Same as race but returns the loser's fiber too:
val (result, loser) = Async.racePair(work1(), work2())

// Map a list in parallel, preserving input order:
val profiles: Seq[UserProfile] = Async.run {
  Async.parTraverse(List(1, 2, 3, 4, 5))(fetchUserProfile)
}

// Same, but at most 4 invocations run at once:
val throttled: Seq[UserProfile] = Async.run {
  Async.parTraverseLimit(ids, concurrency = 4)(fetchUserProfile)
}
```

`race` and `raceSuccess` differ only in how they treat a losing failure.
`race` returns whatever settles first, so a branch that fails fast wins and
the slower success is cancelled unseen. `raceSuccess` ignores that failure and
keeps waiting on the surviving branch; it fails only if **both** branches do,
surfacing the last failure observed. Reach for `raceSuccess` when the branches
are interchangeable — hedged requests, replica reads — and for `race` when
losing genuinely means "stop".

Use `parTraverseLimit` whenever the work hits a bounded resource (a connection
pool, a rate-limited API). Plain `parTraverse` forks one fiber per element,
which for a large collection means every element in flight simultaneously.
Both are fail-fast: on the first failure the remaining fibers are cancelled.

`Async.never` is a computation that never completes on its own — useful as the
branch of a `race`, `raceSuccess`, or `timeout` that exists purely to be
cancelled or timed out:

```scala
// Serve until shutdown cancels us, with nothing to return on the happy path:
Async.race(Async.never, shutdownSignal())
```

> **Warning:** `Async.never` must always be forked and then cancelled, raced
> away, or wrapped in a timeout. Left to itself it blocks the enclosing
> `Async.run` forever.

## Unsupervised scopes

`Async.unsupervised { … }` is a standalone handler like `Async.run`, but its
scope does **not** fail fast: a fiber that throws and is never joined neither
propagates its exception nor cancels its siblings. To observe a failure you
must join the fiber explicitly with `join()` or `value`.

Supervision is a property of the scope, not the fork — there is no separate
"unsupervised fork" operation, and `Async.fork` / `Async.forkNamed` are used
unchanged inside the block. Nesting is safe: an inner `Async.unsupervised` (or
`Async.run`) saves the enclosing scope and restores it on exit, leaving the
outer scope untouched.

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
      val server = Async.forkNamed("server") {
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

## Escaping the scope: the `Unscoped` effect

Genuine fire-and-forget work — best-effort telemetry, a log flush — has to
outlive the scope that started it, which no `Async` operation can do. That is
the `Unscoped` effect, deliberately kept separate and awkward to reach:

```scala
import io.yaes.unsafe.allowUnscoped
import io.yaes.{Async, Unscoped}

allowUnscoped {
  Async.run {
    val strand = Unscoped.spawn {
      sendTelemetry()  // still running after Async.run and allowUnscoped return
    }
    strand.onComplete(_ => println("telemetry sent"))
  }
}
```

`Unscoped.spawn` starts the block on its own daemon virtual thread belonging
to no scope, and returns immediately. It differs from `Async.fork` in every
way that matters:

* **No cancellation, no join.** The returned `Strand[A]` exposes only
  `onComplete` / `onFailure` — there is no `join()`, no `value`, no `cancel()`.
* **Failures are contained.** An exception inside the block is captured for
  observers but never rethrown into the caller, so it can neither fail nor
  cancel the spawning scope.
* **It grants nothing.** The block gets no `Async` capability; work needing
  concurrency of its own opens its own `Async.run`.
* **It requires no `Async` at all** — only the `Unscoped` capability.

Because virtual threads are always daemons, a spawned computation left running
never keeps the JVM alive on its own.

> **Warning:** `allowUnscoped` is the one handler in yaes that does **not**
> bound what it grants — work spawned inside it keeps running after it
> returns. That is the entire point, and the reason there is no
> `Unscoped.run`. Importing it is meant to be a visible, greppable act: every
> grant site turns up under `grep -rn "allowUnscoped"`. Note that grepping for
> `io.yaes.unsafe` does **not** find them all, since a wildcard
> `import io.yaes.*` allows `unsafe.allowUnscoped { … }` at the call site.

Reach for `Unscoped.spawn` only when the work must genuinely outlive its
scope. Anything that should still respect structured concurrency belongs in
`Async.fork` inside `Async.run` or `Async.unsupervised`.

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
