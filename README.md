# Says Who

A Claude Code plugin with two skills for auditing the claims in Claude's most recent reply. Both share a five-verdict scheme, ordered worst-news-first; they differ in how much effort they spend and how they present the result.

**The point:** the single most damaging failure mode in AI assistance is the confident false "done" — claiming a file was edited when it wasn't, a number when it was invented, a fact when it was a vibe. These skills are the antidote.

## The two skills

### `says-who` — fast scan, table output

A quick, scannable audit. Roughly one tool call per claim. Output is a table, one row per claim, ordered worst-news-first.

Fires whenever the user casually challenges the previous turn:

- "says who?"
- "prove it"
- "back that up"
- "show your receipts"
- "did you actually do that?"
- "really? are you sure?"

### `prove-it` — deep research, narrative output

Real investigation: multiple sources per non-trivial Tier 3 claim, primary documents preferred, evidence cited. Output is a per-claim narrative — header, verdict icon, one paragraph stating the claim and findings.

Fires on explicit requests for depth:

- "really prove it"
- "do a thorough check"
- "fact-check this carefully"
- "verify in depth"
- "investigate what you said"
- "research these claims"
- "I want sources"
- "do a comprehensive audit"

Effort is proportional — simple claims still get short checks. The skill aims for the *job* to be heavy and the *writeup* to be light.

## The five verdicts (shared)

| Emoji | Label | Meaning |
|---|---|---|
| ✅ | Verified | Checked against ground truth and holds |
| ❌ | Contradicted | Checked and does NOT hold — the high-value catch |
| ⚠️ | Unsupported | Couldn't confirm with what's available |
| 🔍 | Needs research | Would take real effort to verify (rare in `prove-it`) |
| 💬 | Judgment | Opinion / recommendation — not a factual claim |

## How they stay honest

The auditor never sees the authoring turn's reasoning — only the extracted claims plus access to ground truth (the file, the command, the web). Where subagents are available (Claude Code, Cowork), the audit runs in fresh contexts so the "I intended to do it" bias from the original turn can't leak in. Where they aren't (Claude.ai web chat), a preamble warns the user that self-claims are checked under reduced rigor.

`says-who` caps effort at roughly one tool call per claim; anything heavier gets marked 🔍. `prove-it` removes that cap and actually invests the research, while still bailing to 🔍 when a single claim would require an hour or a literature review.

## Installing

This repo is a Claude Code plugin marketplace. Install it via:

```
/plugin marketplace add wboelaars/says-who
/plugin install says-who
```

For local development, clone the repo and point Claude Code at the directory:

```
/plugin marketplace add /path/to/Says-who
/plugin install says-who
```

If you just want one of the skills on its own (no plugin manifest), drop `skills/says-who/` or `skills/prove-it/` into `~/.claude/skills/` (or a project's `.claude/skills/`).

## Repo layout

```
.claude-plugin/
├── marketplace.json   ← makes the repo a /plugin marketplace add target
└── plugin.json        ← the plugin's own manifest
skills/
├── says-who/
│   └── SKILL.md       ← fast table-style audit
└── prove-it/
    └── SKILL.md       ← deep research, narrative output
evals/                 ← test prompts and fixtures (not packaged)
└── evals.json
README.md
LICENSE
```

## Status

v1.1. Iteratively developed with the `skill-creator` workflow.
