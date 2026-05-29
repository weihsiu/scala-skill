# CLAUDE.md — repo-level guidance

This file tells Claude Code how to work in this repository. It is not a skill
that ships to users; it shapes how Claude behaves when **editing the repo
itself**.

## What this repo is

A fork of [VirtusLab/scala-skill](https://github.com/VirtusLab/scala-skill),
restructured into **three independent Claude Code / Codex plugins** in a
single marketplace:

| Plugin              | Scope                                                                |
| ------------------- | -------------------------------------------------------------------- |
| `scala`             | General Scala 3 language, tooling, FP rules. Foundation.             |
| `softwaremill-scala`| Direct-style Scala on Ox + Tapir + sttp + ox-kafka + MacWire.        |
| `yaes-scala`        | Direct-style Scala with the [yaes](https://github.com/rcardin/yaes) algebraic effect system. |

`scala` is the prerequisite. `softwaremill-scala` and `yaes-scala` are
**alternatives** to each other — both deliver direct-style concurrency on Java
virtual threads, just via different libraries. Pick one per project.

## Layout

```
scala-skill/
├── .claude-plugin/marketplace.json        ← Claude Code marketplace (3 plugins)
├── .agents/plugins/marketplace.json       ← Codex marketplace (mirror)
├── .claude/commands/sync-upstream.md      ← /sync-upstream slash command
├── .sync-state.json                       ← tracked upstream SHAs + path mapping
├── scala/                                 ← plugin: pure Scala
├── softwaremill-scala/                    ← plugin: SoftwareMill stack
├── yaes-scala/                            ← plugin: yaes effect system
├── CLAUDE.md                              ← this file
├── LICENSE                                ← Apache-2.0
├── NOTICE                                 ← attribution
└── README.md                              ← user-facing install instructions
```

Each plugin folder follows the same shape:

```
<plugin>/
├── .claude-plugin/plugin.json
├── .codex-plugin/plugin.json
└── skills/<plugin>/
    ├── SKILL.md       ← skill definition (frontmatter + body)
    ├── README.md      ← human-facing intro
    └── NNN-*.md       ← numbered chapters
```

## Install

End users install through Claude Code's plugin marketplace:

```
/plugin marketplace add weihsiu/scala-skill
/plugin install scala@scala-skill
/plugin install softwaremill-scala@scala-skill   # or
/plugin install yaes-scala@scala-skill
```

Codex users:

```
codex plugin marketplace add weihsiu/scala-skill
# then install the desired plugin(s) from /plugins
```

For local development, point the marketplace at a checkout:

```
/plugin marketplace add /Users/walterchang/Home/projects/scala/scala-skill
codex plugin marketplace add /Users/walterchang/Home/projects/scala/scala-skill
```

Manual install (bypassing the marketplace):

```sh
git clone https://github.com/weihsiu/scala-skill.git /tmp/scala-skill
mkdir -p ~/.claude/skills
cp -r /tmp/scala-skill/scala/skills/scala               ~/.claude/skills/
cp -r /tmp/scala-skill/softwaremill-scala/skills/softwaremill-scala ~/.claude/skills/
# OR cp -r /tmp/scala-skill/yaes-scala/skills/yaes-scala ~/.claude/skills/
```

## Authoring conventions

These rules carry over from upstream's `CONTRIBUTING.md` (now living at
`softwaremill-scala/skills/softwaremill-scala/CONTRIBUTING.md`):

* **Numbering** — chapters use numeric groups: 100s (setup/structure), 200s
  (error handling), 300s (HTTP / endpoints), 400s (data / integration), 500s
  (testing / observability). Within a group, increment by 10. Never re-number
  an existing chapter — it breaks external links and the `.sync-state.json`
  mapping.
* **Frontmatter** — every `SKILL.md` starts with `--- name: <slug> --- description: ...
  ---`. Chapter files do not need frontmatter.
* **Format** — title, optional dependencies, then `##` sections with prose and
  code. One use-case per file.
* **Code is grounded** — examples come from a real reference project (Bootzooka
  for the SoftwareMill stack, the yaes README/modules for yaes), trimmed to
  the essence.
* **Show only the essence** — strip unrelated features and domain noise.
* **`>` callouts for pitfalls** — use `> **Warning:**`, `> **Important:**`,
  `> **Required:**` for things that affect correctness or are easy to miss.

## Syncing with upstreams

This repo derives from two upstreams that move independently:

* **`VirtusLab/scala-skill`** (Apache-2.0) — the source of the `scala` and
  `softwaremill-scala` chapters.
* **`rcardin/yaes`** (MIT) — the source material for the `yaes-scala` chapters.
  yaes does **not** publish skill files itself; we re-derive yaes content from
  its README and source.

Run the sync workflow with `/sync-upstream`. It is a Claude-orchestrated
command (not a shell script) because step-by-step prose merges and yaes
re-derivation need judgment. See `.claude/commands/sync-upstream.md` for the
exact flow.

State is tracked in `.sync-state.json`:

* `upstream_scala_skill.last_synced_sha` — the upstream commit we last merged.
* `upstream_scala_skill.path_mapping` — upstream path → local path. Add an
  entry whenever a new upstream file lands.
* `yaes.last_synced_sha` / `last_synced_version` — what we last derived
  `yaes-scala/` from.

**Never edit `.sync-state.json` by hand.** It is updated only by
`/sync-upstream`. If you must change it manually (e.g. recovering from a
botched sync), commit the change separately with a clear message.

**Never auto-merge upstream diffs.** Every change to a `softwaremill-scala/`
or `scala/` chapter that originates upstream should pass through user
confirmation. The local SKILL.md split, plugin layout, and any local edits
must not be silently undone by a sync.

## Attribution

* Content in `scala/skills/scala/` and `softwaremill-scala/skills/softwaremill-scala/`
  is derived from VirtusLab/scala-skill under Apache-2.0. The original
  copyright is preserved; modifications are local. See `NOTICE`.
* Content in `yaes-scala/skills/yaes-scala/` is hand-written based on the
  rcardin/yaes README and module sources under MIT. See `NOTICE`.
* The repository itself is Apache-2.0.

## Do / Don't when editing this repo

**Do**

* Keep all three plugin folders self-contained — a chapter in `scala/` must
  not link `../softwaremill-scala/...` directly. Reference companion plugins by
  prose name, not relative path.
* Preserve numbering when moving chapters between plugins (used by sync).
* When adding a new chapter from upstream, pick the right plugin first, then
  add a corresponding entry under `path_mapping` in `.sync-state.json`.
* When yaes adds a new module or effect, weigh whether it deserves its own
  chapter or fits an existing one. Aim for ~7–10 chapters total in `yaes-scala/`.

**Don't**

* Don't reintroduce a single combined plugin. The split is the design.
* Don't bump version numbers across all three plugins in lockstep — they version
  independently.
* Don't add cross-skill imports that assume "the other plugin is installed."
  Each plugin must be coherent on its own (with `scala` as documented
  prerequisite).
* Don't edit `.sync-state.json` outside `/sync-upstream` runs.
