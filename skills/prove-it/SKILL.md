---
name: prove-it
description: Thoroughly research the claims in Claude's most recent reply and report findings as a per-claim narrative. Use when the user explicitly asks for deep, research-driven verification — "really prove it", "do a thorough check", "fact-check this carefully", "verify in depth", "investigate what you said", "research these claims", "I want sources", "dig into this", "do a comprehensive audit". Especially appropriate when the turn contained substantive factual, historical, or technical claims that benefit from real research with multiple sources. The companion to says-who: same five-verdict scheme, but deeper effort per claim and a narrative output instead of a table.
---

# Prove It

A claim is a checkable statement Claude made. This skill audits the claims in Claude's
most recent turn(s), invests real effort to verify each one, and reports the result
as a per-claim narrative — one short paragraph per claim, each headed by the claim
and its verdict icon. The governing value is **honesty about effort**: every verdict
tells the user not just whether a claim holds, but what was actually done to check it
and what the evidence was.

Where the companion skill **says-who** does a fast, scannable audit (roughly one tool
call per claim, table output), **prove-it** does the opposite: it spends real research
effort on each non-trivial claim — multiple searches, cross-referenced sources, actual
file re-reads, command re-runs — and writes the result up in prose. Simple claims still
get short checks; the effort is proportional to the claim. The point is to do the
investigation, not to write a long report about it.

## Scope of the audit

By default, audit **only the span between the user's previous message and this skill's
invocation** — i.e. Claude's most recent reply (or replies, if Claude sent several
without an intervening user message).

Offer to widen on request. If the user names earlier turns or says "check the whole
session", expand and state the widened scope in one line before the first claim.

Do not audit the user's own statements. Only Claude's claims are in scope.

## The five verdicts

Classify and label every claim with exactly one of these. Same scheme as says-who,
ordered from most to least confirmed:

- ✅ **Verified** — researched and the claim holds. Name the source(s) — file path,
  command output, primary document, URL.
- ❌ **Contradicted** — researched and the claim does NOT hold. Highest-value result.
  State plainly what is actually true and where you saw it.
- ⚠️ **Unsupported** — investigated but could not reach a definitive answer with what's
  available. State what would settle it.
- 🔍 **Needs research** — even deeper investigation could not converge: contested
  evidence, paywalled primaries, requires domain expertise to weigh. Should be **rare**
  in this skill. The whole point is to do the research; reaching 🔍 means the question
  genuinely doesn't have an answer that a careful pass can produce.
- 💬 **Judgment** — opinion, recommendation, prediction. Not gradeable. Skip
  verification but still list it so the user sees it was set aside.

## How the audit runs (execution model)

The auditor must never be the same mind that made the claims. The context that wrote
the original turn carries the belief that it's right — it remembers *intending* the
claim to be true, and that memory biases the check even when sources are re-read.

**The governing invariant:** no auditor ever sees the authoring turn's reasoning or
intentions. An auditor receives only the extracted claim plus access to ground truth
(the file, the command, the web). It cannot be seduced by narration it never read.

**Default path (subagents available — Claude Code, Cowork):**
1. **Triage stays on the capable model.** The main model reads the turn, extracts
   discrete claims, splits compounds, assigns tiers, and decides — per claim — whether
   it's a "quick check" (one ground-truth lookup will settle it) or a "research check"
   (needs multiple sources or genuine investigation). In this skill, world-facing
   claims default to "research" unless they are unambiguously cheap.
2. **Dispatch fresh-context subagents:**
   - Quick checks → a fast, cheap model with `{claim, source}`; returns
     present/absent/found plus a one-line evidence string.
   - Research checks → the capable model in a fresh subagent context, given the claim
     and tool access. The research agent is expected to use multiple tool calls per
     claim: cross-reference two or more independent sources, prefer primary documents
     (official changelogs, papers, specs) over secondary ones, and return a verdict
     plus a named-sources evidence string.
3. **Batch by shared source** only where natural. In this skill, claims more often have
   distinct sources, so batching is less common than in says-who.
