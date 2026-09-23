## Brain-first protocol

You have a knowledge brain connected over MCP. Before answering any question
about people, companies, decisions, projects, or past context:

1. **Brain first — route by the shape of the question.** Exact names or known
   tokens → `search` (cheap hybrid, no expansion). Concept, landscape, or
   "all the X that do Y" questions → `query` FIRST — it recovers synonym
   phrasings `search` misses, and a populated `search` result set is not proof
   of coverage. On the verbs surface the same split is `recall` (retrieve)
   vs `synthesize` (reasoned answer). Check the brain BEFORE answering from
   memory or asking me. Never ask "who is X?" or "what did we decide about Y?"
   before checking — the brain probably already knows.
2. **Write back.** When I make a decision, mention a new person/company, or land
   on an idea worth keeping, write it to the brain: `remember` on the verbs
   surface (one fact, with provenance), or `put_page` on the full surface
   (entity pages under people/, companies/; decisions under decisions/ or
   notes/). One insight, one page, linked.
3. **Cite.** When you answer from the brain, name the page you used.

## Rules

Project rules live in [`.agents/rules/`](.agents/rules/) and are binding. Read
them before acting in this repo.

No rules are currently active.

Deprecated, kept for history, not binding:

- [Never run `gbrain upgrade`](.agents/rules/no-gbrain-upgrade.md) — retired at
  gbrain v0.50.0.0 together with the
  [`/upgrade-gbrain`](.claude/commands/upgrade-gbrain.md) command. Why:
  [Retiring the `gbrain upgrade` ban](gbrain/retiring-the-gbrain-upgrade-ban.md).

## Upgrading gbrain

Agents may run `gbrain upgrade`. Do not use `/upgrade-gbrain`.

Before running it, read the migration note for the version it will install:
`skills/migrations/v<version>.md` in the incoming tree. To see that tree
without pulling:

```bash
git -C ~/gbrain fetch && git -C ~/gbrain show @{u}:VERSION
```

```bash
git -C ~/gbrain show @{u}:skills/migrations/v<version>.md
```

If a note calls for stopping services or an ordered cutover (v0.50.0.0 did),
follow the note instead of running `gbrain upgrade`. On this install,
`gbrain upgrade` pulls, runs `bun install` with its install scripts, and runs
`post-upgrade` against whatever processes are live.

After upgrading, restart any gbrain process that started before the new
code. The autopilot's job worker restarts with it:

```bash
launchctl kickstart -k gui/$(id -u)/com.gbrain.autopilot
```

The full sequence, with checks:
[Upgrading gbrain from source, end to end](gbrain/upgrading-gbrain-from-source.md).
