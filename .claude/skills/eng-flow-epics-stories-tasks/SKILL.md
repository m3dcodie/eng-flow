---
name: eng-flow-epics-stories-tasks
description: Production Stage 4 — turns a spec's domains and journeys, cross-checked against the architecture's system diagram, into epics and stories in a project-level backlog (not per-spec — epics/stories outlive any one spec run). Task-level breakdown is separate and on-demand, only generated per-story once the user decides to implement it.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Write
  - Bash
  - AskUserQuestion
triggers:
  - epics
  - stories
  - break this down
  - task breakdown
  - eng-flow epics
  - split into stories
  - break story into tasks
---

# eng-flow epics / stories / tasks

Stage 4 of the production track. Unlike Stages 1-3, output here is **project-level**, not per-spec — epics and stories persist and accumulate in a backlog as the project grows, with each story traceable back to the spec that spawned it. Tasks are a separate, on-demand step: generated per-story, only when the user is actually ready to implement that story, not automatically for the whole backlog.

## Analytics

At the start of every numbered step in Steps 0-6 (including Step 0), run `python3 .claude/skills/lib/bin/eng-flow-analytics-checkpoint eng-flow-epics-stories-tasks "<step name>" "<dated-slug>"`. As the last action of Step 6, run `python3 .claude/skills/lib/bin/eng-flow-analytics-finish eng-flow-epics-stories-tasks "<dated-slug>"`. Step 7 is a separate invocation and gets its own analytics calls — see its own note below. See `eng-flow-spec`'s Analytics section for what this logs and why; rollup via `eng-flow-analytics` (Stage 10).

## Step 0 — Find the inputs

Locate the spec folder this run is for (`eng-flow/specs/<dated-slug>/`, containing `spec.md`, `domain-model.md`, `architecture.md`). If any are missing, tell the user which stage to run first.

Check whether `eng-flow/backlog/` already exists. If it does, this may be an update pass (new spec, existing backlog) rather than a first run — don't recreate or duplicate epics that already reference this project; just add what's new.

---

## Step 1 — Propose epics

Candidates come from two places, cross-referenced:
- The spec's **domains touched** list (each domain is often a natural epic boundary)
- The architecture's **system diagram** groupings (confirms which domains are actually coupled vs independent — two domains diagrammed as tightly coupled might be one epic, not two)

Present the candidate epic list to the user and ask them to confirm, merge, or split — don't finalize silently. An epic should represent a real outcome ("users can complete checkout end-to-end"), not just restate a domain name.

For each confirmed epic, capture: name, outcome/goal (what value this delivers, not what gets built), source spec reference, linked domains.

---

## Step 2 — Derive stories per epic

Walk the spec's **user journeys** — each journey, or each meaningful step within one, is a candidate story. Apply the test: **who benefits, and what do they gain?** If you can't answer both, it's not a real story yet — narrow it.

Size against the **1-3 day completable** bar (a story too big for that is really two stories — split it now, don't defer the problem). Ask the user to confirm the split rather than asserting it.

For each story, write acceptance criteria — specific, testable conditions, not "works correctly."

---

## Step 3 — Cross-check against architecture

For each story, check the architecture's system diagram and API contracts: does this story touch two or more independently-diagrammed subsystems? If so, flag it — either as a candidate to split further, or, if it genuinely can't be split, as a coordination note (which subsystems, what the dependency is) so it isn't picked up assuming it's a single-subsystem change.

---

## Step 4 — Traceability

Every story references its parent epic and the source spec path. Every epic references its source spec path and linked domains. This is what makes the backlog navigable later — don't skip it even for a small backlog; it's cheap now and expensive to reconstruct later.

---

## Step 5 — Save (project-level backlog)

Compute slugs the same way as spec (`lowercase, spaces to dashes, strip to [a-z0-9._-]`), from epic/story names — no date prefix here, since these persist independent of when they were written.

```
eng-flow/backlog/
  epics/<epic-slug>.md
  stories/<story-slug>.md
```

Epic file:
```markdown
# Epic: [Name]

## Outcome
[What value this delivers]

## Source
Spec: eng-flow/specs/<dated-slug>/spec.md
Domains: [linked domains]

## Stories
- [story-slug]: [one-line title]
```

Story file:
```markdown
# Story: [Name]

## As a / I want / So that
As a [user], I want to [action], so that [benefit].

## Parent Epic
[epic-slug]

## Source
Spec: eng-flow/specs/<dated-slug>/spec.md

## Acceptance Criteria
- [ ] [Specific, testable condition]

## Architecture Notes
[Subsystems touched; coordination note if it spans 2+ from Step 3, else "single subsystem"]
```

Write files, don't overwrite existing epics/stories from prior runs — if this run's proposals overlap an existing epic/story, ask the user whether to update it or add a new one.

---

## Step 6 — Report back

List the epics and stories created (or updated). Tell the user tasks are separate: "say 'break story `<slug>` into tasks' when you're ready to implement it" — don't generate tasks for the whole backlog now.

Run the Step 6 analytics-finish call (see Analytics section above) before ending.

---

## Step 7 (separate invocation, on-demand, per-story) — Task breakdown

**Analytics:** this is its own run, tagged separately. At the start, run `python3 .claude/skills/lib/bin/eng-flow-analytics-checkpoint eng-flow-epics-stories-tasks-breakdown "task-breakdown" "<story-slug>"`. When done (after saving `tasks.md` below), run `python3 .claude/skills/lib/bin/eng-flow-analytics-finish eng-flow-epics-stories-tasks-breakdown "<story-slug>"`.

Only runs when the user names a specific story to implement. Read that story's file (`eng-flow/backlog/stories/<story-slug>.md`) and the relevant parts of `architecture.md` for that domain.

Break the story into tasks:
- **Vertical slices**, not horizontal layers — each task should deliver a working, testable increment (e.g. "user can submit the form end-to-end," not "build the database schema" as its own task).
- **Dependency order** — map what depends on what, build foundations first.
- **Size each task**: XS (1 file) / S (1-2 files) / M (3-5 files) / L (5-8 files, consider splitting) / XL (8+ files — break it down further, don't leave it XL).
- **Split further if**: it'd take more than ~2 hours of focused work, acceptance criteria needs more than 3 bullets to state, it touches 2+ independent subsystems, or the task title has "and" in it.
- **Checkpoint** every 2-3 tasks: tests pass, build succeeds, core flow works end-to-end.

Task template:
```markdown
## Task [N]: [Short descriptive title]

**Description:** [One paragraph]

**Acceptance criteria:**
- [ ] [Specific, testable condition]

**Verification:**
- [ ] Tests pass: [command]
- [ ] Build succeeds: [command]
- [ ] Manual check: [what to verify]

**Dependencies:** [Task numbers, or "None"]

**Files likely touched:**
- `path/to/file`

**Estimated scope:** [XS | S | M | L | XL]
```

Save to `eng-flow/backlog/stories/<story-slug>/tasks.md`.