4. **Assemble on the capable model.** Collect verdicts, sort worst-news-first, render
   the narrative.

**Fallback path (no subagents — Claude.ai / web chat):**
Run the audit in-context with the same depth — multiple searches, careful sourcing,
primary documents preferred. If the turn contained Tier 1 self-claims, add a single
caveat line before the first claim: *"Audited in-context (no fresh-context reviewer on
this surface), so treat the self-claims below with extra skepticism."* Do NOT add this
caveat when subagents were used, and not on turns without self-claims.

## How to classify each claim by checkability

Same four tiers as says-who; the tier dictates *what kind* of verification work to do.

**Tier 1 — Self-verifiable (claims about what Claude just did).**
"I added a null check to auth.ts." "I renamed the column." Verify against external
ground truth — re-read the file, re-run the command, list the directory — **never
against memory**. Depth here usually means *being careful and complete*, not searching
more: read the whole function, not just the diff; re-run the full test suite, not just
one file's tests. If ground truth is unreachable on this surface, the verdict is
⚠️ Unsupported, not ✅.

**Tier 2 — Context-verifiable (consistent with the conversation or attachments).**
Read carefully, including tables, code, quoted material, and earlier messages. If
the claim is "as you said earlier, the deadline is Friday", actually re-locate where
the user said that. If it isn't there, mark ❌ or ⚠️, not ✅.

**Tier 3 — World-verifiable (claims about external reality).**
This is where prove-it diverges most from says-who. Do real research:
- At least two independent sources for any non-trivial claim.
- Prefer primary sources — official changelogs, government data, the paper itself —
  over secondary write-ups.
- Quote or cite what you found, briefly, in the evidence string.
- If sources disagree, say so and name the disagreement rather than picking a winner
  silently.

**Tier 4 — Unverifiable by nature.**
Opinions, recommendations, predictions, aesthetic or strategic framing. Mark
💬 Judgment and move on. Do not fact-check a value judgment.

## Verification policy (default: invest real effort, proportionally)

Verify with depth proportional to the claim:
- "X library released v3 in 2024" — one quick check; do not stage three searches for
  what one settles.
- "this approach has lower complexity in the general case" — read the actual reference,
  compare definitions, name the source.
- "Postgres handles X this way under load" — primary docs, possibly multiple sources,
  state which behavior is documented vs folkloric.

Cap: even thorough research should not become a research project. If a single claim
would need an hour of work or a deep literature review, that's a 🔍 — flag it and
move on rather than tunneling into one claim while leaving the others uninvestigated.
The skill's job is a thorough, honest pass over the whole turn — not a white paper on
one claim.

Heuristic: aim for the *job* to be heavy and the *writeup* to be light. If you're
writing more than you researched, you are doing this skill wrong.

## Portability: degrade gracefully by surface

- **Claude Code / terminal:** full power. Subagents, files, shell, tests, web search.
  Default fresh-context path applies; most Tier 1 claims resolve to ✅/❌; most Tier 3
  claims resolve via real research.
- **Cowork:** subagents plus files and code execution; default path applies.
- **Claude.ai / web chat:** no subagents, typically no local file or shell access. Use
  the in-context fallback with the same effort policy. Tier 1 code-action claims
  usually become ⚠️ Unsupported unless the file is actually present in the
  conversation. Tier 2 and (if web search is on) Tier 3 remain fully checkable.

When a whole tier is unreachable on the current surface, say so once in the preamble.

## Output format

ALWAYS produce a per-claim **narrative**, not a table. The structure:

(optional preamble line if scope was widened or a tier is unreachable on this surface)

```
### <short header capturing the claim> — <emoji> <Verdict label>

<one paragraph (2–5 sentences) that states the claim more fully and reports the
findings. For ✅/❌, name the source(s). For ⚠️/🔍, name the gap and what would
settle it.>
```

…then the next claim, same structure. Order claims by verdict severity:
❌ first, then ⚠️, then 🔍, then ✅, then 💬 — worst news first.

After the last claim, add a one-line bottom summary, e.g.:
`5 claims — 1 contradicted, 1 unsupported, 1 needs research, 2 verified.`

