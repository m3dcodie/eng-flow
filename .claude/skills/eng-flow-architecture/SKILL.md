---
name: eng-flow-architecture
description: Production Stage 3 — takes a saved eng-flow spec and domain model and produces the technical design — tech stack, a tech-labeled system diagram, API contracts, quantified non-functional requirements, deployment topology, and ADRs. This is where technical decisions actually get made; Stages 1 and 2 deliberately deferred all of them here.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Write
  - Bash
  - AskUserQuestion
  - Artifact
  - Skill
triggers:
  - architecture
  - tech stack
  - system design
  - eng-flow architecture
  - technical design
---

# eng-flow architecture

Stage 3 of the production track. Everything Stage 1 and Stage 2 deferred — tech stack, system diagram, API shape, quantified NFRs, deployment — gets decided here. This is the one production stage where technical questions are not just allowed but the entire point.

## Analytics

At the start of every numbered step below (including Step 0), run `python3 .claude/skills/lib/bin/eng-flow-analytics-checkpoint eng-flow-architecture "<step name>" "<dated-slug>"`. As the last action of Step 10, run `python3 .claude/skills/lib/bin/eng-flow-analytics-finish eng-flow-architecture "<dated-slug>"`. See `eng-flow-spec`'s Analytics section for what this logs and why; rollup via `eng-flow-analytics` (Stage 10).

## Step 0 — Find the inputs

Look for `eng-flow/specs/*/spec.md` and the matching `domain-model.md` in the same folder. If more than one spec folder exists, ask which one this run is for.

If `spec.md` is missing: offer via `AskUserQuestion` — "No spec found for this. eng-flow-spec captures the problem, audience, and scope this stage needs to ground decisions in. Run it now?" Options: A) Run eng-flow-spec now (resume architecture right after) B) Cancel — I'll run it myself. If A, invoke `eng-flow-spec` with the `Skill` tool, then re-run this check once it saves. If B, stop here.

If `spec.md` exists but `domain-model.md` is missing: same offer, naming `eng-flow-domain-model` instead — its entities, relationships, and conceptual diagram are this step's other required input.

Read the spec's constraints (Phase 1, "what constraints came with the approval") and non-functional requirements, and the domain model's entities/relationships/conceptual diagram — these are the inputs to every step below.

---

## Step 1 — Tech stack

If this extends an existing product, use its existing stack — don't relitigate it here. If greenfield, ask, constrained by anything the spec already locked in (e.g. "must use vendor X").

Apply **boring by default**: prefer proven, well-understood technology unless there's a specific, stated reason not to. If the user picks something novel/unproven, ask them to name the reason explicitly (it becomes part of the ADR in Step 7) — an unjustified novel choice is a flag to raise, not silently accept.

---

## Step 2 — System diagram

Take Stage 2's conceptual diagram (domain boxes, no tech) and promote it to a **container-level diagram** (C4 model terminology: context → container → component → code; this stage stops at container, not component/code level): same domains, now labeled with the actual service/technology handling each. This is the one diagram in the whole track where tech labels belong.

Draft it as mermaid first — that's what goes in the saved file. Then ask once: "Want this rendered as a diagram you can actually see, or is the mermaid source in the saved file enough?" Default to skipping if the user doesn't ask. If yes, load the `artifact-diagramming` skill and publish via the `Artifact` tool; note the artifact URL alongside the saved diagram source in Step 8/9.

---

## Step 3 — Frontend/backend separation

Decide based on the actual complexity surfaced in Stage 2 — number of domains, their coupling, team size — not by default. If scope is small, say so and recommend staying monolithic; splitting into separate frontend/backend (or further into services) is a decision to justify, not a default to assume. If it does split, name the boundary explicitly (what runs where).

---

## Step 4 — API contracts

Define the interface between frontend/backend and between domains identified in Stage 2, contract-first — the contract is written before implementation, not derived from it.

