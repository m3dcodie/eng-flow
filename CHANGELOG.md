# Changelog

All notable changes to eng-flow are documented here. Format loosely follows [Keep a Changelog](https://keepachangelog.com/).

## [Unreleased]

## [0.5.0] — 2026-10-29

### Added

- Four plugin agents under `agents/`, picked up by Claude Code without a `plugin.json` entry and listed as `eng-flow:<name>`. `eng-flow-adversarial-reviewer` and `eng-flow-security-reviewer` (code review Step 3.5) and `eng-flow-doc-reviewer` (eng review Step 3, domain model Step 7) get Read, Grep and Glob, plus Bash for git on the two code reviewers; none has Write or Edit. `eng-flow-repro-test-writer` (implement Step 2, bug-fix tasks) can write a new test file but has no Edit. Before, all four passes ran on `general-purpose`, which could edit the code or docs it was meant to review blind. The review prompts moved from the skills into the agent files unchanged; the skills now pass only file paths, diff scope and the security-policy path. Each file sets `model: inherit`; `--review-model` still overrides on the domain-model call. If an agent isn't available, the step falls back to `general-purpose` with the agent file's body as the prompt and logs the fallback. Spec: `eng-flow/specs/2026-10-29-plugin-reviewer-agents/spec.md`.
- `eng-flow-qa` Step 3 runs browser exploration in foreground subagents (`eng-flow-qa-explorer`, one page or a small batch per call, one after another), so Playwright snapshots and screenshots stay out of the main context. Only a structured findings list comes back; `qa-report.md` keeps its shape. Screenshots go to `eng-flow/backlog/stories/<slug>/qa-evidence/`. The explorer has no Write, Edit, Bash or Agent, and returns logins, mutating JS and other user decisions to the skill instead of acting on them. Guided-manual mode stays inline.
- `eng-flow-mvp` Step 4 runs each checklist item in its own foreground subagent (`eng-flow-mvp-item-builder`): implement, test, verify, commit. The main thread still checks hard stops before dispatch, verifies exactly one commit landed, pushes (if enabled), checks the item off and logs decisions. A subagent that hits a hard stop leaves its changes uncommitted and returns it.
- `--inline` on either skill restores the old in-conversation behaviour. Both fall back to `general-purpose` with the agent file's body if the plugin agents aren't loaded.
- Analytics counts subagent tokens. Checkpoint and finish match each `Agent` call in a step to its transcript under `<session>/subagents/` and add its usage to that step's row, with `subagent_calls`, `subagent_usage` and `subagent_tokens: "included"`. If a subagent's usage can't be found, the row carries `subagent_tokens: "untracked"` and `subagent_untracked_calls`, and the report flags the undercount next to the totals. The extraction moved into a shared `lib/bin/eng-flow-usage`. Spec: `eng-flow/specs/2026-10-29-subagent-context-offload/spec.md`.

### Fixed

- Analytics overcounted tokens. Claude Code writes one transcript line per content block of an assistant turn, each repeating that turn's usage, and the old extraction summed every line. Usage is now taken once per message. Rows written before this change are inflated, so don't compare them directly with newer ones.

## [0.4.0] — 2026-10-28

### Added

- A `SessionStart` hook (`hooks/hooks.json`, matcher `compact|resume`), the plugin's first. After a context compaction or a resumed session, the new read-only `eng-flow-session-context` script lists unfinished drafts (with the step each stopped at), started stories with their next task, and open MVP checklists. When a draft is listed, it tells Claude to re-read it and `lib/draft-resume.md` before the next step. This closes the gap drafts leave: a mid-stage compaction doesn't re-run Step 0. The hook also runs the input-drift check itself. A draft whose inputs (e.g. `spec.md`) changed since it started is flagged with the files named, and the model is told to offer *review* or *discard* before saving more. Without this, a live test showed a resumed session took "continue" as approval and wrote `architecture.md` over a changed spec. The hook is silent outside eng-flow projects, capped at 20 lines and never blocks the session. `ENG_FLOW_NO_HOOKS=1` turns it off. No `PreCompact` / `Stop` / `SessionEnd` hooks: they can't save what's only in the model's context. `eng-flow-draft` gains shared `summaries()` and `drift()` functions; `list` and `check` output are unchanged. Design: `docs/improvements/session-start-hook.md`.

## [0.3.0] — 2026-10-27

### Added

- `eng-flow-spec` (Stage 1) now saves as it goes and can resume. Each numbered step ends by writing its agreed answers to `eng-flow/.drafts/spec--<dated-slug>.json`, so a crash or closed terminal loses at most the step in progress instead of the whole interview. Step 0 lists unfinished drafts and resumes one at the next step, with review and discard on request; the draft is deleted only after `spec.md` is saved. Backed by a new shared helper, `eng-flow-draft` (`save`, `list`, `truncate`, `clear`), under `.claude/skills/lib/bin/`, which never blocks the skill on error. The spec folder's dated slug is now computed at Step 0 rather than Step 7. Design: `.claude/skills/lib/draft-resume.md`. Drafts are gitignored: the helper writes `eng-flow/.drafts/.gitignore` (`*`) on first use.
- Draft and resume rolled out to `eng-flow-domain-model` (Stage 2), `eng-flow-architecture` (3), `eng-flow-eng-review` (3.5), `eng-flow-ui-design` (3.6) and `eng-flow-epics-stories-tasks` (4, Steps 0–6; the per-story task breakdown has no draft). Drafts are keyed on the spec folder's slug, so picking a spec with a draft resumes it. Per-item loops (domain-model Steps 1 and 8, eng-review Step 4, ui-design Step 3) save after each item as `in_progress`, so a crash loses only the item in progress. Expensive output (subagent findings) is saved, so a resume doesn't redo it. `eng-flow-draft` gains an `in_progress` status that doesn't count as finished, plus `inputs` / `check` subcommands that hash a stage's input files at Step 0 and report changes on resume. The shared Resume / Review / Discard rules moved from the spec skill's folder to `.claude/skills/lib/draft-resume.md`.

## [0.2.3] — 2026-08-31

### Fixed

- The 0.2.1 `CLAUDE_PLUGIN_ROOT` fix used `${CLAUDE_PLUGIN_ROOT}/skills/lib/bin/<script>`, assuming skills live at the plugin root's default `skills/` directory. eng-flow actually declares a non-default location — `plugin.json`'s `"skills": "./.claude/skills"` — so the real path is `${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/<script>`. `CLAUDE_PLUGIN_ROOT` always resolves to the plugin's repo root regardless of where its `skills` field points, so the `.claude/` segment was missing. Confirmed via a real install: with the 0.2.1 path the script didn't exist on disk; with `.claude/` restored, `eng-flow-analytics-report` runs correctly against a consuming project's `eng-flow/analytics.jsonl`.

## [0.2.2] — 2026-08-31

### Fixed

- `eng-flow-ship` Step 7 opened its PR with plain `gh pr create`, which falls back to GitHub's repo-level default-branch setting. That can silently differ from the `<base>` branch the rest of the skill (Steps 1–3) has actually been operating against — e.g. a project whose real integration base is a staging/develop branch rather than whatever GitHub treats as default. Step 7 now explicitly passes `--base <base>`, targeting the same branch used throughout the skill. Generalized from a project-specific workaround (a consuming project's local skill copy had patched this in independently) into the shared skill so every project benefits, not just that one.

