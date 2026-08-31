# Changelog

All notable changes to eng-flow are documented here. Format loosely follows [Keep a Changelog](https://keepachangelog.com/).

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
