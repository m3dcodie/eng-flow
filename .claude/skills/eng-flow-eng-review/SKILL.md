---
name: eng-flow-eng-review
description: Production Stage 3.5 — reviews a saved architecture.md against engineering-leadership judgment before it becomes epics/stories/tasks. Catches architecture problems while they're still cheap to fix (before backlog breakdown, before implementation), not a code review — no code exists yet at this point in the track.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Write
  - Bash
  - AskUserQuestion
  - Agent
triggers:
  - engineering review
  - review the architecture
  - review this design
  - eng review
  - architecture review
---

# eng-flow engineering review

Stage 3.5 of the production track, between Architecture and Epics/Stories/Tasks. Reviews `architecture.md` for the kind of problems that are cheap to fix now and expensive to fix after a backlog is built on top of them — this is a design review, not a code review; no code exists yet at this point in the track.

## Analytics

At the start of every numbered step below (including Step 0), run `python3 .claude/skills/lib/bin/eng-flow-analytics-checkpoint eng-flow-eng-review "<step name>" "<dated-slug>"`. As the last action of Step 6, run `python3 .claude/skills/lib/bin/eng-flow-analytics-finish eng-flow-eng-review "<dated-slug>"`. See `eng-flow-spec`'s Analytics section for what this logs and why; rollup via `eng-flow-analytics` (Stage 10).

## Decision Ledger

Check `$ARGUMENTS` for a `--guide` token; if present, every decision point below gets an explicit `AskUserQuestion` instead of a silent default, and Step 6's report adds a "Decisions I made / decisions you made" summary. Log every decision point via `python3 .claude/skills/lib/bin/eng-flow-decision-log eng-flow-eng-review "<step>" <reason> <mode> <owner> "<description>" "<dated-slug>"`. See `eng-flow-spec`'s Decision Ledger section for the taxonomy and why. Rollup/analysis: `eng-flow-retro` Step 1 (Stage 9).

## Step 0 — Find the inputs

Look for `eng-flow/specs/*/architecture.md` and the `domain-model.md` / `spec.md` in the same folder. If `architecture.md` is missing, tell the user to run Stage 3 first — there's nothing to review yet. If more than one spec folder exists, ask which one this run is for.

Read all three: the spec's constraints and NFRs, the domain model's entities/relationships, and the architecture's tech stack, diagram, API contracts, NFR targets, and deployment topology.

---

## Step 1 — Architecture review checklist

Walk `architecture.md` against:
- **Component boundaries and coupling** — do the container-level boxes reflect real seams, or is this one thing pretending to be several (or several things pretending to be one)?
- **Data flow bottlenecks** — any path where everything funnels through one component under the stated load/scale targets?
- **Scaling characteristics and single points of failure** — does the deployment topology have one of anything that can't be one of anything?
- **Security architecture** — auth model, data-at-rest/in-transit handling, API boundary trust assumptions — actually specified, not just "secure" as an adjective.
- **Failure scenario per integration point** — for each external dependency or service boundary in the diagram, is there a stated realistic failure mode and whether the design accounts for it?
- **Distribution path** — if this introduces a new deployable artifact, is the build/publish/deploy path part of the doc (or explicitly deferred), not silently assumed?

---

## Step 2 — Cognitive-pattern lenses

Apply these where relevant — not every pattern applies to every architecture, don't force one in. Adapted from gstack's `plan-eng-review` (attribution: `docs/DECISIONS.md`), scoped down to the seven that apply before any code exists:

