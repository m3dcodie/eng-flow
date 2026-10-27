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

At the start of every numbered step in Steps 0-6 (including Step 0), run `python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-analytics-checkpoint" eng-flow-epics-stories-tasks "<step name>" "<dated-slug>"`. As the last action of Step 6, run `python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-analytics-finish" eng-flow-epics-stories-tasks "<dated-slug>"`. Step 7 is a separate invocation and gets its own analytics calls — see its own note below. See `eng-flow-spec`'s Analytics section for what this logs and why; rollup via `eng-flow-analytics` (Stage 10).

## Decision Ledger

Check `$ARGUMENTS` for a `--guide` token; if present, every decision point below gets an explicit `AskUserQuestion` instead of a silent default, and Step 6's report (Step 7's own report for the breakdown run) adds a "Decisions I made / decisions you made" summary. Log every decision point via `python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-decision-log" eng-flow-epics-stories-tasks "<step>" <reason> <mode> <owner> "<description>" "<dated-slug>"` (use `eng-flow-epics-stories-tasks-breakdown` as the skill name for Step 7's own calls, matching its separate analytics tag). See `eng-flow-spec`'s Decision Ledger section for the taxonomy and why. Rollup/analysis: `eng-flow-retro` Step 1 (Stage 9).

## Drafts

Steps 0–6 save as they go to `eng-flow/.drafts/epics-stories-tasks--<dated-slug>.json` (`<dated-slug>` is the spec folder's), so a crash loses at most the step in progress. Step 7 (task breakdown) has no draft — it's short and cheap to re-run. The shared rules (Resume, Review, Discard, input drift) are in `${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/draft-resume.md` — Read it before resuming, reviewing or discarding. Every save in this file is this call, with the step's JSON on stdin — short form, JSON-escaped:

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-draft" save eng-flow-epics-stories-tasks "<dated-slug>" <step> <<'EOF'
{"status": "done", "name": "<step name>", "answers": {…}}
EOF
```

Degrades silently like the analytics scripts — a failed save prints a warning and the run carries on.

## Step 0 — Find the inputs

Locate the spec folder this run is for (`eng-flow/specs/<dated-slug>/`, containing `spec.md`, `domain-model.md`, `architecture.md`). If any are missing, tell the user which stage to run first. Check for drafts: `python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-draft" list eng-flow-epics-stories-tasks` — one line per draft, starting with the spec folder's `<dated-slug>`. If you ask which spec folder this run is for, a folder with a draft gets "draft: stopped after Step n, age" in its option description. If the chosen folder has a draft → **Resume** (`draft-resume.md`) — no further Step 0 questions.

Check whether `eng-flow/backlog/` already exists. If it does, this may be an update pass (new spec, existing backlog) rather than a first run — don't recreate or duplicate epics that already reference this project; just add what's new.

As the last action of this step (new run only), save step 0 (`"name": "Find the inputs"`) with `topic` (the spec's title), `spec_folder` and `update_pass` (true if `eng-flow/backlog/` existed), then record the inputs:

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-draft" inputs eng-flow-epics-stories-tasks "<dated-slug>" "eng-flow/specs/<dated-slug>/spec.md" "eng-flow/specs/<dated-slug>/domain-model.md" "eng-flow/specs/<dated-slug>/architecture.md"
```

**Stale on review:** the epics (Step 1) changed after Step 2 was saved → `truncate ... 1`, re-save step 1, continue at Step 2. A story (Step 2) changed after Step 3 was saved → `truncate ... 2`, re-save step 2, continue at Step 3.

---

## Step 1 — Propose epics

Candidates come from two places, cross-referenced:
- The spec's **domains touched** list (each domain is often a natural epic boundary)
- The architecture's **system diagram** groupings (confirms which domains are actually coupled vs independent — two domains diagrammed as tightly coupled might be one epic, not two)

Present the candidate epic list to the user and ask them to confirm, merge, or split — don't finalize silently. An epic should represent a real outcome ("users can complete checkout end-to-end"), not just restate a domain name.

Log it: `risk open_question <user_confirmed|user_revised> "epics: <confirmed as-is|merged/split: how>"`.

For each confirmed epic, capture: name, outcome/goal (what value this delivers, not what gets built), source spec reference, linked domains.

As the last action of this step, save step 1 (`"name": "Epics"`) with `epics` — one entry per confirmed epic with `name`, `outcome` and `domains`.

---

## Step 2 — Derive stories per epic

Walk the spec's **user journeys** — each journey, or each meaningful step within one, is a candidate story. Apply the test: **who benefits, and what do they gain?** If you can't answer both, it's not a real story yet — narrow it.

Size against the **1-3 day completable** bar (a story too big for that is really two stories — split it now, don't defer the problem). Ask the user to confirm the split rather than asserting it.

Log it: `risk open_question <user_confirmed|user_revised> "story '<name>' sizing: <accepted|split: how>"`.

For each story, write acceptance criteria — specific, testable conditions, not "works correctly."

As the last action of this step, save step 2 (`"name": "Stories"`) with `stories` — one entry per story with `name`, `epic`, `as_a`, `i_want`, `so_that` and `acceptance_criteria` (list).

---

## Step 3 — Cross-check against architecture

For each story, check the architecture's system diagram and API contracts: does this story touch two or more independently-diagrammed subsystems? If so, flag it — either as a candidate to split further, or, if it genuinely can't be split, as a coordination note (which subsystems, what the dependency is) so it isn't picked up assuming it's a single-subsystem change.

Log flagged stories: `risk silent_decide ai_inferred "story '<name>' spans 2+ subsystems: <split candidate | coordination note>"`.

As the last action of this step, save step 3 (`"name": "Architecture cross-check"`) with `architecture_notes` — one entry per story with `story`, `subsystems` and `note` (the coordination note, or `single subsystem`). If a story was split here, re-save step 2 with the split stories first.

---

## Step 4 — Traceability

Every story references its parent epic and the source spec path. Every epic references its source spec path and linked domains. This is what makes the backlog navigable later — don't skip it even for a small backlog; it's cheap now and expensive to reconstruct later.

As the last action of this step, save step 4 (`"name": "Traceability"`) with `files` — every epic and story file Step 5 will write, each with `path` and `markdown` (the full file text per the templates below, JSON-escaped).

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

Write files, don't overwrite existing epics/stories from prior runs — if this run's proposals overlap an existing epic/story, ask the user whether to update it or add a new one. Log overlaps: `risk open_question user_confirmed "'<slug>' overlaps existing: <updated|added new>"`.

Write from step 4's saved `files`. When resuming into this step, a crash may have left some of them written already: a file whose content matches the draft's `markdown` exactly was written by this run — skip it, it isn't an overlap.

As the last action of this step, once every file is written, delete the draft: `python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-draft" clear eng-flow-epics-stories-tasks "<dated-slug>"`. If any write failed, skip this and keep the draft; tell the user which file didn't save.

---

## Step 6 — Report back

List the epics and stories created (or updated). Tell the user tasks are separate: "say 'break story `<slug>` into tasks' when you're ready to implement it" — don't generate tasks for the whole backlog now.

If this run was in guide mode, add a "Decisions I made / decisions you made" summary here, drawn from this run's `eng-flow-decision-log` calls.

Run the Step 6 analytics-finish call (see Analytics section above) before ending.

---

## Step 7 (separate invocation, on-demand, per-story) — Task breakdown

**Analytics:** this is its own run, tagged separately. At the start, run `python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-analytics-checkpoint" eng-flow-epics-stories-tasks-breakdown "task-breakdown" "<story-slug>"`. When done (after saving `tasks.md` below), run `python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-analytics-finish" eng-flow-epics-stories-tasks-breakdown "<story-slug>"`.

Only runs when the user names a specific story to implement. Read that story's file (`eng-flow/backlog/stories/<story-slug>.md`) and the relevant parts of `architecture.md` for that domain.

Break the story into tasks:
- **Vertical slices**, not horizontal layers — each task should deliver a working, testable increment (e.g. "user can submit the form end-to-end," not "build the database schema" as its own task).
- **Dependency order** — map what depends on what, build foundations first.
- **Size each task**: XS (1 file) / S (1-2 files) / M (3-5 files) / L (5-8 files, consider splitting) / XL (8+ files — break it down further, don't leave it XL).
- **Split further if**: it'd take more than ~2 hours of focused work, acceptance criteria needs more than 3 bullets to state, it touches 2+ independent subsystems, or the task title has "and" in it.
- **Checkpoint** every 2-3 tasks: tests pass, build succeeds, core flow works end-to-end.

Log the sizing/splitting pass: `eng-flow-decision-log eng-flow-epics-stories-tasks-breakdown "task breakdown" risk silent_decide ai_inferred "story '<slug>': <N> tasks, split per size/dependency rules" "<story-slug>"`.

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
