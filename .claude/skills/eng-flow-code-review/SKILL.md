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
  - Agent
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

## Step 3.5 — Independent subagent pass

Step 3 ran in this conversation — often the same one that wrote the diff. Counter self-review bias with independent passes, dispatched via the `Agent` tool, foreground (their output feeds Step 5). Give each subagent **only what it needs to review — not this conversation's context or Step 1-3's findings**.

**Always** — one blind adversarial subagent, general-purpose:

> "Read the diff for `<scope from Step 0>` (`git diff <base>...HEAD` or equivalent). You are an independent senior engineer reviewing this — you have not seen any prior review and have no stake in the code. Find what a structured checklist review would miss: assumptions that don't hold, interactions between changed files, error paths that look handled but aren't, anything that works in the demo path but not the edge case. For each finding: what's wrong, severity, file:line, and the fix."

**Conditionally** — one security-specialist subagent, only if the diff touches auth, payments, secrets, PII, data access, or config/session handling. Skip it and say why if the diff clearly doesn't. Prompt adapted from `agent-skills`' `security-auditor` persona (attribution: `docs/DECISIONS.md`), scoped to this diff:

> "Read the diff for `<scope from Step 0>`. You are a security engineer conducting a focused audit. Check: input validation at boundaries, injection vectors, auth/authz on every changed endpoint, secrets in code/logs, encryption in transit/at rest where relevant, IDOR (can a user reach another user's data through this change). Map findings to OWASP Top 10 where relevant. For each: severity (Critical/High/Medium/Low/Info), file:line, exploit scenario for Critical/High, and the fix."

If both trigger, dispatch both `Agent` calls in the same assistant turn so they run in parallel. If a subagent fails or times out, note it and continue — the structured Step 3 review still stands on its own.

Merge each subagent's findings into Step 4's severity list, tagged by source (`five-axis` / `adversarial` / `security specialist`). Drop exact duplicates; keep independent findings that reach the same conclusion — that agreement is informative, not redundant.

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

For each Critical or Required finding, state the issue, its source tag, the recommendation, and why — then stop and wait for the user's explicit response before raising the next one, same discipline as every other eng-flow stage. Don't assume the obvious fix and move on.

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
**Source:** [five-axis | adversarial | security specialist]
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
