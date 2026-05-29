# Resource — Lifecycle Management

The `Resource` effect acquires resources and releases them in **LIFO order**
when the enclosing `Resource.run { … }` block exits — including on exceptions
and on `Raise`-signalled errors.

## Dependencies

```sbt
"in.rcard.yaes" %% "yaes-core"
```

## Acquiring `AutoCloseable` resources

```scala
import in.rcard.yaes.Resource.*
import java.io.{FileInputStream, FileOutputStream}

def copyFile(src: String, dst: String)(using Resource): Unit =
  val in  = Resource.acquire(new FileInputStream(src))
  val out = Resource.acquire(new FileOutputStream(dst))

  val buf = new Array[Byte](1024)
  var n = in.read(buf)
  while n != -1 do
    out.write(buf, 0, n)
    n = in.read(buf)

Resource.run {
  copyFile("source.txt", "target.txt")
}
```

`Resource.acquire(c)` registers `c.close()` to run at scope exit and returns
`c` for immediate use. Resources released in **reverse order of acquisition**
(`out` then `in` above), the same rule as Java try-with-resources.

## Custom acquire / release pairs

When `close()` is not the right cleanup, use `Resource.install`:

```scala
val connection = Resource.install(openDbConnection()) { conn =>
  conn.close()
  println("DB connection closed")
}
```

The second argument is the cleanup function. It runs at scope exit even if the
block raises.

## Fire-and-forget cleanup

For a cleanup action with no acquired value (the moral equivalent of a
`try { … } finally { … }`):

```scala
Resource.ensuring {
  println("Processing finished")
}
```

## Combining effects

`Resource` composes with the other yaes effects. A common shape:

```scala
def importUsers(path: String)(using Resource, Raise[ImportError]): Int =
  val reader = Resource.acquire(openCsv(path))
  var count = 0
  while reader.hasNext do
    val row = reader.next()
    if !row.isValid then Raise.raise(ImportError(s"bad row $count"))
    saveUser(row.toUser)
    count += 1
  count

val outcome: Either[ImportError, Int] = Raise.either {
  Resource.run {
    importUsers("users.csv")
  }
}
```

If `importUsers` raises, the reader still closes — `Resource.run` honours
cleanup before the error escapes.

## Pitfalls

> **Important:** `Resource.run` is the *only* place cleanup runs. Calling
> `Resource.acquire` outside a `Resource.run` will not compile; smuggling the
> capability out of the handler is a bug — the underlying registry is gone the
> moment the handler returns.

> **Warning:** if your "resource" is a long-running fiber, prefer `Async`
> structured concurrency (see chapter 130) over `Resource`. `Resource` is for
> handles that need explicit close/release; fibers cancel via scope exit.

> **Warning:** mixing `Resource.acquire` (closes via `close()`) and
> `Resource.install` (closes via your function) is fine — they share the same
> LIFO registry — but be explicit which one each resource uses. Hidden double
> closes are easy to introduce by accident.
