# eng-flow

A lightweight, tool-agnostic engineering process — from rapid MVP to production-grade — for one engineer or a shared team.

It is not a framework, plugin, or dependency. It's a playbook (`PROCESS.md`) plus fill-in templates. No installation, no external tool required to use it — copy this repo (or just the pieces you need) into any project.

## The model

Two phases, one gate:

- **MVP mode** — move fast, autonomously, minimal ceremony. A spec if you have one; a one-page brief if you don't. No mandated testing/security process.
- **Production mode** — entered deliberately, once the [graduation checklist](templates/graduation-checklist.md) says so. Four staged, non-overlapping documents: pure requirements → domain model → architecture → epics/stories/tasks. Each stage only asks what it owns — requirements never touches tech stack, architecture never re-litigates requirements.

See `PROCESS.md` for the full routing logic and when to use which template.

## Why this exists

This isn't a reinvention of existing SDLC tooling — see `docs/DECISIONS.md` for what was deliberately reused from prior art (gstack, agent-skills, a personal project's own battle-tested skills) versus what's genuinely new here and why.

## Layout

```
PROCESS.md                       # the playbook: routing + phase gate
templates/
  mvp/                           # lightweight, for MVP-mode work
  graduation-checklist.md        # criteria to move from MVP to production mode
  production/                    # staged, for production-mode work
docs/
  DECISIONS.md                   # design rationale, comparisons, external validation
```
