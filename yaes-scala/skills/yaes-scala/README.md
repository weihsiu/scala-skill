# yaes-scala

Direct-style Scala 3 with [yaes (λÆS)](https://github.com/rcardin/yaes), an
algebraic effect system using context parameters and context functions.

This plugin is an **alternative** to `softwaremill-scala` — both deliver
direct-style concurrency on virtual threads, but yaes uses fine-grained
capability effects (`Raise`, `Resource`, `Async`, `State`, `Reader`, `Writer`,
…) while the SoftwareMill stack uses Ox scopes plus Tapir/sttp. Pick one per
project; do not mix.

Builds on the general
[`scala`](https://github.com/weihsiu/scala-skill/tree/master/scala/skills/scala)
plugin — install that first.

## Prerequisites

* Scala 3.x
* Java 24+ (yaes uses Virtual Threads and Structured Concurrency)
* [Metals
  MCP](https://softwaremill.com/a-beginners-guide-to-using-scala-metals-with-its-model-context-protocol-server/)
  server and [sbt](https://www.scala-sbt.org)

## Attribution

Chapters in this plugin are hand-written based on the
[yaes README and modules](https://github.com/rcardin/yaes) (MIT-licensed) and
kept in sync via the `/sync-upstream` slash command in the parent repo.
