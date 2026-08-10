# Shared State Across Fibers

Never use a `var` or a mutable collection for state touched by more than one
fiber. Cats Effect's `std` module provides concurrency-safe primitives.

## Choosing a primitive

| Need | Use |
| --- | --- |
| Mutable value, pure updates | `Ref` |
| Mutable value, effectful updates | `AtomicCell` |
| One-shot "the value is ready" signal | `Deferred` |
| Producer/consumer hand-off with backpressure | `Queue` |
| Limit N concurrent users of something | `Semaphore` |
| Mutual exclusion (N = 1) | `Mutex` |
| Wait for N events | `CountDownLatch` |
| Rendezvous of N fibers, reusable | `CyclicBarrier` |
| Observable current value | `Signal` (fs2) |
| Fan-out to many subscribers | `Topic` (fs2) |

## Ref — concurrent mutable state

```scala
import cats.effect.{IO, Ref}

for
  counter <- Ref[IO].of(0)             // or IO.ref(0)
  _       <- counter.update(_ + 1)      // atomic
  _       <- counter.set(10)
  current <- counter.get
  prev    <- counter.getAndSet(0)
  out     <- counter.modify(n => (n + 1, s"was $n"))  // update + return
yield out
```

`update` and `modify` use compare-and-swap: the function may be **retried** if
another fiber wins the race.

> **Warning:** the function passed to `update` / `modify` MUST be pure and
> cheap. It can run several times. Never put logging, I/O, or an expensive
> computation inside it.

When the update genuinely needs an effect:

```scala
// flatModify — the effect runs once, after the state settles:
counter.flatModify(n => (n + 1, IO.println(s"now ${n + 1}")))

// Or use AtomicCell, which serialises effectful updates:
for
  cell <- AtomicCell[IO].of(initialState)
  _    <- cell.evalUpdate(state => refreshFromDb(state))
yield ()
```

`AtomicCell` is a `Ref` whose updates may be effectful; it takes a lock, so it
is slower than `Ref` and can deadlock if the update touches the same cell.

> **Important:** `Ref` protects a *single* value. Two `Ref`s updated in
> sequence are not atomic together. If two pieces of state must change as a
> unit, put them in one `Ref` holding a case class.

## Deferred — a one-shot promise

```scala
import cats.effect.Deferred

for
  ready <- Deferred[IO, Config]
  _     <- (loadConfig.flatMap(ready.complete)).start
  cfg   <- ready.get           // semantically blocks until completed
yield cfg
```

* `complete` returns `IO[Boolean]` — `false` if it was already completed.
* `get` blocks the *fiber*, not a thread, and is cancellable.
* `tryGet` returns `IO[Option[A]]` without blocking.

`Deferred` is write-once. For a value that changes over time and that fibers
want to observe, use fs2's `SignallingRef`.

## Queue — producer/consumer

```scala
import cats.effect.std.Queue

for
  q <- Queue.bounded[IO, Event](1024)
  _ <- producer(q).start
  _ <- consumer(q).foreverM
yield ()

def producer(q: Queue[IO, Event]): IO[Unit] =
  events.traverse_(q.offer)     // semantically blocks when full — backpressure

def consumer(q: Queue[IO, Event]): IO[Unit] =
  q.take.flatMap(handle)        // semantically blocks when empty
```

Variants:

| Constructor | Behaviour when full |
| --- | --- |
| `Queue.bounded(n)` | `offer` blocks (backpressure) |
| `Queue.unbounded` | never full — risks unbounded memory |
| `Queue.dropping(n)` | new items discarded |
| `Queue.circularBuffer(n)` | oldest item discarded |
| `Queue.synchronous` | `offer` waits for a `take` (rendezvous) |

Non-blocking variants: `tryOffer` / `tryTake` return `Boolean` / `Option`.

> **Important:** prefer `bounded` over `unbounded`. An unbounded queue turns a
> slow consumer into an out-of-memory error; a bounded one turns it into
> backpressure that propagates to the producer.

## Semaphore and Mutex

```scala
import cats.effect.std.{Mutex, Semaphore}

// At most 10 concurrent calls to a fragile downstream service:
for
  sem <- Semaphore[IO](10)
  _   <- requests.parTraverse(r => sem.permit.use(_ => call(r)))
yield ()

// Exclusive access:
for
  mutex <- Mutex[IO]
  _     <- mutex.lock.surround(criticalSection)
yield ()
```

`permit` is a `Resource`, so the permit is returned even on error or
cancellation. Prefer it over manual `acquire` / `release`.

For simple bounded parallelism over a collection, `parTraverseN(n)` is clearer
than a `Semaphore`.

## Latches and barriers

```scala
import cats.effect.std.{CountDownLatch, CyclicBarrier}

// Wait until three subsystems report ready:
for
  latch <- CountDownLatch[IO](3)
  _     <- List(dbReady, cacheReady, queueReady).parTraverse(_ *> latch.release)
  _     <- latch.await
  _     <- startServing
yield ()

// Reusable rendezvous — all N fibers wait for each other each round:
for
  barrier <- CyclicBarrier[IO](4)
  _       <- workers.parTraverse(w => phaseOne(w) *> barrier.await *> phaseTwo(w))
yield ()
```

## A worked example: an in-memory repository

```scala
import cats.effect.{IO, Ref}

trait UserRepo:
  def get(id: UserId): IO[Option[User]]
  def save(user: User): IO[Unit]
  def delete(id: UserId): IO[Boolean]

object UserRepo:
  def inMemory: IO[UserRepo] =
    Ref[IO].of(Map.empty[UserId, User]).map { ref =>
      new UserRepo:
        def get(id: UserId): IO[Option[User]] =
          ref.get.map(_.get(id))

        def save(user: User): IO[Unit] =
          ref.update(_ + (user.id -> user))

        def delete(id: UserId): IO[Boolean] =
          ref.modify(m => (m - id, m.contains(id)))
    }
```

This is also the standard way to build a test double — see the testing chapter.

## IOLocal — fiber-local state

For values that should follow a logical operation across fibers (a request id,
a trace context) without threading a parameter through every signature:

```scala
import cats.effect.IOLocal

for
  local <- IOLocal("no-request-id")
  _     <- local.set("req-123") *> handler(local)
yield ()
```

`IOLocal` is inherited by fibers started from the current one, and reverts on
`scope`. It is the mechanism behind `otel4s` trace propagation.

> **Warning:** `IOLocal` is not a general-purpose dependency injection
> mechanism. Use constructor parameters for dependencies; reserve `IOLocal` for
> genuinely ambient context like tracing and request ids.
