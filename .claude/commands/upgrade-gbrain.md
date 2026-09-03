---
description: Upgrade the source-linked gbrain CLI after you have pulled — bun install, apply migrations, then read the post-upgrade and migration notes
allowed-tools: Bash(cd:*), Bash(gbrain:*), Bash(bun install:*), Bash(ls:*), Bash(cat:*), Bash(wc:*), Bash(sed:*), Bash(sort:*), Bash(printf:*), Read
---

# Upgrade gbrain

`gbrain` on this machine is installed **from source**: `~/.bun/bin/gbrain` is a
`bun link` symlink into the clone at `~/gbrain`, which
runs `src/cli.ts` directly under Bun. There is no build step and no binary to
self-update.

**The user pulls the code themselves.** By the time this command runs, the clone is
already at the version they want.

## The version cannot move during this command

`gbrain --version` reads `VERSION` / `package.json` **out of the working tree**. The
user's pull already updated those files before this command started, and `bun install`
never rewrites them, so reading it before and after the chain returns the same string.
Read it once.

The question worth answering is *which migration notes have not been handled yet*. Two
sources answer it, and the first is authoritative:

- `~/.gbrain/migrations/pending-host-work.jsonl` — entries gbrain's own migrations queue
  when they need agent action, each naming its note file and often the exact remediation
  command. `src/commands/migrations/index.ts` states the contract: the markdown notes are
  read on demand when this file is non-empty. It is append-only — nothing prunes an entry
  — so treat it as a checklist to confirm against, not as a delta.
- A version range over `~/gbrain/skills/migrations/`, bounded by:
  - `NOW` — the version in the clone right now.
  - `LAST` — the version at the end of the last successful run of this command, kept in
    `~/.gbrain/claude-upgrade-stamp`. That directory's `.gitignore` is `*`, so the stamp
    never lands in a repo.

## Steps

### 1. Baseline, current version, outstanding host work

```bash
printf 'LAST=%s\nNOW=%s\nLEDGER_START=%d\n' "$(cat ~/.gbrain/claude-upgrade-stamp 2>/dev/null || sed -n 's/.*"from": *"\([^"]*\)".*/\1/p' ~/.gbrain/upgrade-state.json 2>/dev/null)" "$(gbrain --version | sed 's/^gbrain //')" "$(wc -l < ~/.gbrain/migrations/completed.jsonl)"; cat ~/.gbrain/migrations/pending-host-work.jsonl 2>/dev/null
```

- `LAST` is the stamp, falling back to `.last_upgrade.from` in `upgrade-state.json`
  (`from`, not `to` — the wider range; re-reading a handled note is free, missing one is
  not). An empty `LAST` means no baseline: say so, and rely on the host-work queue plus
  any note matching `NOW` exactly.
- Every `skill` path in the host-work output is a note to read in step 4, whatever the
  version range says.

### 2. Run the upgrade

Run this exactly as one chain — `&&` fail-fast is intentional:

```bash
cd ~/gbrain && bun install && gbrain apply-migrations --yes && gbrain post-upgrade
```

### 3. Which migrations applied

```bash
sed -n '<LEDGER_START + 1>,$s/.*"version":"\([^"]*\)".*/\1/p' ~/.gbrain/migrations/completed.jsonl
```

`~/.gbrain/migrations/completed.jsonl` is the ledger `apply-migrations` appends to.
Slicing from `LEDGER_START` yields the versions that applied this run; empty output means
everything was already applied. Trust this over the stdout wording — "All migrations up
to date." is printed by `bun install`'s postinstall, by `apply-migrations`, and again by
`post-upgrade`, so three identical lines are one fact, not three.

### 4. Migration notes to read

```bash
printf '%s\n' "<LAST>" "<NOW>" $(ls ~/gbrain/skills/migrations/ | sed 's/^v//;s/\.md$//') | sort -V
```

`sort -V` does the version comparison — do not compare these by eye. Filenames mix three-
and four-component forms (`v0.40.5.md`, `v0.46.35.0.md`), so lexical order is wrong and
`0.46.35.0` does not sit next to `0.46.3.0`. `Read` every note from the `LAST` line down
to the `NOW` line, skipping any line equal to `LAST` — the two sentinels are printed into
the list, so a version appearing twice is a sentinel beside a real note at that version.
Read every note named by step 1's host-work queue as well, in or out of range.

Notes are **sparse** — most releases have none, and `LAST` itself is excluded because its
notes were handled on a previous run. Summarize only the steps that still need doing:
backfills, verification commands, config changes.

### 5. Stamp the baseline

Only when the chain in step 2 succeeded:

```bash
gbrain --version | sed 's/^gbrain //' > ~/.gbrain/claude-upgrade-stamp
```

Skip this if the chain failed. A failed run must not advance the baseline, or the next
run silently skips every note in between.

### 6. Report

State plainly:
- `LAST` → `NOW`, or that `NOW` matches `LAST` so this pull carried no version bump
- which migration versions the ledger gained, or that it gained none
- what `post-upgrade` said beyond the migration line
- the actionable items from the notes read, or that there were none

## Rules

- **Never run `git pull` or `git fetch`.** The user handles pulling. If `NOW` equals
  `LAST`, say the pull carried no version bump so they can check whether it landed —
  never pull to "fix" it.
- **Never run `gbrain upgrade`.** Not because it skips this install — `src/commands/upgrade.ts`
  has a first-class `bun-link` branch — but because that branch opens with
  `git pull --ff-only`, which this command must not do.
- **gbrain records an upgrade baseline only inside `gbrain upgrade`**, so
  `upgrade-state.json` goes stale on a pull-only workflow and serves here only as a
  fallback. Until `runPostUpgrade` (`src/commands/upgrade.ts`) stamps one itself, this
  command keeps its own.
- **If the chain fails**, stop, name the step that failed, and do not write the stamp.
  Re-run just that step to get clean output before diagnosing. Do not retry the whole
  chain blindly.
- **If `post-upgrade` prints `[AGENT]` markers or a banner asking for an operator
  decision** (e.g. the v0.32.3 search-mode cost matrix), do not move past it. Show it
  to the user verbatim, ask which option they want, and only then run the config
  change they choose.
- **No migration notes in range is a normal result**, not an error — but check the
  host-work queue before saying so. Say so and move on.
- Do not run `gbrain doctor` unless the upgrade output gives a reason to, or the user
  asks.
