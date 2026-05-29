---
description: Bring scala-skill into sync with VirtusLab/scala-skill and rcardin/yaes upstreams. Updates `.sync-state.json` on completion.
---

You are running the **upstream sync workflow** for this repo. Your job is to
detect changes in two upstream sources, present each diff to the user, apply
the ones they approve, and update `.sync-state.json` at the end. Do **not**
auto-merge anything.

There are two upstreams, tracked independently in `.sync-state.json`:

1. **`VirtusLab/scala-skill`** — source for the `scala/` and `softwaremill-scala/`
   plugins. Diff at the markdown level: each tracked file maps from an upstream
   path to a local path via `upstream_scala_skill.path_mapping`.
2. **`rcardin/yaes`** — source material for `yaes-scala/`. yaes ships no skill
   files; chapters are hand-derived from its README and source. Detect a
   change in the upstream HEAD SHA, summarise what's new, and propose targeted
   updates to the affected `yaes-scala/skills/yaes-scala/*.md` files.

---

## Workflow

### Step 0 — Read state

Read `.sync-state.json`. Cache:
- `upstream_scala_skill.last_synced_sha`, `path_mapping`
- `yaes.last_synced_sha`, `derived_files`, `watched_paths`

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
- `https://api.github.com/repos/rcardin/yaes/commits/main` → extract `sha`
- `https://raw.githubusercontent.com/rcardin/yaes/main/README.md` → full README

```sh
LAST_YAES=$(jq -r '.yaes.last_synced_sha' .sync-state.json)
```

If new SHA `== LAST_YAES`, report "yaes is unchanged" and skip.

Otherwise:

1. Fetch the diff URL or use the GitHub compare API:
   `https://api.github.com/repos/rcardin/yaes/compare/$LAST_YAES...<new sha>`
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

### Step 5 — Cross-link sanity

Run:

```sh
grep -rE '\.\./|direct-style-scala/' --include='*.md' \
  scala/ softwaremill-scala/ yaes-scala/ || true
```

Any hit (other than the deliberate `../../scala/skills/scala/SKILL.md`
prerequisite reference in the two non-`scala` SKILL.mds) is a sign that an
upstream edit reintroduced a stale path. Surface it.

### Step 6 — Write `.sync-state.json` and stage

If the user approved any changes:

```sh
jq --arg sha "$UPSTREAM_HEAD" --arg date "$(date -u +%Y-%m-%d)" \
  '.upstream_scala_skill.last_synced_sha = $sha
   | .upstream_scala_skill.last_synced_date = $date' \
  .sync-state.json > .sync-state.json.tmp && mv .sync-state.json.tmp .sync-state.json
```

(Equivalent for the `yaes` section if it advanced.)

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
