---
name: eng-flow-code-review
description: Production Stage 6 — reviews an implemented diff (branch vs. base, or a specific story's commits) across five axes before it ships. Post-implementation counterpart to Stage 3.5's pre-implementation architecture review — this one reviews actual code, not a design doc.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Write
  - Bash
  - AskUserQuestion
triggers:
  - code review
  - review this pr
  - review this diff
  - check my diff
  - eng-flow code review
---

# eng-flow code review

Stage 6 of the production track. Reviews the actual diff produced by Stage 5's implementation loop — the post-implementation counterpart to Stage 3.5, which reviewed `architecture.md` before any code existed. Runs before the diff ships.

## Analytics

At the start of every step below (including Step 0), run `python3 .claude/skills/lib/bin/eng-flow-analytics-checkpoint eng-flow-code-review "<step name>" "<story-slug>"`. As the last action of Step 7, run `python3 .claude/skills/lib/bin/eng-flow-analytics-finish eng-flow-code-review "<story-slug>"`. See `eng-flow-spec`'s Analytics section for what this logs and why; rollup via `eng-flow-analytics` (Stage 10).

## Step 0 — Scope

Default to the current branch's diff against its base branch. If that's ambiguous (detached HEAD, no clear base, or the user named something specific — a story, a file, a commit range), ask via `AskUserQuestion` rather than guessing.

---

## Step 1 — Context

Find the story this diff implements (`eng-flow/backlog/stories/<story-slug>.md` and its `tasks.md`, if one exists). Read the acceptance criteria — review is grounded in what was supposed to happen, not a vibe check. If no story/task exists for this diff, proceed without it, but say so in the report.

---

## Step 2 — Tests first

Before reading implementation code, check the tests:
- Do tests exist for the change?
- Do they test behavior, not implementation details?
- Are edge cases and error paths covered, not just the happy path?
- Would they actually catch a regression if this code changed later?

---

## Step 3 — Five-axis review

Reused from `agent-skills`' `code-review-and-quality` (attribution: `docs/DECISIONS.md`). For each file in scope:

1. **Correctness** — matches the story's acceptance criteria; edge cases and error paths handled; no off-by-one/race/state-consistency issues.
2. **Readability & simplicity** — names are clear; control flow is straightforward; no clever tricks that should be simplified; could this be done in fewer lines; are abstractions earning their complexity (don't generalize before a third use case); any dead-code artifacts (unused vars, backwards-compat shims, `// removed` comments).
3. **Architecture** — follows existing patterns or justifies a new one; no unjustified duplication; dependencies flow the right direction; abstraction level appropriate; a "cleaner" refactor should reduce the number of concepts a reader holds, not just relocate them.
4. **Security** — input validated at boundaries; no secrets in code/logs; auth/authz checked where needed; queries parameterized; external data treated as untrusted.
5. **Performance** — no N+1 patterns; no unbounded loops/fetches; no unnecessary work in hot paths; pagination on list endpoints where relevant.

When flagging a structural problem, propose the specific remedy (extract a helper, replace a conditional chain with a dispatcher, separate orchestration from business logic) — not just "this is complex."

---

## Step 4 — Severity and gating

Tag every finding:

| Tag | Meaning | Gating |
|---|---|---|
| **Critical** | Blocks merge — security hole, data loss, broken functionality | One `AskUserQuestion` per finding, stop and wait |
| **Required** | Must address before merge | One `AskUserQuestion` per finding, stop and wait |
| **Nit** | Minor, optional — author may ignore | List in report, no individual gate |
| **Optional / Consider** | Worth considering, not required | List in report, no individual gate |
| **FYI** | Informational only | List in report, no individual gate |

Lead with what matters — correctness and security first. Don't bury a real Critical/Required finding under a pile of Nits; if there's one structural problem and ten nits, the structural problem *is* the review.

For each Critical or Required finding, state the issue, the recommendation, and why — then stop and wait for the user's explicit response before raising the next one, same discipline as every other eng-flow stage. Don't assume the obvious fix and move on.

---

## Step 5 — Dead code check

If the diff leaves anything orphaned (a function/component/constant nothing calls anymore), list it explicitly and ask before removing it — don't delete silently, don't leave it lying around unmentioned either.

---

## Step 6 — Verify the verification

Confirm: tests pass (run them), build succeeds (run it), and — if this touches UI — note that a manual check is still owed (this skill doesn't drive a browser; that's `eng-flow-qa`, Stage 7).

---

## Step 7 — Save and report

Write findings to `eng-flow/backlog/stories/<story-slug>/code-review.md` (or, if no story was found in Step 1, report inline without saving):

```markdown
# Code Review: [scope — branch/story name]
(source story: eng-flow/backlog/stories/<story-slug>.md, if found)

## Findings
### [Severity] [Title]
**File:** [path:line]
**Issue:** [...]
**Resolution:** [Fixed | Deferred — reason | Won't fix — reason]

## Verification
- Tests: [pass/fail, command run]
- Build: [pass/fail, command run]

## Verdict
[Approve | Request changes]
```

Report the verdict and a summary of what was fixed vs. deferred.

Run the Step 7 analytics-finish call (see Analytics section above) before ending.
