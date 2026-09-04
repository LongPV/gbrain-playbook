# A placeholder key is a routing decision

I don't have a paid OpenAI account. So when `gbrain models` showed me this,
sitting in a table where every other row named a Claude model on my
subscription, it was worth a second look:

```
models.dream.extract_atoms  → openai:gpt-5.6-luna  [tier.utility (caller-specific)]
```

Nothing I configured asked for that. I found it on gbrain 0.48.2.0, and the
path from "a string I typed during setup" to "a phase of my brain routes to a
provider I can't pay for" turned out to be four hops long, every one of them
individually reasonable.

## The chain

My `~/.gbrain/config.json` carries this line:

```json
"openai_api_key": "sk-dummy"
```

I put it there — or something did, during setup — to get past a check. It is
not a credential. It is the shape of a credential.

`mergedProviderEnv` folds config-file keys into the environment, so
`openai_api_key` becomes `OPENAI_API_KEY` for everything downstream. That's a
good design: it means a launchd job or an MCP server with an empty environment
still sees the keys you configured in a file, and the source comments show
that gap being closed provider by provider over time as each one bit someone.

`resolveTierDefault` then picks a default model per tier by walking an ordered
provider list and taking the first whose env key is **present**:

```ts
export const PROVIDER_TIER_DEFAULTS = [
  { provider: 'anthropic', envKey: 'ANTHROPIC_API_KEY', tiers: (t) => TIER_DEFAULTS[t] },
  { provider: 'openai',    envKey: 'OPENAI_API_KEY',    tiers: discoveredOrStaticOpenAITier },
];
```

Anthropic is first, and on any normal install it wins, which is exactly the
stated intent — the comment says Anthropic-first "preserves today's behavior
byte-for-byte on every keyed install." But I run Claude through the
subscription, via the `claude-cli` recipe, which shells out to the `claude`
binary and has no API key at all. So there is no `ANTHROPIC_API_KEY` in my
environment. Anthropic is skipped. OpenAI is checked, `sk-dummy` is *present*,
and OpenAI wins.

The specific model comes from a fourth hop: with no discovered-model cache to
read from (the account probe against `sk-dummy` had already failed, leaving
`model-cache.json` holding a timestamp and no models), it falls back to the
static ranking derived from the OpenAI recipe, whose cheap tier is
`gpt-5.6-luna`.

So: a placeholder credential became a provider election, which became a model
pin, on a phase I never touched.

## Why only this one row

Most tasks never reach that provider list. They resolve through
`resolveModel()`, which reads `models.tier.*` first — and my tiers are all set
to `claude-cli:` models, so they land correctly.

`extract_atoms` is different, and deliberately so. Its resolver is two steps:
its own config key, else `resolveTierDefault('utility')`. The docstring is
explicit that this is not an oversight:

> Deliberately NOT `resolveModel()`'s fuller chain (which also honors
> `models.tier.utility`, `models.default`, and an env var) — extract_atoms has
> never read those, and unifying the two behaviors is a separate, larger
> change.

I have a lot of sympathy for that comment. It is an honest note about scope,
left by someone who saw the inconsistency and chose not to fix it in that
change. But it means `models.tier.utility` — which I set, which the dashboard
displays, which reads like the thing that governs utility-tier work — does not
govern this. The one task that skips the tier system is the one task that
lands on the provider list, and the provider list is the thing the placeholder
key corrupts.

## What gave it away

Not the model column. `openai:gpt-5.6-luna` is a plausible-looking cheap
utility model; if I'd been skimming for something obviously broken I'd have
skimmed past it.

It was the attribution:

```
[config: models.tier.utility]      ← someone configured this
[tier.utility (caller-specific)]   ← nobody configured this; I guessed
```

Thirteen rows cited a config key I'd set. One cited a *tier fallback*, which
is the report's way of saying it had nothing to go on. The value looked like a
decision. The provenance admitted it was a default.

That column exists because someone thought carefully about it. The code
comments say the resolver returns model and attribution *from the same call*,
specifically so the label can never disagree with the value — an earlier
version re-read the config separately to decide what to print, and could
report a source that didn't match what actually resolved. Someone had been
burned by a dashboard that lied about its own reasoning and closed the hole.

That is the single most useful thing in the whole feature, and it's the part
that has nothing to do with models.

