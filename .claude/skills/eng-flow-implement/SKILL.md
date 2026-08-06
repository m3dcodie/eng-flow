---
name: eng-flow-implement
description: Production Stage 5 — implements one pending task from a story's tasks.md, one at a time, using an incremental TDD loop (implement, test, verify, commit). Terminal stage after eng-flow-epics-stories-tasks. Whole-story auto mode is not built yet — this only ever does the next single task, then stops.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Write
  - Edit
  - Bash
  - AskUserQuestion
triggers:
  - implement this task
  - implement the story
  - build this task
  - eng-flow implement
  - start implementing
  - work on this task
---

# eng-flow implement

Stage 5 of the production track, terminal — consumes a story's `tasks.md` (from `eng-flow-epics-stories-tasks`'s on-demand task breakdown) and implements the next pending task. One task per run, using an incremental TDD loop, then stop — no whole-plan auto mode yet; that's deferred.

## Analytics

At the start of every step below (including Step 0 and Step 0.5), run `python3 .claude/skills/lib/bin/eng-flow-analytics-checkpoint eng-flow-implement "<step name>" "<story-slug>"`. As the last action of Step 5, run `python3 .claude/skills/lib/bin/eng-flow-analytics-finish eng-flow-implement "<story-slug>"`. See `eng-flow-spec`'s Analytics section for what this logs and why; rollup via `eng-flow-analytics` (Stage 10).

## Step 0 — Find the story

Look for `eng-flow/backlog/stories/*/tasks.md`. If none exist, tell the user to break a story into tasks first (`eng-flow-epics-stories-tasks` Step 7).

If exactly one story has pending (unchecked) tasks, use it. If more than one does, ask via `AskUserQuestion` which story to work on. If a story is named in the request, use that one directly without asking.

Read the story file itself (`eng-flow/backlog/stories/<story-slug>.md`) for acceptance criteria and architecture notes, and the relevant parts of `architecture.md` for that domain — this is the context the task was scoped against.

If `eng-flow/learnings.md` exists (built up by `eng-flow-retro`, Stage 9), skim it for entries whose **files** or subject matter overlap what this task touches. A relevant `pitfall` entry means don't repeat it; a relevant `pattern`, `preference`, or `operational` entry means apply it without re-deriving it. This is the step that makes retros worth running — a learnings log nobody reads before starting new work is dead weight.

---

## Step 0.5 — Branch setup

Work on a story never happens directly on the base/default branch. Check the current branch:

- **Already on a feature branch for this story** (name matches or contains the story slug): continue as-is.
- **On base, or on an unrelated branch:** pull latest base (`git fetch origin <base> && git pull`), then create a new branch from it — `feature/<story-slug>` (or `fix/<story-slug>` if the story is a bug fix, judged from its title/type) — and switch to it. Branch naming convention reused from `agent-skills`' `git-workflow-and-versioning` (attribution: `docs/DECISIONS.md`).
- **Uncommitted changes present that aren't part of this task:** stop and ask how to handle them before switching/creating branches — don't carry unrelated work into a new feature branch silently.

This is what makes `eng-flow-ship`'s pre-flight (Stage 8, "abort if on the base branch") meaningful — the branch it expects already exists by the time a story's tasks are done.

---

## Step 1 — Pick the next task

Take the first unchecked task in `tasks.md`, in file order (task numbering already reflects dependency order from Stage 4). If its listed dependencies aren't checked off yet, stop and tell the user which task needs to land first rather than skipping ahead.

---

## Step 2 — Implement (incremental loop)

Adapted from `agent-skills`' `incremental-implementation` and `test-driven-development` (attribution: `docs/DECISIONS.md`):

1. **Read** the task's acceptance criteria, description, and files-likely-touched from `tasks.md`.
2. **Load context** — read the actual files involved, and anything they depend on. Discover the repo's real test/build/lint commands first (package.json scripts, Makefile, CI config) rather than guessing.
3. **Simplest thing that could work** — implement the smallest complete vertical slice for this task. No abstractions ahead of a second use case, no touching files outside this task's scope.
4. **Test** — write a failing test for the expected behavior if the repo has a test setup (RED), then implement to pass it (GREEN). If the repo has no test framework, say so explicitly rather than skipping silently.
5. **Verify** — run the full test suite (regression check, not just the new test), run the build, run lint/typecheck if the stack has them.
6. **Commit** — one task, one commit, descriptive message. Stage only the files this task touched plus the `tasks.md` checkbox update, not a blanket `git add -A`.
7. **Check off the task** in `tasks.md`.

---

## Step 3 — Scope discipline

Touch only what the task requires. If something else needs fixing along the way, don't fix it inline — log it:

```
NOTICED BUT NOT TOUCHING:
- [file] — [what's off, why it's out of scope for this task]
```

Surface these in the report (Step 5); don't silently expand the task.

---

## Step 4 — Hard stop conditions

Stop and ask the user, don't push through, when:
- A test or the build breaks and there's no obvious fix.
- The task needs a decision that isn't covered by `tasks.md`, the story, or `architecture.md`.
- The task is high-risk or hard to reverse — auth/permissions, destructive migrations, payments, deletions, deploys, anything touching secrets, or anything that can't be undone with `git revert`. Get explicit sign-off before writing code, not after.

---

## Step 5 — Report back

Summarize: task completed, tests added, the commit made, anything flagged as "noticed but not touching." State how many tasks remain unchecked in this story's `tasks.md` — if more remain, tell the user to re-invoke this skill to continue with the next one (one task per run, by design).

Run the Step 5 analytics-finish call (see Analytics section above) before ending.

---

## Not built yet

Whole-story "auto" mode (implement every remaining task in one approved pass, per-task commits, single upfront checkpoint — mirroring `agent-skills`' `/build auto`) is deliberately deferred. Don't imply it's available; if the user asks for it, say it's not built yet rather than approximating it by silently chaining several single-task runs without the upfront approval/clean-baseline discipline that mode requires.
