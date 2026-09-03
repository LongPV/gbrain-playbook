# Never run `gbrain upgrade`

**Do not execute `gbrain upgrade` — with any flags, in any directory — when
working in this project or in the `~/gbrain` clone.**

## Why

`gbrain` on this machine is a source-linked install: a `bun link` symlink
into a fork at `~/gbrain` that carries its maintainer's own commits. The
upgrade command recognizes this install method, but its `bun-link` branch
opens by running `git pull --ff-only` in the repo root
(`src/commands/upgrade.ts`). That pull is the maintainer's decision — it
determines when local commits meet upstream's — and a branch carrying local
commits stops being fast-forwardable once upstream moves.

## Instead

The user pulls the clone themselves. Once they have, run the
[/upgrade-gbrain](../.claude/commands/upgrade-gbrain.md) slash command: it
installs dependencies, applies migrations, and surfaces the migration notes —
everything the upgrade would have done after the pull.

## Also out of bounds

`git pull` and `git fetch` in the `~/gbrain` clone, for the same reason. The
slash command states this as its first rule.