## [0.2.1] — 2026-08-31

### Fixed

- Shared scripts under `.claude/skills/lib/bin/` (`eng-flow-analytics-checkpoint`, `-finish`, `-report`, `eng-flow-decision-log`, `eng-flow-findings-log`) were invoked from every skill via a `.claude/skills/lib/bin/<script>` path relative to the consuming repo's cwd. That only worked because the skill files were copied directly into a project's own `.claude/skills/` — installed as a real plugin, skills execute from the plugin's own install location, so the relative path resolved nowhere. All invocations across every `SKILL.md`, `CLAUDE.md`, and `PROCESS.md` now use `"${CLAUDE_PLUGIN_ROOT}/skills/lib/bin/<script>"`, matching the convention used by other Claude Code plugins. The scripts' own data paths (`eng-flow/analytics.jsonl` etc.) were already resolved relative to cwd and needed no change — they still write into the consuming project, not the plugin cache.

## [0.2.0] — 2026-08-18

### Added

- Optional project-level security policy (`eng-flow/security-policy.md`) — `eng-flow-architecture` (Stage 3) offers to establish standing security rules (credential handling, least-privilege access, write confirmation, etc.) when none exist yet; `eng-flow-eng-review` (Stage 3.5) checks `architecture.md` against each stated rule, and `eng-flow-code-review` (Stage 6) checks the actual diff against each rule in its Security axis and now dispatches the security-specialist subagent whenever the policy file exists, not only when the diff looks security-shaped by heuristic. See `docs/DECISIONS.md`.

## [0.1.0] — 2026-08-18

Initial tracked release.

### Added

- Two-phase process (`PROCESS.md`): MVP mode and a ten-stage production track (spec → domain model → architecture → eng review → UI design → epics/stories/tasks → implementation → code review → QA → ship → retro → analytics), each implemented as an `eng-flow-*` Claude Code skill.
- `eng-flow-browse` — general-purpose visual-verification skill wrapping a Playwright MCP server bundled directly with the plugin (`@playwright/mcp@0.0.79`, declared in `plugin.json`'s `mcpServers`) — no manual MCP setup required, usable standalone by any skill or ad hoc.
- `eng-flow-qa` (Stage 7) now drives the browser through `eng-flow-browse` by default, falling back to a guided-manual checklist only if MCP is unavailable in a given session.
- Distributed as a Claude Code plugin (`.claude-plugin/plugin.json` + `marketplace.json`) rather than a copy/symlink template — see `docs/DECISIONS.md` for why.
