# Setting up gbrain locally on a MacBook

Every plane gbrain needs, this laptop provides. The database is a container.
The embedding model and the reranker are files on disk. The chat calls shell
out to the `claude` binary already logged into my subscription. The pages are a
git repo in my home directory. Nothing in the retrieval path calls a metered
API, and nothing in it depends on a service staying up.

That is a deliberate build, not the default one. gbrain's own quickstart is
two commands and a PGLite file — [ready in about two
seconds](https://github.com/garrytan/gbrain/blob/main/docs/INSTALL.md), no
Docker, no server. If you don't intend to modify the tool or run it from
several processes at once, take that path and skip this note. What follows is
what I traded those two seconds for, and why.

## The four planes

Setup is easier to reason about once you stop thinking of it as one install
and start thinking of it as four independent choices. Each is configured
somewhere different, which is the part that bites.

| Plane | What runs it here | Configured in |
|---|---|---|
| CLI | `bun link` symlink into a clone at `~/gbrain` | the filesystem |
| Database | Postgres 16 + pgvector in Docker, `localhost:5432` | `~/.gbrain/config.json` |
| Models | Ollama for embeddings, llama.cpp for reranking, `claude-cli` for chat | the file for embeddings, the Postgres `config` table for the rest |
| Pages | a git repo of markdown I own | `~/.gbrain/config.json` |

Note that the third row names two places. The embedding pins sit in
`~/.gbrain/config.json`; the reranker and the chat tiers do **not** — they
live in a table inside the brain they configure. That split surprised me more
than once, and §5 has the command that tells you which plane answered.

## Prerequisites

Bun (1.4.0 here) for the CLI. A Docker runtime for Postgres — Docker Desktop,
OrbStack, or Colima all work; nothing in the compose file is engine-specific.
Ollama for embeddings and llama.cpp (`brew install llama.cpp`) for reranking.
The `claude` CLI, already signed in, for chat. No vendor API keys.

For reference when a number below sounds heavy: this is a 24GB M4 Pro MacBook
Pro. Both local models resident at once come to under 2GB.

## 1. The CLI: a clone, not a release

```bash
git clone https://github.com/garrytan/gbrain.git ~/gbrain
cd ~/gbrain && bun install && bun link
```

`bun link` puts a symlink at `~/.bun/bin/gbrain` pointing at `src/cli.ts`,
which Bun runs directly — no build step, no binary. Edits to the clone are
live the moment I save them, which is the entire reason I install this way.

Two things to know before you copy it. First, **do not** `npm install -g
gbrain`; the npm package of that name is an unrelated project that will
shadow the real binary on your PATH. Second, running from a clone means you
have opted out of `gbrain upgrade` — the cost of that, and the upgrade
routine I ended up writing instead, is the subject of [Upgrading a
source-linked CLI](upgrading-a-source-linked-cli.md).

## 2. The database: Postgres, because of the process count

PGLite is the default engine and it is a good default: embedded Postgres
compiled to WASM, a data directory instead of a server, ready instantly. It
also accepts exactly one connection at a time. gbrain knows this and ships an
advisory file lock (`src/core/pglite-lock.ts`) whose header says so plainly —
concurrent access throws `Aborted()`, so the lock serializes callers rather
than letting them collide.

Here is the count that decided it for me:

```bash
$ ps aux | grep -c '[g]brain serve'
7
```

Seven MCP servers, one per editor project with gbrain wired in, each a
long-lived stdio subprocess holding the brain open — plus whatever I'm typing
in a terminal, plus the autopilot daemon when it runs. On a single-writer
engine that is a queue. A real server is not a luxury at that point; it is the
thing that makes the topology legal.

The compose file lives in this repo:

```bash
docker compose -f gbrain/docker-compose.yml up -d
```

`pgvector/pgvector:pg16`, one named volume, one published port. Then point
gbrain at it:

```bash
export GBRAIN_DATABASE_URL=postgresql://gbrain:<password>@localhost:5432/gbrain
gbrain init --prefer-postgres
```

Reach for the env var rather than `gbrain config set database_url`, because
that flag runs a ladder whose **first** rung is the env URL. The later rungs
are Supabase discovery, then a local server, then a container gbrain starts
itself — and the last two sit behind explicit `--local-postgres` /
`--allow-docker` flags, on the principle that gbrain should not mutate a
server it doesn't own without being told to. When you have brought your own
container, rung one is the whole conversation, and init persists the URL into
the config file for you. An existing PGLite brain moves across with
`gbrain migrate --to supabase --url <postgres url>`; the flag is named for
the common case but takes any URL.

One caveat about that compose file, which is mine and which I am flagging
rather than hiding: it publishes `5432:5432`, meaning all interfaces, with a
password that is committed to a public repo. On a laptop that joins café
wi-fi that is an open Postgres. `127.0.0.1:5432:5432` binds it to the loopback
instead and costs nothing, because every client here is local anyway.

## 3. Embeddings: a local model, pinned before the first import

```bash
ollama pull bge-m3
gbrain config set embedding_model ollama:bge-m3:latest
gbrain config set embedding_dimensions 1024
```

This is the one decision that is expensive to revisit. Embeddings are stored
in a fixed-width `vector(N)` column, so changing the dimension count later is
a schema transition plus a full re-embed of every chunk — a supported
operation (`gbrain migrate embeddings --to <provider:model> --dim <N>`) but
not a free one. `gbrain init`, `doctor`, and `embed --stale` all refuse to
proceed on a mismatch rather than silently writing the wrong width, which is
the right call and also why you meet this decision at setup time.

Picking a local model is what makes ingestion free and offline — a bulk import
of a few thousand pages costs nothing but fan noise. bge-m3 is 1.2GB on disk
and 1024 dimensions, the same width as `voyage:voyage-4`, so if I ever want a
hosted embedder the escape hatch is a same-dimension swap rather than a column
migration. Choosing a local model and choosing its width are the same
decision; make it once, deliberately.

Picking *this* local model is about language. Most of what's in my brain is
written in my first language (L1) rather than English — pages about people,
companies, and family — while the questions I put to it arrive in whichever
language I happen to be thinking in. A query in English against an L1 page is
the normal case here, not the edge case, and that rules out the English-first
models that dominate the small-and-fast tier: they will embed the L1 page
somewhere its own English paraphrase never goes.

bge-m3 is multilingual by construction rather than by translation — an
XLM-RoBERTa backbone trained across 100+ languages, with an 8192-token context
so a long page doesn't get chopped mid-thought. The two languages land in one
space, so cross-language retrieval is just retrieval. It gives me better
results in my L1 than the English-first alternatives, which is the only
comparison this brain cares about.

The general form of that decision: pick your embedder for the language your
*corpus* is in, not the language the model's README is in. It is the one
setting a monolingual-English default gets wrong for everyone else, and the
dimension pin above means you get to be wrong about it exactly once.

## 4. The reranker: a cross-encoder on port 8081

Hybrid search returns candidates; the reranker decides which of them you
actually see. It is a cross-encoder — it reads the query and a candidate
*together* and scores the pair, which is slower per document than a vector
comparison and much better at telling near-misses from real hits. On a
bilingual corpus this is where quality is won or lost, so it gets the same
language test the embedder got.

Two config keys — and they land in the DB plane, not the file, so
`gbrain config show` will not show them:

```bash
gbrain config set search.reranker.enabled true
gbrain config set search.reranker.model llama-server-reranker:bge-reranker-v2-m3
```

The model behind that name is llama.cpp serving a GGUF you download once —
mine is a Q8_0 quant of `bge-reranker-v2-m3` pulled from Hugging Face into
`~/Developer/models/`:

```bash
brew install llama.cpp
llama-server -m ~/Developer/models/bge-reranker-v2-m3-Q8_0.gguf \
  --alias bge-reranker-v2-m3 --reranking --pooling rank \
  -c 8192 -b 8192 --ubatch-size 8192 -fa on -ngl 99 \
  --host 127.0.0.1 --port 8081
```

Three flags carry real weight. `--alias` sets the id llama-server reports at
`/v1/models`, and gbrain's model string after the colon must match it —
without an alias the id defaults to the gguf's file path, which technically
works and reads horribly. `--reranking` and `--embeddings` are mutually
exclusive at launch, which is the real reason this is a second server rather
than another Ollama model. And `-ngl 99` puts every layer on the GPU, which on
Apple silicon means unified memory and no copy.

**Why this model.** `bge-reranker-v2-m3` is the cross-encoder sibling of the
embedder — same M3 family, same multilingual backbone — so the shortlist and
the re-scoring share one view of language instead of handing my L1 to a
retriever that understands it and then to a scorer that doesn't. A reranker
that is weaker in your corpus language than your embedder is worse than no
reranker at all: it takes a good shortlist and confidently reorders it into a
bad one.

**Why it fits an ordinary MacBook.** The Q8_0 quantization is 606MB on disk —
against a 24GB machine, noise. It runs in bursts rather than continuously,
because a cross-encoder only ever sees the shortlist, a few dozen pairs per
query. The 8192-token context and matching batch sizes mean a full chunk plus
query fits in one pass with nothing truncated. On an M4 Pro with all layers
offloaded, reranking is not the slow part of any query I run — chat is (see
§5).

The bigger option is real and I went looking at it: the download cache in that
same folder still records a Qwen3-Reranker-4B GGUF, and llama.cpp serves those
just as happily. But a 4B cross-encoder is roughly four times the weights for
a job the 568M multilingual model already does well on my corpus, and it wants
that memory permanently, on a laptop that is also running Postgres, an
editor, and a browser. Small enough to leave running beats large enough to
impress. The 606MB model is the one that stayed.

**Keep it running.** A reranker that is down is a search quality regression
you feel and can't easily name, so it belongs in launchd rather than in a
terminal tab:

```xml
<!-- ~/Library/LaunchAgents/com.local.llama-rerank.plist — excerpt;
     wrap it in the usual <plist version="1.0"><dict> boilerplate -->
<key>ProgramArguments</key><array>
  <string>/opt/homebrew/bin/llama-server</string>
  <string>-m</string><string>~/Developer/models/bge-reranker-v2-m3-Q8_0.gguf</string>
  <string>--alias</string><string>bge-reranker-v2-m3</string>
  <string>--reranking</string>
  <string>--pooling</string><string>rank</string>
  <string>-c</string><string>8192</string>
  <string>-b</string><string>8192</string>
  <string>--ubatch-size</string><string>8192</string>
  <string>-fa</string><string>on</string>
  <string>-ngl</string><string>99</string>
  <string>--host</string><string>127.0.0.1</string>
  <string>--port</string><string>8081</string>
</array>
<key>RunAtLoad</key><true/>
<key>KeepAlive</key><true/>
<key>ProcessType</key><string>Background</string>
```

`launchctl load -w ~/Library/LaunchAgents/com.local.llama-rerank.plist`, and
it comes back at every login and after every crash. Bind it to `127.0.0.1`,
not `0.0.0.0` — same reasoning as the Postgres port in §2, and here there is
no excuse at all, since the only client is on the same machine.

One happy asymmetry with the section above: swapping rerankers is cheap.
Nothing is stored, so changing the model is a `config set` and a server
restart, with no re-embed and no migration. The embedder is the decision to
agonize over; the reranker is the one to experiment with.

## 5. Chat: the subscription, through a subprocess

Claude has no first-party embedding model, so the model plane is genuinely
three choices, not one. Embeddings come from Ollama and reranking from
llama.cpp, above. Everything chat-shaped —
synthesis, fact extraction, query expansion, the dream cycle — routes through
the `claude-cli` recipe, which spawns `claude --print` and rides the CLI's own
OAuth session. Its `auth_env.required` is empty by design: the binary on PATH
*is* the auth surface, and there is no `ANTHROPIC_API_KEY` anywhere on this
machine.

```bash
gbrain config set models.tier.utility   claude-cli:claude-haiku-4-5-20251001
gbrain config set models.tier.reasoning claude-cli:claude-sonnet-5
gbrain config set models.tier.deep      claude-cli:claude-opus-5
gbrain config set models.tier.subagent  claude-cli:claude-sonnet-5
```

Those four writes do not land in `~/.gbrain/config.json`. They go to the
`config` table in Postgres, so `gbrain config show` — which prints the file
plane — will not show them, and `gbrain config get models.tier.utility` will.
Two planes, one command surface. If you go looking for a setting in the file
and find nothing, you have not lost it; you are reading the wrong plane.

`config get` is the one that tells you, on stderr, which plane answered — and
occasionally that both did:

```
$ gbrain config get embedding_model
ollama:bge-m3:latest
[config] source: file/env plane — a DB-plane value also exists and is shadowed at runtime
```

A shadowed duplicate is not an error, but it is a trap you set for your future
self: change the DB value, watch nothing happen, lose twenty minutes. When you
see that line, delete the loser.

gbrain is honest about what this path costs, and says so unprompted the
moment you pin the subagent tier to it:

```
[models] tier.subagent resolved to "claude-cli:claude-sonnet-5" — provider
does not support prompt caching. The loop will run hot (cost scales linearly
with conversation length).
```

On a flat-rate subscription that warning is about latency rather than money,
but it is the correct shape of warning: a subprocess-per-call provider has no
cache to hit, so long loops re-send their whole context every turn.

The table that actually tells you what is wired is `gbrain models`, and the
column worth reading is the attribution on the right:

```
tier.utility    claude-cli:claude-haiku-4-5-20251001   [config: models.tier.utility]
```

`[config: <key>]` means a pin you set was read. `[tier.<t> (caller-specific)]`
means a narrow resolver fell through to a key-aware default and elected a
provider on its own — which can be one you cannot pay for, from a placeholder
you typed months ago. That failure is worth understanding before it happens to
you: [A placeholder key is a routing
decision](a-placeholder-key-is-a-routing-decision.md).

## 6. Pages: markdown in a repo you own

```bash
gbrain sources list
```

```
  default               federated         56 pages  last sync 2026-09-05T10:48:32Z
                        /Users/<username>/Developer/github.com/<GitHubUsername>/<RepoName>
```

The brain is a Postgres database, but the source of truth for pages is that
directory: markdown under `people/`, `companies/`, `projects/`, `notes/`,
`daily/`, registered as a source and synced into the DB. Binary attachments go
to a bucket beside them — `storage.localPath` in `~/.gbrain/config.json`,
`_files` inside the same repo — so the whole brain is one tree.

Making it a git repo is the part I'd repeat anywhere. Every enrichment the
daemon writes overnight lands as a diff I can read in the morning, and undoing
one is `git revert` rather than a database question. Clone the repo to a
second machine and the brain comes with you. Delete it and the brain is gone.
That symmetry is the point: the database is an index, and an index should be
rebuildable from something you can read.

## 7. Wire it into the agent

```bash
claude mcp add gbrain -- gbrain serve --surface verbs
```

`--surface verbs` publishes the seven-verb memory protocol — `recall`,
`remember`, `entity`, `synthesize`, `forget`, `context_pack`, `delta` —
instead of the full tool catalog, which is the right size for a coding agent
that mostly needs to look things up and write facts back. Registration is
per-project, and each project that has it spawns its own server against the
same database. That is the seven processes from §2; the wiring step and the
engine choice are the same decision seen twice.

Then give the agent a protocol for *when* to use those verbs. This repo's
[`AGENTS.md`](../AGENTS.md) is mine: check the brain before answering from
memory, write back decisions and new people, cite the page you used. Without
it the tools are present and unused.

## 8. Autopilot: the part that runs without me

Everything above is a brain you have to talk to. `gbrain autopilot` is the
daemon that keeps it current between conversations — sync the repo, extract
facts from what changed, embed what's new, on an interval.

Run it in the foreground first, because a daemon whose first execution is
unattended is a daemon you are debugging through a log file:

```bash
gbrain autopilot --repo ~/braindata --interval 300
```

`--interval` is seconds and defaults to 300. On Postgres with minions enabled
— this install — that command doesn't do the work itself: it forks a
`gbrain jobs work` child and submits one `autopilot-cycle` job per interval
with an idempotency key, so a cycle that runs long is skipped rather than
stacked on top of the previous one. On PGLite, or with `--inline`, the phases
run in-process instead. Worth knowing which shape you're in before you read
the logs, because they look nothing alike — and worth knowing that the switch
between them lives in a *third* file, `~/.gbrain/preferences.json`
(`"minion_mode": "pain_triggered"` here), which is neither of the two planes
§5 talks about. Three places, one mental model to keep straight.

Then install it:

```bash
gbrain autopilot --install --repo ~/braindata
gbrain autopilot --status
```

`--install` writes two files: `~/.gbrain/autopilot-run.sh`, a generated
wrapper, and a launchd agent at
`~/Library/LaunchAgents/com.gbrain.autopilot.plist` with `RunAtLoad`,
`KeepAlive`, and `ThrottleInterval 60`. That throttle is a scar: without it,
an unrecoverable error — a missing `database_url`, a malformed config —
relaunched instantly and re-hit itself in a tight loop. Stdout and stderr land
in `~/.gbrain/autopilot.log` and `.err`.

**The env problem is the whole reason the wrapper exists.** A launchd process
gets a non-interactive shell, and `~/.zshrc` is for interactive shells, so
nothing you exported there reaches the daemon. Keys that work perfectly in
your terminal are simply absent at 3am, and the LLM phases don't crash — they
no-op quietly, which is worse. The wrapper sources `~/.zshenv`, `~/.zshrc`,
and `~/.bashrc` independently, then `~/.gbrain/env` last so it wins:

```bash
printf 'GBRAIN_DATABASE_URL=postgresql://gbrain:<password>@localhost:5432/gbrain\n' >> ~/.gbrain/env
gbrain autopilot --install --repo ~/braindata   # re-run so the daemon reloads
```

That file is created 0600 by `--install` and is the deterministic place for
process-level env the daemon needs before boot — keys, proxy vars,
`NODE_EXTRA_CA_CERTS`. It is the same lesson as `models.tier.*` living in
Postgres: **a value is only set if it is set in the plane that reads it.**
This note is largely one long variation on that theme.

**What it looks like when it stops.** Mine is currently off, and the status
line says exactly why:

```
Autopilot: DISABLED — repo path gone: /var/folders/.../gbrain-install-test-.../repo-behavior
  It stopped itself. Fix the path, then `gbrain autopilot --install --repo <path>`.
Last log: [autopilot] repo path missing (strike 1 of 3, disabling at 3): /var/folders/...
```

An install test had pointed the daemon at a scratch repo under `/var/folders`.
macOS cleaned the temp directory, the repo stopped existing, and the daemon
stopped itself rather than respawning against a path that was never coming
back. The generated wrapper is what counts: a miss increments a strike file,
three strikes writes the disabled marker, and any successful probe resets the
count — so a repo on a drive you unplug over lunch doesn't trip it.

I like this design more than I expected to. A `KeepAlive` daemon that cannot
disable itself is a machine that heats your lap all night against a path that
is never coming back; three strikes distinguishes gone from briefly missing;
and the marker turns "why is my brain stale?" — a question that would
otherwise cost an hour — into one line of status output that names the fix.
Re-pointing it is the command it already printed.

One setting to check before you leave a daemon running against a fork:

```bash
gbrain config get self_upgrade.mode   # notify
```

Autopilot is where gbrain decides whether to update itself. On a clone with my
own commits on it, an unattended self-upgrade would be the pull I explicitly
reserved for myself, so `notify` is the only correct value here — see
[Upgrading a source-linked CLI](upgrading-a-source-linked-cli.md) and the rule
it points at.

## 9. Verify at the call site, not in the dashboard

`gbrain doctor` catches the structural problems — unreachable database, stale
schema, a shadowing npm install — and names the fix command in the message.
Run it. But do not mistake a green health check for a working feature.

`gbrain models doctor` reported `expansion ✔ ok` on this machine while query
expansion was completely dead, because the probe calls the model directly with
an explicit override and never asks whether the recipe declares that
touchpoint at all. The label was cosmetic. The bug was two fixes deep, and no
amount of re-running the probe would have shown it.

What actually verifies an install is a real round trip: write one fact, start
a fresh session so the chat context is gone, and ask for it back. An answer
proves the planes that carried it — MCP wiring, the engine, the pages on disk
— and a fresh session is what makes the proof mean anything, because nothing
but the brain could have supplied the answer. Then ask something
`synthesize`-shaped, which is the only call that has to reach a chat model,
and watch it either cite pages or fail loudly. Two queries, four planes, both
of them facts about state rather than claims in a log.

## What it costs

A Postgres container running all day, a 1.2GB embedder paged in on demand, a
606MB reranker resident from login, and a daemon waking every five minutes.
Under 2GB of models on a 24GB machine; I have never noticed it while working.
The real cost is latency: `claude-cli` is a subprocess per call with no prompt
caching, and query expansion alone adds
roughly eight seconds because a small model spends hundreds of thinking tokens
rewriting a short query. Flat-rate, but slow — that is the trade a
subscription-routed brain makes, and on interactive queries you feel it.

The other cost is that you own the seams. A local-first install exercises code
paths the maintainers run less often: a chat provider with no structured
output, an embedder and reranker nobody upstream is benchmarking against a
corpus in my L1, an engine choice the default install avoids, a daemon whose
environment you assemble by hand. Some of those seams you will find by
hitting them. Budget for that, or take the two-second path.
