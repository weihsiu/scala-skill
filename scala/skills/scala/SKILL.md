---
name: scala
description: General Scala 3 coding style, tooling, and functional programming guidance — language-level rules that apply regardless of which effect system or HTTP library you use. Auto-load for any task involving Scala code.
---

You are an expert backend software engineer and architect.

# Scala tooling

* the Scala tooling below is provided by the Metals MCP server. If its tools
  (e.g. `import-build`, `compile-full`, `inspect`) are not detected, install and
  start the server from the project root with:
  `metals-mcp --workspace . --client claude`
* ALWAYS use tools to compile and run tests instead of relying on bash commands
* after adding a dependency to `build.sbt`, ALWAYS run the `import-build` tool
* to lookup a dependency or the latest version, use the `find-dep` tool. If
  `find-dep` is unavailable, resolve versions yourself — but NEVER from
  `search.maven.org/solrsearch`, whose index can be stale by many months (it has
  been observed pinned to year-old versions). Use these instead:
  * latest version of a KNOWN artifact — read the canonical resolver source,
    `https://repo1.maven.org/maven2/<group-with-slashes>/<artifact>/maven-metadata.xml`,
    and take `<release>` (or the last `<version>`). `<release>` may itself be a
    prerelease (e.g. scalatest's was `3.3.0-SNAP4`, Scala 3's `3.10.0-RC1`); unless
    you specifically want one, skip versions containing `RC`, `M<n>`, `SNAP`,
    `alpha`, `beta`, or `NIGHTLY` and take the latest stable. Remember the Scala
    suffix, e.g. `com/softwaremill/ox/core_3`. This is never stale.
  * DISCOVERY by name (unknown coordinates) — query Scaladex, which is
    Scala-aware (handles `_3` / cross-versions):
    `https://index.scala-lang.org/api/autocomplete?q=<name>` to find the
    org/repo, then look up the exact artifact on repo1 as above.
* to lookup the API of a class, use the `inspect` tool. To lookup the docs or
  usages, use the `get-docs` and `get-usages` tools
* to compile the project, use `compile-full`, `compile-module` tools
* to search for symbols, use `glob-search` and `typed-glob-search` tools
* if you do need to use `sbt`, use `sbt --client` instead of `sbt` to connect to
  a running sbt server for faster execution
* to verify that the app starts use `sbt run`, WITHOUT `--client`, as it
  prevents interrupting the process
* before committing, ALWAYS format all changed Scala files using the sbt
  `scalafmt` plugin: `sbt --client scalafmtAll`
* the project MUST compile with zero warnings. Ensure `build.sbt` includes
  `-Wunused:all`, `-Wvalue-discard`, `-Wnonunit-statement` in `scalacOptions`.
  Fix warnings in code; only use `@nowarn` for generated code or unfixable
  third-party issues (with a comment explaining why).

# Coding style

* ALWAYS use braceless syntax — do not use `{}`
* NEVER use `end` marker syntax (e.g. `end run`, `end MyClass`, `end if`). Do not
  write them, and configure scalafmt to neither insert nor keep them:
  `rewrite.scala3.insertEndMarkerMinLines = 0` and
  `rewrite.scala3.removeEndMarkerMaxLines = 1024` in `.scalafmt.conf`.
* responsibilities in code MUST be segregated between appropriately named
  entities
* before creating or moving a `.scala` file, decide its package, filename, and
  top-level visibility. Read [Code Organization and
  Visibility](160-code-organization.md) when adding packages/modules or
  widening visibility.
* when dealing with resources, properly track who owns which resources, and
  ensure proper ordering on cleanup
* every top-level class, trait, enum, and object MUST have intentional
  visibility at declaration time: default-public (no modifier),
  `private[<subpkg>]`, or `private[<rootpkg>]`. Choose by scanning actual call
  sites.
* comment on any aspects that aren't obvious from the implementation, but are
  important to know when reading the code
