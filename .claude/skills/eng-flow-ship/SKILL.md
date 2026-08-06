---
name: eng-flow-ship
description: Production Stage 8, terminal — ships a feature branch. Gate-checks Stage 6/7 verdicts (code-review.md, qa-report.md) instead of re-reviewing, merges base before a final regression pass, bumps version/changelog if the project tracks them, commits, pushes, and opens a PR. Never deploys to a cloud target — matches local/container/GitHub practice.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Write
  - Bash
  - AskUserQuestion
triggers:
  - ship it
  - ship this
  - create a pr
  - eng-flow ship
  - ready to ship
---

# eng-flow ship

Stage 8 of the production track, terminal. Doesn't re-review — Stages 6 and 7 already did that work and left evidence behind; this stage checks that evidence is clear, then closes the loop: merge base, final regression pass, version/changelog, commit, push, PR.

## Step 1 — Pre-flight

Check the current branch. If it's the base branch or the repo's default branch, **abort**: "You're on the base branch — nothing to ship from here." Otherwise summarize what's shipping: `git status`, `git diff <base>...HEAD --stat`, `git log <base>..HEAD --oneline`.

---

## Step 2 — Gate check (reuses Stage 6/7 evidence, doesn't re-review)

Find the story this branch implements and look for:
- `eng-flow/backlog/stories/<story-slug>/code-review.md` — verdict must be **Approve**. Missing, or verdict is "Request changes," or the file predates the latest commit on this branch → stop, tell the user to run `eng-flow-code-review` (or resolve its open findings) first.
- If the branch touches front-end code: `eng-flow/backlog/stories/<story-slug>/qa-report.md` — no unresolved Critical/Required issues. Missing and the branch is clearly front-end-touching → tell the user `eng-flow-qa` hasn't been run; ask whether to proceed anyway or run it first, don't assume.

If no story is associated with this branch at all, ask the user to confirm they want to skip the gate check rather than silently proceeding.

---

## Step 3 — Merge base branch

```bash
git fetch origin <base> && git merge origin/<base> --no-edit
```

Catches integration conflicts against the latest base before the final test run, not after. If conflicts are simple (lockfiles, changelog ordering) attempt to resolve; if they touch actual logic, **stop** and show them — don't guess at a resolution.

---

## Step 4 — Final regression pass

Run the full test suite and the build against the merged state — this is the last check before anything ships, not a repeat of Stage 5/6's per-task checks. If anything fails, stop and fix it (route through `eng-flow-implement` if it's a real bug, not a merge artifact) before continuing.

---

## Step 5 — Version and changelog (only if the project already tracks these)

Check for a `VERSION` file or a `version` field in `package.json`/equivalent, and a `CHANGELOG.md`. If neither exists, skip this step silently — don't impose semantic versioning on a project that doesn't use it.

If they exist: bump per semver (patch/minor/major — ask if the right level isn't obvious from the diff) and add a changelog entry in the project's existing format.

---

## Step 6 — Commit, push

Commit any remaining changes from Steps 3-5 (merge resolution, version bump) with a descriptive message. Push the branch.

---

## Step 7 — Open a PR

If the remote has a PR workflow (GitHub), open one (`gh pr create`) with a description covering what changed and why. This is the default — matches local-dev/containers/GitHub practice. Not a cloud deploy step; if the project's actual deployment is more than "merge triggers CI," that's already documented in `architecture.md`'s deployment section, not this skill's concern.

---

## Step 8 — Report: GO/NO-GO with rollback plan

```markdown
## Ship Decision: GO | NO-GO

### Blockers (if NO-GO)
- [what's blocking, from Step 2 or 4]

### Shipped
- Branch: [name] → PR: [link]
- Version: [bumped to X, or "not tracked"]

### Rollback plan
- Trigger conditions: [what would prompt a revert]
- Rollback steps: [git revert / redeploy previous version — whatever applies]
```

Adapted from `agent-skills`' `/ship` decision format (attribution: `docs/DECISIONS.md`) — without its parallel subagent fan-out, since no other eng-flow skill uses that mechanism; the gate check in Step 2 does the equivalent job by reading Stage 6/7's saved verdicts instead.
