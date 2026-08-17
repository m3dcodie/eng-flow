# Changelog

All notable changes to eng-flow are documented here. Format loosely follows [Keep a Changelog](https://keepachangelog.com/).

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