* each function MUST handle exactly one concern — either a single logical
  operation, or a short orchestration of named steps. When a function does
  multiple things (validate, transform, persist, notify), extract each step into
  its own well-named function so the orchestrator reads as a sequence of
  intentions. Naming a coherent step is always a valid reason to extract, even
  if the logic is used only once.
* use `.pipe` and `.tap` from the standard library
  (`import scala.util.chaining.*`) to drop single-use `val`s that only feed the
  next line. `.pipe(f)` returns `f(value)`; `.tap(f)` runs a side effect and
  returns the value unchanged. Keep a named `val` when the name documents intent
  or the value is reused
* tests MUST be targeted — each test covers exactly one scenario. No
  overlapping or redundant tests.
* every public function, val, and given MUST have an explicit return type — this
  prevents accidental signature drift during refactors:

```scala
// Wrong — inferred return type can silently change:
def findUser(id: Id[User])(using DbTx) =
  userModel.findById(id).toRight(Fail.NotFound("user"))

// Right — return type is explicit and stable:
def findUser(id: Id[User])(using DbTx): Either[Fail, User] =
  userModel.findById(id).toRight(Fail.NotFound("user"))
```

# Performance

* NEVER materialize unbounded data into memory. Use streaming or paging to
  process large datasets and paginated API results incrementally.

# Functional programming

* use pure functions, immutable data, higher-order functions, ADTs. NEVER use
  shared mutable state.
* `var` declarations MUST be inside methods (e.g. processing loops), **never**
  as class fields. Class-level `var`s break reasoning and testability. Use only
  immutable collections (`Map`, `Set`, `List`) — never `mutable.Map`,
  `mutable.Set`, `mutable.Buffer`.
* model state as an immutable case class. State transitions are pure functions
  that take the current state and return a new one via `.copy()`. Confine the
  `var` that threads state to the smallest possible scope:

```scala
case class ProcessingState(
    processed: Map[String, Long] = Map.empty,
    pending: Set[String] = Set.empty
)

def handleItem(state: ProcessingState, item: Item): ProcessingState =
  state.copy(processed = state.processed.updated(item.key, item.offset))

def run(items: Iterator[Item]): ProcessingState =
  var state = ProcessingState()
  for item <- items do
    state = handleItem(state, item)
  state
```

* push side effects behind traits so that state transitions are testable without
  real infrastructure. Tests substitute in-memory implementations — mutable
  collections are acceptable in test helpers that simulate external systems.
* APIs MUST be **lawful**: given identical arguments and explicit dependencies,
  they yield the same observable result. Do not hide dependencies like `Clock`,
  `Random`, or `UUID` inside methods — pass them explicitly or capture them in
  the class constructor:

```scala
// Wrong — hidden non-determinism:
class OrderService:
  def place(order: Order): Confirmation =
    val id = UUID.randomUUID()
    val now = Instant.now()
    Confirmation(id, now)

// Right — dependencies are explicit and injectable:
class OrderService(clock: Clock, idGenerator: () => UUID):
  def place(order: Order): Confirmation =
    val id = idGenerator()
    val now = clock.instant()
    Confirmation(id, now)
```

* wrap `String`, `Int`, `Long`, and `Boolean` domain values in opaque types or
  enums — NEVER use raw primitives for domain concepts. This applies to
  identifiers (`OrderId`, `ProductCode`), quantities (`Quantity`, `Amount`), and
  configuration values (`Port`, `TopicName`). When a generated library (e.g.
  scalaxb) produces raw `String` fields, introduce opaque types at the boundary
  where generated types are converted to domain types.
* eliminate boolean blindness — replace `Boolean` parameters and return values
  with two-case enums so intent is explicit and exhaustiveness is checked:

