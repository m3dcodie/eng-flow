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

At the start of every numbered step below (including Step 0), run `python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-analytics-checkpoint" eng-flow-spec "<step name>" "<dated-slug-if-known>"`. As the last action of Step 8, run `python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-analytics-finish" eng-flow-spec "<dated-slug>"`. Logs time and token usage per step, incrementally, to `eng-flow/analytics.jsonl` — a killed terminal or restart loses at most the in-flight step, not the whole run. Degrades silently if unavailable; never blocks real work. Rollup: `eng-flow-analytics` (Stage 10).

## Decision Ledger

**Guide mode:** check `$ARGUMENTS` for a `--guide` token before treating the remainder as the topic (strip it out either way). When present, every decision point below gets an explicit `AskUserQuestion` instead of a silent default, and Step 8's report adds a "Decisions I made / decisions you made" summary before saving.

**Logging:** at each decision point below, run `python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-decision-log" eng-flow-spec "<step name>" <reason> <mode> <owner> "<description>" "<dated-slug-if-known>"` — guide mode or not, every decision point gets logged; the flag only changes whether it's surfaced live.

- `reason` — why this was routed the way it was: `risk` (getting it wrong is costly or hard to reverse downstream) | `knowledge_asymmetry` (stakeholder/customer context the AI can't infer) | `stated_preference` (the user has told the assistant they want to be asked about this category)
- `mode` — how it was handled: `silent_decide` | `surface_existing` (pull real candidates from what already exists — brand guide, prior spec, existing pattern — never invent) | `generate_options` (synthesize novel candidates; only on explicit user request or when nothing exists to surface) | `open_question` | `must_escalate`
- `owner` — what actually happened this run: `ai_default` | `ai_inferred` | `user_confirmed` | `user_revised` | `user_delegated`

Appends to `eng-flow/decisions.jsonl`, same degrade-silently discipline as analytics — never blocks real work. Every other eng-flow skill's Decision Ledger section points back here for the taxonomy and just logs calls at its own decision points. Consumed by `eng-flow-retro` (Stage 9) Step 1, which cross-references `ai_default`/`ai_inferred` entries against later friction to tell whether a routing default is actually working or needs promoting to `must_escalate`.

## Step 0 — Topic, resume, and routing

`$ARGUMENTS` is the rough topic — strip a leading/trailing `--guide` token first (see Decision Ledger above) before treating the remainder as the topic. It may be empty.

### Check for unfinished drafts

Before asking anything, list this skill's drafts (design: `${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/draft-resume.md`):

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-draft" list eng-flow-spec
```

One tab-separated line per unfinished draft, newest first: `<dated-slug>  <topic>  stopped after Step <n> (<name>)  <age>`. No output means no drafts.

| Situation | What to do |
|---|---|
| No topic, no drafts | Ask "What are we speccing?", then **New run** below. |
| No topic, one or more drafts | One `AskUserQuestion` listing each draft (label: its topic, or `<topic> (<dated-slug>)` when two drafts share a topic; description: "stopped after Step n (name), age") plus **Something new**. `AskUserQuestion` takes at most 4 options, so with more than 3 drafts show the 3 newest and name the rest in the question text — the user can pick one through "Other". A draft → **Resume** straight away — no second question. Something new → ask "What are we speccing?", then **New run**. |
| Topic given, a draft has the same topic (compare trimmed, case-insensitive) | One `AskUserQuestion`: **Resume it** (stopped after Step n, age) / **Start a separate spec** / **Discard it**. Resume → **Resume**; separate → **New run**; discard → **Discard**. If more than one draft matches (an earlier "separate spec" gave it a `-2` slug), list the matching drafts instead, each labelled with its slug, plus **Start a separate spec**; a picked draft → **Resume**. |
| Topic given, no draft matches | **New run**. |

### Resume and discard

Picking a draft means resume — don't ask again. Discard is reached from the topic-match question above, or when the user says "discard" at any point after a resume starts.

**Discard** — confirm first: "Delete the draft for `<topic>`? Its saved answers can't be recovered." Only on yes:

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-draft" clear eng-flow-spec "<dated-slug>"
```

Log it: `eng-flow-decision-log eng-flow-spec "Step 0" risk open_question user_confirmed "discarded draft <dated-slug>" "<dated-slug>"`. Then go back to the top of Step 0 — with a topic given, that's a **New run** on it; without one, the draft list again (or "What are we speccing?" if none are left). On no, carry on where the user was.

**Resume** — Read `eng-flow/.drafts/spec--<dated-slug>.json`. Take `dated_slug`, `topic` and `path` from it; reuse `dated_slug` exactly as saved — it carries the original start date, and Step 7 names the spec folder with it.

Don't ask whether the saved answers are still right — continue straight away. Run Step `<n+1>`'s analytics checkpoint first, then say in one line, directly above that step's content (so tool output doesn't separate them), where it's picking up and how to review, e.g.: "Resuming **json export as csv** at Step 6 (Draft). Steps 0–5 are saved; say *review* to go over or change them, or *discard* to drop this draft." Then run Step `last_completed_step + 1` as normal, including its skip rules and draft save. Resuming at Step 7 writes `spec.md` from Step 6's saved `spec_markdown`.

