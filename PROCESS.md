# Process

## The model

Two phases, one gate between them.

```
                    ┌─────────────┐
  new idea/feature  │   ROUTING   │
  ─────────────────>│  (Phase 0)  │
                    └──────┬──────┘
                           │
        ┌──────────────────┼──────────────────┐
        v                  v                  v
  not yet decided     stakeholder-        feature/story in
  (you decide)        greenlit            existing product
        │                  │                  │
        v                  │                  │
  idea validation           │                  │
  (trimmed forcing          │                  │
  questions)                │                  │
        │                  │                  │
        └────────┬─────────┘                  │
                 v                            │
           ┌───────────┐                       │
           │ MVP MODE  │<──────────────────────┘
           └─────┬─────┘
                 │
                 v
       graduation checklist met?
                 │ yes
                 v
         ┌───────────────┐
         │ PRODUCTION MODE│
         └───────┬────────┘
                 v
   1. Spec/BRD (pure requirements)
                 v
   2. Domain model
                 v
   3. Architecture
                 v
   3.5. Engineering Review
                 v
   3.6. UI/UX Design (only if the spec has UI journeys)
                 v
   4. Epics / Stories / Tasks
                 v
   5. Implementation (per task, on demand)
                 v
   6. Code Review (diff, before it ships)
                 v
   7. QA (browser-based, front-end only)
                 v
   8. Ship (gate check, merge, version, push, PR)
                 v
   9. Retro (capture learnings, feeds back into Stage 5)
                 v
  10. Analytics (on-demand rollup, read-only)
```

MVP mode and production mode are not a maturity ranking — they're a match to context. A one-off internal tool never needs to graduate. A stakeholder-greenlit product might skip straight past MVP-mode ceremony into the production track, still carving an MVP-cut inside it. Route on what the work actually needs, not on ambition.

---

## Phase 0 — Routing

Before anything else, answer: **has the build decision already been made by someone with the authority to make it?**

