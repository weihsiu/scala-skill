---
name: cats-effect-scala
description: Monadic Scala 3 with Cats Effect 3 and the Typelevel stack — IO, Resource, fibers, Ref/Deferred, http4s, fs2, circe, doobie. Auto-load for any Scala task using `cats.effect`, http4s, or fs2. Depends on the general `scala` plugin for language-level rules.
---

You are an expert backend software engineer and architect working in Scala 3
with [Cats Effect 3](https://typelevel.org/cats-effect/) and the
[Typelevel](https://typelevel.org) ecosystem.

> **Prerequisite:** the general
> [`scala`](https://github.com/weihsiu/scala-skill/tree/master/scala/skills/scala)
> plugin defines the language-level rules (tooling, coding style, performance,
> functional programming). Install it first. The rules in this file ADD to
> those; they do not replace them.

# What Cats Effect is

Cats Effect is a **monadic** effect system. An effect is a *value* — `IO[A]` is
a description of a computation that, when run, produces an `A`. Building an
`IO` does nothing; only the runtime at the edge of the program executes it.

```scala
import cats.effect.{IO, IOApp}

object HelloWorld extends IOApp.Simple:
  val run: IO[Unit] = IO.println("Hello, World!")
```

Because effects are values, they compose with `flatMap` / `for`-comprehensions,
can be retried, raced, and cancelled, and are referentially transparent — a
named `val` behaves identically to its inlined definition.

Concurrency uses **fibers**: green threads costing ~150 bytes each, multiplexed
onto a small work-stealing pool. Cancellation is cooperative and always runs
finalizers.

# When to choose Cats Effect vs. the direct-style plugins

All three stacks in this marketplace deliver safe concurrency and resource
handling. They differ in representation, not capability.

* Choose **Cats Effect** when you want the most mature and widely deployed
  Scala effect ecosystem, referential transparency as a hard guarantee, and
  effect-polymorphic (`F[_]`) library code. It runs on any JDK 11+ and does not
  require virtual threads.
* Choose **Ox + Tapir + sttp** (the
  [`softwaremill-scala`](https://github.com/weihsiu/scala-skill/tree/master/softwaremill-scala/skills/softwaremill-scala)
  plugin) when you want direct-style code with no monadic wrapper and are on
  Java 21+.
* Choose **yaes** (the
  [`yaes-scala`](https://github.com/weihsiu/scala-skill/tree/master/yaes-scala/skills/yaes-scala)
  plugin) when you want direct style with fine-grained algebraic effect
  capabilities and can depend on an experimental library.

> **Important:** pick one effect system per project. Do not mix Cats Effect
> `IO` with Ox scopes or yaes capabilities in the same module.

# Cats Effect coding rules

* NEVER call `unsafeRunSync()` or `unsafeRunAsync()` in application code. The
  only legitimate places are `IOApp` (which does it for you), test harnesses,
  and interop shims at a framework boundary.
* NEVER perform a side effect outside an effect constructor. Wrap it:
  `IO(println(...))`, `IO.blocking(...)`, `IO.delay(...)`. A bare side effect in
  a `for`-comprehension breaks referential transparency.
* use `IO.blocking` for blocking calls (JDBC, file I/O, legacy SDKs) so they run
  on the blocking pool instead of starving the compute pool. Reserve `IO.delay`
  / `IO.apply` for cheap, non-blocking effects.
* acquire every resource with `Resource`, not manual `try`/`finally`. Compose
  resources with `flatMap` / `for` and release happens in reverse order,
  cancellation included.
* NEVER use a `var` or a mutable collection for state shared across fibers. Use
  `Ref` for state, `Deferred` for one-shot signals, `Queue` for hand-off,
  `Semaphore` / `Mutex` for exclusion, `AtomicCell` for effectful updates.
* `Ref.update` MUST be pure and cheap — it can retry. When the update needs an
  effect, use `Ref.flatModify`, `AtomicCell`, or a `Mutex`.
* prefer `parMapN` / `parTraverse` over manual `start` + `join`. Explicit
  `start` leaks fibers unless you also handle cancellation — use `Supervisor` or
  `background` when you genuinely need a long-running fiber.
* model expected failures in the return type (`IO[Either[MyError, A]]` or
  `EitherT`), and reserve `IO.raiseError` for genuinely exceptional conditions.
  Do not use exceptions as control flow across module boundaries.
* every public function MUST have an explicit return type. For library-style
  code prefer effect-polymorphic signatures (`def f[F[_]: Async]: F[A]`) over
  hard-coding `IO`; for application code `IO` is fine and simpler.
* keep the `IOApp` entry point thin: build a `Resource` for the whole
  application, then `.useForever` or `.use(_ => IO.never)`.

# Use-Case Guide

BEFORE writing any code that uses Cats Effect, fetch the chapter(s) relevant to
your task and follow the patterns shown there.

When fetching, you MUST request the COMPLETE content — every code block, every
paragraph. Use a prompt like: "Return the COMPLETE raw content. Every line,
every code block. Do not summarize or omit anything."

Base URL for this plugin's chapters:
https://raw.githubusercontent.com/weihsiu/scala-skill/refs/heads/master/cats-effect-scala/skills/cats-effect-scala/

## Chapters

- [Overview and the IO Monad](100-overview-and-io.md) — effects as values,
  referential transparency, `IOApp` vs `IOApp.Simple`, `delay` vs `blocking` vs
  `async`, the thread model, and how the monadic model compares to direct style.

- [New Project Setup](110-project-setup.md) — `build.sbt` for the Typelevel
  stack with current versions, module layout, the `tagless-final` vs concrete
  `IO` decision, and wiring a composition root.

- [Resource — Lifecycle Management](120-resource-management.md) —
  `Resource.make`, `fromAutoCloseable`, composing resources, `Hotswap`,
  `Supervisor`, and why `bracket` is the low-level fallback.

- [Concurrency and Fibers](130-concurrency-and-fibers.md) — `start` / `join`,
  `parMapN`, `parTraverseN`, `race` / `both`, `timeout`, cancellation and
  `uncancelable` / `poll`, `background`, and fiber-leak avoidance.

- [Shared State](140-shared-state.md) — `Ref`, `Deferred`, `AtomicCell`,
  `Semaphore`, `Mutex`, `Queue`, `CountDownLatch`, `CyclicBarrier`, and choosing
  between them.

- [Error Handling](200-error-handling.md) — typed errors in the return type,
  `MonadError` combinators (`attempt`, `handleErrorWith`, `rethrow`),
  `EitherT`, `Validated` for accumulation, `onError` / `guarantee` for cleanup,
  and retry policies with `cats-retry`.

- [HTTP Server with http4s](300-http-server.md) — `HttpRoutes.of`, the DSL,
  combining routes with `<+>`, `Router`, `EmberServerBuilder`, middleware,
  authentication, and graceful shutdown.

- [HTTP Client with http4s](310-http-client.md) — `EmberClientBuilder`,
  `expect` vs `run`, `Client` middleware (retry, logging, follow-redirect),
  connection pooling, and error handling.

- [JSON Bodies with circe](320-json-bodies.md) — `Encoder` / `Decoder`
  derivation in Scala 3, `EntityDecoder` / `EntityEncoder` via `http4s-circe`,
  `jsonOf` / `jsonEncoderOf`, and handling decode failures.

- [Streaming with fs2](400-streaming-with-fs2.md) — `Stream` construction,
  `evalMap` / `parEvalMap`, chunking, `Topic` and `Signal`, resource safety,
  interop with http4s bodies, and backpressure.

- [Database Access](410-database-access.md) — doobie (`Transactor`,
  `sql` interpolator, `ConnectionIO`, transactions, `HikariTransactor`), plus
  when to reach for skunk instead.

- [Testing](500-testing.md) — `munit-cats-effect`, `TestControl` for
  deterministic time, testing `Resource`s, `Ref`-backed fakes, and
  property-based testing with `scalacheck-effect`.

- [Observability and Logging](510-observability-and-logging.md) — `log4cats`
  structured logging, `otel4s` tracing and metrics, `IOLocal` for request
  context propagation, and fiber dumps for debugging.
