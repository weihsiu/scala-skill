# Cats Effect Overview and the IO Monad

[Cats Effect](https://typelevel.org/cats-effect/) is a purely functional
asynchronous runtime for Scala. Effects are **values**: `IO[A]` describes a
computation that will produce an `A` when the runtime executes it.

## Dependencies

```sbt
libraryDependencies += "org.typelevel" %% "cats-effect" % "3.7.1"
```

`cats-effect` transitively pulls in `cats-core` and `cats-effect-kernel` (the
typeclasses) and `cats-effect-std` (`Queue`, `Random`, `Console`, …).

> **Note:** on Scala 2 add `-Wnonunit-statement` and the `better-monadic-for`
> compiler plugin. On Scala 3 neither is needed — use `-Wvalue-discard`
> instead to catch discarded effects.

## Effects are descriptions, not actions

This is the single most important property. Constructing an `IO` runs nothing:

```scala
import cats.effect.IO

val printed: IO[Unit] = IO.println("hi")   // nothing printed yet
val twice: IO[Unit] = printed >> printed   // still nothing printed
```

Because `printed` is a value, substituting it for its definition changes
nothing — that is **referential transparency**. Compare with a side-effecting
`Future`, which starts running the moment it is constructed.

The practical payoff: an `IO` can be retried, timed out, raced, cancelled, or
run N times in parallel, all after the fact and without changing its
definition.

> **Warning:** never write a bare side effect inside a `for`-comprehension. The
> statement runs at *construction* time, once, in the wrong order.
>
> ```scala
> // WRONG — println happens while building the IO
> for
>   x <- fetch
>   _ = println(x)     // runs during construction
> yield x
>
> // RIGHT
> for
>   x <- fetch
>   _ <- IO.println(x)
> yield x
> ```

## Choosing the right constructor

| Constructor | Use for | Runs on |
| --- | --- | --- |
| `IO.pure(a)` | an already-computed value | — (no effect) |
| `IO(...)` / `IO.delay(...)` | cheap, non-blocking side effects | compute pool |
| `IO.blocking(...)` | blocking calls: JDBC, file I/O, legacy SDKs | blocking pool |
| `IO.interruptible(...)` | blocking calls that respond to `Thread.interrupt` | blocking pool |
| `IO.async_` / `IO.async` | callback-based APIs | caller's fiber |
| `IO.fromFuture(IO(f))` | an existing `Future` | compute pool |

```scala
import cats.effect.IO
import java.sql.{Connection, ResultSet}

// Cheap and non-blocking:
def now: IO[Long] = IO(System.currentTimeMillis())

// Blocking — MUST be IO.blocking, not IO():
def query(c: Connection, q: String): IO[ResultSet] =
  IO.blocking(c.createStatement().executeQuery(q))
```

> **Important:** putting a blocking call in `IO(...)` parks a thread from the
> small work-stealing compute pool. Enough of them and the whole application
> stalls — Cats Effect will log a *starvation* warning when it detects this.
> The fix is always `IO.blocking`.

`IO.pure` is strict: `IO.pure(println("x"))` prints immediately. Use `IO.pure`
only for values already in hand.

## Wrapping callback-based APIs

```scala
import cats.effect.IO

def fetchAsync(url: String, cb: Either[Throwable, String] => Unit): Unit = ???

// async_ — for callbacks with no cancellation support:
def fetch(url: String): IO[String] =
  IO.async_[String](cb => fetchAsync(url, cb))

// async — when the underlying API can be cancelled; return a finalizer:
def fetchCancelable(url: String): IO[String] =
  IO.async[String] { cb =>
    IO {
      val handle = startRequest(url, cb)
      Some(IO(handle.cancel()))   // Some(finalizer) = cancellable
    }
  }
```

## Entry points

`IOApp` runs the program and installs the runtime, thread pools, and signal
handlers.

```scala
import cats.effect.{ExitCode, IO, IOApp}

// No arguments, no exit code:
object Simple extends IOApp.Simple:
  val run: IO[Unit] = IO.println("hello")

// Full form — arguments and an explicit exit code:
object Main extends IOApp:
  def run(args: List[String]): IO[ExitCode] =
    IO.println(s"args: $args").as(ExitCode.Success)
```

`IOApp` installs a SIGTERM/SIGINT handler that **cancels** the main fiber, so
every `Resource` finalizer and `guarantee` runs on shutdown. That is what makes
graceful shutdown work — see the HTTP server chapter.

> **Warning:** do NOT write your own `def main` that calls `unsafeRunSync()`.
> You lose graceful cancellation, the tuned runtime, and starvation checking.

## Composition

`IO` is a monad, so ordinary Cats syntax applies:

```scala
import cats.effect.IO
import cats.syntax.all.*

val program: IO[Int] =
  for
    a <- IO(1)
    b <- IO(2)
  yield a + b

// Sequencing when you discard the left result:
val seq: IO[Int] = IO.println("starting") *> IO.pure(42)
val seqLazy: IO[Int] = IO.println("starting") >> IO.pure(42)

// Independent effects in parallel:
val par: IO[Int] = (IO(1), IO(2)).parMapN(_ + _)
```

> **Important:** `*>` evaluates its right-hand side eagerly when building the
> `IO`; `>>` takes it by name. In a recursive definition you MUST use `>>`, or
> the construction loops forever:
>
> ```scala
> lazy val loop: IO[Unit] = IO.println("tick") >> IO.sleep(1.second) >> loop
> ```

## The thread model

Cats Effect runs three pools:

* **compute** — a work-stealing pool sized to the number of CPUs. All `IO`
  runs here by default. Never block it.
* **blocking** — an unbounded cached pool for `IO.blocking`. On JDK 21+ Cats
  Effect 3.7 can back this with virtual threads.
* **scheduler** — a single thread driving `IO.sleep` and timeouts.

Fibers yield automatically at `flatMap` boundaries (auto-cede), so a long chain
of effects never monopolises a carrier thread. A tight *synchronous* loop
inside one `IO(...)` still can — break it up or move it to `IO.blocking`.

## A worked example

Counter incremented once a second, with three independent observers, all
cancelled together when the app exits:

```scala
import cats.effect.{IO, IOApp}
import scala.concurrent.duration.*

object FizzBuzz extends IOApp.Simple:
  val run: IO[Unit] =
    for
      ctr  <- IO.ref(0)
      wait  = IO.sleep(1.second)
      poll  = wait *> ctr.get
      _    <- poll.flatMap(IO.println(_)).foreverM.start
      _    <- poll.map(_ % 3 == 0).ifM(IO.println("fizz"), IO.unit).foreverM.start
      _    <- poll.map(_ % 5 == 0).ifM(IO.println("buzz"), IO.unit).foreverM.start
      _    <- (wait *> ctr.update(_ + 1)).foreverM.void
    yield ()
```

## Compared to direct-style Scala

| Concern | Cats Effect | Ox / yaes (direct style) |
| --- | --- | --- |
| Effect tracking | `IO[A]` wrapper in the return type | `using` capability; return type unchanged |
| Composition | `flatMap` / `for`-comprehensions | plain `;` and expressions |
| Errors | `IO.raiseError` + typed error in the value | `Either` + `.ok()`, or `Raise[E]` |
| Concurrency unit | fiber on a work-stealing pool | virtual thread in a scope |
| Interpretation | `IOApp` at the edge | handler at the boundary |
| JDK floor | 11 | 21 (Ox) / 24 (yaes) |
| Ecosystem | largest in Scala FP | growing |

The trade-off: Cats Effect asks you to accept the monadic wrapper and
`for`-comprehension syntax; in exchange you get referential transparency,
effect-polymorphic libraries, and by far the most mature ecosystem. Stack
traces are reconstructed from fiber tracing rather than the JVM stack, which
reads differently from direct-style traces but is usually more informative
across async boundaries.
