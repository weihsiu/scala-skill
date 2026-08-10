# Scala Skills

A collection of [Scala](https://www.scala-lang.org) skills for [Claude
Code](https://claude.ai/claude-code) and [Codex](https://openai.com/codex/),
packaged as four independent plugins in a single marketplace:

- a foundational **`scala`** plugin (language and FP rules)
- a **`softwaremill-scala`** plugin for the Ox / Tapir / sttp stack
- a **`yaes-scala`** plugin for the [yaes](https://github.com/yaes-io/yaes)
  effect system
- a **`cats-effect-scala`** plugin for [Cats
  Effect](https://typelevel.org/cats-effect/) and the Typelevel stack

The last three are alternatives to each other at the same level. Install
`scala` plus whichever one your project uses.

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
  [yaes (λÆS)](https://github.com/yaes-io/yaes) — algebraic effects via Scala
  3 context parameters, structured concurrency on virtual threads.
- **[cats-effect-scala](cats-effect-scala/skills/cats-effect-scala/)** —
  Monadic Scala 3 with [Cats Effect 3](https://typelevel.org/cats-effect/) and
  the [Typelevel](https://typelevel.org) stack:
  [http4s](https://http4s.org), [fs2](https://fs2.io),
  [circe](https://circe.github.io/circe/),
  [doobie](https://typelevel.org/doobie/).

### Choosing a stack

| Plugin | Style | Effects are… | JDK floor |
| --- | --- | --- | --- |
| `softwaremill-scala` | direct | plain expressions inside `Ox` scopes | 21 |
| `yaes-scala` | direct | capabilities in `using` clauses | 24 |
| `cats-effect-scala` | monadic | values (`IO[A]`) composed with `flatMap` | 11 |

Pick exactly one per project — they have overlapping but incompatible takes on
concurrency and resource handling. Broadly: `cats-effect-scala` for the most
mature ecosystem and the widest JDK support, `softwaremill-scala` for
production-ready direct style on virtual threads, `yaes-scala` for fine-grained
algebraic effects if you can accept an experimental library.

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
/plugin install cats-effect-scala@scala-skill
```

(Install `scala` for any Scala project. Then pick **one** of
`softwaremill-scala`, `yaes-scala`, or `cats-effect-scala` depending on which
effect/concurrency stack your project uses.)

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
# then one of:
cp -r /tmp/scala-skill/softwaremill-scala/skills/softwaremill-scala ~/.claude/skills/
cp -r /tmp/scala-skill/yaes-scala/skills/yaes-scala ~/.claude/skills/
cp -r /tmp/scala-skill/cats-effect-scala/skills/cats-effect-scala ~/.claude/skills/
```

### Cursor, OpenCode, and Pi

These editors load [Agent Skills](https://agentskills.io) (the `SKILL.md`
format) directly. All three discover skills from the vendor-neutral
`~/.agents/skills/` directory, so a single copy makes the plugins available in
each of them. Copy the foundation plus whichever stack you use:

```bash
git clone https://github.com/weihsiu/scala-skill.git /tmp/scala-skill
mkdir -p ~/.agents/skills
cp -r /tmp/scala-skill/scala/skills/scala ~/.agents/skills/
# then one of:
cp -r /tmp/scala-skill/softwaremill-scala/skills/softwaremill-scala ~/.agents/skills/
cp -r /tmp/scala-skill/yaes-scala/skills/yaes-scala ~/.agents/skills/
cp -r /tmp/scala-skill/cats-effect-scala/skills/cats-effect-scala ~/.agents/skills/
```

Restart the editor (or your `cursor-agent` CLI session); the agent loads a
skill automatically when a task looks relevant. You can also trigger one
explicitly: `/scala`, `/softwaremill-scala`, `/yaes-scala`, or
`/cats-effect-scala` in Cursor's Agent chat, or `/skill:scala` (etc.) in Pi.

Each tool also has its own native skills directory if you prefer to scope the
install per tool — use it instead of `~/.agents/skills/` above:

- **[Cursor](https://cursor.com/docs/skills)**: `~/.cursor/skills/` (global) or
  `.cursor/skills/` (per project)
- **[OpenCode](https://opencode.ai/docs/skills/)**:
  `~/.config/opencode/skills/` (global) or `.opencode/skills/` (per project)
- **[Pi](https://pi.dev/docs/latest/skills)**: `~/.pi/agent/skills/` (global) or
  `.pi/skills/` (per project)

For per-project use, copy the skill folders into the project-level
`.agents/skills/` directory instead, and commit them alongside your code.

#### Staying up to date

The `cp` above installs a snapshot — it does not auto-update. None of these
editors poll a remote for skill updates. To stay current, clone the repository
once and symlink the skill folders into the skills directory, then `git pull` to
refresh:

```bash
git clone https://github.com/weihsiu/scala-skill.git ~/scala-skill
ln -s ~/scala-skill/scala/skills/scala ~/.agents/skills/scala
ln -s ~/scala-skill/softwaremill-scala/skills/softwaremill-scala ~/.agents/skills/softwaremill-scala
# update later (refreshes Cursor, OpenCode, and Pi together):
git -C ~/scala-skill pull
```

Wrap the `git pull` in a cron job or shell-login hook if you want it to run
automatically.

## Updating an installed plugin

Marketplace installs (Claude Code and Codex) are **version-gated**: an already
installed plugin only picks up new content when its `version` changes. Pushing
new commits to this repo is **not** enough on its own.

For users with the plugins installed:

```
# Claude Code — updating the marketplace also updates its installed plugins:
/plugin marketplace update scala-skill     # refresh + update installed plugins
# (equivalently, from a shell:)
claude plugin marketplace update scala-skill
# Restart Claude Code afterwards for the new version to apply.

# To update a single plugin instead of the whole marketplace:
/plugin update scala@scala-skill           # or: claude plugin update scala@scala-skill

# Codex
codex plugin marketplace upgrade scala-skill
# then reinstall/refresh the plugin from /plugins
```

> **Maintainer note — bump the version, or agents won't see the change.**
> Because installs are version-gated, every time you change a plugin's content
> you MUST bump that plugin's `version` (`0.1.0` → `0.1.1`) in **all** of:
>
> - `<plugin>/.claude-plugin/plugin.json`
> - `<plugin>/.codex-plugin/plugin.json`
> - the matching entry in `.claude-plugin/marketplace.json`
>
> Bump only the plugins you actually changed — the four plugins version
> independently (see `CLAUDE.md`). Until the version is bumped and pushed,
> `/plugin marketplace update` finds nothing new and agents keep running the
> old skill.

## Keeping up with upstream

The `softwaremill-scala` and `scala` plugins are derived from
[VirtusLab/scala-skill](https://github.com/VirtusLab/scala-skill); the
`yaes-scala` plugin is derived from
[yaes-io/yaes](https://github.com/yaes-io/yaes); the `cats-effect-scala`
plugin is derived from the official documentation of the Typelevel projects it
covers. All move independently.

Inside this repo, run the `/sync-upstream` slash command in Claude Code to
detect upstream changes, review each diff, and merge approved updates. State
(last-synced SHAs, the upstream→local path mapping) is tracked in
`.sync-state.json`. See `CLAUDE.md` for the full workflow.

## Attribution

Content derived from VirtusLab/scala-skill is Apache-2.0. Content derived
from yaes-io/yaes is MIT-licensed. Content in `cats-effect-scala` is
hand-written from the Typelevel projects' documentation (Apache-2.0 and MIT).
The repository itself is Apache-2.0. See `NOTICE` and `LICENSE`.
