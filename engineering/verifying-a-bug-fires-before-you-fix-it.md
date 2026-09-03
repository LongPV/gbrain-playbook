# Verifying a bug fires before you fix it

I asked an agent, "does gbrain's `slugify()` turn `đ` and `Đ` into `d`?"
It ran the function on a Vietnamese phrase, saw `Đường Đông` come out as
`uong-ong` — the Đ dropped, silently — and reported: no. I said, open a PR
fixing it. The agent did: a transliteration table for `đ/ð → d`, `ł → l`,
`ø → o`, and the other Latin stroke letters and ligatures that NFKD leaves
alone; a fresh test file; a clean commit; a PR against my fork. Nine minutes
end to end.

Then I asked the diagnostic question I should have asked first: my brain has
a page at `people/phan-dang-khoa` for a Vietnamese name. If `slugify()` really
does drop `Đ`, how did that page get its correct slug?

## The primary path had already routed around the bug

The answer sat in one line of the extractor prompt in
`src/core/facts/extract.ts`:

> entity = a canonical slug (e.g. `people/alice-example`, `companies/acme`,
> `travel`) when known, else a display name the caller can canonicalize.

The LLM that extracts facts from a transcript is told to emit a
pre-canonicalized slug whenever it can, and Claude knows the Vietnamese
romanization convention that maps Đ to D. It emits `people/phan-dang-khoa`
directly. That string is slug-shaped — it contains a `/` — so a later ticket
(#3447) special-cased slug-shaped input to preserve the separator instead of
running the full slugify pass. Between those two behaviors, `slugify()`
almost never sees a Vietnamese display name on the write path.

The function is buggy in isolation. It is also a fallback whose caller has
already worked around the bug. The PR I opened would have added a
transliteration table, a list of Latin ligatures to keep current, and a
maintenance surface, all to fix a code path that does not misfire on real
data. There was no wrong slug in my brain to point at.

I closed the PR.

## A demo is not a defect

The trap was in how I decided the fix was worth writing. I saw a
`bun -e` reproduction — a function called in isolation, producing wrong
output — and treated it as evidence of a bug. It is not. It is evidence of
a code smell: the function's behavior is wrong for those inputs *if* those
inputs reach it. Whether they do is a separate question, and it is the one
that determines whether a fix is worth the diff.

For a fallback in particular, "wrong output when called directly" is close
to the design. Fallbacks catch the cases the primary path did not handle;
they are not required to be smart, only to be safe. If the primary path in
practice handles everything you care about, the fallback's degraded output
is a latent code smell rather than an active bug — and shipping a fix
without an observed defect trades a real maintenance cost for an imagined
correctness win.

## The question that would have stopped me

The one question that would have caught this before I wrote a line of the
fix: *who calls this function in production, and does the buggy path fire
on real data?* Ten seconds of grep across the callers would have surfaced
the extractor prompt and the #3447 slug-shaped preservation. Ten seconds of
asking "is there a bad slug in the brain I can point at?" would have made
it obvious there wasn't.

I did neither, because a `bun -e` reproduction feels like enough evidence.
It is not enough evidence. It tells you *how* the function behaves in
isolation. It tells you nothing about *whether* that behavior matters.

## The angle for AI coding assistants

Every step of that PR would have taken a human developer longer than it
took the agent — and the friction of those minutes is where a person
usually pauses long enough to ask whether the fix is worth writing. When
that friction goes to zero, the pause has to be deliberate. A code smell
you can fix in nine minutes still costs something to review, to
counter-review later, and to keep in maintenance; a fix without an
observed defect is a debt you took on for no benefit.

So the discipline is not "be slower." It is: before writing a fix, name
the observed failure. Not the reproducible-in-isolation failure. The one
that showed up on real data — a bad row, a wrong result the user pointed
at, a log line. If you cannot name it, the code smell is a note-to-self,
not a PR.

## What this generalizes to

Two rules, and one habit for the fallback case in particular.

**A fix needs a defect, not a smell.** "This function returns the wrong
thing when I call it directly" is a code smell. "This function returned
the wrong thing on this real input and here is the row it wrote" is a
defect. The first is a note; the second is a PR. Do not conflate them,
and do not accept a synthetic reproduction as evidence of the second.

**Trace the callers before writing the fix.** Ten seconds of grep can
reveal that the primary path has already worked around the bug, or that
the function is called somewhere its output no longer matters. Neither
outcome is visible from reading the function alone.

**Fallbacks are shadowed by design.** A function whose whole purpose is to
be a last-resort answer is *expected* to have degraded behavior; the
question worth asking is whether the primary path lets that behavior
matter. If the primary path handles the class of input in question,
"fixing" the fallback closes a code smell you could have left as a
comment.

The agent that opened the PR was doing exactly what I told it to. The
mistake was mine, in the sentence before: "open a PR fixing this," when
I had not yet asked whether there was anything to fix.
