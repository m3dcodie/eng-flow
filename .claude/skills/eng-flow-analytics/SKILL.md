---
name: eng-flow-analytics
description: Production Stage 10 — on-demand rollup report of eng-flow/analytics.jsonl, the incremental time/token log every other eng-flow skill writes to. Reports per-story and per-stage totals, tokens broken down by category (input/output/cache-write/cache-read). Read-only; doesn't instrument itself.
allowed-tools:
  - Read
  - Bash
  - AskUserQuestion
triggers:
  - analytics
  - how much time did this take
  - token usage
  - eng-flow analytics
  - engineering metrics
---

# eng-flow analytics

Stage 10, on-demand and read-only. Every other eng-flow skill (Stages 1 through 9) logs its own time and token usage incrementally, step by step, to a project-level `eng-flow/analytics.jsonl` — this skill just rolls that log up into something readable. It doesn't log anything itself.

## Step 0 — Scope

Ask (or infer from the request) whether this is a whole-project report or scoped to one story. Default: whole project.

---

## Step 1 — Run the rollup

```bash
python3 .claude/skills/lib/bin/eng-flow-analytics-report [story-slug]
```

If `eng-flow/analytics.jsonl` doesn't exist yet or is empty, say so plainly — no data has accumulated yet, nothing to report. This isn't an error.

---

## Step 2 — Present

Show the per-story and per-skill/stage breakdown as returned. Points worth calling out explicitly, not just dumping the raw output:
- Which stage is consuming the most time/tokens across the project — this is the kind of signal worth acting on (e.g., if `eng-flow-implement` dominates every story's total, that's expected; if `eng-flow-code-review` does, something about the review loop may be thrashing).
- `cache_read_input_tokens` is typically the largest number by far in a long-running session — that's expected (cached context being re-read each turn) and much cheaper per-token than `input_tokens`/`output_tokens`/`cache_creation_input_tokens`; don't alarm the user over a large cache-read figure alone.
- Any "recovered" segments the report flags — these came from an interrupted run (killed terminal, system restart) rather than a clean finish. The data is still real, just call out that it exists so the user knows why a story's step count might look odd.

Don't compute or state a dollar-cost estimate — pricing changes and isn't tracked here; point to Anthropic's current pricing page or the Claude Code `/cost` command if the user wants that conversion.

---

## How the data gets there (for context, not something this skill does)

Every other eng-flow skill runs `eng-flow-analytics-checkpoint` at the start of each of its own steps and `eng-flow-analytics-finish` at the end (see any other skill's "Analytics" section, e.g. `eng-flow-spec`). Checkpointing happens step by step, not just once at skill start/end, specifically so a killed terminal or system restart loses at most the current in-flight step's data — not the whole run. An orphaned marker from a crash gets flushed and tagged `"outcome":"recovered"` the next time that skill starts, rather than silently discarded.

Both scripts degrade silently if `$CLAUDE_CODE_SESSION_ID` or the session transcript aren't available (non-Claude-Code host, or the transcript format changes in a future version) — they log time only, `usage: null`, and never block the calling skill's real work.
