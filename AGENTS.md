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

Project rules live in [`agent-rules/`](agent-rules/) and are binding. Read
them before acting in this repo.

- [Never run `gbrain upgrade`](agent-rules/no-gbrain-upgrade.md)
