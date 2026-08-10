# Resource — Lifecycle Management

`Resource[F, A]` pairs an acquire step with a release step. It guarantees the
release runs — on success, on error, and on cancellation.

## Constructing resources

```scala
import cats.effect.{IO, Resource}

// Explicit acquire / release:
def file(path: String): Resource[IO, BufferedReader] =
  Resource.make(
    IO.blocking(new BufferedReader(new FileReader(path)))   // acquire
  )(reader =>
    IO.blocking(reader.close()).handleErrorWith(_ => IO.unit)  // release
  )

// Anything AutoCloseable:
def file2(path: String): Resource[IO, BufferedReader] =
  Resource.fromAutoCloseable(IO.blocking(new BufferedReader(new FileReader(path))))

// An effect with no cleanup, lifted into the Resource monad:
val config: Resource[IO, AppConfig] = Resource.eval(AppConfig.load)

// A pure value:
val name: Resource[IO, String] = Resource.pure("service")
```

> **Important:** the release effect should not fail. If it can, handle the error
> inside the finalizer (as above) — an exception thrown from release replaces
> the original error and hides the real failure.

## Using a resource

```scala
val program: IO[String] =
  file("data.txt").use { reader =>
    IO.blocking(reader.readLine())
  }
// reader is closed here, whatever happened inside
```

`use` is the only place the resource is live. Do **not** let the `A` escape:

```scala
// WRONG — the reader is closed by the time you use it
val leaked: IO[BufferedReader] = file("data.txt").use(IO.pure)
```

Other consumers:

| Method | Meaning |
| --- | --- |
| `.use(f)` | run `f`, then release |
| `.useForever` | hold the resource until cancelled — for servers |
| `.use_` | acquire and immediately release (run for the side effect) |
| `.allocated` | escape hatch returning `(A, F[Unit])` — you must call the finalizer |
| `.surround(fa)` | run `fa` while the resource is open, ignoring the `A` |

> **Warning:** `.allocated` defeats the point of `Resource` — the compiler no
> longer enforces cleanup. Use it only at framework boundaries that demand an
> imperative lifecycle, and pair it with `Ref`/`Supervisor` bookkeeping.

## Composing resources

`Resource` is a monad, so resources compose with `for`. Release happens in
**reverse** order of acquisition:

```scala
import cats.effect.{IO, Resource}

def app: Resource[IO, Server] =
  for
    config <- Resource.eval(AppConfig.load)     // 1st acquired
    pool   <- connectionPool(config.db)         // 2nd
    client <- EmberClientBuilder.default[IO].build  // 3rd
    server <- serverResource(config.http, pool, client)  // 4th
  yield server
// released: server, client, pool  (config had no finalizer)
```

If `connectionPool` fails, `config` is released and the rest is never acquired.
This is the reason to build the whole application as one `Resource`.

Independent resources can be acquired in parallel with `parMapN` — but only
when neither depends on the other:

```scala
import cats.syntax.all.*

val both: Resource[IO, (Db, Cache)] = (dbResource, cacheResource).parTupled
```

## Resources and fibers

Starting a fiber inside a `Resource` requires care: a bare `.start` outlives
the resource scope. Use `background`, which is a `Resource` itself:

```scala
// Runs the fiber for the lifetime of the resource, cancels it on release:
val worker: Resource[IO, IO[Outcome[IO, Throwable, Unit]]] =
  pollingLoop.background
```

For a dynamic number of fibers, use `Supervisor`:

```scala
import cats.effect.std.Supervisor

Supervisor[IO].use { supervisor =>
  // Fibers started here are cancelled when the Supervisor closes.
  supervisor.supervise(backgroundJob).flatMap { fiber =>
    handleRequests(supervisor)
  }
}
```

`Supervisor[IO](await = true)` waits for supervised fibers to finish instead of
cancelling them — useful for fire-and-forget work you want to drain on
shutdown.

## Hotswap — replacing a resource without closing the scope

When you need to cycle through resources one at a time (log-file rotation, a
re-established connection) without nesting ever deeper:

```scala
import cats.effect.std.Hotswap

def rotatingLog(paths: Stream[IO, String]): IO[Unit] =
  Hotswap.create[IO, Writer].use { hotswap =>
    paths.evalMap { path =>
      hotswap.swap(fileWriter(path))   // old writer closed, new one opened
        .flatMap(writer => write(writer))
    }.compile.drain
  }
```

`swap` releases the previous resource only after the new one is acquired.

## bracket — the low-level form

`bracket` is `Resource` without the composability. Reach for it only when the
acquire/use/release triple is genuinely local:

```scala
IO.blocking(openConnection()).bracket { conn =>
  useConnection(conn)
} { conn =>
  IO.blocking(conn.close()).voidError
}

// bracketCase gives you the ExitCase — Succeeded, Errored, or Canceled:
acquire.bracketCase(use) {
  case (a, Outcome.Canceled())  => rollback(a)
  case (a, Outcome.Errored(e))  => rollback(a) *> log(e)
  case (a, Outcome.Succeeded(_)) => commit(a)
}
```

> **Important:** prefer `Resource` over `bracket` in almost all cases. Nested
> `bracket` calls produce the pyramid of doom that `Resource`'s monad instance
> exists to flatten.

## Cleanup without a resource

To run an effect on completion without managing a value:

```scala
// Always, regardless of outcome:
action.guarantee(cleanup)

// Branch on the outcome:
action.guaranteeCase {
  case Outcome.Succeeded(_) => IO.unit
  case Outcome.Errored(e)   => logError(e)
  case Outcome.Canceled()   => logCancellation
}

// Only on error (the error still propagates):
action.onError { case e => logError(e) }

// Only on cancellation:
action.onCancel(logCancellation)
```

`guarantee` and friends are cancellation-safe — the finalizer runs even if the
fiber is cancelled mid-`action`.
