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
   4. Epics / Stories / Tasks
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

Move fast, autonomously, minimal ceremony.

- Spec: if the user already has a detailed spec, use it as-is. Otherwise, fill `templates/mvp/00-mvp-brief.md` — a one-pager, not an interrogation.
- Tasks: `templates/mvp/01-task-list.md` — a flat checklist. No epics, no stories, no formal breakdown.
- No mandated testing or security ceremony. Add it ad hoc if the work warrants it.
- Ship, iterate, don't look back unless something breaks.

---

## Graduation gate

Before moving to production mode, check `templates/graduation-checklist.md`. Don't graduate speculatively — graduate when the checklist says so, not because the code has gotten big or because production mode feels more "serious."

---

## Production mode

Four stages, strictly staged — each owns different questions, none overlap. A stage never asks what a later stage owns.

### Stage 1 — Spec / Business Requirements (pure requirements, zero technical decisions)

Written in plain, non-technical language — the artifact both business and engineering sign off on as "this is what we agreed." Template: `templates/production/00-business-requirements.md`.

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

Exception: if this is a feature/story in an *existing* product (routing path 3), a technical-grounding pass is appropriate here, because real code already embodies real constraints worth reading before asking. That pass reads the relevant code first, then asks about data model, API, and integration points — see `templates/mvp/00-mvp-brief.md` for the lighter version of this same idea at MVP scale.

### Stage 2 — Domain Model

Owned by an "architect" role. Takes Stage 1 as input, translates it into structure — still conceptual, no tech stack yet. Template: `templates/production/01-domain-model.md`.

- Domain model — entities, relationships, ubiquitous language, per domain named in Stage 1's domain list
- Data flow diagram(s) — how information moves between domains/subsystems
- Conceptual system diagram — how subsystems/domains interact (a context map), deliberately not naming specific technologies
- Mockups/wireframes (lightweight, optional) — validated against Stage 1's user journeys
- Refinement Q&A — surfaces gaps/ambiguity in Stage 1 that only become visible once someone tries to draw the structure

### Stage 3 — Architecture

Only here do tech stack, deployment, and API/microservice shape get decided. Template: `templates/production/02-architecture.md`.

- Tech stack and key dependencies
- Frontend/backend separation (or services, if warranted by scope — don't split prematurely)
- API contracts
- Deployment strategy — local/containers now, cloud-ready notes for later; match the user's actual current practice (local dev, containers, GitHub) rather than assuming cloud deploy is imminent
- ADRs for significant decisions (`templates/production/04-adr.md`) — one per decision, as they come up, not retrofitted

### Stage 3.5 — Engineering Review

Reviews `architecture.md` before any backlog work is built on top of it — catches design problems while they're still cheap to fix, not a code review (no code exists yet at this point). Skill: `.claude/skills/eng-flow-eng-review/SKILL.md`.

- Architecture-review checklist: component boundaries/coupling, data flow bottlenecks, scaling/SPOF, security architecture, per-integration-point failure scenarios, distribution path
- Cognitive-pattern lenses (adapted from gstack's `plan-eng-review`, see `docs/DECISIONS.md`): boring by default, blast radius, Conway's Law, essential vs. accidental complexity, two-week smell test, incremental over revolutionary, reversibility preference
- Interactive, one issue at a time — each finding grounded in the specific section of `architecture.md` it reacts to, explicit accept/change/reject from the user before moving on
- Findings and resolutions logged to `eng-review.md` in the same spec folder; accepted/changed findings also update `architecture.md` directly

### Stage 4 — Epics / Stories / Tasks

Breaks Stage 2/3 into executable work. Template: `templates/production/03-epics-stories-tasks.md`. Epics should map to the domains/functional areas named back in Stage 1.

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
  specs/
    2026-08-06-csv-export/
      spec.md
      competitor-analysis.md     # only if that sub-step ran
```

The directory name is `{date}-{slug}`, where `slug` is the spec's topic, lowercased, spaces to dashes, stripped to `[a-z0-9._-]` — the date prefix avoids collisions between specs on the same topic over time.
