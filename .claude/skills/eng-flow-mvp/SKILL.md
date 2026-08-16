---
name: eng-flow-mvp
description: MVP mode — the fast, low-ceremony counterpart to the ten-stage production track (PROCESS.md). Captures a one-pager, breaks it into a flat checklist (no epics/stories), gives a ballpark time/token estimate and gets an explicit go/no-go, then runs an auto-continuous implement loop that keeps committing (and, if the user opts in, pushing) until the checklist is done or a hard-stop condition hits, then hands off to eng-flow-ship. Self-triggerable — reuses an existing eng-flow-spec spec.md if one exists instead of re-capturing it, but doesn't require one.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Write
  - Edit
  - Bash
  - AskUserQuestion
  - Agent
triggers:
  - mvp this
  - build mvp
  - quick build
  - move fast on this
  - prototype this
  - eng-flow mvp
  - ship this fast
---

# eng-flow mvp

Implements "MVP mode" from `PROCESS.md`: move fast, autonomously, minimal ceremony. One continuous run — one-pager → checklist → estimate/go-no-go → branch → auto-continuous implement loop → local check → ship — instead of the production track's ten gated stages. Skips Stages 2, 3, 3.5, 3.6, 4, 6 (full 5-axis), 7 (formal), 9, 10 outright; still logs Analytics and the Decision Ledger the same as every other eng-flow skill, and still hard-stops on irreversible/high-risk work — MVP mode trims ceremony, not the safety floor.

## Analytics

At the start of every numbered step below (including Step 0), run `python3 .claude/skills/lib/bin/eng-flow-analytics-checkpoint eng-flow-mvp "<step name>" "<dated-slug-if-known>"`. As the last action of Step 6, run `python3 .claude/skills/lib/bin/eng-flow-analytics-finish eng-flow-mvp "<dated-slug>"`. See `eng-flow-spec`'s Analytics section for what this logs and why; rollup via `eng-flow-analytics` (Stage 10).

## Decision Ledger

Check `$ARGUMENTS` for a `--guide` token; if present, every decision point below gets an explicit `AskUserQuestion` instead of a silent default, and Step 6's report adds a "Decisions I made / decisions you made" summary. Log every decision point via `python3 .claude/skills/lib/bin/eng-flow-decision-log eng-flow-mvp "<step>" <reason> <mode> <owner> "<description>" "<dated-slug-if-known>"`. See `eng-flow-spec`'s Decision Ledger section for the taxonomy and why. Rollup/analysis: `eng-flow-retro` Step 1 (Stage 9), same as every other skill's ledger entries.

## Step 0 — Entry: resume check, then scope sanity check

