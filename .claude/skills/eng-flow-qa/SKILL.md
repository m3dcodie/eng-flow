---
name: eng-flow-qa
description: Production Stage 7 — browser-based, front-end-only behavioral QA. Exercises the running app (not the diff) to find bugs a code review can't catch. Browser-agnostic on Playwright specifically (gstack's own browser daemon is a wrapper around it) — falls back to a guided manual checklist if no browser automation is available. Documents bugs; does not fix them — fixes route through eng-flow-implement.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Write
  - Bash
  - AskUserQuestion
triggers:
  - qa this
  - qa test this
  - test the site
  - browser test
  - eng-flow qa
  - find bugs on site
---

# eng-flow qa

Stage 7 of the production track, browser-based and front-end only — skip entirely for backend-only, CLI, or library work with no UI. Exercises the *running* app rather than reading the diff (that's Stage 6); catches the class of bug that only shows up when something actually renders and gets clicked. Documents findings; does not fix them here.

## Step 0 — Detect browser automation

Check, in order:
1. An MCP-provided Playwright (or equivalent browser-automation) tool registered in this environment.
2. Playwright installed in the target project itself (`@playwright/test` or `playwright` in `package.json`, or `node_modules/playwright`) — drive it directly via a small script run through Bash.
3. Neither present.

If (1) or (2), use it for navigation, screenshots, and console-error checks. If neither, fall back to **guided-manual mode**: present the per-page checklist (Step 3) to the user, ask them to walk through it in their own browser and report back what they see (descriptions or pasted screenshots) — don't skip QA just because there's no automation, degrade the mechanism, not the coverage.

---

## Step 1 — Mode

- **Diff-aware (default)** — on a feature branch, or a story is named: scope to the pages/routes affected by that story/branch's changes. Cross-reference changed files (routes, views/components, styles) against what they render.
- **Full** — systematic exploration of the whole app. Use for first QA pass on a project, or when explicitly asked.
- **Quick** — homepage + top 5 navigation targets, console errors and broken links only. No detailed per-page checklist.

Ask which mode if it's not obvious from context (e.g., no branch diff and no story named).

---

## Step 2 — Orient

Navigate to the target (ask for the local dev URL if none is running or known). Map the navigation structure. Note whether it's server-rendered (page navigations) or a client-side-routed SPA (affects how "page" is defined for Step 3).

---

## Step 3 — Explore (per-page checklist)

For each page in scope:
- **Visual scan** — obvious layout issues from a screenshot, if the tool supports one.
- **Interactive elements** — do buttons, links, and controls actually do what they claim?
- **Forms** — fill and submit; test empty, invalid, and edge-case input.
- **Navigation** — every path in and out of the page.
- **States** — empty state, loading state, error state, overflow (long text, large lists).
- **Console errors** — any new JS errors after interaction, not just on load.
- **Responsiveness** — check a mobile viewport if the project has responsive/mobile scope.

Quick mode: only loads-without-error + console errors + visibly broken links, skip the rest.

---

## Step 4 — Document

Document each issue **immediately when found**, not batched at the end. For interactive bugs: screenshot (or description, in guided-manual mode) before the action, the action taken, the result, and repro steps referencing them.

Tag severity (reuses `eng-flow-code-review`'s vocabulary for consistency):
- **Critical** — broken core flow, data loss, crash.
- **Required** — real bug, not core-breaking.
- **Nit** — cosmetic, doesn't block.

---

## Step 5 — Save and hand off

Write `eng-flow/backlog/stories/<story-slug>/qa-report.md`:

```markdown
# QA Report: [scope — story/branch/app name]
Mode: [diff-aware | full | quick]

## Issues

### [Severity] [Title]
**Page:** [route/page]
**Repro:** [steps]
**Evidence:** [screenshot path or description]

## Summary
[pages tested, issues found by severity]
```

**Do not fix issues in this skill.** Report the list, then tell the user: "Route these through `eng-flow-implement` as tasks against the story" (or, if a Critical issue is a straightforward code-review-catchable regression, note that too) — QA's job ends at documentation, same separation as every other stage in this track.
