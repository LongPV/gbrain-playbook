# Upgrading gbrain from source, end to end

My gbrain runs from a clone. `~/.bun/bin/gbrain` is a symlink to
`~/.bun/install/global/node_modules/gbrain`, which is itself a `bun link`
back to `~/gbrain`. That puts the install on the `bun-link` path in
`src/commands/upgrade.ts`: `git pull --ff-only`, then `bun install`, then
`gbrain post-upgrade`.

Why I stopped hand-rolling this and let `gbrain upgrade` run is in
[Retiring the `gbrain upgrade` ban](retiring-the-gbrain-upgrade-ban.md). This
note is the sequence itself, as I ran it on 2026-09-23. It includes the part
neither of the earlier notes covered: an upgrade isn't done while the old
code is still running.

## Look before you pull

Fetching is safe. It moves the remote-tracking ref and leaves the checkout
alone, so I can read what's coming before any of it lands:

```bash
git -C ~/gbrain fetch && git -C ~/gbrain show @{u}:VERSION
```

```bash
git -C ~/gbrain log --oneline HEAD..@{u}
```

```bash
git -C ~/gbrain show @{u}:skills/migrations/v<version>.md
```

Most releases ship no migration note. When one does, it can ask for
something `gbrain upgrade` won't do, like stopping every process that shares
the database before migrating. v0.50.0.0 asked for exactly that. If a note
asks for a cutover, I follow the note and skip the one-liner.

## Two routes, the same steps

The one-liner:

```bash
gbrain upgrade
```

Or the same steps by hand, when I want to stop between them:

```bash
git -C ~/gbrain pull --ff-only
```

```bash
cd ~/gbrain && bun install
```

```bash
gbrain --version
```

```bash
gbrain post-upgrade
```

`bun install` does more than install packages. Its postinstall hook runs the
schema migrations, so by the time `post-upgrade` runs, it often has nothing
left to do and says "All migrations up to date." If I don't want postinstall
to rewrite the autopilot services, I prefix the install with
`GBRAIN_NO_AUTOPILOT_INSTALL=1`.

## A matching version is not a migrated database

This is the part that surprised me. After the fetch, `VERSION` in my checkout
and on `upstream/master` both read 0.52.2.0, which looked like nothing to do.
I ran `bun install` anyway, and it applied 14 schema migrations, taking the
database from 149 to 163.

The version string describes the checkout, and the schema version describes
the database. They are two separate records, and something had moved the
first without the second. Most likely an earlier pull landed and nothing ran
the migrations afterwards. `VERSION` can't show that gap. Running the install
and migrate steps is what shows it, and on a current install they cost next
to nothing, so I run them even when the version says I'm up to date.

## Restart what's still running the old code

The migrations changed the database underneath processes that were already
running. On this Mac that means:

- the launchd agent `com.gbrain.autopilot`, and the `gbrain jobs work`
  worker it starts as a child;
- `gbrain serve` processes, one per Claude session that has the brain
  connected over MCP.

Bun reads the source when a process starts, so a process that started before
the pull is still running the old code. The check compares each process's
start time with the time of the commit it should be running:

```bash
git -C ~/gbrain log -1 --format=%cd
```

```bash
ps -o pid,lstart,command -p $(pgrep -d, -f "bin/gbrain")
```

My autopilot had started the previous morning, hours before the commit. One
kickstart replaces it:

```bash
launchctl kickstart -k gui/$(id -u)/com.gbrain.autopilot
```

The worker went with it, because it is the autopilot's child. There was no
separate process to kill, and the new worker came up in the same second as
the new autopilot. `launchctl list` then showed `143` for the agent. That's
the exit status of the old process, killed by the kickstart's SIGTERM, and it
isn't an error.

`serve` processes belong to the Claude sessions that launched them. The fix
is to reconnect the brain in those sessions, or close them. Mine had started
after the commit, so they could stay.

For a release that asks for a cutover, this order is backwards: the stops
come first, before anything migrates. See
[Retiring the `gbrain upgrade` ban](retiring-the-gbrain-upgrade-ban.md) for
how v0.50.0.0 laid that out.

## What this generalizes to

A tool you run from source has three things to upgrade: the code on disk, the
state it keeps, and the processes that are running it. Each has its own
version and its own way to check it. For gbrain those are `VERSION`, the
schema version, and process start times. Pulling updates only the first.
Before I call an upgrade done, I check all three.
