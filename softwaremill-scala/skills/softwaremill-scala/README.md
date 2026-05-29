# softwaremill-scala

Direct-style Scala on the [SoftwareMill](https://softwaremill.com) stack:
[Ox](https://ox.softwaremill.com) for structured concurrency, synchronous
[Tapir](https://tapir.softwaremill.com) for HTTP, [sttp
client](https://sttp.softwaremill.com), [ox-kafka](https://ox.softwaremill.com),
and [MacWire](https://github.com/softwaremill/macwire) for compile-time DI.

This plugin builds on the general
[`scala`](https://github.com/weihsiu/scala-skill/tree/master/scala/skills/scala)
plugin — install that first.

## Prerequisites

Assumes the [Metals
MCP](https://softwaremill.com/a-beginners-guide-to-using-scala-metals-with-its-model-context-protocol-server/)
server is available and uses [sbt](https://www.scala-sbt.org) as the build tool.

## Attribution

The chapters in this plugin are derived from
[VirtusLab/scala-skill](https://github.com/VirtusLab/scala-skill) (Apache-2.0)
and kept in sync via the `/sync-upstream` slash command in the parent repo.
