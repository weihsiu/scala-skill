# Testing yaes Code

The big payoff of capability-based effects: tests substitute **deterministic
handlers** instead of mocking calls. There is no `IO[A]` to interpret — the
function is plain Scala under a different `given`.

## Dependencies

```sbt
"in.rcard.yaes" %% "yaes-core-test-scalatest" % Test
```

## Testing `Raise`

The most useful trick: a function with `Raise[E]` is its own test scaffold.
Just call the handler.

```scala
import in.rcard.yaes.Raise.*
import org.scalatest.flatspec.AnyFlatSpec
import org.scalatest.matchers.should.Matchers

class DivideSpec extends AnyFlatSpec with Matchers:
  "divide" should "produce a result for non-zero divisors" in:
    Raise.either { divide(10, 2) } shouldBe Right(5)

  it should "raise DivisionByZero for zero" in:
    Raise.either { divide(10, 0) } shouldBe Left(DivisionByZero())
```

`shouldBe Left(...)` works because `Raise.either` produces a plain `Either`.
No `IO.unsafeRunSync`, no `Await.result`.

## Testing with mixed effects

When the function under test uses several effects, install handlers in nested
order — outermost handler returns the bare value to ScalaTest:

```scala
def randomWalk(steps: Int)(using State[Int], Random): Int =
  if steps <= 0 then State.get[Int]
  else
    val direction = if Random.nextBoolean then 1 else -1
    State.update(_ + direction)
    randomWalk(steps - 1)

"randomWalk" should "be deterministic with a seeded Random" in:
  given java.util.Random = new java.util.Random(42)  // pin RNG
  val (finalPos, result) = State.run(0) {
    Random.run { randomWalk(10) }
  }
  finalPos shouldBe result
```

## Pinning `Clock`

`Clock` handler picks up the ambient `java.time.Clock`. Override it with a
`given` to make assertions deterministic:

```scala
import java.time.{Clock as JClock, Instant, ZoneOffset}

"order placement" should "use the test clock" in:
  given JClock = JClock.fixed(Instant.parse("2026-01-01T00:00:00Z"), ZoneOffset.UTC)
  val order = Clock.run { placeOrder(item = "book") }
  order.placedAt shouldBe Instant.parse("2026-01-01T00:00:00Z")
```

## Capturing `Writer` output

```scala
import in.rcard.yaes.Writer.*

def report(): Int writes String =
  Writer.write("start")
  Writer.write("compute")
  42

"report" should "log start and compute" in:
  val (log, value) = Writer.run[String, Int] { report() }
  value shouldBe 42
  log shouldBe Vector("start", "compute")
```

## Asserting cleanup ran

When a `Resource.install` cleanup matters for the test:

```scala
"importer" should "close the reader even on Raise" in:
  var closed = false
  val outcome = Raise.either {
    Resource.run {
      Resource.install(())(_ => closed = true)
      Raise.raise(ImportError("boom"))
    }
  }
  outcome shouldBe Left(ImportError("boom"))
  closed shouldBe true
```

## Testing `Async` — prefer `parTraverse` over manual forks

Tests for parallel logic become flaky when you reach for `Thread.sleep`. Use
`Async.parTraverse` to assert on the *set* of outcomes, not on timing:

```scala
"loader" should "load all profiles in parallel" in:
  val ids = List(1, 2, 3, 4, 5)
  val profiles = Async.run { Async.parTraverse(ids)(loadProfile) }
  profiles.map(_.id) should contain theSameElementsAs ids
```

For timing-sensitive code, fix the `Clock`/`Random` handlers and use the
`Schedule` policies — never `Thread.sleep` in tests.

## Pitfalls

> **Important:** if a function uses an effect, but no handler is installed in
> the test, the code *will not compile*. That is the point — but it also
> means a refactor that adds a new effect may suddenly break a test by
> requiring an extra handler in the composition.

> **Warning:** do not test handler implementations alongside business logic.
> Trust the yaes library's handlers; assert on the **business value** the
> function returned, not on how many times a handler was called.

> **Important:** `Async` tests run on real virtual threads — they may
> interleave nondeterministically. Assert on outcomes (set equality, sums,
> presence/absence) rather than on order, unless your code explicitly
> establishes order.
