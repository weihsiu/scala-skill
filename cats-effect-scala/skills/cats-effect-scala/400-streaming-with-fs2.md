# Streaming with fs2

[fs2](https://fs2.io) provides `Stream[F, A]` — a lazy, resource-safe,
constant-memory sequence of effects. http4s bodies, doobie result sets, and
Kafka consumers are all fs2 streams, so it is the connective tissue of the
Typelevel stack.

## Dependencies

```sbt
libraryDependencies ++= Seq(
  "co.fs2" %% "fs2-core" % "3.13.0",
  "co.fs2" %% "fs2-io"   % "3.13.0",   // files, sockets, stdin/stdout
)
```

## Building streams

```scala
import cats.effect.IO
import fs2.Stream
import scala.concurrent.duration.*

Stream.emit(1)                       // single element
Stream.emits(List(1, 2, 3))          // from a collection
Stream.eval(IO(readLine()))          // one effect
Stream.repeatEval(IO(readLine()))    // that effect, forever
Stream.iterate(0)(_ + 1)             // pure infinite
Stream.awakeEvery[IO](1.second)      // a tick every second
Stream.range(0, 100)
Stream.resource(fileResource)        // a Resource as a one-element stream
Stream.never[IO]                     // never emits, never terminates
```

Nothing runs until the stream is compiled and the resulting effect is executed:

```scala
val program: IO[List[Int]] = Stream.range(0, 10).covary[IO].compile.toList
val drained: IO[Unit]      = stream.compile.drain      // run for effects only
val counted: IO[Long]      = stream.compile.count
val folded: IO[Int]        = stream.compile.fold(0)(_ + _)
val last: IO[Option[A]]    = stream.compile.last
```

## Transforming

```scala
// Pure transformations:
stream.map(_ * 2).filter(_ > 10).take(100).drop(5)

// Effectful, sequential:
stream.evalMap(id => fetchUser(id))
stream.evalFilter(user => isActive(user))
stream.evalTap(user => logger.info(s"processing $user"))   // keeps the element

// Effectful, concurrent — n at a time, order preserved:
stream.parEvalMap(8)(fetchUser)

// Same, without preserving order (faster):
stream.parEvalMapUnordered(8)(fetchUser)

// Grouping:
stream.chunkN(100)                       // fixed-size chunks
stream.groupWithin(100, 5.seconds)       // size OR time, whichever first
stream.sliding(3)

// Combining:
s1 ++ s2                                 // sequential
s1.merge(s2)                             // concurrent interleave
s1.zip(s2)
Stream(s1, s2, s3).parJoin(3)            // run N streams concurrently
```

> **Important:** `groupWithin` is the right tool for batched writes. It emits
> as soon as either the size or the timeout is reached, so a trickle of events
> is still flushed promptly instead of sitting in a half-full buffer.

## Chunks and performance

A `Stream` is internally a sequence of `Chunk`s, and operations work chunk-wise.
This is why fs2 is fast — but `evalMap` breaks chunking, since each effect is
run per element.

```scala
// Chunk-aware, much faster for pure work:
stream.mapChunks(_.map(_ * 2))

// Batch the effect instead of running it per element:
stream.chunkN(500).evalMap(chunk => repo.insertMany(chunk.toList)).unchunks
```

## Resource safety

Streams release resources deterministically, including on early termination:

```scala
import fs2.io.file.{Files, Path}

val lineCount: IO[Long] =
  Files[IO].readAll(Path("big.csv"))
    .through(fs2.text.utf8.decode)
    .through(fs2.text.lines)
    .compile
    .count
// the file handle is closed even if the stream fails or is cancelled

// bracket inside a stream:
Stream.bracket(acquire)(release).flatMap(use)
```

`.take(10)` on a stream reading a 100 GB file reads only what it needs and then
closes the handle.

## Concurrency primitives

```scala
import fs2.concurrent.{Signal, SignallingRef, Topic}

// Topic — publish/subscribe fan-out:
for
  topic <- Topic[IO, Event]
  _     <- Stream(
             producer.through(topic.publish),
             topic.subscribe(1024).through(consumerA),
             topic.subscribe(1024).through(consumerB),
           ).parJoinUnbounded.compile.drain
yield ()

// SignallingRef — a Ref you can observe as a stream:
for
  signal <- SignallingRef[IO, Boolean](false)
  _      <- work.interruptWhen(signal).compile.drain.start
  _      <- signal.set(true)      // stops the stream
yield ()
```

Interruption:

```scala
stream.interruptAfter(30.seconds)
stream.interruptWhen(shutdownSignal)
stream.takeWhile(_.isValid)
```

## Error handling

```scala
stream.handleErrorWith(e => Stream.eval(logger.error(e)("failed")) >> Stream.empty)
stream.attempt                       // Stream[F, Either[Throwable, A]]
stream.rethrow

// Keep going past individual failures:
stream.evalMap(process(_).attempt).collect { case Right(a) => a }

// Retry the whole stream:
Stream.retry(makeStream, delay = 1.second, nextDelay = _ * 2, maxAttempts = 5)

// Always run cleanup:
stream.onFinalize(IO.println("done"))
```

> **Warning:** an error terminates the stream. If one bad record should not
> kill the pipeline, `attempt` *inside* the `evalMap` — as above — rather than
> around the whole stream.

## Interop with http4s

```scala
// Streaming response — constant memory regardless of row count:
case GET -> Root / "export" =>
  val body: Stream[IO, Byte] =
    repo.streamAll
      .map(_.asJson.noSpaces + "\n")
      .through(fs2.text.utf8.encode)
  Ok(body)

// Streaming request body:
case req @ POST -> Root / "bulk" =>
  req.body
    .through(fs2.text.utf8.decode)
    .through(fs2.text.lines)
    .evalMap(line => process(line))
    .compile
    .drain *> Accepted()
```

## A worked pipeline

Read a CSV, look records up concurrently, batch the writes, log progress:

```scala
def importUsers(path: Path): IO[Unit] =
  Files[IO].readAll(path)
    .through(fs2.text.utf8.decode)
    .through(fs2.text.lines)
    .drop(1)                                  // header
    .filter(_.nonEmpty)
    .map(parseRow)
    .parEvalMap(16)(enrichFromApi)            // 16 concurrent calls
    .groupWithin(500, 10.seconds)             // batch the inserts
    .evalTap(batch => logger.info(s"writing ${batch.size}"))
    .evalMap(batch => repo.insertMany(batch.toList))
    .compile
    .drain
```

Memory stays flat whether the file has 100 rows or 100 million.
