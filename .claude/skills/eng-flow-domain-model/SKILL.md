---
name: eng-flow-domain-model
description: Production Stage 2 — takes a saved eng-flow spec and turns its named domains into a domain model, data flows, and a conceptual system diagram. No tech stack, no deployment, no API shape — that's Stage 3. Run by an "architect" persona, after eng-flow-spec, before eng-flow-architecture.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Write
  - Bash
  - AskUserQuestion
  - Artifact
triggers:
  - domain model
  - model the domain
  - map the domains
  - eng-flow domain model
---

# eng-flow domain model

Stage 2 of the production track. Translates a spec's plain-language requirements into structure — entities, relationships, flows — while staying strictly conceptual. If a question about tech stack, database choice, deployment, or API shape comes up here, defer it explicitly: "that's Stage 3 (architecture), not this stage" — don't answer it now even if you know the answer.

## Analytics

At the start of every numbered step below (including Step 0), run `python3 .claude/skills/lib/bin/eng-flow-analytics-checkpoint eng-flow-domain-model "<step name>" "<dated-slug>"`. As the last action of Step 8, run `python3 .claude/skills/lib/bin/eng-flow-analytics-finish eng-flow-domain-model "<dated-slug>"`. See `eng-flow-spec`'s Analytics section for what this logs and why; rollup via `eng-flow-analytics` (Stage 10).

## Step 0 — Find the spec

Look for `eng-flow/specs/*/spec.md`. If none exist, tell the user to run `eng-flow-spec` first — this stage has nothing to model without one. If more than one exists, ask which spec this run is for. If exactly one, use it.

Read the spec's **Domains Touched** list and **User Journeys** — these are the inputs to everything below. If the spec has no domains list (e.g. it used the lightweight feature/story template), ask the user directly: "Which domains/functional areas does this touch?"

---

## Step 1 — Domain model, per domain

For each domain named in the spec, ask (don't invent):

1. What are the core entities in this domain? (the nouns the business actually uses — pull from the spec's own language, don't rename them into something more "technical")
2. What are the key relationships between these entities, and between entities in this domain and entities in other named domains?
3. Is there any term here that means something different to different stakeholders? (surface ubiquitous-language conflicts now — cheap to fix here, expensive once code exists)

Keep this at the conceptual level: entity names, relationships, cardinality if it's non-obvious. No fields, no types, no persistence — that's implementation, not modeling.

---

## Step 2 — Data flow

Using the spec's user journeys and functional requirements: for each journey, trace which domains it touches and in what order. Ask the user to confirm or correct the sequence rather than asserting it — the spec describes user-facing behavior, not internal flow, so this is genuinely new information, not a restatement.

Represent as a simple flow list or diagram (mermaid `flowchart` is fine if the host renders it, otherwise plain ordered steps):

```
User action → Domain A (does X) → Domain B (does Y) → outcome
```

---

## Step 3 — Conceptual system diagram

One diagram showing how the named domains/subsystems relate — a context map, not a deployment diagram. Boxes are domain names from Step 1 (e.g. "Checkout," "Inventory," "Billing"), arrows show which domain calls/depends on which. **Do not label boxes with technology** (no "Postgres," "React," "Lambda") — if a box needs a technology label to make sense, that's a sign the diagram has drifted into Stage 3's territory; pull it back to domain names only.

Draft it as mermaid first — that's what goes in the saved file. Then ask once: "Want this rendered as a diagram you can actually see, or is the mermaid source in the saved file enough?" Default to skipping if the user doesn't ask. If yes, load the `artifact-diagramming` skill and publish via the `Artifact` tool; note the artifact URL alongside the saved diagram source in Step 6/7 so it isn't lost.

---

## Step 4 (optional) — Mockups / wireframes

Ask: "Would a rough wireframe help validate any of these journeys?" Default: skip unless the user asks or a journey is genuinely hard to follow in text. If yes, keep it low-fidelity — boxes and labels showing flow and content, not visual design (no colors, no branding, no component library references). If produced, offer the same Artifact-rendering treatment as Step 3.

---

## Step 5 — Refinement Q&A

While modeling, gaps in the spec surface that weren't visible when it was pure prose — e.g. "the spec's journey says a user cancels an order, but doesn't say what happens to reserved inventory." Collect these as explicit questions, ask the user, and note the answer here rather than silently amending the spec file itself (the spec stays the record of what was agreed at Stage 1; this stage's answers extend it, they don't retroactively rewrite it).

---

## Step 6 — Draft and confirm

```markdown
# Domain Model: [Name]
(source spec: eng-flow/specs/<dated-slug>/spec.md)

## Domains

### [Domain name]
**Entities:** [...]
**Relationships:** [...]
**Ubiquitous language notes:** [...]

(repeat per domain)

## Data Flow

[Per-journey flow traces from Step 2]

## System Diagram

[Conceptual diagram from Step 3 — domain names only, no tech]
[Artifact URL, if rendered; omit line otherwise]

## Wireframes

[If Step 4 ran; omit section otherwise. Artifact URL if rendered.]

## Open Questions From Modeling

[Step 5 gaps + answers, or "none surfaced" if genuinely none]
```

Show the draft, ask: "Does this match how you think about the domains, or anything to correct?"

---

## Step 7 — Save

Write to the same spec's folder: `eng-flow/specs/<dated-slug>/domain-model.md`.

---

## Step 8 — Report back

Confirm the saved path. Tell the user this feeds Stage 3 (architecture — tech stack, deployment, API shape), not yet run.

Run the Step 8 analytics-finish call (see Analytics section above) before ending.