## The fix

Pin the key the narrow resolver actually reads:

```bash
gbrain config set models.dream.extract_atoms claude-cli:claude-haiku-4-5-20251001
```

The row flips, and the attribution flips with it — from
`[tier.utility (caller-specific)]` to `[config: models.dream.extract_atoms]`,
which is the part that proves the fallback is no longer in play.

Deleting `sk-dummy` does **not** fix this, and that's worth sitting with. With
no provider key present at all, `resolveTierDefault` returns the hardcoded
`TIER_DEFAULTS.utility` — `anthropic:claude-haiku-4-5-20251001` — which needs
an Anthropic API key I also don't have. I'd trade an unusable OpenAI pin for an
unusable Anthropic one. The placeholder is still worth removing, because it
biases every other call site that ends in `?? resolveTierDefault('utility')`,
but removing it is hygiene, not the repair.

## The deeper shape

Here is the part I keep turning over. The provider list can only elect a
provider that has an env key to check. My actual working provider —
`claude-cli` — declares `auth_env.required: []`, because the CLI owns its own
auth and there is nothing for the gateway to forward. It is keyless *by
design*, and it is the only provider on this machine that can actually serve a
request.

Which means the one provider that works is structurally ineligible for the
fallback that exists to find a provider that works. The mechanism's entire
theory of "can I use this?" is "is there an env var," and a provider that
authenticates out-of-band cannot express itself in that language. It isn't
ranked last. It cannot be ranked.

This is what happens when a capability check gets built out of the most
convenient proxy for the capability. Presence of a key is a decent proxy for
"account configured" — right up until you meet a placeholder (present, not
valid) or a subscription (valid, not present). Both ends of the proxy fail,
and they fail in opposite directions.

The same proxy shows up one layer down, too. `providerKeyReady` checks that
every required env var is *set*; `isAvailable('chat', model)` gates on that.
So in `facts/classify.ts`, a dummy key passes the availability gate, the call
is attempted, the call fails, and the `catch` degrades to a cosine-similarity
fallback. No error surfaces. The system works, slightly worse, forever.

## What this generalizes to

**A credential check that tests for existence is a check a placeholder
passes.** If your setup flow lets someone type a fake key to get past
validation, that fake key is now load-bearing input to every downstream
decision that asks "do we have a key for X." The placeholder was a lie told to
one validator, and validators don't stay put — they get folded into env maps
and read by resolvers that were written years apart. Either verify usability at
the point where it matters, or refuse recognizable sentinels outright, but do
not let "a string is present" mean "an account exists."

**Report provenance, not just values.** A config dump that shows what resolved
is a fraction as useful as one that shows *why* it resolved. Every row in my
table looked configured; only the attribution column distinguished a setting
from a guess. And derive the provenance from the same call that produced the
value — a label computed by re-walking the chain is a label that can disagree
with reality, which is worse than no label, because now you trust it.

**Watch for the call site that opts out of the general mechanism.** A codebase
with a good central resolver will still have two or three callers with their
own narrower chain, usually with a candid comment explaining that unifying them
is a bigger change. Those comments are honest and the decision is often right.
But those call sites are where your central configuration silently stops
applying, and they're disproportionately where this class of bug lives — the
general path is well-tested precisely because everything uses it.

**A fallback ordered by provider needs a way to say "none of these."** Mine
couldn't represent my situation at all: not "no provider available," but "the
available provider doesn't participate in this ranking." When you add a new
kind of thing — a keyless provider, an out-of-band auth path, a local model —
check whether your existing selection mechanisms can even *see* it, or whether
it's invisible to them by construction.

**Silent degradation is the expensive failure.** Every hop here failed
politely. The bad key produced no startup error. The provider election printed
no warning. The failed call fell back to cosine similarity. If any single step
had been loud, the chain would have been ten minutes of work instead of a
column I nearly skimmed past. Fallbacks make systems resilient and they make
misconfiguration invisible, and you generally get to pick only one.

---

*Verified on gbrain 0.48.2.0. Source references —
`src/core/ai/provider-env.ts`, `src/core/model-config.ts`,
`src/core/cycle/extract-atoms.ts`, `src/commands/models.ts`,
`src/core/facts/classify.ts` — point into the gbrain clone, not this repo.*