- **No — you're deciding whether to build it.** (Solo idea, side project, "should we build this?") → run idea validation first.
- **Yes — stakeholders already approved it.** (Business case made elsewhere, you're capturing what was decided.) → skip validation, go straight into the production track's Stage 1 (Spec/BRD), still carve an MVP-cut inside the approved scope.
- **This is a feature or story in an existing product.** (Domain model and architecture already exist.) → skip validation, straight to a light spec, extend what's there.

### Idea validation (only for the "not yet decided" path)

A trimmed set of forcing questions — not a full interrogation, just enough to catch a idea that doesn't hold up before investing in a spec:

1. Who has this problem, concretely — not "everyone," a specific person/role?
2. What do they do today without this? (status quo)
3. Why now — what's changed that makes this worth building today?
4. What's the narrowest version that would prove it's wanted?

If it doesn't survive these, don't spec it yet — go find sharper answers first.

---

## MVP mode

Move fast, autonomously, minimal ceremony. Skill: `.claude/skills/eng-flow-mvp/SKILL.md`.

- Spec: if the user already has a detailed spec, use it as-is (confirmed with the user, not assumed). Otherwise, capture a one-pager — problem, scope, done-when — not a full interrogation.
- Tasks: a flat checklist. No epics, no stories, no formal breakdown.
- A ballpark time/token estimate and an explicit go/no-go gate the checklist before any branch or code work starts.
- Implementation runs as one auto-continuous loop across the whole checklist (not one task per run) — commits (and, if the user opts in, pushes) as it goes, stopping only for test/build breaks with no obvious fix, undecided requirements, high-risk/irreversible work, or a base-branch merge/push.
- No mandated testing or security ceremony beyond what the loop already does. Add more ad hoc if the work warrants it.
- Ship, iterate, don't look back unless something breaks.

---

## Graduation gate

Before moving to production mode, confirm the work actually needs it — multiple stakeholders involved, non-technical people need to sign off on requirements, the codebase needs to outlive this sprint, or a handoff to another engineer is likely. No checklist artifact for this; it's a judgment call. Don't graduate speculatively just because the code has gotten big or production mode feels more "serious."

---

## Production mode

Ten stages (numbered 1–10, with sub-stages 3.5 and 3.6), strictly staged — each owns different questions, none overlap. A stage never asks what a later stage owns.

### Stage 1 — Spec / Business Requirements (pure requirements, zero technical decisions)

Written in plain, non-technical language — the artifact both business and engineering sign off on as "this is what we agreed." Skill: `.claude/skills/eng-flow-spec/SKILL.md`.

Covers, via a phased interrogation:

**Phase 1 — Decision capture** (for the stakeholder-greenlit path; for the "you're deciding" path this is folded into idea validation above):
1. What did stakeholders approve, in their own words?
2. Who are the stakeholders/sponsors — is there a written source (deck, doc, email) to cite?
3. What business outcome is expected, and how will success be measured?
4. What constraints came with the approval (budget, timeline, compliance, must-integrate-with-X)?

**Phase 2 — Audience & scope lock:**
5. Who is the actual end user (often not the same person as the approving stakeholder)?
6. What's the smallest slice of scope that delivers real value — MVP-cut inside the approved scope?
7. What's explicitly out of scope for this first cut?
8. Which high-level domains/functional areas does this touch or introduce? (e.g. checkout, inventory, billing, support — a flat list, no modeling yet. Seeds Stage 2.)

Also captured: user journeys/stories (narrative scenarios), functional requirements (what the system must do), non-functional requirements (performance, security, availability, compliance, scalability — ask explicitly, these get forgotten otherwise).

**Phase 3 — Draft + confirm.** Present the draft, confirm accuracy, quick manual skim for secrets/PII before saving. No technical grounding phase here — there's no code yet to ground against, and asking tech questions this early re-mixes concerns Stage 3 owns.

**Optional — competitor/market analysis** (opt-in, bounded — see "Bounded research" below).

Exception: if this is a feature/story in an *existing* product (routing path 3), a technical-grounding pass is appropriate here, because real code already embodies real constraints worth reading before asking. That pass reads the relevant code first, then asks about data model, API, and integration points — the same idea applies at MVP scale, just lighter.

### Stage 2 — Domain Model

Owned by an "architect" role. Takes Stage 1 as input, translates it into structure — still conceptual, no tech stack yet. Skill: `.claude/skills/eng-flow-domain-model/SKILL.md`.

- Domain model — entities, relationships, ubiquitous language, per domain named in Stage 1's domain list
- Data flow diagram(s) — how information moves between domains/subsystems
- Conceptual system diagram — how subsystems/domains interact (a context map), deliberately not naming specific technologies
- Mockups/wireframes (lightweight, optional) — validated against Stage 1's user journeys
- Refinement Q&A — surfaces gaps/ambiguity in Stage 1 that only become visible once someone tries to draw the structure

### Stage 3 — Architecture

Only here do tech stack, deployment, and API/microservice shape get decided. Skill: `.claude/skills/eng-flow-architecture/SKILL.md`.

- Tech stack and key dependencies
- Frontend/backend separation (or services, if warranted by scope — don't split prematurely)
- API contracts
- Deployment strategy — local/containers now, cloud-ready notes for later; match the user's actual current practice (local dev, containers, GitHub) rather than assuming cloud deploy is imminent
- ADRs for significant decisions — one per decision, as they come up, not retrofitted
- Project-level security policy (`eng-flow/security-policy.md`, optional) — standing rules (credential handling, least-privilege access, write confirmation, etc.), established here if the project doesn't already have one, reused across specs like `design-system.md`, checked by Stage 3.5 and Stage 6

### Stage 3.5 — Engineering Review

Reviews `architecture.md` before any backlog work is built on top of it — catches design problems while they're still cheap to fix, not a code review (no code exists yet at this point). Skill: `.claude/skills/eng-flow-eng-review/SKILL.md`.

- Architecture-review checklist: component boundaries/coupling, data flow bottlenecks, scaling/SPOF, security architecture (checked against `eng-flow/security-policy.md` when one exists, not just generically), per-integration-point failure scenarios, distribution path
- Cognitive-pattern lenses (adapted from gstack's `plan-eng-review`, see `docs/DECISIONS.md`): boring by default, blast radius, Conway's Law, essential vs. accidental complexity, two-week smell test, incremental over revolutionary, reversibility preference
- Interactive, one issue at a time — each finding grounded in the specific section of `architecture.md` it reacts to, explicit accept/change/reject from the user before moving on
- Findings and resolutions logged to `eng-review.md` in the same spec folder; accepted/changed findings also update `architecture.md` directly

### Stage 3.6 — UI/UX Design

Only runs if the spec's user journeys have a UI surface — skipped entirely for backend/CLI/library work, same skip discipline Stage 7 applies. Skill: `.claude/skills/eng-flow-ui-design/SKILL.md`.

- Design system — typography, color, spacing, motion — project-level (`eng-flow/design-system.md`), reused across specs rather than rebuilt each run, same "boring by default" discipline Stage 3 applies to tech stack
- Per-journey wireframes (markdown/ASCII, low-fidelity) — starts from Stage 2's optional wireframes if present
- Design gate — interactive, one dimension at a time (hierarchy, consistency, accessibility, responsiveness), scored 0-10, fix-to-10, explicit per-journey approval before moving on
- Only journeys that pass the gate get promoted to a static HTML/CSS mockup, styled against the design system and targeting Stage 3's chosen frontend stack

### Stage 4 — Epics / Stories / Tasks

Breaks Stage 2/3 into executable work, and Stage 3.6's gated wireframes/mockups where that stage ran. Skill: `.claude/skills/eng-flow-epics-stories-tasks/SKILL.md`. Epics should map to the domains/functional areas named back in Stage 1. UI-touching stories get acceptance criteria grounded in the approved mockup rather than inventing layout at Stage 5. Task breakdown per story is on-demand, not automatic for the whole backlog.

### Stage 5 — Implementation

Terminal stage, on-demand per task. Skill: `.claude/skills/eng-flow-implement/SKILL.md`.

- Consumes a story's `tasks.md` (Stage 4's on-demand output), implements the next pending task using an incremental TDD loop (implement → test → verify → commit), one task per run
- Scope discipline: touch only what the task requires, log anything else noticed rather than fixing it inline
- Hard stop before high-risk/irreversible changes (auth, destructive migrations, payments, deletions, deploys, secrets) — explicit sign-off required
- Whole-story "auto" mode (implement every remaining task in one approved pass) is not built yet — deferred

### Stage 5 note — branching

Any new feature/bug/task starts the same way: pull latest base, create a branch (`feature/<slug>` or `fix/<slug>`), then work — never directly on the base branch. `eng-flow-implement`'s Step 0.5 handles this before the first task of a story begins.

### Stage 6 — Code Review

Post-implementation counterpart to Stage 3.5 — reviews the actual diff Stage 5 produced, not a design doc. Skill: `.claude/skills/eng-flow-code-review/SKILL.md`.

- Five-axis review: correctness, readability, architecture, security (checked against `eng-flow/security-policy.md` when one exists), performance
- Severity-tagged findings — Critical/Required block merge and get an individual sign-off gate; Nit/Optional/FYI are listed, not gated
- Tests reviewed first, dead code flagged before removal, verdict (Approve / Request changes) saved alongside the story

### Stage 7 — QA

Browser-based, front-end only — skip for backend/CLI/library work with no UI. Skill: `.claude/skills/eng-flow-qa/SKILL.md`.

- Exercises the *running* app, not the diff — catches what code review can't
- Detects Playwright (MCP or project-local) if available; falls back to a guided manual checklist otherwise, never skips QA for lack of tooling
- Modes: diff-aware (default, scoped to the current story), full, quick
- Documents bugs with evidence and severity, does not fix them — fixes route through Stage 5 (`eng-flow-implement`)

### Stage 8 — Ship

Terminal. Skill: `.claude/skills/eng-flow-ship/SKILL.md`.

- Gate check reuses Stage 6/7's saved verdicts (`code-review.md`, `qa-report.md`) instead of re-reviewing
- Merge base branch before a final regression pass (catch integration conflicts before merge, not after)
- Version/changelog bump only if the project already tracks them — never imposed
- Commit, push, open a PR — no cloud-deploy step, matches actual local/container/GitHub practice
- Reports GO/NO-GO with a rollback plan

### Stage 9 — Retro

Skill: `.claude/skills/eng-flow-retro/SKILL.md`.

- Per-story, blameless, root-cause-focused — not a weekly velocity dashboard
- Gathers signal from `tasks.md`, `code-review.md`, `qa-report.md`, and the branch's git log rather than working from memory
- Durable learnings (typed: pattern/pitfall/preference/architecture/tool/operational, confidence-scored) appended to project-level `eng-flow/learnings.md`
- Only useful if read later: `eng-flow-implement`'s Step 0 checks `learnings.md` before starting a task

### Analytics (cross-cutting, not a sequential stage)

Every stage above (1 through 9) logs its own time and token usage, incrementally, step by step, to project-level `eng-flow/analytics.jsonl` — via `eng-flow-analytics-checkpoint` at the start of each step and `eng-flow-analytics-finish` at the end, shared scripts under `.claude/skills/lib/bin/`. Logging incrementally (not just once at start/end of a whole skill run) means a killed terminal or system restart loses at most the current in-flight step, not the whole run — an orphaned marker from a crash gets recovered and flushed the next time that skill starts.

Token counts come from Claude Code's own session transcript (`$CLAUDE_CODE_SESSION_ID`), which carries real per-turn usage data — not something gstack or agent-skills does. Degrades to time-only if unavailable; never blocks the actual work.

### Decision Ledger (cross-cutting, not a sequential stage)

Every stage above logs each judgment call it makes — not just risky/irreversible ones (already gated elsewhere), but anywhere the AI decides something the user might reasonably have wanted a say in. Logged via `eng-flow-decision-log`, shared script under `.claude/skills/lib/bin/`, appending to project-level `eng-flow/decisions.jsonl`. Full taxonomy lives in `eng-flow-spec`'s Decision Ledger section (canonical copy); every other skill's own section is a short pointer back to it.

Three fields per entry:
- **reason** — why this was routed the way it was: `risk` (costly or hard to reverse downstream) | `knowledge_asymmetry` (stakeholder/customer context the AI structurally can't infer — e.g. a color scheme) | `stated_preference` (the user's told the assistant they want to be asked about this category)
- **mode** — how it was handled: `silent_decide` | `surface_existing` (pull real candidates from what already exists — never invent when something real can be surfaced instead) | `generate_options` (synthesize novel candidates — only on explicit request or when nothing exists to surface) | `open_question` | `must_escalate`
- **owner** — what actually happened this run: `ai_default` | `ai_inferred` | `user_confirmed` | `user_revised` | `user_delegated`

**Guide mode:** any skill invocation carrying a `--guide` token in `$ARGUMENTS` raises every decision point in that run to an explicit `AskUserQuestion` instead of a silent default, and closes with a "Decisions I made / decisions you made" summary. Without it, skills behave exactly as documented per-stage above — the flag only changes whether a decision is surfaced live, not whether it's logged.

**Closing the loop:** a ledger nobody reads is dead weight, same principle `learnings.md` already runs on. Stage 9 (`eng-flow-retro`) is the consumer — its Step 1 reads `decisions.jsonl` back and cross-references `ai_default`/`ai_inferred` entries against friction found elsewhere (rework, code-review findings, QA bugs). A correlation is the evidence to promote that decision point to `must_escalate` by default, rather than a permanent, never-revisited categorization.

No dedicated rollup skill yet (unlike Analytics' Stage 10) — one can be added once real `decisions.jsonl` data exists to design a report format against.

### Stage 10 — Analytics

Read-only, on-demand. Skill: `.claude/skills/eng-flow-analytics/SKILL.md`.

- Rolls up `analytics.jsonl` by story and by stage — time and tokens, broken down by category (input/output/cache-write/cache-read), not flattened into one number
- No dollar-cost estimate — check Anthropic's pricing page or Claude Code's `/cost` for that

---

## Bounded research (competitor analysis, and any other web-research sub-step)

Reused directly from prior art (see `docs/DECISIONS.md`) rather than invented fresh:

- **User-seeded only.** Never open-ended "go find competitors" discovery — only research what the user names (product names, text, URLs).
- **Registry/seed-first, one-fallback-max.** If a seed doesn't resolve directly, one fallback search per item, max. Needing several unrelated searches to resolve one item is a signal something's wrong with the input, not normal behavior.
- **Tiered, opt-in depth.** Level 1 (shallow, one fetch per item) runs by default once opted in. Deeper levels are opt-in *per item*, only after the user sees Level 1 and picks what to expand. Never bulk-expand without an explicit "expand all."
- **Propose, don't run.** End by listing what could be dug into further; wait for the user to pick rather than auto-continuing.

---

## Save conventions

Everything lives inside the project's own repo — no global/external store.

```
<project-root>/eng-flow/
  design-system.md               # project-level, reused across specs (Stage 3.6)
  security-policy.md             # project-level, optional, reused across specs (Stage 3; checked by 3.5 and 6)
  specs/
    2026-08-06-csv-export/
      spec.md
      competitor-analysis.md     # only if that sub-step ran
      wireframes/                # only if Stage 3.6 ran (spec had UI journeys)
        checkout-flow.md
      mockups/                   # only for wireframes gated at Stage 3.6
        checkout-flow.html
```

The directory name is `{date}-{slug}`, where `slug` is the spec's topic, lowercased, spaces to dashes, stripped to `[a-z0-9._-]` — the date prefix avoids collisions between specs on the same topic over time.
