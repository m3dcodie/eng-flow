---
name: eng-flow-analytics
description: Production Stage 10 — on-demand rollup report of eng-flow/analytics.jsonl (time/token log) and eng-flow/findings.jsonl (bug counts), both incrementally written by other eng-flow skills. Reports per-story and per-stage totals, tokens by category, cycle time (elapsed vs. active), stage-transition gaps, review/QA rework cycles, and bug rate. Read-only; doesn't instrument itself.
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

Stage 10, on-demand and read-only. Every other eng-flow skill (Stages 1 through 9) logs its own time and token usage incrementally, step by step, to a project-level `eng-flow/analytics.jsonl`; `eng-flow-code-review` and `eng-flow-qa` additionally log severity-tagged finding counts to `eng-flow/findings.jsonl`. This skill just rolls both logs up into something readable. It doesn't log anything itself.

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

Show the per-story and per-skill/stage breakdown as returned, plus the cycle-time, rework-cycle, and bug-rate sections. Points worth calling out explicitly, not just dumping the raw output:
- Which stage is consuming the most time/tokens across the project — this is the kind of signal worth acting on (e.g., if `eng-flow-implement` dominates every story's total, that's expected; if `eng-flow-code-review` does, something about the review loop may be thrashing).
- `cache_read_input_tokens` is typically the largest number by far in a long-running session — that's expected (cached context being re-read each turn) and much cheaper per-token than `input_tokens`/`output_tokens`/`cache_creation_input_tokens`; don't alarm the user over a large cache-read figure alone.
- A story's **elapsed** cycle time far exceeding its **active** time isn't automatically a problem — it usually just means the story sat idle between sessions/stages. The per-stage-transition gaps underneath it show *where* the idle time landed, which is the actionable part (e.g., a large spec→architecture gap vs. a large implement→code-review gap point at different bottlenecks).
- Any skill flagged with more than one pass (`eng-flow-code-review`/`eng-flow-qa` rework/re-review loops) — call these out by name; a story that went through 3 review passes is a different signal than one that passed clean the first time.
- The bug-rate section (only shown if `eng-flow/findings.jsonl` has data): Critical/Required findings are the ones worth trending — a rising rate across stories may point at a process gap earlier in the pipeline (spec ambiguity, architecture review missing something) rather than an implementation-quality problem per se. Don't editorialize about Nit counts; they're noise-level by design.
- Any "recovered" segments the report flags — these came from an interrupted run (killed terminal, system restart) rather than a clean finish. The data is still real, just call out that it exists so the user knows why a story's step count might look odd.

Don't compute or state a dollar-cost estimate — pricing changes and isn't tracked here; point to Anthropic's current pricing page or the Claude Code `/cost` command if the user wants that conversion.

---

## How the data gets there (for context, not something this skill does)

Every other eng-flow skill runs `eng-flow-analytics-checkpoint` at the start of each of its own steps and `eng-flow-analytics-finish` at the end (see any other skill's "Analytics" section, e.g. `eng-flow-spec`). Checkpointing happens step by step, not just once at skill start/end, specifically so a killed terminal or system restart loses at most the current in-flight step's data — not the whole run. An orphaned marker from a crash gets flushed and tagged `"outcome":"recovered"` the next time that skill starts, rather than silently discarded.

Both scripts degrade silently if `$CLAUDE_CODE_SESSION_ID` or the session transcript aren't available (non-Claude-Code host, or the transcript format changes in a future version) — they log time only, `usage: null`, and never block the calling skill's real work.

Separately, `eng-flow-code-review` (Step 7) and `eng-flow-qa` (Step 5) each run `eng-flow-findings-log` once, at save time, logging the severity-tagged finding counts they already computed for their own `.md` reports to `eng-flow/findings.jsonl`. This is a single point-in-time append, not a checkpoint/marker pair — there's no in-progress state to crash-recover.

Cycle time, stage-transition gaps, and rework-cycle counts are pure derivations of `analytics.jsonl`'s existing `ts_start`/`ts_end`/`run_id` fields — no separate instrumentation feeds them.