Apply the same discipline `agent-skills`' `api-and-interface-design` documents:
- **Hyrum's Law** — every observable behavior becomes a de facto commitment once something depends on it; be deliberate about what's exposed, don't leak implementation details.
- **Contract first** — define request/response shapes before writing the implementation.
- **One-Version Rule** — design so consumers never have to choose between multiple versions of the same interface.

Depth matches scope — a small feature needs a short list of endpoint shapes, not a full OpenAPI spec, unless the scope actually warrants one.

---

## Step 5 — Non-functional requirements (quantified)

Stage 1 captured NFRs at the business level ("fast," "secure," "available") — this step translates them into technical targets:
- Performance: response time / latency budget under what load
- Scale: concurrent users, data volume, growth assumptions
- Availability: uptime target
- Security: specific controls, not just "secure" (auth model, data-at-rest/in-transit handling)
- Compliance: specific standards if the spec named any

Ask for numbers if the spec didn't give any. If none are available and the user doesn't have a target in mind, state the assumption explicitly rather than inventing a number silently.

---

## Step 6 — Deployment

Default to what the user actually does, not an assumed cloud target: local development, containers, GitHub (per the project's real practice) — document that as the deployment topology (dev/staging/prod environments, or fewer if that's the real setup), the CI pipeline (GitHub Actions or equivalent), and any infrastructure-as-code in use.

Add a short **cloud-ready notes** subsection: what would need to change to deploy this to a cloud target later (which components would need it, roughly what'd be involved) — informational, not a decision to act on now. This is what makes the doc useful to another engineer picking this up later without forcing a cloud decision that isn't warranted yet.

---

## Step 7 — ADRs

Reuse `agent-skills`' convention rule: **before creating an ADR, check for an existing convention in the repo** (an `adr/` or `docs/decisions/` directory, an `.adr-dir` file, existing numbered ADRs) and match its location, numbering, and section headings. Only apply the default below if no convention exists.

Write one ADR per decision from Steps 1-6 that would be expensive to reverse — framework/library choice, data model, auth strategy, API architecture style, deployment/hosting choice. Not every small choice needs one; a Step 1 "boring by default" pick with an obvious reason usually doesn't.

Default (no existing convention found): `eng-flow/specs/<dated-slug>/adr/NNNN-title.md`, sequential numbering starting at `0001`:

```markdown
# ADR-NNNN: [Decision]

## Status
Accepted

## Context
[What problem/question this decision answers]

## Decision
[What was chosen]

## Alternatives Considered
[What else was on the table, and why not]

## Consequences
[What this makes easier, harder, or forecloses]
```

---

## Step 8 — Draft and confirm

```markdown
# Architecture: [Name]
(source: spec.md, domain-model.md in this folder)

## Tech Stack
[Choices + justification, especially anything not "boring by default"]

## System Diagram
[Container-level, domains from Stage 2 now tech-labeled]
[Artifact URL, if rendered; omit line otherwise]

## Frontend/Backend Separation
[Decision + reasoning — or "staying monolithic, and why"]

## API Contracts
[Endpoint/interface shapes]

## Non-Functional Requirements
[Quantified targets per Step 5]

## Deployment
[Topology, CI pipeline, IaC — actual current practice]
[Cloud-ready notes]

## ADRs
[List of ADR files written in Step 7, or "none warranted" if genuinely none]
```

Show the draft, ask: "Does this match what you'd actually build, or anything to change?"

---

## Step 9 — Save

Write `architecture.md` to the same spec folder (`eng-flow/specs/<dated-slug>/architecture.md`), and any ADR files to `eng-flow/specs/<dated-slug>/adr/` (or the matched existing convention's location).

---

## Step 10 — Report back

Confirm the saved paths. Tell the user this feeds Stage 4 (epics/stories/tasks — breaking this into executable work), not yet run.

Run the Step 10 analytics-finish call (see Analytics section above) before ending.
