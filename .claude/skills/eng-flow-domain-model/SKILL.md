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
  - Agent
triggers:
  - domain model
  - model the domain
  - map the domains
  - eng-flow domain model
---

# eng-flow domain model

Stage 2 of the production track. Translates a spec's plain-language requirements into structure — entities, relationships, flows — while staying strictly conceptual. If a question about tech stack, database choice, deployment, or API shape comes up here, defer it explicitly: "that's Stage 3 (architecture), not this stage" — don't answer it now even if you know the answer.

## Analytics

At the start of every numbered step below (including Step 0), run `python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-analytics-checkpoint" eng-flow-domain-model "<step name>" "<dated-slug>"`. As the last action of Step 10, run `python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-analytics-finish" eng-flow-domain-model "<dated-slug>"`. See `eng-flow-spec`'s Analytics section for what this logs and why; rollup via `eng-flow-analytics` (Stage 10).

## Decision Ledger

Check `$ARGUMENTS` for a `--guide` token; if present, every decision point below gets an explicit `AskUserQuestion` instead of a silent default, and Step 10's report adds a "Decisions I made / decisions you made" summary. Log every decision point via `python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-decision-log" eng-flow-domain-model "<step>" <reason> <mode> <owner> "<description>" "<dated-slug>"`. See `eng-flow-spec`'s Decision Ledger section for the taxonomy and why. Rollup/analysis: `eng-flow-retro` Step 1 (Stage 9).

Also check `$ARGUMENTS` for `--review-model=<opus|sonnet|haiku|fable>` — see Step 7 for what it controls.

## Drafts

