---
description: Bring scala-skill into sync with the VirtusLab/scala-skill, yaes-io/yaes, and Typelevel upstreams. Updates `.sync-state.json` on completion.
---

You are running the **upstream sync workflow** for this repo. Your job is to
detect changes in three upstream sources, present each diff to the user, apply
the ones they approve, and update `.sync-state.json` at the end. Do **not**
auto-merge anything.

There are three upstreams, tracked independently in `.sync-state.json`:

1. **`VirtusLab/scala-skill`** — source for the `scala/` and `softwaremill-scala/`
   plugins. Diff at the markdown level: each tracked file maps from an upstream
   path to a local path via `upstream_scala_skill.path_mapping`.
2. **`yaes-io/yaes`** — source material for `yaes-scala/`. yaes ships no skill
   files; chapters are hand-derived from its README and source. Detect a
   change in the upstream HEAD SHA, summarise what's new, and propose targeted
   updates to the affected `yaes-scala/skills/yaes-scala/*.md` files.
3. **The Typelevel projects** — source material for `cats-effect-scala/`.
   There is no single repo, so drift is tracked by **released library
   version** in `cats_effect.last_synced_versions`, not by commit SHA.

---

## Workflow

### Step 0 — Read state

Read `.sync-state.json`. Cache:
- `upstream_scala_skill.last_synced_sha`, `path_mapping`
- `yaes.last_synced_sha`, `derived_files`, `watched_paths`
- `cats_effect.last_synced_versions`, `sources`, `derived_files`

### Step 1 — Git remote setup (idempotent)

Confirm git remotes are wired:

```sh
git remote -v
```

Expected:
- `origin` → the user's fork (e.g. `weihsiu/scala-skill`)
- `upstream` → `VirtusLab/scala-skill.git`

If `origin` still points at `VirtusLab/scala-skill`:
> Tell the user — do not silently retarget. Recommend:
> ```sh
> git remote rename origin upstream
> git remote add origin git@github.com:<their-fork>/scala-skill.git
> ```
> Then stop and let them confirm before proceeding.

If `upstream` is missing but `origin` is theirs:
```sh
git remote add upstream https://github.com/VirtusLab/scala-skill.git
```

### Step 2 — VirtusLab fetch + per-file diff

```sh
git fetch upstream master
UPSTREAM_HEAD=$(git rev-parse upstream/master)
LAST=$(jq -r '.upstream_scala_skill.last_synced_sha' .sync-state.json)
```

If `UPSTREAM_HEAD == LAST`, report "scala-skill upstream is unchanged since
`$LAST`" and skip to Step 5.

Otherwise, list changed files in the upstream skill tree:

```sh
git diff --name-status "$LAST..$UPSTREAM_HEAD" -- \
  direct-style-scala/skills/direct-style-scala/
```

For each line:

- **M (modified)** — look up the local path via `path_mapping`. If present,
  fetch upstream's diff for that file (`git show $UPSTREAM_HEAD:<path>` vs
  the version at `$LAST`) and present BOTH the upstream diff AND the current
  local file content. Ask: "Apply this upstream change?"
  - If the user says yes, propose specific `Edit` calls that merge the
    upstream change into the local file. Preserve local structural changes
    (renamed sections, different intro paragraphs, different example
    framing). Avoid wholesale `Write` overwrites.
  - If the user says no, skip the file and continue.

- **A (added)** — present the new upstream file. Ask:
  - "Which plugin does this belong to: `scala`, `softwaremill-scala`, or skip?"
  - On choice, fetch the file content
    (`git show $UPSTREAM_HEAD:direct-style-scala/skills/direct-style-scala/<file>`),
    write it to the new local path, and add an entry to
    `upstream_scala_skill.path_mapping`.

- **D (deleted)** — present the local file that would be removed. Ask
  "Delete? (Removing it from `path_mapping` is required either way to mark it
  no longer tracked.)" Apply the user's choice.

- **R (renamed)** — surface as a paired deletion + addition; let the user
  confirm both halves.

After processing all files, the new `path_mapping` and the new
`UPSTREAM_HEAD` will be written in Step 6.

### Step 3 — VirtusLab CONTRIBUTING.md and README.md

These were not preserved with their original semantics — `CONTRIBUTING.md`
now lives at `softwaremill-scala/skills/softwaremill-scala/CONTRIBUTING.md`
and the original `README.md` was replaced. Still surface upstream changes if
present (use the mapping for CONTRIBUTING; for README, show the upstream diff
and let the user merge anything worth keeping into the root `README.md` or
the per-plugin README).

### Step 4 — yaes fetch + summarise + propose

Fetch yaes HEAD SHA from GitHub (no local clone needed):

