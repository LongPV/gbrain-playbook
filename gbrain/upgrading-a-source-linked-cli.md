# Upgrading a source-linked CLI

I run gbrain from a clone rather than from a release. `~/.bun/bin/gbrain` is a
`bun link` symlink into that clone, and it runs `src/cli.ts` directly under
Bun — no build step, no binary, nothing to reissue. I set it up that way for
one reason: I wanted to change the tool. The clone is a fork, with my own repo
as `origin`, the upstream project as a second remote, and my own commits on my
own branch. Edits are live the moment I save them.

Everything awkward about upgrading follows from that single choice. The slash
command I eventually wrote —
[.claude/commands/upgrade-gbrain.md](../.claude/commands/upgrade-gbrain.md) —
is really a list of answers to problems I created for myself by forking.

## Customizing is what makes upgrading hard

A tool you only configure can update itself. A tool you fork cannot, and the
break shows up in two places.

First, the pull becomes mine. Pulling into a tree that carries my own commits
is a decision — about when my changes meet upstream's, and what happens if
they disagree — and it is not a decision I want made on my behalf while I am
looking elsewhere. So nothing in my upgrade automation is allowed to run
`git pull` or `git fetch`. That is the first rule in the command file, and it
is a rule about ownership rather than about safety.

Second, the tool's own upgrade path stops fitting. gbrain does recognize this
install method — there is a first-class `bun-link` branch in
`src/commands/upgrade.ts` — but that branch opens by running
`git pull --ff-only` in the repo root. It pulls from whatever the branch
tracks, and it refuses anything that is not a fast-forward, which is precisely
what a branch carrying local commits becomes once upstream moves. The code
half-concedes the point: when that pull fails, it prints a fallback telling me
to go and do it by hand.

So `gbrain upgrade` is off the table — not because it is broken, but because
its first action is the one action I reserved for myself.

That prohibition is now written down as a rule of its own —
[agent-rules/no-gbrain-upgrade.md](../agent-rules/no-gbrain-upgrade.md) —
linked from `AGENTS.md` and `CLAUDE.md`, the files an agent reads before
touching this repo. The rule states what not to run, why, and what to do
instead, placed where an agent meets it before it can be helpfully wrong.

I want to be plain about what that is: documentation, not enforcement. No
mechanism stops the command from running — the rule works only if it is read
and obeyed. A pre-flight hook could make it mechanical; I chose prose because
the failure I am defending against is habit and helpfulness, not defiance,
and because a rule written as markdown travels with the repo to any agent
that can read, not just the ones that honor one vendor's settings file.

What remains is the part the tool would have done *after* pulling: install
dependencies, apply migrations, and work out what the new version now expects
of me that it cannot do on its own. That reconciliation is the real work.

## The upgrade itself is one line

By the time the command runs, I have already pulled, so the clone is at the
version I want and the mechanical part is short:

```bash
cd ~/gbrain && bun install && gbrain apply-migrations --yes && gbrain post-upgrade
```

The `&&` is deliberate. If the dependency install fails, migrations must not
run against a half-installed tree, and post-upgrade must not print success
over the top of the wreckage. When a link in that chain breaks I want to know
which one, so the rule is to stop, name the failing step, and re-run only that
step to get clean output — never to retry the whole chain and hope.

Everything else is bookkeeping around those four commands.

## Read the version once

My first instinct was to read the version before the upgrade and again after,
then report the difference. That is wrong here, and it took me a beat to see
why: the version is read out of the working tree, my pull already moved it
before the command started, and nothing in the chain rewrites it. Both reads
return the same string. I had borrowed the before-and-after framing from
package managers that do the fetching themselves.

The habit worth keeping is more general: know which of your observations can
actually change during an operation, and stop re-measuring the ones that
cannot. The interesting question was never "what version am I on" — it was
"which migration notes have I not dealt with yet."

## Keep your own baseline, and only advance it on success

Answering that means knowing where I was last time. gbrain records an upgrade
baseline only inside its own upgrade command — the one I do not run — so on a
pull-only workflow that record goes stale, and it serves me as a fallback at
best. The bookkeeping is mine to keep.