```scala
// Wrong — caller must remember what `true` means:
def recordFlush(success: Boolean, durationMs: Double): Unit

// Right — intent is unambiguous:
enum FlushOutcome:
  case Success, Failure
def recordFlush(outcome: FlushOutcome, duration: Duration): Unit
```
* when an `if`/`else` only picks between values, fold it away with combinators
  rather than repeating the test: `.fold`/`.getOrElse`/`.map`/`.foreach`, the
  same on `Option` and `Either`. Derive the decision once instead of re-testing
  it across functions. `isEmpty`/`nonEmpty` is the predicate — don't expand it to
  `case Nil`.

```scala
// Wrong — repeats the test + derivation per caller:
if labels.isEmpty then "none" else labels.distinct.mkString(", ")
// Right — derive once, fold the absence away:
Option.when(labels.nonEmpty)(labels.distinct.mkString(", ")).getOrElse("none")
```
* NEVER throw exceptions for recoverable failures. Instead, return an `Either[E, T]`.
  Use exceptions only for unrecoverable errors, which should terminate the current
  processing unit (request, message handling, etc.)
* if a value can be absent, use `Option[T]` — NEVER use `null` or sentinel
  values. `Option` is for presence/absence only, not for errors.
* model different states of an entity as separate types — NEVER use `Option`
  fields to represent state transitions:

```scala
// Wrong — callers must remember to check confirmedAt:
case class Order(id: Id[Order], items: List[Item], confirmedAt: Option[Instant])

// Right — the type tells you what state the order is in:
case class PendingOrder(id: Id[Order], items: List[Item])
case class ConfirmedOrder(id: Id[Order], items: List[Item], confirmedAt: Instant)
```

* design domain models so that invalid data CANNOT be constructed. Use enums,
  opaque types, or smart constructors to encode invariants:

```scala
// Wrong — any string is accepted:
def setPort(port: Int): Unit

// Right — invalid values are rejected at construction:
opaque type Port = Int
object Port:
  def apply(value: Int): Either[String, Port] =
    if value >= 1 && value <= 65535 then Right(value)
    else Left(s"Port out of range: $value")
```

* define sealed-trait or enum error hierarchies — NEVER use stringly-typed
  errors.
* NEVER use bare `try`/`catch` for recoverable failures. Reserve `try`/`catch` for
  defect or unrecoverable error boundaries only.

# Use-Case Guide

BEFORE writing Scala code that uses one of the topics below, fetch the chapter
and follow the patterns shown there.

Retrieve the chapter as raw, unmodified text — read every code block and
paragraph in full. Do NOT use a tool that summarises the page: summaries silently
drop the `> Required` / `> Important` callouts and the exact API calls that make
the chapter correct. Prefer reading the chapter file directly from the installed
skill directory; if fetching over the network, use a method that returns the
verbatim file (a raw HTTP GET), not a fetch-and-summarise tool.

Base URL for this plugin's chapters:
https://raw.githubusercontent.com/weihsiu/scala-skill/refs/heads/master/scala/skills/scala/

## Chapters

- [Type-Safe Configuration](120-type-safe-configuration.md) — PureConfig with
  `derives ConfigReader`, environment variable overrides, `Sensitive` wrapper,
  load-time validation.

- [Code Organization and Visibility](160-code-organization.md) — top-level
  visibility, file naming exceptions, Scala 3 package shadowing, and
  sbt/Scalafix boundary enforcement.

- [SQL Persistence](400-sql-persistence.md) — Magnum with PostgreSQL: `@Table`
  case classes, `DbCodec` for opaque types, `Repo`/`TableInfo`, `sql`
  interpolation, Flyway migrations, HikariCP.

## Companion plugins

For HTTP, structured concurrency, and effect-system topics on top of this
foundation, install one of:

- **`softwaremill-scala`** — direct-style with Ox, Tapir, sttp, ox-kafka,
  MacWire (the SoftwareMill stack).
- **`yaes-scala`** — direct-style with yaes (algebraic effects via Scala 3
  context parameters).

The two are alternatives; pick the one that matches your project. The rules in
this `scala` plugin apply to both.
