# Changelog

All notable changes to eng-flow are documented here. Format loosely follows [Keep a Changelog](https://keepachangelog.com/).

## [Unreleased]

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
