---
name: eng-flow-spec
description: Front door for new work — routes an idea, a stakeholder-greenlit product, or a feature/story into the right requirements process, interrogates for a spec, optionally runs bounded competitor research, and saves to eng-flow/specs/. Pure requirements only — never asks technical questions except on the feature/story path where code already exists.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Write
  - Bash
  - AskUserQuestion
  - WebSearch
  - WebFetch
triggers:
  - spec this
  - write a spec
  - new feature
  - new product idea
  - eng-flow spec
  - start a spec
  - is this worth building
---

# eng-flow spec

Turns a rough idea, an already-approved product/feature, or a new story into a saved spec — asking only what the current stage owns. See `PROCESS.md` at the repo root for the full model this implements.

## Analytics

At the start of every numbered step below (including Step 0), run `python3 .claude/skills/lib/bin/eng-flow-analytics-checkpoint eng-flow-spec "<step name>" "<dated-slug-if-known>"`. As the last action of Step 8, run `python3 .claude/skills/lib/bin/eng-flow-analytics-finish eng-flow-spec "<dated-slug>"`. Logs time and token usage per step, incrementally, to `eng-flow/analytics.jsonl` — a killed terminal or restart loses at most the in-flight step, not the whole run. Degrades silently if unavailable; never blocks real work. Rollup: `eng-flow-analytics` (Stage 10).

## Step 0 — Topic and routing

`$ARGUMENTS` is the rough topic. If empty, ask: "What are we speccing?"

Then ask the routing question — this determines everything downstream, so don't skip or infer it silently:

> "Has this already been decided by someone with the authority to approve it, or are you the one deciding?"

Offer three paths via `AskUserQuestion`:
- **Not decided yet** — you're weighing whether to build this
- **Already approved** — a stakeholder greenlit this, you're capturing what was decided
- **Feature/story in an existing product** — domain model and architecture already exist, this extends them

The path chosen governs which steps below run. Do not let "this looks like a startup idea" or "this looks big" substitute for actually asking — greenlit enterprise work and solo weekend ideas can look identical from the topic string alone.

---

## Step 1 (path: not decided) — Idea validation

Trimmed forcing questions, asked until answered concretely — not restated requests:

1. Who has this problem, concretely? (a specific person/role, not "everyone")
2. What do they do today without this? (status quo)
3. Why now — what's changed that makes this worth building today?
4. What's the narrowest version that would prove it's actually wanted?

If these can't be answered without hand-waving, say so plainly and stop — offer to keep refining the idea rather than proceeding to a spec that will just encode the vagueness. If they survive, continue to Step 2.

Paths "already approved" and "feature/story" skip this step entirely — the demand question isn't the engineer's to re-litigate on those paths.

---

## Step 2 — Phase 1: Decision capture / Why

**Path: already approved** — ask until answered:
1. What did stakeholders approve, in their own words?
2. Who are the stakeholders/sponsors — is there a written source (deck, doc, email) to cite?
3. What business outcome is expected, and how will success be measured?
4. What constraints came with the approval (budget, timeline, compliance, must-integrate-with-X)?

**Path: not decided** (idea already validated in Step 1) or **feature/story** — lighter, mirrors the validation answers:
1. What problem does this solve, concretely, for whom?
2. What happens if this doesn't get built?
3. How will we know it worked?

Don't move on until answers are specific and non-circular.

---

## Step 3 — Phase 2: Audience, scope, and domains

All paths, ask until answered:

1. Who is the actual end user? (may differ from the approving stakeholder)
2. What's the smallest slice of scope that delivers real value — the MVP-cut?
3. What's explicitly out of scope for this first cut? Lock this now — it prevents the spec from growing later.
4. Which high-level domains/functional areas does this touch or introduce? (e.g. checkout, inventory, billing, support — a flat list, not a model. This seeds the domain-modeling stage, if this work reaches production mode.)

For the "already approved" or "not decided" paths (new product/feature, not a small story), also capture:
- User journeys/stories — narrative scenarios ("As a [user], I want to [action], so that [benefit]")
- Functional requirements — what the system must do, as a list
- Non-functional requirements — performance, security, availability, compliance, scalability. Ask explicitly; these get skipped if not prompted for.

---

## Step 4 (path: feature/story only) — Technical grounding

**Hard requirement: read the relevant code before asking any technical question here.** Grep/Read the actual files this touches first — this is the one path where code already exists and embodies real constraints worth reading before asking.

Ask targeted questions about whichever apply (skip what clearly doesn't):
- Where this plugs into existing code (cite real files/functions found)
- Data model / API shape changes
- Edge cases the existing code already handles that this must not break

Other paths skip this entirely — there's no code yet, and asking technical questions this early re-mixes concerns that belong to the (separate, not-yet-built) domain-modeling and architecture stages.

---

## Step 5 (optional, opt-in) — Competitor / market research

Ask once: "Want competitor analysis? If so, give me names, text, or URLs to start from — I won't go searching blind." Default: no, if the user doesn't raise it.

If yes:
- **Seed-only intake.** Only research what the user named. Never independently discover competitors via open search.
- **Level 1 (default, once opted in):** one fetch/search per named competitor. Shallow summary only — what it does, pricing if visible, one differentiator. If a seed doesn't resolve directly, one fallback search max — needing several searches to resolve one seed means the input was too vague, ask the user to clarify rather than searching harder.
- **Level 2 (opt-in per competitor):** after showing the Level 1 list, ask which (if any) to go deeper on. Never expand all without an explicit "expand all."
- End by proposing what could be dug into further; don't auto-continue.

Save findings to `competitor-analysis.md` alongside the spec (Step 7) if this ran.

---

## Step 6 — Draft and confirm

Draft using the shape appropriate to the path:

**Already-approved / not-decided (new product/feature):**
```markdown
# Spec: [Name]

## Decision
[What was approved, by whom, business outcome, success metric, constraints]

## Problem & Audience
[Problem, end user, stakeholders]

## User Journeys
[Narrative scenarios]

## Functional Requirements
[List]

## Non-Functional Requirements
[List]

## Domains Touched
[Flat list]

## Scope
- MVP cut: [...]
- Out of scope: [...]

## Success Criteria
[Specific, testable conditions]

## Open Questions
[Anything unresolved]
```

**Feature/story:**
```markdown
# Spec: [Name]

## Context
## Current State
## Proposed Change
## Acceptance Criteria
## Out of Scope
## Files Reference
```

Show the draft, ask: "Does this accurately capture it, or anything to change?"

Quick manual check before saving: skim for anything that looks like a secret, credential, internal URL, or PII. If something looks off, flag it and ask before continuing — don't save it silently.

---

## Step 7 — Save

Compute the slug from the topic: lowercase, spaces to dashes, strip to `[a-z0-9._-]`. Prefix with today's date.

```bash
SLUG=$(printf '%s' "$TOPIC" | tr '[:upper:]' '[:lower:]' | tr ' ' '-' | tr -cd 'a-z0-9._-')
DATED_SLUG="$(date +%Y-%m-%d)-$SLUG"
mkdir -p "eng-flow/specs/$DATED_SLUG"
```

Write the spec to `eng-flow/specs/<dated-slug>/spec.md`, and `competitor-analysis.md` alongside it if Step 5 ran.

---

## Step 8 — Report back

Tell the user the saved path, and what's next:
- **MVP path:** ready to build — capture a flat task checklist and go, no formal breakdown needed.
- **Production path** (already-approved, or a validated idea past a scope threshold): note this spec is Stage 1 of the production track; domain modeling is Stage 2, not yet run.

Run the Step 8 analytics-finish call (see Analytics section above) before ending.
