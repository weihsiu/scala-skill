# Channels and Flow

`yaes-data` adds two streaming primitives on top of `Async`: **`Channel`**
(producer/consumer with backpressure) and **`Flow`** (lazy reactive-streams
operators).

## Dependencies

```sbt
"io.yaes" %% "yaes-data"
```

## Channel — inter-fiber communication

Three flavours:

```scala
import io.yaes.Channel
import io.yaes.Channel.OverflowStrategy

val unbounded  = Channel.unbounded[Int]()                 // grows without bound
val bounded    = Channel.bounded[Int](capacity = 2)       // blocks producer at capacity
val droppy     = Channel.bounded[Int](
  capacity = 2,
  onOverflow = OverflowStrategy.DROP_OLDEST                // or DROP_LATEST
)
val rendezvous = Channel.rendezvous[String]()             // no buffer: synchronous handoff
```

### Send / receive

```scala
val ch = Channel.bounded[Int](capacity = 4)

ch.send(42)
val v = ch.receive()    // may raise ChannelClosed
ch.foreach(v => println(v))
ch.close()              // no more sends; receivers drain buffer
ch.cancel()             // drop buffer immediately
```

`receive()` raises `ChannelClosed` once the channel is closed and empty. Either
handle it with `Raise.either` or use `foreach`, which exits cleanly.

### Producer DSL

For "produce items, return a channel", use the `Channel.produce` builder:

```scala
import io.yaes.Channel.Producer

val squares = Channel.produce[Int] {
  (1 to 10).foreach(i => Producer.send(i * i))
}

squares.foreach(v => println(s"square: $v"))
```

A custom channel shape:

```scala
val ch = Channel.produceWith(Channel.Type.Bounded(5)) {
  var n = 0
  while n < 100 do
    Producer.send(n)
    n += 1
}
```

## Flow — reactive streams of operations

`Flow` is a declarative, lazy stream. Operators compose; nothing runs until
`collect`:

```scala
import io.yaes.Flow

val pipeline = Flow(1, 2, 3, 4, 5)
  .map(_ * 2)
  .filter(_ > 4)
  .take(2)

val out = scala.collection.mutable.ArrayBuffer[Int]()
pipeline.collect(out += _)   // ArrayBuffer(6, 8)
```

### Backpressured buffering

To detach producer and consumer rates, insert a `buffer`:

```scala
import io.yaes.Channel
import io.yaes.Channel.buffer

flow.buffer().collect { println(_) }                                 // unbounded
flow.buffer(Channel.Type.Bounded(2)).collect { println(_) }          // capped
flow.buffer(Channel.Type.Bounded(2, OverflowStrategy.DROP_OLDEST))   // capped + drop
  .collect { println(_) }
```

### Merging concurrent flows

`Channel.channelFlow` lets you build a `Flow` that hosts multiple producer
fibers:

```scala
def merge[T](a: Flow[T], b: Flow[T]): Flow[T] =
  Channel.channelFlow[T] {
    Async.fork { a.collect(v => Channel.Producer.send(v)) }
    Async.fork { b.collect(v => Channel.Producer.send(v)) }
  }
```

> **Required:** `channelFlow` runs in an `Async` scope when consumed. Make
> sure the consumer's outer handler installs `Async`, or you will get a
> capability error.

## Reactive Streams interop

Bridge a `Flow` to a `java.util.concurrent.Flow.Publisher`:

```scala
import io.yaes.{Flow, FlowPublisher}
import io.yaes.FlowPublisher.asPublisher
import java.util.concurrent.Flow.{Subscriber, Subscription}

val src = Flow(1, 2, 3, 4, 5)

Async.run {
  val publisher = src.asPublisher()
  publisher.subscribe(new Subscriber[Int]:
    var sub: Subscription = _
    def onSubscribe(s: Subscription): Unit =
      sub = s; s.request(10)
    def onNext(item: Int): Unit =
      println(s"got $item"); sub.request(1)
    def onError(t: Throwable): Unit = println(s"err: $t")
    def onComplete(): Unit = println("done")
  )
}
```

For an explicit buffer policy: `FlowPublisher.fromFlow(flow, Channel.Type.Bounded(64))`.

## Pitfalls

> **Warning:** `Channel.send` on a closed channel raises; `Channel.receive` on
> a closed *and empty* channel raises. Decide which signal you want — close
> for "no more items" or cancel for "drop everything now."

> **Important:** `Flow` is lazy; `collect` is the only thing that drives it.
> If your operators include side effects expected to run "for free," they will
> not run until something consumes the flow.

> **Warning:** the `State` effect is single-threaded. Do not share a `State`
> capability across fibers — use a `Channel` to serialise state changes, or
> hold the state in an `AtomicReference`.