**Review** — only when the user asks (they say "review", or want to change an earlier answer): play back a short summary of the saved answers — a line or two per `done` step, each `skipped` step named with its reason — and take any corrections:

- **Corrected** — re-save each changed step with its corrected answers (the same `save` call, same step number), then continue at `last_completed_step + 1`. Two exceptions, where the correction makes later saved steps stale — drop those steps *first*, so a crash before they're redone can't leave them counted as finished:
  - **Routing path changed** — the saved skips no longer fit. Run `truncate eng-flow-spec "<dated-slug>" 0`, re-save step 0 with the new path, and continue at Step 1.
  - **Step 6 was saved and an earlier answer changed** — its spec text is out of date. Run `truncate eng-flow-spec "<dated-slug>" 5`, re-save the changed steps, and continue at Step 6.

  ```bash
  python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-draft" truncate eng-flow-spec "<dated-slug>" <keep-up-to-step>
  ```

Log it: `eng-flow-decision-log eng-flow-spec "Step 0" risk open_question user_confirmed "resumed draft at Step <n+1>" "<dated-slug>"`, and on a review with corrections, `... user_revised "resumed draft, corrected: <what changed>" ...`. A resume skips the routing question and the step-0 save below — the draft already has both.

### New run

Compute the slug from the topic now, not at Step 7 — the draft file (end of this step) and the spec folder (Step 7) both use it, so it carries the start date. Lowercase, spaces to dashes, strip to `[a-z0-9._-]`, prefix with today's date:

```bash
SLUG=$(printf '%s' "$TOPIC" | tr '[:upper:]' '[:lower:]' | tr ' ' '-' | tr -cd 'a-z0-9._-')
DATED_SLUG="$(date +%Y-%m-%d)-$SLUG"
```

If `eng-flow/.drafts/spec--$DATED_SLUG.json` already exists (the same topic started again on the same day, and the user chose a separate spec), append `-2`, `-3`, … until the name is free — a separate spec must never write into the other one's draft.

Shell variables don't survive between Bash calls — carry the literal `<dated-slug>` value forward and reuse it in every later step instead of recomputing it.

Then ask the routing question — this determines everything downstream, so don't skip or infer it silently:

> "Has this already been decided by someone with the authority to approve it, or are you the one deciding?"

Offer three paths via `AskUserQuestion`:
- **Not decided yet** — you're weighing whether to build this
- **Already approved** — a stakeholder greenlit this, you're capturing what was decided
- **Feature/story in an existing product** — domain model and architecture already exist, this extends them

The path chosen governs which steps below run. Do not let "this looks like a startup idea" or "this looks big" substitute for actually asking — greenlit enterprise work and solo weekend ideas can look identical from the topic string alone.

Log it: `eng-flow-decision-log eng-flow-spec "Step 0" risk open_question user_confirmed "routing path: <path chosen>" "<dated-slug>"` — wrong routing here cascades into every downstream step, so this is always an explicit ask, guide mode or not.

As the last action of this step, save it to the draft — this creates `eng-flow/.drafts/spec--<dated-slug>.json`, so a crash from here on keeps the topic and path (design: `${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/draft-resume.md`). `<path>` is `idea`, `approved` or `feature`:

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-draft" save eng-flow-spec "<dated-slug>" 0 <<'EOF'
{"status": "done", "name": "Topic and routing", "answers": {"topic": "<topic>", "path": "<path>"}}
EOF
```

Degrades silently like the analytics scripts — a failed save prints a warning and the interview carries on.

---

## Step 1 (path: not decided) — Idea validation

Trimmed forcing questions, asked until answered concretely — not restated requests:

1. Who has this problem, concretely? (a specific person/role, not "everyone")
2. What do they do today without this? (status quo)
3. Why now — what's changed that makes this worth building today?
4. What's the narrowest version that would prove it's actually wanted?

If these can't be answered without hand-waving, say so plainly and stop — offer to keep refining the idea rather than proceeding to a spec that will just encode the vagueness. If they survive, continue to Step 2.

Paths "already approved" and "feature/story" skip this step entirely — the demand question isn't the engineer's to re-litigate on those paths.

As the last action of this step, save it to the draft. Steps 1–6 all save the same way — `save eng-flow-spec "<dated-slug>" <step>` with the step's JSON on stdin — as `"done"` with the agreed answers in short form (not the conversation), or as `"skipped"` with a reason so a resume doesn't ask it again. JSON-escape quotes and newlines inside answer strings:

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-draft" save eng-flow-spec "<dated-slug>" 1 <<'EOF'
{"status": "done", "name": "Idea validation", "answers": {"who": "<who>", "status_quo": "<status quo>", "why_now": "<why now>", "narrowest_version": "<narrowest version>"}}
EOF
```

Off the idea path: `{"status": "skipped", "name": "Idea validation", "reason": "not the idea path"}`. If the idea doesn't survive and you stop here, don't save Step 1 — the draft stays at Step 0, so a later run can resume and retry validation.

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

