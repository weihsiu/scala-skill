# Scala Skills

A collection of [Scala](https://www.scala-lang.org) skills for [Claude
Code](https://claude.ai/claude-code) and [Codex](https://openai.com/codex/),
packaged as three independent plugins in a single marketplace:

- a foundational **`scala`** plugin (language and FP rules)
- a **`softwaremill-scala`** plugin for the Ox / Tapir / sttp stack
- a **`yaes-scala`** plugin for the [yaes](https://github.com/rcardin/yaes)
  effect system — an alternative to the SoftwareMill stack at the same level

Install one, two, or all three.

## Available plugins

- **[scala](scala/skills/scala/)** — General Scala 3 coding style, tooling, and
  functional programming guidance. The foundation; other plugins assume it.
- **[softwaremill-scala](softwaremill-scala/skills/softwaremill-scala/)** —
  Direct-style Scala on the SoftwareMill stack:
  [Ox](https://ox.softwaremill.com) structured concurrency, synchronous
  [Tapir](https://tapir.softwaremill.com), [sttp
  client](https://sttp.softwaremill.com), ox-kafka,
  [MacWire](https://github.com/softwaremill/macwire).
- **[yaes-scala](yaes-scala/skills/yaes-scala/)** — Direct-style Scala 3 with
  [yaes (λÆS)](https://github.com/rcardin/yaes) — algebraic effects via Scala
  3 context parameters, structured concurrency on virtual threads.

## Installation

### Claude Code

Add the marketplace:

```
/plugin marketplace add weihsiu/scala-skill
```

Install the plugins you want:

```
/plugin install scala@scala-skill
/plugin install softwaremill-scala@scala-skill
/plugin install yaes-scala@scala-skill
```

(Install `scala` for any Scala project. Then pick `softwaremill-scala` **or**
`yaes-scala` depending on which effect/concurrency stack your project uses.)

### Codex

```sh
codex plugin marketplace add weihsiu/scala-skill
```

Then open Codex and install the desired plugin(s) from `/plugins`.

If you added the marketplace before, refresh it first:

```sh
codex plugin marketplace upgrade scala-skill
```

For local development, add a checkout as a marketplace instead:

```sh
git clone https://github.com/weihsiu/scala-skill.git /tmp/scala-skill
codex plugin marketplace add /tmp/scala-skill
```

### Manual install

```sh
git clone https://github.com/weihsiu/scala-skill.git /tmp/scala-skill
mkdir -p ~/.claude/skills
cp -r /tmp/scala-skill/scala/skills/scala ~/.claude/skills/
cp -r /tmp/scala-skill/softwaremill-scala/skills/softwaremill-scala ~/.claude/skills/
# or
cp -r /tmp/scala-skill/yaes-scala/skills/yaes-scala ~/.claude/skills/
```

## Keeping up with upstream

The `softwaremill-scala` and `scala` plugins are derived from
[VirtusLab/scala-skill](https://github.com/VirtusLab/scala-skill); the
`yaes-scala` plugin is derived from
[rcardin/yaes](https://github.com/rcardin/yaes). Both move independently.

Inside this repo, run the `/sync-upstream` slash command in Claude Code to
detect upstream changes, review each diff, and merge approved updates. State
(last-synced SHAs, the upstream→local path mapping) is tracked in
`.sync-state.json`. See `CLAUDE.md` for the full workflow.

## Attribution

Content derived from VirtusLab/scala-skill is Apache-2.0. Content derived
from rcardin/yaes is MIT-licensed. The repository itself is Apache-2.0. See
`NOTICE` and `LICENSE`.
