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

At the start of every numbered step below (including Step 0), run `python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-analytics-checkpoint" eng-flow-architecture "<step name>" "<dated-slug>"`. As the last action of Step 10, run `python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-analytics-finish" eng-flow-architecture "<dated-slug>"`. See `eng-flow-spec`'s Analytics section for what this logs and why; rollup via `eng-flow-analytics` (Stage 10).

## Decision Ledger

Check `$ARGUMENTS` for a `--guide` token; if present, every decision point below gets an explicit `AskUserQuestion` instead of a silent default, and Step 10's report adds a "Decisions I made / decisions you made" summary. Log every decision point via `python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-decision-log" eng-flow-architecture "<step>" <reason> <mode> <owner> "<description>" "<dated-slug>"`. See `eng-flow-spec`'s Decision Ledger section for the taxonomy and why. Rollup/analysis: `eng-flow-retro` Step 1 (Stage 9).

## Drafts

This stage saves as it goes to `eng-flow/.drafts/architecture--<dated-slug>.json` (`<dated-slug>` is the spec folder's), so a crash loses at most the step in progress. The shared rules (Resume, Review, Discard, input drift) are in `${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/draft-resume.md` — Read it before resuming, reviewing or discarding. Every save in this file is this call, with the step's JSON on stdin — agreed answers in short form, JSON-escaped:

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-draft" save eng-flow-architecture "<dated-slug>" <step> <<'EOF'
{"status": "done", "name": "<step name>", "answers": {…}}
EOF
```

Degrades silently like the analytics scripts — a failed save prints a warning and the run carries on.

## Step 0 — Find the inputs

Look for `eng-flow/specs/*/spec.md` and the matching `domain-model.md` in the same folder. Check for drafts: `python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-draft" list eng-flow-architecture` — one line per draft, starting with the spec folder's `<dated-slug>`. If more than one spec folder exists, ask which one this run is for; a folder with a draft gets "draft: stopped after Step n, age" in its option description. If the chosen folder (or the only one) has a draft → **Resume** (`draft-resume.md`) — no further Step 0 questions.

If `spec.md` is missing: offer via `AskUserQuestion` — "No spec found for this. eng-flow-spec captures the problem, audience, and scope this stage needs to ground decisions in. Run it now?" Options: A) Run eng-flow-spec now (resume architecture right after) B) Cancel — I'll run it myself. If A, invoke `eng-flow-spec` with the `Skill` tool, then re-run this check once it saves. If B, stop here.

If `spec.md` exists but `domain-model.md` is missing: same offer, naming `eng-flow-domain-model` instead — its entities, relationships, and conceptual diagram are this step's other required input.

Read the spec's constraints (Phase 1, "what constraints came with the approval") and non-functional requirements, and the domain model's entities/relationships/conceptual diagram — these are the inputs to every step below.

As the last action of this step (new run only — not if it stopped for a missing input), save step 0 (`"name": "Find the inputs"`) with `topic` (the spec's title) and `spec_folder`, then record the inputs:

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-draft" inputs eng-flow-architecture "<dated-slug>" "eng-flow/specs/<dated-slug>/spec.md" "eng-flow/specs/<dated-slug>/domain-model.md"
```

**Stale on review:** an answer from Steps 1–6 changed after Step 7 was saved — the ADRs and the draft may no longer match it. `truncate ... 6`, re-save the changed steps, continue at Step 7.

---

## Step 1 — Tech stack

If this extends an existing product, use its existing stack — don't relitigate it here. If greenfield, ask, constrained by anything the spec already locked in (e.g. "must use vendor X").

Apply **boring by default**: prefer proven, well-understood technology unless there's a specific, stated reason not to. If the user picks something novel/unproven, ask them to name the reason explicitly (it becomes part of the ADR in Step 7) — an unjustified novel choice is a flag to raise, not silently accept.

Log it: `risk silent_decide ai_default "tech stack: reused existing / boring-by-default choice"` if no relitigation was needed, otherwise `risk open_question user_confirmed "tech stack: <choice> — reason: <stated reason>"`.

As the last action of this step, save step 1 (`"name": "Tech stack"`) with `stack` (one entry per layer with `choice` and `reason`) and `reused_existing` (true/false).

---

## Step 2 — System diagram

Take Stage 2's conceptual diagram (domain boxes, no tech) and promote it to a **container-level diagram** (C4 model terminology: context → container → component → code; this stage stops at container, not component/code level): same domains, now labeled with the actual service/technology handling each. This is the one diagram in the whole track where tech labels belong.

Draft it as mermaid first — that's what goes in the saved file. Then ask once: "Want this rendered as a diagram you can actually see, or is the mermaid source in the saved file enough?" Default to skipping if the user doesn't ask. If yes, load the `artifact-diagramming` skill and publish via the `Artifact` tool; note the artifact URL alongside the saved diagram source in Step 8/9.

Log it: `risk silent_decide ai_default "diagram rendering: skipped, mermaid source only"` if the default applies, otherwise `open_question user_confirmed`.

As the last action of this step, save step 2 (`"name": "System diagram"`) with `mermaid` (the diagram source) and `artifact_url` (or `null`).

---

## Step 3 — Frontend/backend separation

Decide based on the actual complexity surfaced in Stage 2 — number of domains, their coupling, team size — not by default. If scope is small, say so and recommend staying monolithic; splitting into separate frontend/backend (or further into services) is a decision to justify, not a default to assume. If it does split, name the boundary explicitly (what runs where).

Log it: `risk silent_decide ai_inferred "frontend/backend separation: <monolith|split> — <reasoning from Stage 2 complexity>"`.

As the last action of this step, save step 3 (`"name": "Separation"`) with `decision` (`monolith` or `split`), `boundary` (what runs where, if split) and `reasoning`.

---

## Step 4 — API contracts

Define the interface between frontend/backend and between domains identified in Stage 2, contract-first — the contract is written before implementation, not derived from it.

Apply the same discipline `agent-skills`' `api-and-interface-design` documents:
- **Hyrum's Law** — every observable behavior becomes a de facto commitment once something depends on it; be deliberate about what's exposed, don't leak implementation details.
- **Contract first** — define request/response shapes before writing the implementation.
- **One-Version Rule** — design so consumers never have to choose between multiple versions of the same interface.

Depth matches scope — a small feature needs a short list of endpoint shapes, not a full OpenAPI spec, unless the scope actually warrants one.

As the last action of this step, save step 4 (`"name": "API contracts"`) with `contracts_markdown` — the agreed contracts as written for the API Contracts section, JSON-escaped.

---

## Step 5 — Non-functional requirements (quantified)

Stage 1 captured NFRs at the business level ("fast," "secure," "available") — this step translates them into technical targets:
- Performance: response time / latency budget under what load
- Scale: concurrent users, data volume, growth assumptions
- Availability: uptime target
- Security: specific controls, not just "secure" (auth model, data-at-rest/in-transit handling)
- Compliance: specific standards if the spec named any

Ask for numbers if the spec didn't give any. If none are available and the user doesn't have a target in mind, state the assumption explicitly rather than inventing a number silently.

Log it: `risk silent_decide ai_default "NFR target for <dimension>: assumed <value> — no user-provided number"` for each assumed target — this is exactly the kind of silent default `eng-flow-retro` should be watching for correlation against later rework.

**Project-level security policy.** Look for `eng-flow/security-policy.md` — project-level, sits alongside `design-system.md` (reused across specs, not rebuilt per run). If it exists, the Security NFR above must address each of its stated rules explicitly for this spec's scope, not just restate "secure." If it doesn't exist, offer via `AskUserQuestion`: "No project-level security policy found. Establish standing rules now (e.g. credential handling, least-privilege data access, write confirmation) that this and future specs' architecture/eng-review/code-review stages will check against?" Options: A) Yes — capture rules now B) No — this spec's Security NFR stands alone. If A and the user has no strong opinion yet, offer common local-first defaults as a starting point (edit to taste): local/MCP servers never forward credentials off the local machine; DB access defaults to least-privilege/read-only unless a story states a write is required; secrets load from environment variables and never enter committed config; state-mutating operations require explicit confirmation before executing. Save the agreed rules as a short numbered list to `eng-flow/security-policy.md`. This is a knowledge-asymmetry call — never invent or silently skip project security policy.

Log it: `risk knowledge_asymmetry open_question user_confirmed "security policy: established — N rules | declined | already existed"`.

As the last action of this step, save step 5 (`"name": "NFRs"`) with `nfrs` (one entry per dimension with `target` and `assumed`: true when no user-given number), and `security_policy` (`established`, `declined` or `existed`). `security-policy.md` is written during this step, not at Step 9, so a resume past Step 5 finds it already on disk.

---

## Step 6 — Deployment

Default to what the user actually does, not an assumed cloud target: local development, containers, GitHub (per the project's real practice) — document that as the deployment topology (dev/staging/prod environments, or fewer if that's the real setup), the CI pipeline (GitHub Actions or equivalent), and any infrastructure-as-code in use.

Add a short **cloud-ready notes** subsection: what would need to change to deploy this to a cloud target later (which components would need it, roughly what'd be involved) — informational, not a decision to act on now. This is what makes the doc useful to another engineer picking this up later without forcing a cloud decision that isn't warranted yet.

Log it: `risk silent_decide ai_inferred "deployment topology: matched actual practice (<local/containers/GitHub>)"`.

As the last action of this step, save step 6 (`"name": "Deployment"`) with `topology`, `ci`, `iac` and `cloud_ready_notes`.

---

## Step 7 — ADRs

Reuse `agent-skills`' convention rule: **before creating an ADR, check for an existing convention in the repo** (an `adr/` or `docs/decisions/` directory, an `.adr-dir` file, existing numbered ADRs) and match its location, numbering, and section headings. Only apply the default below if no convention exists.

Write one ADR per decision from Steps 1-6 that would be expensive to reverse — framework/library choice, data model, auth strategy, API architecture style, deployment/hosting choice. Not every small choice needs one; a Step 1 "boring by default" pick with an obvious reason usually doesn't.

Log it: `risk silent_decide ai_inferred "ADR convention: <matched existing repo convention | applied default location/numbering>"`.

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

As the last action of this step, save step 7 (`"name": "ADRs"`) with `convention` (matched location/numbering, or `default`) and `adrs` — one entry per ADR with `path` and `markdown` (the full ADR text, JSON-escaped); an empty list if none are warranted. Step 9 writes these files.

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

Show the draft, ask: "Does this match what you'd actually build, or anything to change?" Log it: `risk open_question <user_confirmed|user_revised> "draft accepted|draft revised: <what changed>"`.

As the last action of this step, once the user has confirmed, save step 8 (`"name": "Draft"`) with `architecture_markdown` — the full confirmed text, JSON-escaped — and `revisions` (empty if accepted as-is). If the revisions changed an ADR, re-save step 7 with the updated ADR text first.

---

## Step 9 — Save

Write `architecture.md` to the same spec folder (`eng-flow/specs/<dated-slug>/architecture.md`), and any ADR files to `eng-flow/specs/<dated-slug>/adr/` (or the matched existing convention's location). When resuming, write them from the draft's saved `architecture_markdown` (step 8) and `adrs` (step 7).

As the last action of this step, once every file is written, delete the draft: `python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-draft" clear eng-flow-architecture "<dated-slug>"`. If any write failed, skip this and keep the draft; tell the user which file didn't save.

---

## Step 10 — Report back

Confirm the saved paths. Tell the user this feeds Stage 4 (epics/stories/tasks — breaking this into executable work), not yet run.

If this run was in guide mode, add a "Decisions I made / decisions you made" summary here, drawn from this run's `eng-flow-decision-log` calls.

Run the Step 10 analytics-finish call (see Analytics section above) before ending.