As the last action of this step, save it (step 2, `"name": "Why"`) with one short answer per question asked, keyed by what it answers — e.g. `approved`, `stakeholders`, `source`, `outcome`, `success_metric`, `constraints` on the approved path; `problem`, `if_not_built`, `success_signal` otherwise.

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

As the last action of this step, save it (step 3, `"name": "Audience and scope"`) with `end_user`, `mvp_cut`, `out_of_scope` and `domains` (a list), plus `journeys`, `functional_requirements` and `non_functional_requirements` (lists) when they were captured.

---

## Step 4 (path: feature/story only) — Technical grounding

**Hard requirement: read the relevant code before asking any technical question here.** Grep/Read the actual files this touches first — this is the one path where code already exists and embodies real constraints worth reading before asking.

Ask targeted questions about whichever apply (skip what clearly doesn't):
- Where this plugs into existing code (cite real files/functions found)
- Data model / API shape changes
- Edge cases the existing code already handles that this must not break

Other paths skip this entirely — there's no code yet, and asking technical questions this early re-mixes concerns that belong to the (separate, not-yet-built) domain-modeling and architecture stages.

As the last action of this step, save it (step 4, `"name": "Technical grounding"`) with `files` (the real files/functions cited), `integration_points`, `data_api_changes` and `edge_cases`. Off the feature path: `{"status": "skipped", "name": "Technical grounding", "reason": "not the feature path"}`.

---

## Step 5 (optional, opt-in) — Competitor / market research

Ask once: "Want competitor analysis? If so, give me names, text, or URLs to start from — I won't go searching blind." Default: no, if the user doesn't raise it.

Log it: `eng-flow-decision-log eng-flow-spec "Step 5" risk silent_decide ai_default "competitor research: skipped, not raised by user" "<dated-slug>"` if the default applies (in guide mode, ask explicitly instead and log `open_question`/`user_confirmed`).

If yes:
- **Seed-only intake.** Only research what the user named. Never independently discover competitors via open search.
- **Level 1 (default, once opted in):** one fetch/search per named competitor. Shallow summary only — what it does, pricing if visible, one differentiator. If a seed doesn't resolve directly, one fallback search max — needing several searches to resolve one seed means the input was too vague, ask the user to clarify rather than searching harder.
- **Level 2 (opt-in per competitor):** after showing the Level 1 list, ask which (if any) to go deeper on. Never expand all without an explicit "expand all."
- End by proposing what could be dug into further; don't auto-continue.

Save findings to `competitor-analysis.md` alongside the spec (Step 7) if this ran.

As the last action of this step, save it (step 5, `"name": "Research"`). If research ran, `answers` holds `competitors` — one entry per competitor with `name`, `summary`, `level` (1 or 2) and `source` — because `competitor-analysis.md` isn't written until Step 7 and a crash before then would lose the fetches. If it didn't: `{"status": "skipped", "name": "Research", "reason": "user declined competitor research"}`.

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

Log it: `eng-flow-decision-log eng-flow-spec "Step 6" risk open_question <user_confirmed|user_revised> "draft accepted as-is|draft revised: <what changed>" "<dated-slug>"`.

Quick manual check before saving: skim for anything that looks like a secret, credential, internal URL, or PII. If something looks off, flag it and ask before continuing — don't save it silently. This check runs before the draft save below too, so nothing flagged lands in `eng-flow/.drafts/` either.

As the last action of this step, once the user has confirmed the draft, save it (step 6, `"name": "Draft"`) with `spec_markdown` — the full confirmed spec text, JSON-escaped — and `revisions`, a short note of what the user changed (empty if accepted as-is). Step 7 writes this text as `spec.md`.

---

## Step 7 — Save

Use the `<dated-slug>` from Step 0 — don't recompute it, or a run that crosses midnight gets a different date than its draft.

```bash
mkdir -p "eng-flow/specs/<dated-slug>"
```

Write the spec to `eng-flow/specs/<dated-slug>/spec.md`, and `competitor-analysis.md` alongside it if Step 5 ran.

As the last action of this step, once every file above is written, delete the draft — the spec folder now holds everything it did:

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-draft" clear eng-flow-spec "<dated-slug>"
```

If any write failed, skip this call and keep the draft, so a re-run can resume from Step 6 instead of re-interviewing. Tell the user which file didn't save.

---

## Step 8 — Report back

Tell the user the saved path, and what's next:
- **MVP path:** ready to build — run `eng-flow-mvp`, which will read this spec, confirm it's still what's wanted, and take it from checklist through ship.
- **Production path** (already-approved, or a validated idea past a scope threshold): note this spec is Stage 1 of the production track; domain modeling is Stage 2, not yet run.

If this run was in guide mode (Decision Ledger section), add a short "Decisions I made / decisions you made" summary here, drawn from this run's `eng-flow-decision-log` calls, before the report is considered done.

Run the Step 8 analytics-finish call (see Analytics section above) before ending.