So the command keeps a stamp: one file holding the version as of the end of
the last successful run. It lives in the tool's state directory, whose
`.gitignore` is `*`, so it can never accidentally land in a repo.

Two details matter more than the file does.

**Only advance it on success.** A failed run that stamps a new baseline
silently swallows every note between the old baseline and the new one, and
nothing will ever surface them again. A stamp is a claim that the work got
done, so it has to be written where that claim is true.

**With no stamp, take the wider range.** The fallback record carries both a
`from` and a `to`; the command reads `from`. Re-reading a note I already
handled costs me a minute. Missing one costs me a silent misconfiguration I
discover weeks later. When the two error directions are that lopsided, bias
hard toward the cheap one.

## Trust the ledger, not the transcript

Which migrations actually applied? The obvious move is to read the output, and
the obvious move is wrong: "All migrations up to date." is printed by the
postinstall hook, again by `apply-migrations`, and again by `post-upgrade`.
Three identical lines in the transcript are one fact repeated by three
callers, and I have caught myself reading them as three confirmations.

The reliable answer is the ledger — an append-only file that
`apply-migrations` writes to. Count its lines before the chain, read
everything past that mark afterwards. Empty means nothing applied. That is a
fact about state rather than a claim in a log.

Prefer a machine-readable record over parsed human output whenever a tool
offers both. Stdout is written for a person watching in real time: it repeats
itself, it reassures, and it was never meant to be a transaction log.

## Two sources of outstanding work, one authoritative

Some migrations cannot finish by themselves — they need something done on the
host. gbrain queues those in a pending-host-work file, each entry naming the
note that explains it and often the exact remediation command. The source
comments are explicit that the notes are read on demand whenever that file is
non-empty.

The trap is that the file is append-only. Nothing prunes an entry once the
work is done, so it is a checklist to confirm against rather than a list of
things definitely still outstanding. Read it as a delta and you will redo
work.

The second source is a version range over the notes directory, running from my
stamp to the current version. Neither source subsumes the other: the queue
catches host work from versions I crossed long ago and never finished, and the
range catches notes from versions that queued nothing. I read the union, and I
treat a sparse result as normal — most releases ship no note at all.

## Let the tool do the version comparison

There are 38 note files in my clone as I write this, and their names mix three-
and four-component versions — `v0.40.5.md` sitting beside `v0.46.35.0.md`.
Sorted as text, that ordering is wrong in ways that look plausible right up
until they bite. `v0.22.14.md` lands before `v0.22.4.md`. Every single-digit
minor version — `0.5.0` through `0.9.1`, six of the thirty-eight — sorts
*after* `0.46.35.0`, because `4` precedes `5` one character in. Skim that
listing for "anything newer than 0.40" and you miss them all.

`sort -V` knows how to compare version numbers. I do not, reliably, at speed,
on the tenth filename.

The part I like is the sentinel trick: print the two range bounds into the
same list as the filenames, sort the whole thing, and "which notes are in
range" collapses into "read the lines between these two." No comparison logic
of my own, so nothing of my own to get subtly wrong. The lower bound is
excluded, because its notes were handled on the previous run.

## Stop at decisions that aren't yours

Some migrations do not want a command run so much as a choice made — now and
then the post-upgrade output prints a banner asking which of several options I
want, with real cost consequences either way. The rule is to show it to me
verbatim and stop, rather than pick whichever default looks sensible.

This matters more on a fork than it otherwise would. Upstream's migrations are
written against upstream's code, and mine is not quite that any more. A
migration that assumes a file it wrote is a migration that can collide with
something I changed. Automation that guesses where it should ask is fine right
up until the moment it is expensive.

## What this generalizes to

Forking a tool is cheap to start and quietly ongoing. The visible cost is
merge conflicts, and everyone prices that in. The invisible cost, which took
me longer to notice, is that you also opt out of the tool's self-update story
— the version bookkeeping, the migration prompts, the "here is what changed
and here is what you must do about it" — and you end up rebuilding whichever
part of it you actually needed.

That is a fair trade and I would make it again. But it is worth pricing in
before you fork, rather than discovering it on the third upgrade, when you
realize you have no idea which of the last dozen versions' notes you have
read.
