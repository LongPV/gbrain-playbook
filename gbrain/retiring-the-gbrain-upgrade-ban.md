# Retiring the `gbrain upgrade` ban

Until gbrain v0.50.0.0 I kept a rule that no agent may run `gbrain upgrade` —
[never run it](../.agents/rules/no-gbrain-upgrade.md) — and a slash command,
[/upgrade-gbrain](../.claude/commands/upgrade-gbrain.md), that did the
post-pull half of an upgrade by hand. Both are now deprecated. Agents may run
`gbrain upgrade`. The files stay in the repo with a banner on top, for
history; this note is why.

## The workaround's one line broke first

The command rested on one claim, made in
[Upgrading a source-linked CLI](upgrading-a-source-linked-cli.md): once I have
pulled, the mechanical part of an upgrade is a single chain.

```bash
cd ~/gbrain && bun install && gbrain apply-migrations --yes && gbrain post-upgrade
```

On 2026-09-11 I upgraded from 0.48.4.0 to 0.50.0.0, and the v0.50.0.0
migration note forbids exactly that chain on a host with live services. Every
process sharing the database has to be stopped first. The install has to skip
lifecycle scripts (`bun install --ignore-scripts`), because `bun install`'s
postinstall applies migrations too. Only schema migrations may run
(`apply-migrations --force-schema --yes`), queued jobs get reviewed, and
`post-upgrade` waits until everything is back on the new version. Old workers
must never touch the migrated queue.

When the chain was about to run, autopilot, its job worker, and seven MCP
`serve` processes were attached to the database. What stopped it was my
agent's permission classifier, not anything in the command. The command had
guarded against the pull and left the other risk — running setup against
live services — with no guard at all.

## The ban's premise had already moved

The rule said the pull inside `gbrain upgrade` was mine to make, because it
decides when my commits meet upstream's. But the pull takes no arguments; it
follows the branch's tracking config. The branch I actually run,
`longpv/gbrain`, tracks `origin/longpv/gbrain` — my own fork — not upstream.
So the pull never meets upstream. It fast-forwards to what I have already
pushed, and `--ff-only` refuses anything else, so it fails closed instead of
merging on my behalf. The meeting with upstream happens where I do it: that
day, a `git merge upstream/master` I ran myself.

[When `gbrain upgrade` becomes safe again](when-gbrain-upgrade-becomes-safe-again.md)
imagined a pristine `master` tracking `upstream/master` as the condition for
lifting the ban. The branch I run never tracked upstream in the first place.

## What `gbrain upgrade` gives back

- **Its own baseline.** It records the from/to pair in
  `~/.gbrain/upgrade-state.json` — the record the command's private stamp,
  `~/.gbrain/claude-upgrade-stamp`, existed to stand in for. The stamp is no
  longer maintained.
- **The "what changed" print.** `post-upgrade` pitches the features of every
  version crossed, driven by that from/to pair. A hand-pulled tree makes both
  reads return the same version, and the print goes silent. For it to show, the
  pull has to happen *inside* the command: push the upstream merge to the fork
  and let `gbrain upgrade` fast-forward to it, rather than merging into the
  live clone first.
- **One less thing of mine to keep in step with upstream.** The command
  reimplemented the tail of `src/commands/upgrade.ts` and would have drifted
  from it.

## What it does not fix

`gbrain upgrade` runs the same risky sequence the command did. In
`src/commands/upgrade.ts`, the `bun-link` path pulls, runs `bun install` with
lifecycle scripts, then runs `gbrain post-upgrade` — which runs
`apply-migrations --yes` unconditionally — and `gbrain features`.
`--swap-only` is no escape on a source install: the v0.50.0.0 note points out
that Bun's install scripts still run.

So a release shaped like v0.50.0.0 is exactly as unsafe through
`gbrain upgrade` as it was through the command. Lifting the ban moves the
guard rather than removing the risk. The guard now lives in
[AGENTS.md](../AGENTS.md): before running `gbrain upgrade`, read the migration
note for the version it will install, and if the note asks for a
stopped-service cutover, follow the note instead.

## What this generalizes to

A workaround inherits its author's threat model. Mine was about who gets to
pull, so it guarded the pull and nothing else — and the next release's real
risk was somewhere the workaround had never looked. When the tool's own path
and the hand-rolled path run the same steps, the hand-rolled one is only
worth keeping if it is guarding something the tool's path does not. By
v0.50.0.0 mine was not.
