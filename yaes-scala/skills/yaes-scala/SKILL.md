---
name: yaes-scala
description: Direct-style Scala 3 with the yaes (λÆS) effect system — algebraic effects via context parameters, capability-based handlers, structured concurrency on Java virtual threads. Auto-load for any Scala task using `in.rcard.yaes`. Depends on the general `scala` plugin for language-level rules.
---

You are an expert backend software engineer and architect working in direct-style
Scala with [yaes (λÆS)](https://github.com/rcardin/yaes), an algebraic effect
system.

> **Prerequisite:** the general
> [`scala`](https://github.com/weihsiu/scala-skill/tree/master/scala/skills/scala)
> plugin defines the language-level rules (tooling, coding style, performance,
> functional programming). Install it first. The rules in this file ADD to
> those; they do not replace them.

# What yaes is

yaes is a Scala 3 effect system that uses **context parameters** and **context
functions** to express effects as capabilities. Effectful code is written in
direct style — plain expressions, no monadic `for`-comprehensions — and effects
are **handled** at the boundary by handler functions (`Raise.either { ... }`,
`Async.run { ... }`, `Resource.run { ... }`, etc.).

```scala
// Effect-polymorphic signature: this function needs Raise[DbError] and Async.
def loadUser(id: UserId)(using Raise[DbError], Async): User =
  Async.fork(fetchFromDb(id)).value

// Handled at the boundary:
val result: Either[DbError, User] = Async.run {
  Raise.either {
    loadUser(UserId(1))
  }
}
```

Runtime requires **Java 24+** (Virtual Threads, Structured Concurrency).

# When to choose yaes vs. the SoftwareMill stack

Both yaes and Ox give you direct-style concurrency on virtual threads, so they
are alternatives — pick one per project, do not mix.

* Choose **yaes** when you want fine-grained, composable effect capabilities
  (`Raise`, `Resource`, `Async`, `State`, `Reader`, `Writer`, `Clock`, …) with
  explicit handlers, and you are willing to depend on an experimental library.
* Choose **Ox + Tapir + sttp** (the
  [`softwaremill-scala`](https://github.com/weihsiu/scala-skill/tree/master/softwaremill-scala/skills/softwaremill-scala)
  plugin) when you want a mature, production-tested HTTP stack with structured
  concurrency, scope-based resource management, and rich integrations.

# yaes coding rules

* effectful code MUST be written in direct style — plain expressions, plain
  control flow. Do NOT wrap results in `Future`, `IO`, or any monadic wrapper.
* every effect a function uses MUST appear in its `using` clause. Hiding an
  effect inside a function body (e.g. calling `Async.fork` without declaring
  `Async`) breaks the contract.
* use the **infix syntax** for clarity when there is one error type:
  `def f: A raises E = ...` is preferred over `def f(using Raise[E]): A`.
* effect handlers (`Raise.run`, `Async.run`, `Resource.run`, …) MUST be called
  at the application boundary — typically in `YaesApp.run` or a single
  composition root — not scattered through business logic.
* every public function MUST have an explicit return type AND an explicit
  `using` clause listing every effect it requires. Inferred capability lists
  are a major source of refactor drift.
* prefer the higher-level entry point `YaesApp` for application `main`s — it
  installs `Sync`, `Output`, `Input`, `Random`, `Clock`, and `System` for you.

# Use-Case Guide

BEFORE writing any code that uses yaes, fetch the chapter(s) relevant to your
task and follow the patterns shown there.

When fetching, you MUST request the COMPLETE content — every code block, every
paragraph. Use a prompt like: "Return the COMPLETE raw content. Every line,
every code block. Do not summarize or omit anything."

Base URL for this plugin's chapters:
https://raw.githubusercontent.com/weihsiu/scala-skill/refs/heads/master/yaes-scala/skills/yaes-scala/

## Chapters

- [Overview and Direct Style](100-overview-and-direct-style.md) — yaes mental
  model: effects as capabilities, context functions, handlers at the boundary,
  comparison to monadic effect systems and to Ox direct-style.

- [Raise — Error Handling](110-raise-error-handling.md) — `Raise[E]`,
  `Raise.raise`, `Raise.ensure`, `Raise.catching`, handlers
  (`either`/`option`/`nullable`/`run`), error mapping via `MapError`, error
  accumulation, error tracing.

- [Resource — Lifecycle Management](120-resource-effect.md) — `Resource.acquire`
  for `AutoCloseable`, `Resource.install` for custom acquire/release,
  `Resource.ensuring`, LIFO cleanup, `Resource.run` at the boundary.

- [Async — Structured Concurrency](130-async-structured-concurrency.md) —
  `Async.fork`, `Async.par`, `Async.race`, `Async.parTraverse`, fiber
  cancellation, `Async.delay`, graceful shutdown via `Shutdown` +
  `Async.withGracefulShutdown`.

- [Channels and Flow](140-channels-and-flow.md) — `Channel.bounded` /
  `unbounded` / `rendezvous`, `OverflowStrategy`, the `Channel.produce` DSL,
  `Flow` operators (`map`, `filter`, `take`, `buffer`), reactive-streams
  interop via `FlowPublisher`.

- [Side-Effecting Services](200-side-effecting-services.md) — `Sync`, `Output`,
  `Input`, `Random`, `Clock`, `System`, `Log` — the standard runtime effects
  and how to compose their handlers. SLF4J backend via `yaes-slf4j`.

- [Testing yaes Code](300-testing.md) — `yaes-core-test-scalatest` integration,
  capability substitution in tests, deterministic handlers for `Clock` and
  `Random`, asserting on `Raise` outcomes.

## Additional yaes modules

Beyond `yaes-core` and `yaes-data`, the following are covered briefly in the
chapters above and in the yaes README:

* **`yaes-cats`** — Cats / Cats Effect interop and `RaiseNel` polymorphic
  accumulation
* **`yaes-slf4j`** — SLF4J backend for the `Log` effect
* **`yaes-http-core`** / **`yaes-http-server`** / **`yaes-http-client`** /
  **`yaes-http-circe`** / **`yaes-http-jsoniter`** — HTTP stack on top of yaes