1. **Boring by default** — every project gets a few "innovation tokens." Is anything novel/unproven in Step 1 of the architecture spending one without a stated reason?
2. **Blast radius** — for each major decision, what's the worst case, and how much does it affect if wrong?
3. **Conway's Law** — does the container boundary match how the team is actually structured, or does it assume coordination that won't happen?
4. **Essential vs. accidental complexity** — for anything non-obvious in the design, is it solving a real constraint from the spec, or one the design created?
5. **Two-week smell test** — could a competent engineer joining the project ship a small, representative feature within this architecture in about two weeks? If not, that's an onboarding problem wearing an architecture costume.
6. **Incremental over revolutionary** — does the deployment/rollout path allow incremental delivery (staged rollout, feature flags), or does it require a big-bang cutover?
7. **Reversibility preference** — for the decisions marked expensive-to-reverse (the ones that got ADRs in Stage 3), is the cost of being wrong actually low, or does the doc lock in something hard to undo without saying so?

---

## Step 3 — Independent subagent review

Steps 1-2 ran in this conversation — often the same one that just wrote `architecture.md` in Stage 3. That's a self-review risk: the reasoning that produced a decision is the reasoning most likely to rubber-stamp it. Counter it with one blind pass.

Spawn a single `Agent` call (foreground — its output feeds Step 4, so wait for it), general-purpose, with a prompt that gives it **only the file paths, not this conversation's context or Steps 1-2's findings**:

> "Read `eng-flow/specs/<dated-slug>/spec.md`, `domain-model.md`, and `architecture.md`. You are an independent senior engineer reviewing this architecture — you have not seen any prior discussion of it and have no stake in the decisions. Evaluate: (1) component boundaries and coupling, (2) data flow bottlenecks and scaling, (3) security architecture, (4) failure modes per integration point, (5) hidden or accidental complexity, (6) whether the ADRs' 'alternatives considered' actually hold up. For each finding: what's wrong, severity, the `architecture.md` section it reacts to, and a fix."

Adapted from gstack's `autoplan` dual-voice eng review (attribution: `docs/DECISIONS.md`) — without its Codex cross-check or consensus-table mechanics, since eng-flow has no second external reviewer and Step 4 already gets each finding an explicit human accept/change/reject.

Merge the subagent's findings with Steps 1-2's into one list for Step 4, tagging each by source (`checklist` / `cognitive lens` / `independent review`). Drop exact duplicates, but keep two findings that reach the same conclusion by different reasoning — that agreement is itself informative.

---

## Step 4 — Interactive review, one issue at a time

For each issue surfaced in Steps 1-3, call `AskUserQuestion` individually — one issue per call, never batched. For each: name the issue, note its source tag, ground it in the specific section/decision of `architecture.md` it's reacting to (quote or point to the relevant line — a finding that can't be tied to actual doc text doesn't get raised), state options, give an opinionated recommendation, explain why.

**Stop and wait for the user's answer before raising the next issue.** Don't assume the "obvious fix" and move on — every issue gets an explicit accept/change/reject from the user, same discipline as every other stage in this track.

Log each: `risk open_question <user_confirmed|user_revised> "issue '<title>': accepted|changed|rejected"`.

If Steps 1-3 turn up nothing worth raising, say so plainly — a clean pass is a valid outcome, don't manufacture issues to fill the step.

---

## Step 5 — Record the outcome

Write `eng-flow/specs/<dated-slug>/eng-review.md`:

```markdown
# Engineering Review: [Name]
(source: architecture.md in this folder)

## Findings

### [Issue title]
**Source:** [checklist | cognitive lens | independent review]
**Section:** [architecture.md section/decision this reacts to]
**Issue:** [what's wrong and why]
**Resolution:** [Accepted — architecture.md updated | Changed — describe | Rejected — reason]
```

For any finding marked "Accepted" or "Changed," update `architecture.md` directly to reflect the resolution — the review doc is the log, `architecture.md` stays the current source of truth.

---

## Step 6 — Report back

Summarize what was reviewed, what changed (if anything), and confirm `architecture.md` is up to date. Tell the user this feeds Stage 4 (epics/stories/tasks).

If this run was in guide mode, add a "Decisions I made / decisions you made" summary here, drawn from this run's `eng-flow-decision-log` calls.

Run the Step 6 analytics-finish call (see Analytics section above) before ending.