**Resume check first.** Look for `eng-flow/mvp/*/checklist.md` with unchecked items whose topic matches `$ARGUMENTS` (or, if none named, ask which to resume if more than one has unchecked items). If found, confirm resuming it with the user, then skip straight to Step 3 (branch setup — re-verify the branch wasn't lost) and continue from Step 4. This is what makes an interrupted run resumable: the checklist's own checkboxes are the state, nothing else to reconstruct.

**Otherwise, scope sanity check.** If `eng-flow/specs/` or `eng-flow/backlog/` already contain production-track work for this project, ask once whether this new piece really belongs in MVP mode or should go through Stage 4/5 instead — don't silently downgrade rigor on a project that's already graduated. Log it: `risk open_question user_confirmed "MVP mode confirmed for <topic> despite existing production-track work"` when that applies.

---

## Step 1 — One-pager

**If a matching `eng-flow/specs/<slug>/spec.md` already exists** (from `eng-flow-spec`), show its key sections (problem, scope, success criteria) to the user and ask whether it still reflects what they want built — confirm or revise, never assume it's still current silently. Log it: `risk open_question <user_confirmed|user_revised> "existing spec <slug>: <confirmed as-is|revised: what changed>"`.

**Then, regardless of whether a spec existed,** walk the trimmed set from `PROCESS.md`'s "Idea validation" + MVP mode sections:
1. Who has this problem, concretely?
2. What do they do today without it? (status quo)
3. Why now?
4. What's the narrowest version that proves it's wanted?
5. What's explicitly out of scope for this first cut?
6. Done-when — what's the specific, testable condition that means this is finished?

Where an existing spec already answers one of these clearly, restate the AI's understanding of that answer and ask for a quick confirm/correct instead of re-asking from scratch — the goal here is closing ambiguity before committing to a checklist, not a redundant re-interrogation. If an answer is hand-wavy, press once for specifics; don't draft around vagueness.

Draft:
```markdown
# MVP: [Name]
## Problem
## Who / Status Quo
## MVP Cut (narrowest scope)
## Out of Scope
## Done-When
```
Show the draft, ask "does this capture it?" Quick manual skim for secrets/credentials/PII before saving — don't save silently if something looks off. Log it: `risk open_question <user_confirmed|user_revised> "one-pager: <accepted|revised: what changed>"`.

Save to `eng-flow/mvp/<dated-slug>/one-pager.md`, dated-slug computed the same way as `eng-flow-spec` Step 7 (lowercase, spaces to dashes, strip to `[a-z0-9._-]`, date-prefixed) — a sibling of `eng-flow/specs/`, not nested inside it, so MVP and production-track artifacts stay visually distinct.

---

## Step 2 — Flat checklist

Break the MVP cut into a flat, dependency-ordered checklist. Reuse the task-quality bar from Stage 4 Step 7 (`eng-flow-epics-stories-tasks`) — vertical slices not horizontal layers, sized XS/S/M/L (split anything that'd land XL), acceptance criteria, files likely touched — but skip epic/story wrapping entirely; there's no backlog here, just a list.

```markdown
- [ ] Task: [title] — acceptance: [specific, testable] — files: [path, path] — size: [XS|S|M|L]
```

Save to `eng-flow/mvp/<dated-slug>/checklist.md`. This file's checkboxes are the only state Step 4's loop needs to be resumable.

---

## Step 2.1 — Estimate + go/no-go

Before touching a branch or writing any code, give the user a ballpark derived from the checklist's task-size mix — order-of-magnitude calibration, not a committed estimate:

- **Human-dev-time equivalent** — rough hours/days if a person did this by hand.
- **AI-assisted wall-clock estimate** — rough minutes/hours for this loop to run.
- **Token-usage range** — rough order of magnitude (e.g. "~150k–300k tokens"), scaled from task count and size mix.

Present as a range with the task-size breakdown that produced it (e.g. "6 tasks: 2 XS, 3 S, 1 M → ~X"), not a bare number — the breakdown is what makes the estimate legible enough to sanity-check. Ask an explicit go/no-go via `AskUserQuestion`. On no-go, stop here and let the user revise the one-pager or checklist before re-running this skill. Log it: `risk open_question user_confirmed "estimate presented (<ballpark>): <go|no-go>"`.

---

## Step 3 — Branch setup

Ask which base branch to target — check `git branch -r` / `git remote show origin` first and offer the real candidates found (typically `main` or `develop`) rather than assuming one; teams differ on this convention and guessing wrong here is expensive to unwind later. Log it: `knowledge_asymmetry open_question user_confirmed "base branch: <chosen>"`.

Once confirmed: pull latest base (`git fetch origin <base> && git pull`), then — unless already on a branch matching this MVP's slug — create `feature/<slug>` (or `fix/<slug>` if this is a bug fix) and switch to it. Never work directly on the base branch. If uncommitted changes unrelated to this MVP are present, stop and ask how to handle them before switching/creating branches.

---

## Step 4 — Auto-continuous implement loop

This is the actual gap versus Stage 5 (`eng-flow-implement`), which deliberately does one task per run and stops. Here, once started, the loop runs through **all** unchecked checklist items in one pass.

**Before looping**, check `git remote -v`. If a remote exists, ask once whether to push each completed task's commit to remote as it lands, or keep commits local-only until Step 5's ship step — recommend pushing (protects in-progress work against a local machine or session failure) but let the user decide. If no remote is configured, skip the question, commit locally only, and log it: `risk silent_decide ai_default "no remote configured: committing locally only"`.

**Per checklist item:**
1. Implement the smallest complete vertical slice for that item.
2. Write a test if the repo has a test framework and the item implies observable behavior (not mandatory TDD ceremony — if there's no test setup, say so once and proceed, don't ask per item).
3. Run the existing test suite (regression check) and build/lint if the stack has them.
4. Commit — one item, one commit, descriptive message. Push if enabled above.
5. Check the item off in `checklist.md`.

**Never stop for:** moving to the next checklist item once the current one's tests/build pass, committing a completed item, pushing the feature branch (if enabled), minor lint/format auto-fixes.

**Always stop and ask** (identical floor to Stage 5 Step 4 — MVP mode does not loosen this):
- A test or the build breaks with no obvious fix.
- A checklist item needs a decision the one-pager doesn't cover.
- The item is high-risk or hard to reverse: auth/permissions, destructive migrations, payments, deletions, deploys, anything touching secrets.
- Merging or pushing to the base/main branch — always explicit confirmation, per the user's global `CLAUDE.md` hard exception; never bypassed regardless of mode.

Log each hard stop: `risk must_escalate user_confirmed "<which condition>: <resolution>"`.

If interrupted mid-loop (crash, closed terminal), re-invoking this skill resumes at Step 0's resume check and picks up at the first unchecked item — no separate crash-recovery file needed.

---

## Step 5 — Local verification, then ship

Once every checklist item is checked and the full suite/build passes, don't auto-invoke ship. Ask the user whether they want to test it locally first (run the app, smoke-test the MVP cut) — this is the normal MVP pattern: all tasks done, green build, then a human look before it goes out. Once the user confirms satisfied (or explicitly says to skip local testing), delegate to `eng-flow-ship` (Stage 8) rather than reimplementing merge/version/PR logic here — it already degrades correctly when no story is associated with the branch (its Step 2 asks once to confirm skipping the code-review/QA gate check, then proceeds normally).

---

## Step 6 — Report + graduation check

Summarize: checklist items completed, commits made, anything flagged as noticed-but-not-touched, and (if Step 5 stopped early) how many items remain unchecked. Only if real signals suggest it — multiple stakeholders now involved, the codebase needs to outlive this sprint, a handoff to another engineer looks likely — mention the Graduation gate (`PROCESS.md`) exists and ask if the user wants to continue from here in production mode. Don't raise it by default just because the MVP worked.

If this run was in guide mode, add a "Decisions I made / decisions you made" summary here, drawn from this run's `eng-flow-decision-log` calls.

Run the Step 6 analytics-finish call (see Analytics section above) before ending.
