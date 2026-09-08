# When `gbrain upgrade` becomes safe again

I have a standing rule against running `gbrain upgrade` —
[never run it](../.agents/rules/no-gbrain-upgrade.md) — and the reasoning
behind it is in
[Upgrading a source-linked CLI](upgrading-a-source-linked-cli.md). The rule is
real. But it is worth being precise about what it prohibits, because the
prohibition was never about the command. It was about the branch.

The `bun-link` path in `src/commands/upgrade.ts` opens with
`git pull --ff-only` in the repo root. A fast-forward is impossible on a
branch carrying commits upstream does not have, so on such a branch the pull
fails, the tool prints a fallback telling me to go do it by hand, and I end up
doing the pull myself anyway — except now in the middle of a command that had
already decided the pull was its business.

Put the checkout on a branch that carries nothing of mine and the objection
evaporates. The pull becomes a genuine fast-forward. The tool's own path fits
again.

That is the case I expect to be in once my open PRs land upstream and a
release ships carrying them. My changes arrive *through* upstream rather than
sitting on top of it, and there is nothing left on `master` for a fast-forward
to trip over.

## The procedure

**Precondition:** every change of mine that this upgrade should carry is
already merged upstream and present in the release I am pulling. Anything
still unmerged lives on its own topic branch and is not part of this.

Start on `master`:

```bash
cd ~/gbrain && git checkout master
```

Confirm it is clean and tracks upstream:

```bash
git status -sb
```

Expect `## master...upstream/master` with no `[ahead N]` and no file lines
under it. The tracking target matters as much as the cleanliness: `master`
tracks `upstream/master` — `garrytan/gbrain` — not `origin/master`, which is
my fork. The pull inside `gbrain upgrade` takes no arguments, so it follows
that tracking config and goes straight to upstream. My fork never enters it.

Confirm `master` carries nothing I wrote:

```bash
git log --oneline upstream/master..master
```

Expect no output. `upstream/master` is a cached ref, only as fresh as my last
fetch, so this proves `master` carries nothing of mine *as of that snapshot* —
not that upstream has stood still since. That is the right question anyway:
upstream having moved is the reason to upgrade.

Confirm the tree sits at the version I am running **now**, not the one I am
upgrading to:

```bash
gbrain --version      # e.g. gbrain 0.48.5.0
git log --oneline -1  # the v0.48.5.0 commit
```

If those disagree — I pulled ahead by hand, or a checkout left me somewhere
else — reset to the tag for the version I am running. Release tags are already
in the clone, and the tag for my current version is there by definition:

```bash
git reset --hard v0.48.5.0
```

> `git reset --hard` throws away every uncommitted change in the working tree
> and drops any commit on `master` not reachable from another ref. Run
> `git status --porcelain` first and expect it silent. On a source-linked
> install that working tree *is* my live CLI's source, so "uncommitted
> changes" and "edits I made to the tool" are the same thing. Committed work
> stays recoverable through `git reflog`; uncommitted work does not.

Then, in Terminal:

```bash
gbrain upgrade
```

The first two lines to expect:

```
Detected install method: bun-link
Upgrading bun-link source clone at /Users/photon/gbrain...
```

From there it runs `git pull --ff-only` and `bun install` in the clone,
re-reads the version by shelling out to `gbrain --version`, records the
transition in `~/.gbrain/upgrade-state.json`, then runs `gbrain post-upgrade`
and `gbrain features`.

Afterwards my topic branches are behind a moved `master`. Rebasing them is a
separate decision on a separate day, and nothing above touches them.

## What "bun-link" is, and what it isn't

The printed line names how the install was *made* — `git clone` plus
`bun link` — rather than what the upgrade is about to do. Under `bun-link`
there is no binary to swap. The git pull *is* the swap: `~/.bun/bin/gbrain`
resolves through `~/.bun/install/global/node_modules/gbrain` to
`/Users/photon/gbrain` and runs `src/cli.ts` directly, so new commits *are*
the new version the moment they land.

Detection is thinner than it looks. `detectBunLink()` walks up from `argv[1]`
looking for a `.git/config` whose text contains `garrytan/gbrain`,
case-insensitively — and stops at the first `.git/config` it finds, matching
or not. On my fork `origin` is `LongPV/gbrain`; the substring that makes this
install detectable comes entirely from the `upstream` remote.

Which is a booby trap for exactly the moment this document describes. Once my
PRs are merged, tidying up by removing the now-redundant `upstream` remote
would make `detectInstallMethod()` fall past `bun-link`, past the
`node_modules` check (Bun resolves the symlink chain before setting `argv[1]`,
so the path it sees is the plain clone), past the compiled-binary check, and
land on `unknown` — whose advice is to run `bun update gbrain`. Following that
would install the published package over a source clone. The remote that looks
vestigial is load-bearing.

## Why the current version, and not the new one

Resetting to the version I am *running* rather than the one I *want* is the
step most likely to look backwards, so it is worth saying why.

`runUpgrade` captures `oldVersion` from the constant compiled into the code
that is executing — the old tree, read before the pull. After the pull it
shells out to `gbrain --version`, a fresh process reading the new tree, and
saves the pair as `last_upgrade` in `~/.gbrain/upgrade-state.json`.

Pull by hand first and both reads return the same string. The pull inside the
command reports "Already up to date", `upgraded` is still set — `execFileSync`
exited zero — and the rest of the chain runs over a transition that never
happened.

The cost is narrower than that sounds, and precision matters more than alarm
here. `gbrain post-upgrade` invokes `apply-migrations --yes`
**unconditionally**; the source comments say so and name an older bug where a
missing state file made it early-return and left broken installs broken. The
mechanical side still executes. The from/to pair drives only the feature-pitch
headlines — the "here is what changed in the versions you just crossed" print.

So pre-pulling does not leave the brain unmigrated. It makes the upgrade
*silent* about what changed. Which is precisely the part I keep a fork and a
manual pull in order to read.

## The rule still stands — I run this, not an agent

This document reads like permission, so let me be plain: nothing here repeals
[the rule](../.agents/rules/no-gbrain-upgrade.md). I type `gbrain upgrade`
into a terminal myself. No agent runs it on my behalf, and no agent decides
that `master` looks clean enough to qualify.

The split is the whole point. The precondition — "`master` carries nothing of
mine" — is cheap for me to check and easy for an agent to get wrong in the one
direction that costs something: `git status` came back clean, so it proceeded,
and the pull it thereby triggered was the decision I had reserved for myself.
An agent can read the state and tell me whether the preconditions hold. Acting
on that reading stays mine.

## What this generalizes to

A blanket prohibition is usually a condition somebody got tired of restating.
"Never run `gbrain upgrade`" is really "never run it on a branch carrying
local commits" — but the short form is the one that survives contact with a
hurried reader, so the short form is what I wrote down, deliberately.

The cost of the short form is that it goes stale invisibly. The world moves —
PRs merge, the branch stops being special — and the rule keeps prohibiting
something that has stopped being a problem, with nothing in its own text to
say so. A bare prohibition cannot tell you when it has expired, because the
condition that justified it was never written into it.

The fix is not to loosen the rule. It is to write the condition down beside
it, and leave the prohibition as the thing a reader lands on by default. The
exception should cost a deliberate read; the default should cost nothing.
