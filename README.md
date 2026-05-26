# Says Who

A Claude skill that audits the claims in Claude's most recent reply and reports — in a short, scannable table — which ones actually hold up. One row per claim, five possible verdicts, ordered worst-news-first.

**The point:** the single most damaging failure mode in AI assistance is the confident false "done" — claiming a file was edited when it wasn't, a number when it was invented, a fact when it was a vibe. Says Who is the antidote.

## When it fires

Whenever the user challenges Claude's previous turn:

- "says who?"
- "prove it"
- "back that up"
- "show your receipts"
- "did you actually do that?"
- "really? are you sure?"

…or any other casual expression of doubt about what Claude just said. It's especially useful after Claude reports that it finished work or made factual/numerical assertions.

## The five verdicts

| Emoji | Label | Meaning |
|---|---|---|
| ✅ | Verified | Checked against ground truth and holds |
| ❌ | Contradicted | Checked and does NOT hold — the high-value catch |
| ⚠️ | Unsupported | Couldn't confirm with what's available |
| 🔍 | Needs research | Would take real effort to verify |
| 💬 | Judgment | Opinion / recommendation — not a factual claim |

## How it stays honest

The auditor never sees the authoring turn's reasoning — only the extracted claims plus access to ground truth (the file, the command, the web). Where subagents are available (Claude Code, Cowork), the audit runs in fresh contexts so the "I intended to do it" bias from the original turn can't leak in. Where they aren't (Claude.ai web chat), a preamble warns the user that self-claims are checked under reduced rigor.

The skill caps effort at roughly one tool call per claim. Anything heavier gets marked 🔍 rather than burning tokens on a deep investigation. If the user wants a deep dive on a flagged claim, they'll ask.

## Installing

Drop the `says-who/` folder into your skills directory:

- **Claude Code:** `~/.claude/skills/` (or a project's `.claude/skills/`)
- **Cowork:** install as a plugin skill

Or package it into a `.skill` file:

```bash
python -m scripts.package_skill says-who
```

…and install the resulting `says-who.skill`.

## Repo layout

```
says-who/             ← the installable skill
└── SKILL.md
evals/                ← test prompts and fixtures (not packaged)
└── evals.json
README.md
LICENSE
```

## Status

v1. Iteratively developed with the `skill-creator` workflow.