This stage saves as it goes to `eng-flow/.drafts/domain-model--<dated-slug>.json` (`<dated-slug>` is the spec folder's), so a crash loses at most the step in progress — in Steps 1 and 8, only the domain or finding in progress. The shared rules (Resume, Review, Discard, `in_progress` loops, input drift) are in `${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/draft-resume.md` — Read it before resuming, reviewing or discarding. Every save in this file is this call, with the step's JSON on stdin — agreed answers in short form, JSON-escaped:

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-draft" save eng-flow-domain-model "<dated-slug>" <step> <<'EOF'
{"status": "done", "name": "<step name>", "answers": {…}}
EOF
```

Degrades silently like the analytics scripts — a failed save prints a warning and the run carries on.

## Step 0 — Find the spec

Look for `eng-flow/specs/*/spec.md`. If none exist, tell the user to run `eng-flow-spec` first — this stage has nothing to model without one. Then check for drafts: `python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-draft" list eng-flow-domain-model` — one line per draft, starting with the spec folder's `<dated-slug>`. If more than one spec exists, ask which spec this run is for; a spec with a draft gets "draft: stopped after Step n, age" in its option description. If exactly one, use it. If the chosen spec has a draft → **Resume** (`draft-resume.md`) — no further Step 0 questions.

Read the spec's **Domains Touched** list and **User Journeys** — these are the inputs to everything below. If the spec has no domains list (e.g. it used the lightweight feature/story template), ask the user directly: "Which domains/functional areas does this touch?" Log it: `risk open_question user_confirmed "domains list: <what was named>"`.

As the last action of this step (new run only), save step 0 (`"name": "Find the spec"`) with `topic` (the spec's title), `spec_folder`, `domains` and `journeys` (lists, as read or as the user named them), then record the input:

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-draft" inputs eng-flow-domain-model "<dated-slug>" "eng-flow/specs/<dated-slug>/spec.md"
```

**Stale on review:** an answer from Steps 1–5 changed after Step 6 was saved — the draft text and its review are out of date. `truncate ... 5`, re-save the changed steps, continue at Step 6.

---

## Step 1 — Domain model, per domain

For each domain named in the spec, ask (don't invent):

1. What are the core entities in this domain? (the nouns the business actually uses — pull from the spec's own language, don't rename them into something more "technical")
2. What are the key relationships between these entities, and between entities in this domain and entities in other named domains?
3. Is there any term here that means something different to different stakeholders? (surface ubiquitous-language conflicts now — cheap to fix here, expensive once code exists)

Keep this at the conceptual level: entity names, relationships, cardinality if it's non-obvious. No fields, no types, no persistence — that's implementation, not modeling.

Log it: `knowledge_asymmetry open_question user_confirmed "domain '<name>': entities/relationships/language conflicts"` — the business's own vocabulary for its entities isn't something the AI can infer, it has to be asked per domain.

After each domain, save step 1 as `{"status": "in_progress", "name": "Domains", "answers": {"domains": [...]}}` — every domain finished so far, each with `name`, `entities`, `relationships`, `language_notes`. After the last domain, save it once more as `"done"`.

---

## Step 2 — Data flow

Using the spec's user journeys and functional requirements: for each journey, trace which domains it touches and in what order. Ask the user to confirm or correct the sequence rather than asserting it — the spec describes user-facing behavior, not internal flow, so this is genuinely new information, not a restatement.

Log it: `risk open_question <user_confirmed|user_revised> "journey '<name>' flow: <domain sequence>"`.

Represent as a simple flow list or diagram (mermaid `flowchart` is fine if the host renders it, otherwise plain ordered steps):

```
User action → Domain A (does X) → Domain B (does Y) → outcome
```

As the last action of this step, save step 2 (`"name": "Data flow"`) with `flows` — one entry per journey with `journey` and the confirmed `sequence`.

---

## Step 3 — Conceptual system diagram

One diagram showing how the named domains/subsystems relate — a context map, not a deployment diagram. Boxes are domain names from Step 1 (e.g. "Checkout," "Inventory," "Billing"), arrows show which domain calls/depends on which. **Do not label boxes with technology** (no "Postgres," "React," "Lambda") — if a box needs a technology label to make sense, that's a sign the diagram has drifted into Stage 3's territory; pull it back to domain names only.

Draft it as mermaid first — that's what goes in the saved file. Then ask once: "Want this rendered as a diagram you can actually see, or is the mermaid source in the saved file enough?" Default to skipping if the user doesn't ask. If yes, load the `artifact-diagramming` skill and publish via the `Artifact` tool; note the artifact URL alongside the saved diagram source in Step 6/7 so it isn't lost.

Log it: `risk silent_decide ai_default "diagram rendering: skipped, mermaid source only"` if the default applies, otherwise `open_question user_confirmed`.

As the last action of this step, save step 3 (`"name": "System diagram"`) with `mermaid` (the diagram source) and `artifact_url` (or `null`).

---

## Step 4 (optional) — Mockups / wireframes

Ask: "Would a rough wireframe help validate any of these journeys?" Default: skip unless the user asks or a journey is genuinely hard to follow in text. If yes, keep it low-fidelity — boxes and labels showing flow and content, not visual design (no colors, no branding, no component library references). If produced, offer the same Artifact-rendering treatment as Step 3.

Log it: `risk silent_decide ai_default "wireframes: skipped"` if the default applies, otherwise `open_question user_confirmed`.

As the last action of this step, save step 4 (`"name": "Wireframes"`) with `wireframes` — one entry per journey with `journey`, `wireframe` and `artifact_url` (or `null`). If skipped: `{"status": "skipped", "name": "Wireframes", "reason": "user declined wireframes"}`.

---

## Step 5 — Refinement Q&A

While modeling, gaps in the spec surface that weren't visible when it was pure prose — e.g. "the spec's journey says a user cancels an order, but doesn't say what happens to reserved inventory." Collect these as explicit questions, ask the user, and note the answer here rather than silently amending the spec file itself (the spec stays the record of what was agreed at Stage 1; this stage's answers extend it, they don't retroactively rewrite it).

Log each gap: `risk open_question user_confirmed "spec gap: <question> → <answer>"`.

As the last action of this step, save step 5 (`"name": "Refinement"`) with `gaps` — one entry per gap with `question` and `answer` (an empty list if none surfaced).

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

Show the draft, ask: "Does this match how you think about the domains, or anything to correct?" Log it: `risk open_question <user_confirmed|user_revised> "draft accepted|draft revised: <what changed>"`.

As the last action of this step, once the user has confirmed, save step 6 (`"name": "Draft"`) with `domain_model_markdown` — the full confirmed text, JSON-escaped — and `revisions` (empty if accepted as-is).

---

## Step 7 — Independent subagent review

Steps 1-6 ran in this conversation, built up interactively, one domain/journey at a time — good for capturing the user's mental model, bad at catching things that only show up when you look at the whole doc at once: a domain named in the spec but dropped from the model, a relationship asserted from one side but not the other, a diagram box with no corresponding prose section. Counter it with one blind pass, same pattern Stage 3.5 (`eng-flow-eng-review`) uses on `architecture.md`.

Spawn a single `Agent` call (foreground — its output feeds Step 8, so wait for it), general-purpose, with a prompt that gives it **only the file paths, not this conversation's context or Steps 1-6's findings**:

> "Read `eng-flow/specs/<dated-slug>/spec.md` and the just-drafted `domain-model.md` (pass its content directly, or the path if saved). You are an independent reviewer checking the domain model for internal consistency — you have not seen how it was built and have no stake in the choices made. Check: (1) does every domain in the spec's 'Domains Touched' list appear in the model, and does every domain in the model trace back to the spec — no silent additions or drops; (2) is every cross-domain relationship reflected consistently on both sides, no dangling references or contradicting cardinality; (3) is the same entity/term used consistently across domain sections and the data flow section, or does the language drift; (4) does every user journey in the spec have a matching entry in the Data Flow section, and vice versa; (5) does the system diagram's boxes and arrows match the prose 1:1, and does it stay free of technology labels (this stage is conceptual-only — a 'Postgres' or 'Lambda' box is itself a finding); (6) is there an edge case a journey implies but the model never addresses. For each finding: what's wrong, the section of `domain-model.md` it's in, and a fix."

**Reviewer model:** omit `model` on the `Agent` call by default — it inherits whichever model is running this conversation, the same convention `eng-flow-eng-review` and `eng-flow-code-review` use. If `$ARGUMENTS` carries `--review-model=<opus|sonnet|haiku|fable>`, pass that as the call's `model` instead — e.g. `--review-model=haiku` for a cheap pass on a small/low-risk domain model, `--review-model=opus` when the domain model is large or the spec flagged this as high-risk. Log the choice: `risk silent_decide ai_default "review model: inherited (no override)"` when no override was given, or `risk open_question user_confirmed "review model: <name> (--review-model override)"` when one was.

Fold the subagent's findings into Step 8. If it turns up nothing, say so — a clean pass is a valid outcome, don't manufacture findings to fill the step.

As the last action of this step, save step 7 (`"name": "Independent review"`) with `review_model` and `findings` — one entry per finding with `title`, `section`, `issue` and `fix` — so a resume doesn't spawn the reviewer again.

---

## Step 8 — Issue resolution

For each finding from Step 7, call `AskUserQuestion` individually — one finding per call, never batched. Name the issue, ground it in the specific `domain-model.md` section it reacts to (quote or point to the actual text — a finding that can't be tied to real doc content doesn't get raised), state options, give an opinionated recommendation, explain why.

**Stop and wait for the user's answer before raising the next finding.** Every finding gets an explicit accept/change/reject, same discipline as every other stage in this track.

Log each: `risk open_question <user_confirmed|user_revised> "finding '<title>': accepted|changed|rejected"`.

For anything accepted or changed, update the draft directly before Step 9 — `domain-model.md` gets saved once, already reflecting the review, not saved-then-patched.

After each finding, save step 8 as `{"status": "in_progress", "name": "Findings", "answers": {"resolved": [...], "domain_model_markdown": "…"}}` — `resolved` lists every finding answered so far (`title`, `resolution`: accepted/changed/rejected, `note`), and `domain_model_markdown` is the current text with those resolutions applied, so a resume doesn't apply them twice. After the last finding (or straight away if Step 7 found none), save it as `"done"` with the same fields.

---

## Step 9 — Save

Write to the same spec's folder: `eng-flow/specs/<dated-slug>/domain-model.md` — from step 8's saved `domain_model_markdown` when resuming.

As the last action of this step, once the file is written, delete the draft: `python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-draft" clear eng-flow-domain-model "<dated-slug>"`. If the write failed, skip this and keep the draft; tell the user.

---

## Step 10 — Report back

Confirm the saved path. Tell the user this feeds Stage 3 (architecture — tech stack, deployment, API shape), not yet run.

If this run was in guide mode, add a "Decisions I made / decisions you made" summary here, drawn from this run's `eng-flow-decision-log` calls.

Run the Step 10 analytics-finish call (see Analytics section above) before ending.