Use the WebFetch tool on:
- `https://api.github.com/repos/yaes-io/yaes/commits/main` → extract `sha`
- `https://raw.githubusercontent.com/yaes-io/yaes/main/README.md` → full README

```sh
LAST_YAES=$(jq -r '.yaes.last_synced_sha' .sync-state.json)
```

If new SHA `== LAST_YAES`, report "yaes is unchanged" and skip.

Otherwise:

1. Fetch the diff URL or use the GitHub compare API:
   `https://api.github.com/repos/yaes-io/yaes/compare/$LAST_YAES...<new sha>`
   and read which paths changed.
2. Filter to `watched_paths` in `.sync-state.json`. If nothing in
   `watched_paths` changed, report "no material yaes changes for our derived
   chapters" and update only `last_synced_sha` + `last_synced_date`.
3. Otherwise, summarise the upstream change to the user: what effect / module
   / API was added or modified. For each item in `derived_files`, decide
   whether it is affected. Propose targeted edits (new section, new code
   example, updated import path) and let the user confirm one by one.

**Do NOT** wholesale-overwrite the yaes chapters. They are hand-written; the
user's edits matter.

### Step 4b — Typelevel library versions

`cats-effect-scala/` has no single upstream repo. Check the current released
version of each library in `cats_effect.last_synced_versions` and compare
against the recorded value.

Look them up with WebFetch against Scaladex (or the project's own docs):

- `https://index.scala-lang.org/typelevel/cats-effect`
- `https://index.scala-lang.org/http4s/http4s` (use the stable `0.23.x` line,
  NOT the `1.0.0-Mxx` milestones)
- `https://index.scala-lang.org/typelevel/fs2`
- `https://index.scala-lang.org/circe/circe`
- `https://index.scala-lang.org/typelevel/doobie`
- `https://index.scala-lang.org/typelevel/log4cats`
- `https://index.scala-lang.org/typelevel/otel4s`

If nothing moved, report "Typelevel stack is unchanged" and skip.

For each library that advanced:

1. Fetch its release notes / changelog and summarise what changed.
2. Decide whether the change is **cosmetic** (a patch bump — only the version
   string in `110-project-setup.md` and the chapter that names it) or
   **material** (a renamed package, a changed API, a new capability worth
   documenting).
3. For a version-only bump, propose the `Edit` calls that update the
   dependency lines. Note that the version appears in more than one chapter —
   grep for the old version string rather than assuming a single site:
   ```sh
   grep -rn "3\.7\.0" cats-effect-scala/ || true
   ```
4. For a material change, present it and propose targeted chapter edits, one
   at a time, exactly as for yaes.

> Watch for groupId/package migrations specifically — doobie's move from
> `org.tpolecat` to `org.typelevel` in `1.0.0-RC13` is the kind of change that
> silently invalidates every code example in a chapter.

**Do NOT** wholesale-overwrite the cats-effect chapters. They are hand-written.

### Step 5 — Cross-link sanity

Run:

```sh
grep -rE '\.\./|direct-style-scala/' --include='*.md' \
  scala/ softwaremill-scala/ yaes-scala/ cats-effect-scala/ || true
```

Any hit (other than the deliberate `../../scala/skills/scala/SKILL.md`
prerequisite reference in the three non-`scala` SKILL.mds) is a sign that an
upstream edit reintroduced a stale path. Surface it.

### Step 6 — Write `.sync-state.json` and stage

If the user approved any changes:

```sh
jq --arg sha "$UPSTREAM_HEAD" --arg date "$(date -u +%Y-%m-%d)" \
  '.upstream_scala_skill.last_synced_sha = $sha
   | .upstream_scala_skill.last_synced_date = $date' \
  .sync-state.json > .sync-state.json.tmp && mv .sync-state.json.tmp .sync-state.json
```

(Equivalent for the `yaes` section if it advanced. For `cats_effect`, update
the individual entries under `last_synced_versions` that moved, plus
`last_synced_date`.)

Then stage everything modified for the user to review:

```sh
git status --short
git add -p   # if interactive
```

**Do not commit.** End with: "Sync complete. Review with `git diff --cached`
and commit when ready."

---

## Constraints

- **Never** apply an upstream edit without explicit user confirmation.
- **Never** overwrite a local file with `Write` when an `Edit` would
  preserve local divergence. Reach for `Write` only when adding a brand-new
  file.
- **Never** edit `.sync-state.json` outside this command's Step 6.
- **Never** commit on the user's behalf — they review and commit.
- If the user aborts mid-flow, leave `.sync-state.json` untouched. The next
  run will start over from `last_synced_sha`.
- If a sync run results in zero approved changes, still update
  `last_synced_date` (and the SHAs, since the comparison was made) so the
  next run starts from the most recent comparison point.
