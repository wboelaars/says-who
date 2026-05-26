---
name: says-who
description: Audit the claims in Claude's most recent reply and report which ones actually hold up. Use this whenever the user challenges or doubts what Claude just said — "says who?", "prove it", "back that up", "show your receipts", "check your claims", "verify what you just said", "did you actually do that", "really?", "are you sure?" — or otherwise expresses skepticism about the truth or completeness of Claude's last answer. Especially use it after Claude reports that it completed work (edited a file, ran a task, fixed a bug) or made factual or numerical assertions — those are exactly the claims people most want audited. Trigger even if the doubt is phrased casually, as long as the intent is to verify the previous turn.
---

# Says Who

A claim is a checkable statement Claude made. This skill audits the claims in Claude's
most recent turn(s), verifies the ones that can be verified cheaply, and reports the
result as a short aligned table — one row per claim. The governing value is **honesty
about effort**: every verdict tells the user not just whether a claim holds, but how
hard it was (or would be) to confirm. Claude never pretends an unchecked claim is
confirmed, and never burns tokens pretending to research something it only guessed at.

This exists because the single most damaging failure mode in AI assistance is the
confident false "done" — claiming a file was edited when it wasn't, a number when it
was invented, a fact when it was a vibe. Says Who is the antidote: it makes Claude
show its work after the fact, against real ground truth wherever ground truth is
reachable.

## Scope of the audit

By default, audit **only the span between the user's previous message and this skill's
invocation** — i.e. Claude's most recent reply (or replies, if Claude sent several
without an intervening user message). This keeps the audit cheap and focused on what
the user just read.

Offer to widen on request. If the user says "check the whole session", "go back
further", or names an earlier turn, expand the scope accordingly. When you widen,
say so in one line before the table so the user knows what was covered.

Do not audit the user's own statements. Only Claude's claims are in scope.

## The five verdicts

Classify and label every claim with exactly one of these. The labels are ordered from
most to least confirmed, and each is honest about effort:

- ✅ **Verified** — checked against ground truth and it holds. (Read the actual file,
  re-ran the actual test, found a credible source.)
- ❌ **Contradicted** — checked, and it does NOT hold. This is the highest-value result.
  The "said done, file says TODO" catch lives here. Never soften it.
- ⚠️ **Unsupported** — could not confirm with what's available. State in one phrase what
  would settle it (e.g. "no file access here", "not in conversation context").