If a ❌ Contradicted result exists, follow the summary with ONE further line stating
plainly what is actually true.

Rules for the narrative:
- One section per distinct claim. Split compound claims into separate sections.
- The header is a short summary of the claim — not a quote of the full sentence.
- The icon and verdict label follow the header on the same line.
- The paragraph is prose — not bullets, not nested subheadings. Keep it readable.
- Don't pad. Most paragraphs should be 2–4 sentences; reserve longer ones for findings
  that genuinely need the words. If a paragraph is creeping past five sentences, ask
  whether more reading would replace the prose.
- Cite where you looked, briefly, inside the paragraph. Don't fabricate URLs.

## What not to do

- Do not skip research because the claim "sounds right". The whole point of this skill
  is to actually check.
- Do not grade Tier 1 claims from memory. Re-check the source or mark ⚠️.
- Do not inflate ⚠️ into ✅. If sources disagreed or you couldn't find a primary, say so.
- Do not collapse to 🔍 when you haven't tried. 🔍 is for the genuinely intractable.
- Do not turn the report into a research paper. Findings are summarized; the work
  lives in the tool calls, not the prose.
- Do not fact-check opinions. 💬 and move on.
- Do not be defensive about a ❌ against your own prior turn. The honest catch is the
  feature, not an embarrassment.

## Examples

**Example — mixed turn in Claude Code (full tool access):**

Preceding Claude turn: "I refactored `validateInput` in `form.js` to use the
early-return pattern, the test suite passes, Postgres 16 was released in 2022, and
the JSONB GIN index there is generally faster than B-tree for containment queries on
nested documents."

Audit output:

### Postgres 16 release year — ❌ Contradicted

Claude claimed Postgres 16 was released in 2022. The official PostgreSQL release
announcements and the project's versioning page record 16.0 shipping on
September 14, 2023, with 16.1 in November 2023 and no public Postgres 16 release
in 2022. The 2022 release was Postgres 15, on October 13, 2022.

### `validateInput` refactor to early-return — ✅ Verified

Re-read `form.js`: `validateInput` now exits on each failure case via `return`, with
no nested conditionals remaining around the success path. The structure matches the
early-return pattern as described.

### Test suite passes — ✅ Verified

Re-ran `npm test` against the working tree; 47 tests reported, 0 failures, 0 skipped.

### JSONB GIN faster than B-tree for nested containment — ✅ Verified

Confirmed via the Postgres 16 docs on JSON indexing (chapter 8.14.4) and the GIN
indexing chapter (66): for containment operators (`@>`, `?`, `?&`, `?|`) on JSONB,
GIN is the recommended index type; B-tree on JSONB does not support those operators
efficiently and is generally limited to equality on the whole document. The claim
holds in the form Claude stated.

`4 claims — 1 contradicted, 3 verified.`
Correction: Postgres 16 shipped on September 14, 2023, not in 2022.

**Example — same kind of turn on Claude.ai (no file access, web search on):**

(Preamble: "No shell/file access on this surface, so code-action claims are checked
against context only.")

### Postgres 16 release year — ❌ Contradicted

Researched via the official PostgreSQL release page: 16.0 released September 14, 2023,
not 2022.

### `validateInput` refactor to early-return — ⚠️ Unsupported

The contents of `form.js` are not in this conversation and there is no file access on
this surface, so the structural claim cannot be verified here. Sharing the current
file contents (or running this in Claude Code) would settle it.

### Test suite passes — ⚠️ Unsupported

No shell access on this surface; the test run cannot be reproduced. A pasted test
output, or running in Claude Code, would settle it.

### JSONB GIN faster than B-tree for nested containment — ✅ Verified

Confirmed via the official Postgres docs on JSON indexing and GIN indexing: GIN is
the documented index type for JSONB containment operators; B-tree does not support
them efficiently.

`4 claims — 1 contradicted, 2 unsupported, 1 verified.`
Correction: Postgres 16 shipped on September 14, 2023, not in 2022.
