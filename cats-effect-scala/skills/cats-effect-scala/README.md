# cats-effect-scala

Monadic Scala 3 with [Cats Effect 3](https://typelevel.org/cats-effect/) and the
surrounding [Typelevel](https://typelevel.org) stack — http4s, fs2, circe,
doobie.

This plugin is an **alternative** to `softwaremill-scala` and `yaes-scala`. All
three cover concurrent backend Scala, but they differ in how effects are
represented:

| Plugin | Style | Effects are… |
| --- | --- | --- |
| `cats-effect-scala` | monadic | values (`IO[A]`) composed with `flatMap` |
| `softwaremill-scala` | direct | plain expressions inside `Ox` scopes |
| `yaes-scala` | direct | capabilities in `using` clauses |

Pick one per project; do not mix.

Builds on the general
[`scala`](https://github.com/weihsiu/scala-skill/tree/master/scala/skills/scala)
plugin — install that first.

## Prerequisites

* Scala 3.3+ (the LTS line the Typelevel stack targets)
* Java 11+ (Java 21+ recommended — Cats Effect 3.7 can run its blocking pool on
  virtual threads)
* [Metals
  MCP](https://softwaremill.com/a-beginners-guide-to-using-scala-metals-with-its-model-context-protocol-server/)
  server and [sbt](https://www.scala-sbt.org)

## Attribution

Chapters in this plugin are hand-written based on the official documentation for
[Cats Effect](https://typelevel.org/cats-effect/) (Apache-2.0),
[http4s](https://http4s.org) (Apache-2.0), [fs2](https://fs2.io) (MIT),
[circe](https://circe.github.io/circe/) (Apache-2.0), and
[doobie](https://typelevel.org/doobie/) (MIT), and kept in sync via the
`/sync-upstream` slash command in the parent repo.
