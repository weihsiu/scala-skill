---
name: softwaremill-scala
description: Direct-style Scala with the SoftwareMill ecosystem — Ox structured concurrency, synchronous Tapir, sttp, ox-kafka, MacWire. Auto-load for any Scala task that uses Ox, Tapir, or sttp. Depends on the general `scala` plugin for language-level rules.
---

You are an expert backend software engineer and architect working in direct-style
Scala on the SoftwareMill stack (Ox, Tapir, sttp, MacWire, ox-kafka).

> **Prerequisite:** the general
> [`scala`](https://github.com/weihsiu/scala-skill/tree/master/scala/skills/scala)
> plugin defines the language-level rules (tooling, coding style, performance,
> functional programming). Install it first. The rules in this file ADD to
> those; they do not replace them.

# Direct-style Scala

* in Tapir, use `.handle` / `.handleSecurity` / `.handleSuccess` to wire
  endpoint logic — NEVER use `.serverLogic` / `.serverSecurityLogic`. The
  `.handle` family is the direct-style API. The `.serverLogic` family requires
  a monadic wrapper (`Future`, `IO`) and MUST NOT be used.
* ALWAYS use Ox for threading, channels, and async coordination. Avoid raw
  `Thread.ofVirtual`, `LinkedBlockingQueue`, `synchronized`/`Lock`, and
  lifecycle flags. Use `java.util.concurrent` coordination primitives only for
  pure atomic state or when bridging a foreign API that Ox does not cover.
* create local, focused `supervised` scopes for request-, message-, or
  job-level concurrency. Accept a parent `(using Ox)` only when a fork or
  resource must be tied to that parent scope's lifetime.
* keep constructors plain; use factories that take `(using Ox)` and return
  values that do not carry the capability.

# Use-Case Guide

BEFORE writing any code that uses Tapir, Ox, sttp, or direct-style Scala, you MUST
fetch the chapter(s) relevant to your current task from this guide and follow the
patterns shown there. This is not optional — code that ignores guide patterns will
be rejected in review.

When fetching, you MUST request the COMPLETE content — every code block, every
paragraph. Summaries lose critical details (e.g. required factory overrides,
specific API calls). Use a prompt like: "Return the COMPLETE raw content. Every
line, every code block. Do not summarize or omit anything."

Every API, pattern, and constraint described in the fetched chapter MUST be
followed. If the chapter says to use a specific API (e.g. `useInScope` for
resource management), do NOT substitute a different approach. If the chapter
marks something as required, it is required.

Base URL for this plugin's chapters:
https://raw.githubusercontent.com/weihsiu/scala-skill/refs/heads/master/softwaremill-scala/skills/softwaremill-scala/

## Application Structure

- [New Project Setup](140-new-project-setup.md) — minimal direct-style
  Scala project skeleton with sbt and Ox: directory layout, `build.sbt`,
  required `scalacOptions`, `OxApp.Simple` entry point. adopt-tapir as a
  starting point for HTTP projects.

- [Resource Management](100-resource-management.md) — `useInScope`,
  `useCloseableInScope`, reverse-order release, scope-based cleanup.

- [Background Processes](110-background-processes.md) — `OxApp` entry point,
  `forkDiscard`/`forkUserDiscard` for daemon vs. user threads,
  `forever`/`sleep` for periodic loops, orderly shutdown.

- [Compile-Time Dependency Injection](130-compile-time-dependency-injection.md)
  — MacWire `autowire`, `autowireMembersOf` for config extraction, `wireList`
  for collecting endpoints.

- [Concurrency and Inter-Thread Communication](150-shared-state-across-threads.md)
  — Flows for declarative concurrent pipelines (`mapPar`, `merge`,
  `mapStateful`), Ox primitive selection, channels for worker mailboxes and
  shutdown, actors for serialized mutable state.

## Error Handling

- [Error Handling](200-error-handling.md) — `Fail` ADT, Ox `either` blocks with
  `.ok()` short-circuiting, `transactEither`, `.catching`, nesting rules.

- [Error Output Customisation](210-error-output-customisation.md) — JSON error
  responses for all error types. Bidirectional `Fail` → HTTP status code
  mapping, `failOutput`, `defaultHandlers` for decode failures and 404s.

- [Decode Failure Handling](220-decode-failure-handling.md) —
  `DefaultDecodeFailureHandler` customisation: respond/message/response pipeline,
  `onDecodeFailureNextEndpoint`, custom failure messages,
  `hideEndpointsWithAuth`.

## HTTP & Endpoints

- [Authentication](300-authentication.md) — `secureEndpoint[T]`,
  `AuthTokenOps[T]` trait, `Auth[T]` authenticator, `handleSecurity` wiring.

- [HTTP Server Configuration](310-http-server-configuration.md) — Security
  headers, CORS, serving static files for SPAs, request cancellation,
  `NettySyncServer` startup.

- [Version API](320-version-api.md) — `sbt-buildinfo` generating `BuildInfo`
  with git commit hash, served from a Tapir endpoint.

- [Compile-Time OpenAPI Generation](330-compile-time-openapi-generation.md) —
  Build-time OpenAPI YAML generation for frontend client codegen (not runtime
  Swagger UI). `EndpointsForDocs`, `@main` generator, sbt task wiring.

- [SOAP with scalaxb](340-soap-with-scalaxb.md) — XSD-to-Scala code generation,
  SOAP envelope wrapping/unwrapping, Tapir XML codecs for scalaxb types,
  `SOAPAction`-based endpoint routing, SOAP fault error handlers.

## Data & Integration

- [Sending Emails](410-sending-emails.md) — `EmailScheduler` trait, pluggable
  senders (SMTP, Mailgun, dummy), email templates, background batch processing.

- [Kafka Streaming](420-kafka-streaming.md) — `KafkaFlow.subscribe`, `mapPar`,
  `KafkaDrain` publishing, offset commits, transactional produce-and-commit,
  graceful shutdown.

> For SQL persistence (Magnum), see chapter `400-sql-persistence.md` in the
> [`scala`](https://github.com/weihsiu/scala-skill/tree/master/scala/skills/scala)
> plugin — Magnum is not a SoftwareMill library, so it lives there.

## Testing & Observability

- [Testing HTTP Endpoints](500-testing-http-endpoints.md) —
  `TapirSyncStubInterpreter` stub backend, `SttpClientInterpreter` for
  type-safe requests, testing public and secured endpoints in-process.

- [OpenTelemetry Observability](510-opentelemetry-observability.md) — SDK
  auto-configuration, Tapir tracing/metrics interceptors, sttp client
  instrumentation, custom metrics, `PropagatingVirtualThreadFactory` for
  context propagation, MDC log correlation.
