---
name: eng-flow-retro
description: Production Stage 9 — blameless retrospective on a shipped (or in-progress) story, capturing durable learnings to a project-level log so future implementation work doesn't repeat the same mistakes. Distinct from a weekly velocity report — this is per-story, root-cause-focused, and only as useful as whether later stages actually consult what it logs.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Write
  - Bash
  - AskUserQuestion
triggers:
  - retro
  - retrospective
  - what did we learn
  - reflect on this
  - eng-flow retro
---

# eng-flow retro

Stage 9 of the production track. Reflects on a story that just shipped (or a specific bug/incident) and captures anything worth remembering into a project-level learnings log — not a weekly team-metrics dashboard, a blameless, root-cause-focused look at *this* piece of work. Only useful if what it logs actually gets read later — `eng-flow-implement` checks this log before starting new tasks (see its Step 0).

## Analytics

At the start of every step below (including Step 0), run `python3 .claude/skills/lib/bin/eng-flow-analytics-checkpoint eng-flow-retro "<step name>" "<story-slug>"`. As the last action of Step 5, run `python3 .claude/skills/lib/bin/eng-flow-analytics-finish eng-flow-retro "<story-slug>"`. See `eng-flow-spec`'s Analytics section for what this logs and why; rollup via `eng-flow-analytics` (Stage 10).

## Decision Ledger

Check `$ARGUMENTS` for a `--guide` token; if present, every decision point below gets an explicit `AskUserQuestion` instead of a silent default, and Step 5's report adds a "Decisions I made / decisions you made" summary. Log every decision point via `python3 .claude/skills/lib/bin/eng-flow-decision-log eng-flow-retro "<step>" <reason> <mode> <owner> "<description>" "<story-slug>"`. See `eng-flow-spec`'s Decision Ledger section for the taxonomy and why.

This skill is also the **consumer**, not just another logger — Step 1 below reads `eng-flow/decisions.jsonl` back and Step 3 turns what it finds into durable learnings. That's what makes the whole ledger worth running elsewhere: a decision nobody analyzes is dead weight, same principle `learnings.md` itself already runs on.

## Step 0 — Scope

Default: the story that just shipped (most recent `eng-flow-ship` run, or the story named in the request). Can also target a specific bug fix or incident directly, without a full story context, if that's what the user names.

Log it: `risk silent_decide ai_default "retro scope: <story/incident> — <how selected>"`.

---

## Step 1 — Gather signal

Don't work from memory or vibes — read what already exists:
- `tasks.md` — any task that needed rework, took multiple attempts, or hit a dependency that wasn't caught at breakdown time.
- `code-review.md` — findings raised, especially anything deferred or that recurred from a prior story.
- `qa-report.md` — bugs found that implementation or code review should have caught earlier.
- `git log <base>..<branch> --oneline` for the story's branch — commits like "fix," "revert," "oops," or multiple commits touching the same file in quick succession are signals of friction, not just noise.
- **`eng-flow/decisions.jsonl`** — filter to entries for this story/spec-slug (if the file doesn't exist yet, treat this source as empty, don't error). Look specifically for `owner: ai_default` or `ai_inferred` entries whose `description`, skill, or step overlaps a friction point already surfaced from the four sources above. A correlation there is a root-caused finding worth its own entry: "AI defaulted on X without asking (Step Y of skill Z); this caused/contributed to friction point W" — carry it into Step 2's reflection and Step 3's learnings, with a concrete Step 4 action item (e.g. "promote this decision point to `must_escalate`" or "add it to guide-mode's required-ask list"). This is the mechanism that lets the reason/mode/owner taxonomy get evidence-corrected over time instead of staying a static, never-revisited categorization.

---

## Step 2 — Reflect (blameless, system-focused)

For each friction point surfaced in Step 1:

1. **What happened** — factual, no blame. "System language, not person language" (a process/check/step was missing, not "X made a mistake").
2. **What let it through** — which stage, or absence of a stage, should have caught this but didn't?
3. **Root cause** — ask "why" until reaching the actual cause, not where it happened to surface (reused from `agent-skills`' `debugging-and-error-recovery`, attribution: `docs/DECISIONS.md`). A symptom fix isn't the answer here; the process gap is.

If nothing genuinely notable surfaced in Step 1, say so — a clean retro is a valid outcome, don't manufacture friction to fill the step.

Log it: `risk silent_decide ai_default "friction found: <yes, N points|none — clean retro>"`.

---

## Step 3 — Capture durable learnings

For each genuine discovery, log an entry — **the bar is "would this save time in a future session?"**, not "is this true." Don't log things the user already knows or that are obvious from reading the code.

Type each entry:
- **pattern** — a reusable approach that worked
- **pitfall** — something to not do again
- **preference** — something the user explicitly stated
- **architecture** — a structural decision and why
- **tool** — a library/framework insight
- **operational** — project environment/workflow knowledge

Score confidence 1-10 (an observed, verified pattern is 8-9; a user-stated preference is 10; an inference you're not fully sure of is 4-5).

Log each entry: `risk silent_decide ai_inferred "learning '<key>' (<type>, confidence <N>): <one-line insight>"`.

---

## Step 4 — Action items

For anything that should change a checklist, a stage's steps, or a convention — not just "be more careful next time" — write a concrete action item with an owner (defaults to "next implementer" if no one else is named). If the fix is a missing regression test, name it explicitly rather than leaving it as a vague intention.

Log each: `risk silent_decide ai_default "action item: <what> — owner: <who>"`.

---

## Step 5 — Save and report

Write the full reflection to `eng-flow/backlog/stories/<story-slug>/retro.md`:

```markdown
# Retro: [story name]

## What happened
[Step 2, per friction point]

## Root causes
[Step 2.3]

## Action items
- [ ] [Action] — owner: [who] — [what stage/checklist this changes]
```

Append the distilled entries from Step 3 to the project-level `eng-flow/learnings.md` (create it if it doesn't exist yet):

```markdown
### [date] — [type] — [short key]
**Insight:** [description]
**Confidence:** [1-10]
**Source:** [observed | user-stated | inferred]
**Files:** [relevant paths, if any — enables noticing when a learning goes stale]
```

Report a summary: what was learned, what action items were created, and confirm both files were saved.

If this run was in guide mode, add a "Decisions I made / decisions you made" summary here, drawn from this run's `eng-flow-decision-log` calls.

Run the Step 5 analytics-finish call (see Analytics section above) before ending.
