# Concurrency and Fibers

A **fiber** is a lightweight thread of execution — roughly 150 bytes,
multiplexed onto the work-stealing compute pool. Millions can exist at once.

## Prefer the high-level combinators

Most concurrency does not need explicit fibers.

```scala
import cats.effect.IO
import cats.syntax.all.*
import scala.concurrent.duration.*

// Two independent effects, in parallel, combined:
val user: IO[(Profile, Settings)] = (fetchProfile, fetchSettings).parTupled
val greeting: IO[String] = (fetchName, fetchTitle).parMapN((n, t) => s"$t $n")

// A collection, in parallel:
val all: IO[List[User]] = ids.parTraverse(fetchUser)

// Bounded parallelism — at most 8 at a time:
val bounded: IO[List[User]] = ids.parTraverseN(8)(fetchUser)

// Discard results:
val fired: IO[Unit] = events.parTraverseN(4)(publish).void
```

> **Important:** `parTraverse` on a large list starts *every* element at once.
> Against a database or a rate-limited API that is a self-inflicted outage.
> Default to `parTraverseN` with an explicit bound.

If any parallel branch fails, the others are cancelled and the first error
propagates.

## Racing and timeouts

```scala
// First to finish wins; the loser is cancelled:
val fastest: IO[Either[A, B]] = IO.race(fetchFromCache, fetchFromDb)

// Same, but you get the loser's fiber to join or cancel yourself:
val pair: IO[Either[(OutcomeIO[A], FiberIO[B]), (FiberIO[A], OutcomeIO[B])]] =
  IO.racePair(a, b)

// Both, in parallel, cancelling the other on failure:
val both: IO[(A, B)] = IO.both(a, b)

// Timeout — raises TimeoutException and cancels the action:
val quick: IO[Response] = callService.timeout(2.seconds)

// Timeout with a fallback instead of an error:
val orDefault: IO[Response] = callService.timeoutTo(2.seconds, IO.pure(cached))

// Only interrupt if the action is cancellable; otherwise let it finish:
val soft: IO[Response] = callService.timeoutAndForget(2.seconds)
```

> **Warning:** `timeout` cancels the underlying action. If it holds a resource
> or a database transaction, cancellation must be safe — verify the finalizers
> do the right thing rather than assuming.

## Explicit fibers

```scala
import cats.effect.{IO, FiberIO, Outcome}

val program: IO[Unit] =
  for
    fiber <- longRunning.start          // returns immediately
    _     <- doSomethingElse
    outcome <- fiber.join               // wait for it
    result <- outcome match
                case Outcome.Succeeded(fa) => fa
                case Outcome.Errored(e)    => IO.raiseError(e)
                case Outcome.Canceled()    => IO.raiseError(new CancellationException)
  yield ()
```

`joinWithNever` and `joinWith(fallback)` handle the cancellation case for you:

```scala
val result: IO[A] = fiber.joinWithNever   // canceled => never completes
```

> **Warning:** a `start`ed fiber is NOT tied to the scope that created it. If
> the parent fails before `join`, the child keeps running — a fiber leak. Use
> `background` (a `Resource` that cancels on release) or `Supervisor` instead
> of raw `start` whenever the fiber outlives a single expression.

```scala
// Preferred: the fiber is cancelled when the Resource scope closes.
pollingLoop.background.use { joinHandle =>
  serveRequests
}
```

## Cancellation

Cancellation is **cooperative**: a fiber can only be cancelled at a
cancellation boundary, which is every `flatMap` step by default. Finalizers
always run.

```scala
// Mark a region uninterruptible:
val safe: IO[Unit] = IO.uncancelable { poll =>
  for
    resource <- acquire                 // cannot be cancelled here
    _        <- poll(useResource(resource))  // cancellable inside poll
                  .guarantee(release(resource))
  yield ()
}
```

`poll` re-enables cancellation for the wrapped region. This is exactly how
`Resource` and `bracket` are implemented — acquire and release are masked, use
is pollable.

```scala
// Explicit checkpoint in a long uncancelable stretch:
IO.canceled            // self-cancel
IO.cede                // yield to the scheduler
```

> **Important:** never wrap an entire request handler in `uncancelable`. You
> lose timeouts, graceful shutdown, and racing. Mask only the critical section
> that must not be interrupted — typically resource acquisition or a
> commit/rollback pair.

## Interruptible blocking

`IO.blocking` is *not* cancellable — a blocked thread cannot be interrupted
mid-call. When the underlying API responds to `Thread.interrupt`:

```scala
IO.interruptible(socket.read())        // sends interrupt once
IO.interruptibleMany(socket.read())    // interrupts repeatedly until it returns
```

## Common patterns

**Fire-and-forget with supervision:**

```scala
Supervisor[IO].use { sup =>
  routes(sup)   // handlers call sup.supervise(auditLog(...)) 
}
```

**Parallel with per-item error isolation** — collect successes and failures
instead of failing the whole batch:

```scala
val results: IO[List[Either[Throwable, User]]] =
  ids.parTraverseN(8)(id => fetchUser(id).attempt)
```

**Periodic work:**

```scala
lazy val everyMinute: IO[Unit] =
  doWork.handleErrorWith(logError) >> IO.sleep(1.minute) >> everyMinute
```

Note `>>` (by-name), not `*>` — with `*>` the recursive reference is evaluated
during construction and never terminates.

**Waiting for a signal:**

```scala
for
  latch <- Deferred[IO, Unit]
  _     <- (waitForReady *> latch.complete(())).start
  _     <- latch.get           // blocks the fiber, not the thread
yield ()
```

## Parallelism vs concurrency

`parMapN` expresses *concurrency* — the effects are independent. Whether they
run in *parallel* depends on available cores. On Scala.js there is only ever
one thread, so `parMapN` still interleaves correctly but never runs
simultaneously. Write for concurrency; let the runtime decide parallelism.
