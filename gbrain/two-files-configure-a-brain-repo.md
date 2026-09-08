# Two files configure a brain repo

A GBrain data repo needs exactly two files at its root, and they are not the
two you'd guess:

```
<brain-repo>/
├── .gitignore
├── gbrain.yml
├── people/
├── companies/
└── notes/
```

`gbrain.yml` is the one gbrain parses. `.gitignore` is the one that decides
what gets indexed. Those are different jobs, done by different files, and the
second one is not obvious at all — `.gitignore` looks like it's there for git's
benefit and happens to sit in the same directory. It isn't. It's the only
exclusion mechanism gbrain has.

Everything below is checked against
[garrytan/gbrain](https://github.com/garrytan/gbrain) **0.48.5.0**, at master
commit `43597b19e`. Where behavior matters I've named the file that implements
it, so you can re-check it against whatever version you're on — this is a
young tool and these are young features.

## The short version

| File | Who reads it | What it controls |
|---|---|---|
| `gbrain.yml` | gbrain | Storage tiers. `db_only` keeps directories out of git while the database keeps the pages |
| `.gitignore` | git — and *therefore* gbrain | Everything that does or doesn't get indexed |

If you only take one thing: **`.gitignore` is gbrain's ignore file.** There is
no second one, and the plausible-sounding candidate doesn't exist — see the
note at the end before you go looking for it.

## `gbrain.yml` — the only file gbrain parses itself

`gbrain.yml` lives at the brain repo root and holds sections keyed by feature.
Two exist today: `storage:` (tiering) and `archive-crawler:` (an explicit
allow-list of directories the archive-crawler skill may scan — the skill
refuses to run without one, deliberately, so an agent can't over-scope a scan
into your tax PDFs). A brain repo that never crawls archives needs only the
first.

A starter `storage:` section:

```yaml
storage:
  # Version-controlled: human-curated pages you edit by hand.
  db_tracked:
    - people/
    - companies/
    - projects/
    - notes/

  # Database-only: bulk machine-generated content. Written to disk as a local
  # cache, kept out of git, restorable with `gbrain export --restore-only`.
  db_only:
    - conversations/
    - media/articles/
```

### `db_only` is the half that carries force

`db_only` is the load-bearing key. It drives the auto-managed `.gitignore`
block, it is passed as a `git add` pathspec exclusion when gbrain creates a
baseline commit, and it tells the `undeclared_db_only_pages` doctor check
which DB pages are *supposed* to have no file on disk.

`db_tracked` is, as far as the current code is concerned, documentation. Grep
its consumers and you find `gbrain storage status` — page counts, disk usage,
the tier listing — and the tier classifier. Nothing excludes, nothing writes,
nothing warns on its behalf. Declaring it is still worth doing: it makes
`storage status` legible and it states the intent for the next person. Just
don't expect it to enforce anything.

The one rule that spans both: a directory cannot appear in both tiers. That
throws a `StorageConfigError` rather than picking a winner, which is the right
call — ambiguous routing for a page is not a thing you want resolved silently.

### The parser is deliberately narrow, and that has a sharp edge

`src/core/storage-config.ts` does not use a YAML library. It is a hand-written
line scanner that understands exactly the shape gbrain controls: a top-level
`storage:` key, indented list keys under it, and block-style `- item` entries.
The comment explaining why is worth reading — the predecessor used
`gray-matter`, which silently returned `{data: {}}` on delimiter-less YAML and
broke the feature on every install.

The trade-off is that valid YAML gbrain doesn't recognize resolves to nothing,
quietly. Flow style is the one you'll actually type by accident:

```yaml
storage:
  db_only: [conversations/]   # valid YAML — silently resolves to zero directories
```

That config is not rejected. It parses to an empty tier, and every behavior
that depends on `db_only` simply doesn't happen. One code path guards against
it: when gbrain is about to create a sync baseline commit, it checks whether
the file *mentions* a `db_only:` key while resolving no directories, and
refuses rather than commit content that was meant to stay out of git. That
guard doesn't cover the ordinary sync path. Write block style.

Two smaller behaviors, so they don't surprise you:

- **Trailing slashes are canonical.** `notes` is auto-normalized to `notes/`
  with a one-time note on stderr. Write the slash and the note goes away. The
  matcher is segment-aware either way — `media/x/` matches `media/x/foo` and
  not `media/xerox/foo`.
- **`git_tracked` / `supabase_only` still load.** They're the pre-v0.22.11
  names, deprecated, mapped to the canonical keys with a warning. If both old
  and new appear, canonical wins and the old ones are ignored. Rename them.

## `.gitignore` — git's file, and gbrain's too

Here is the connection that gives `.gitignore` its second job. When gbrain
enumerates files to index, and the directory is a git work tree, it does not
walk the filesystem. It shells out:

```
git ls-files --cached --others --exclude-standard -z
```

Tracked files plus untracked-but-not-ignored files. `--exclude-standard` means
git applies `.gitignore`, `.git/info/exclude`, and your global excludes file.
Every one of those becomes an input to what lands in your brain.

This started as a cost fix rather than a design. The old FS walk descended into
`vendor/`, `storage/`, `public/build/` — a Laravel repo's code sync would try
to import ~50k dependency files — and swapping the walk for `git ls-files`
fixed that by inheriting git's exclusions wholesale. The ignore mechanism the
tool never separately grew is the one it borrowed.

The consequence runs in both directions:

- **Good:** `.gitignore` is a real, familiar, expressive exclusion file for
  your brain. Patterns you already know work, and `git check-ignore` debugs
  them.
- **Sharp:** gitignoring a directory you *wanted* indexed removes it from the
  brain silently. No error, no warning, just fewer pages than you expected.
  `gbrain sync` and `gbrain import` both take `--include-gitignored` when you
  want the walk to ignore the ignore file.

gbrain warns about exactly one instance of that trap: if a configured collector
writes its output into a `db_only` directory, that directory is auto-gitignored,
so the collector runs green while nothing reaches the DB. You get a warning at
sync time and a `db_only_collector_collision` doctor check. Everywhere else,
the silence is on you.

### The auto-managed block

Declare `db_only` directories and `gbrain sync` appends them to `.gitignore`:

```gitignore
# Auto-managed by gbrain (db_only directories)
conversations/
media/articles/
```

Worth knowing about that block:

- It is written **only after a successful sync**, never before the first
  import. The ordering is deliberate, and it's the reason step 2 below tells
  you to leave those lines out of your hand-written file: `.gitignore` gates
  what gets indexed, so writing `db_only` entries before the first import would
  exclude those pages from the database entirely rather than merely from git.
- Skipped on `--dry-run`, skipped when the sync is `blocked_by_failures`,
  skipped when the repo is a git submodule (a submodule's `.gitignore` changes
  don't survive parent updates), and skipped entirely under
  `GBRAIN_NO_GITIGNORE=1` — the escape hatch for shared repos where a
  maintainer wants gbrain to keep its hands off.
- Deduplication is an exact line match on `dir` or `/dir`. Add a second
  `db_only` directory six months later and you get a *second* header block
  appended below the first. Cosmetic, idempotent, slightly untidy.
- Write failures are caught and logged with the lines you need to add by hand.
  They never crash a sync.

A reasonable hand-written `.gitignore` for a brain repo, before gbrain adds its
block:

```gitignore
# Local-only working files — drafts, scratch, anything ending .local
*.local
*.local.*

# Binary attachments cached on disk, not carried in git history
/_files

# Editor and OS noise
.idea
.DS_Store
```

## What gbrain skips no matter what you write

Before either config file matters, a hardcoded gate in `src/core/sync.ts`
prunes paths during descent, and a matching path-level check applies on the
`git ls-files` fast path. Both routes funnel through the same helper
specifically so full and incremental sync can't drift apart. Skipped:

- **Any path segment starting with `.`** — files as well as directories, so
  `.claude/`, `.agents/`, and `.gitignore` itself are all out
- `node_modules`, `vendor`, `dist`, `build`, `venv`
- `.raw` and any `*.raw` directory (the sidecar convention: `people/pedro.raw/`
  holds raw source for `pedro.md`)
- Git submodules, detected by `.git` being a *file* rather than a directory
- Symlinks, always — never followed
- Filenames containing brackets or control characters

A small list of filenames is treated as directory scaffolding rather than brain
pages, on every route: `README.md`, `index.md`, `log.md`, `schema.md`,
`RESOLVER.md`. Put a `README.md` in each content directory to orient a human
reader; it will never become a page.

And under the default `markdown` strategy only `.md` files are collected at
all — which means most of what you'd instinctively reach for an ignore file to
exclude (configs, lockfiles, images) was never a candidate.

## Wiring it together

For a fresh brain repo:

1. Write `gbrain.yml` with block-style `db_tracked` and `db_only` lists,
   trailing slashes on every path, no directory in both.
2. Write `.gitignore` by hand for local-only and editor noise. Leave the
   `db_only` directories out — let gbrain add them after the first successful
   sync, for the ordering reason above.
3. Run `gbrain sync`, then check your work:

```bash
gbrain storage status --repo <brain-repo>
```

Page counts and disk usage by tier, missing files that need restoring, and
config warnings. If a directory you expected is missing from the brain, ask git
why before you ask gbrain:

```bash
git -C <brain-repo> check-ignore -v path/to/file.md
```

That prints the exact ignore file and line responsible. Nine times in ten the
answer is a `.gitignore` pattern doing precisely what it says — which is the
whole point. The ignore file you have to reason about is the one you already
knew how to read.

## A note on `.gbrainignore`

You may reach for a third file. Don't: **`.gbrainignore` is not implemented.**

Grep the whole tree at master `43597b19e`, `node_modules` included, and the
name appears only in prose — a design-doc backlog entry and one source comment:

```
docs/designs/COMMUNITY_IDEAS.md:278:- **`.gbrainignore` / per-repo exclusion** (#1483 @eepaul; …)
docs/designs/COMMUNITY_IDEAS.md:281:  … gitignore-parity `.gbrainignore` + per-source
src/commands/import.ts:1201:  // v0.42.x (#1159 --respect-gitignore / #1483 .gbrainignore): when `dir` is a
```

No loader, no parser, no call site. The file is never opened. It isn't hiding
behind a constructed filename either — `brainignore` as a bare substring, and
the usual handles (`IGNORE_FILE`, `ignoreFile`, `loadIgnore`, `parseIgnore`),
all come back empty.

What happened is easy to reconstruct and entirely reasonable. Three issues
asked for the same thing from different angles: a `.gbrainignore` (#1483),
repo-local code filters (#1011), and `--respect-gitignore` (#1159). The
maintainer shipped the third — the `git ls-files` swap this whole write-up
rests on — which solved most of the pain for most people, and credited all
three in the comment. The named file never got built, and COMMUNITY_IDEAS.md
still lists it as open, high priority.

That leaves a trap with no warning attached. A file called `.gbrainignore` at
the root of a brain repo is a convincing artifact: right name, gitignore-shaped
contents, and nothing about it announcing that it's a wish rather than a config.
It sits there being obeyed by nobody while you assume it explains why something
isn't indexed. The actual explanation is always the `.gitignore` next to it.

If you have one, fold its entries into `.gitignore` and delete it. For
exclusions you don't want committed, `.git/info/exclude` gets the same
`--exclude-standard` treatment and stays local. For one-off runs, `gbrain sync`
and `gbrain import` accept `--exclude <glob>`.

Before trusting any of this, given the version churn, one command settles it
against whatever revision you have:

```bash
git -C <gbrain-clone> grep -n "brainignore" master
```

Prose hits in `docs/` and a comment in `src/commands/import.ts` mean it's still
unimplemented. A non-comment hit under `src/` means it shipped, and this
section is out of date.

---

*Verified against garrytan/gbrain 0.48.5.0, master commit `43597b19e`
(2026-09-08). Implementation lives in `src/core/storage-config.ts`,
`src/core/sync.ts`, `src/core/sync-git.ts`, `src/commands/sync.ts`, and
`src/commands/import.ts`. Claims are about master; an unmerged PR could change
any of this.*