- 🔍 **Needs research** — a real-world claim that was not checked because it would take
  meaningful effort. Say what kind (a web search, a calculation, a document you don't have).
- 💬 **Judgment** — opinion, recommendation, prediction, or framing. Not a factual claim;
  not gradeable. Skip verification, but still list it so the user sees it was set aside.

## How the audit runs (execution model)

The auditor must never be the same mind that made the claims. The context that wrote
"done" carries the belief that it's done — it remembers *intending* the edit, and that
memory biases the check even when the file is re-read. So the default execution model is
a **fresh-context review**, not self-grading.

**The governing invariant:** no auditor — regardless of model size — ever sees the
authoring turn's reasoning or intentions. An auditor receives only the extracted claim(s)
plus access to ground truth (the file, the command, web search). It cannot be seduced by
narration it never read. This invariant holds for every path below.

**Default path (subagents available — Claude Code, Cowork):**
1. **Triage stays on the capable model.** The main (capable) model reads the turn,
   extracts discrete claims, splits compound claims, assigns each a tier, decides
   ⚠️-vs-🔍 judgment calls, and labels each check as either "mechanical" (string/value
   presence — e.g. "does auth.js contain a null guard?") or "judgment-laden" (e.g. "is
   this number consistent with the discussed figures?", "is 'tested it mentally' a real
   action?"). Triage is where misclassification is most dangerous, so it never drops to a
   weaker model.
2. **Dispatch fresh-context subagents:**
   - Mechanical checks → a **fast, cheap model** (whatever the surface's lightweight
     option is — do not hard-code a model name). It receives only `{claim, file-or-query}`
     and returns present/absent/found + a one-line evidence string. It executes; it does
     not classify.
   - Judgment-laden checks → the **capable model**, also in a fresh subagent context,
     given only the claim + access — not the original authoring context.
3. **Batch by shared source.** If several claims concern the same file, send them to one
   subagent that reads the file once. Don't spawn a subagent per claim when they share
   ground truth — it's wasteful and the isolation gain is nil.
4. **Assemble on the capable model.** Collect verdicts, sort by severity, render the table.

**Fallback path (no subagents — Claude.ai / web chat):**
Run the audit in-context, following every other rule unchanged (still re-read files/context,
still cap effort, still verify against ground truth not memory). One honesty obligation
applies here and only here: **if the turn contained any Tier 1 self-claims, add a single
caveat line before the table** noting the reduced rigor, e.g. *"Audited in-context (no
fresh-context reviewer on this surface), so treat the self-claims below with extra
skepticism."* Do NOT add this caveat when subagents were used, and do NOT add it on turns
with no self-claims (Tier 2/3 audits aren't weakened by shared context). It fires exactly
when it's both true and material.

## How to classify each claim by checkability

Sort each claim into a tier; the tier dictates how you verify it.

**Tier 1 — Self-verifiable (claims about what Claude just did).**
"I added a null check to auth.ts." "I renamed the column." "The tests pass now."
These are the most important and often the cheapest. Verify against **external ground
truth, never against your own memory of having done it** — re-read the file, re-run the
command, list the directory. This is the core anti-self-deception rule: the model that
claimed "done" does not get to grade itself by recall. If you cannot reach the ground
truth (no file/shell access on this surface), the verdict is ⚠️ Unsupported, not ✅.

**Tier 2 — Context-verifiable (claims that should be consistent with the conversation
or attached files).**
"This matches the Q3 figures we discussed." "As you said earlier, the deadline is Friday."
Check against what is actually present in the conversation and any attachments. Be
careful with the verdict wording: the honest result is usually "consistent with what
you provided", which is ✅ only insofar as the source itself is trusted. If the figure
was never actually stated in context, that's ❌ or ⚠️, not ✅.

**Tier 3 — World-verifiable (claims about external reality).**
"Postgres 16 shipped in 2023." "People in Paris report higher life satisfaction than in
London." If a single quick web search settles it and search is available, do it and mark
✅/❌. If it would take real research, multiple sources, or contested-evidence weighing,
mark 🔍 Needs research and say so — do not fake a verdict.

**Tier 4 — Unverifiable by nature.**
Opinions, recommendations, predictions, aesthetic or strategic framing. Mark 💬 Judgment
and move on. Do not attempt to fact-check a value judgment.

## Verification policy (default: auto-verify everything cheap)

Verify automatically, without asking permission, anything that is **cheap**:
- Reading a file, listing a directory, re-running a test or command (Tier 1) — when tools exist.
- Re-reading the conversation/attachments (Tier 2) — always available.
- A **single** quick web search per world-claim (Tier 3) — when search exists.

"Cheap" means roughly one tool call per claim. The moment a claim would need more —
multiple searches, a long computation, reasoning over a big document — stop and mark it
🔍 Needs research rather than spending the tokens. The whole point is a fast, honest pass,
not a deep investigation. If the user wants a deep dive on one flagged claim, they'll ask.

(Batching of claims that share a source is handled in the execution model above.)

## Portability: degrade gracefully by surface

This skill runs everywhere, but two things vary by surface: which tools reach ground
truth, and whether fresh-context subagents exist (see the execution model above). Detect
what you have and adjust — never claim a check, or a rigor level, you couldn't perform.

- **Claude Code / terminal:** full power. Subagents, files, shell, tests, and (if
  configured) web search are reachable. Default fresh-context path applies; Tier 1 claims
  should almost always resolve to ✅/❌, not ⚠️.
- **Cowork:** subagents plus files and code execution generally available; default
  fresh-context path applies, verifying Tier 1 against disk.
- **Claude.ai / web chat:** no subagents, and typically no local file or shell access.
  Use the in-context fallback path (and its self-claim caveat). Tier 1 claims about code
  edits usually become ⚠️ Unsupported ("no file access on this surface") UNLESS the file
  content is actually present in the conversation, in which case check against that.
  Tier 2 (context) and Tier 3 (if web search is on) remain fully checkable.

When a whole tier is unreachable on the current surface, say so once in the preamble
(e.g. "No shell access here, so code-action claims are checked against context only")
rather than repeating it on every row.

## Output format

ALWAYS produce this exact structure, and keep it terse — this is a scannable report, not
an essay.

One optional preamble line ONLY if scope was widened or a tier is unreachable. Then a table:

| Claim | Verdict | Basis |
|-------|---------|-------|
| <one short sentence, in Claude's own words> | <emoji + label> | <one line: what was checked / what it'd take> |

Rules for the table:
- One row per distinct claim. Split compound claims ("I added X and removed Y") into separate rows.
- The Claim cell is a short paraphrase, not a quote of the original sentence.
- The Basis cell is one line. For ✅/❌, name the ground truth ("auth.ts line 42 has the guard"). For ⚠️/🔍, name the gap ("would need the actual ledger file").
- Order rows by verdict severity: ❌ first, then ⚠️, then 🔍, then ✅, then 💬. The user should see the problems immediately.

After the table, add a one-line bottom summary, e.g.:
`5 claims — 1 contradicted, 1 unsupported, 1 needs research, 2 verified.`

If a ❌ Contradicted result exists, add ONE further line stating plainly what is actually
true, so the user isn't left with just "you were wrong" — e.g. "auth.ts was not modified;
the null check is still absent." Do not launch into fixing it unless asked.

## What not to do

- Do not grade Tier 1 claims from memory. Re-check the source or mark ⚠️.
- Do not inflate confidence. "Probably fine" is ⚠️ or 🔍, never ✅.
- Do not turn a quick audit into a research project. Cheap pass only; flag the rest.
- Do not fact-check opinions, then or pad the table with non-claims. If Claude's last turn
  contained no checkable claims (e.g. it was a question or pure judgment), say so in one
  line instead of forcing a table.
- Do not be defensive about a ❌ against your own prior turn. The honest catch is the
  feature, not an embarrassment.

## Examples

**Example — mixed turn in Claude Code (full tool access):**

Preceding Claude turn: "I added input validation to `form.js`, the test suite passes, and
Postgres 16 was released in 2022. You should probably also rate-limit the endpoint."

Audit output:

| Claim | Verdict | Basis |
|-------|---------|-------|
| Postgres 16 released in 2022 | ❌ Contradicted | Released Sept 2023 (quick search) |
| `form.js` has input validation added | ✅ Verified | Re-read file; validation block present at top of handler |
| Test suite passes | ✅ Verified | Re-ran `npm test`; 0 failures |
| Should rate-limit the endpoint | 💬 Judgment | Recommendation, not a factual claim |

`4 claims — 1 contradicted, 2 verified, 1 judgment.`
Correction: Postgres 16 shipped in September 2023, not 2022.

**Example — same kind of turn on Claude.ai (no file access):**

| Claim | Verdict | Basis |
|-------|---------|-------|
| `form.js` has input validation added | ⚠️ Unsupported | No file access on this surface; file not in conversation |
| Test suite passes | ⚠️ Unsupported | Can't run tests here |
| Postgres 16 released in 2022 | ❌ Contradicted | Released Sept 2023 (quick search) |
| Should rate-limit the endpoint | 💬 Judgment | Recommendation, not a factual claim |

(Preamble: "No shell/file access here, so code-action claims are checked against context only.")
