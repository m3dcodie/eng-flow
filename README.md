# eng-flow

A lightweight, tool-agnostic engineering process — from rapid MVP to production-grade — for one engineer or a shared team.

It is not a framework, plugin, or dependency. It's a playbook (`PROCESS.md`) plus a set of live Claude Code skills that implement it. No installation beyond having this repo's `.claude/skills/` on the path — copy this repo (or just the pieces you need) into any project.

## The model

Two phases, one gate:

- **MVP mode** — move fast, autonomously, minimal ceremony. A spec if you have one; a one-page brief if you don't. No mandated testing/security process.
- **Production mode** — entered deliberately, as a judgment call (see "Graduation gate" in `PROCESS.md`), not a checklist artifact. Five staged, non-overlapping stages: pure requirements → domain model → architecture → engineering review → epics/stories/tasks → implementation. Each stage only asks what it owns — requirements never touches tech stack, architecture never re-litigates requirements.

See `PROCESS.md` for the full routing logic and what each stage does.

## Why this exists

This isn't a reinvention of existing SDLC tooling — see `docs/DECISIONS.md` for what was deliberately reused from prior art (gstack, agent-skills, a personal project's own battle-tested skills) versus what's genuinely new here and why.

## Layout

```
PROCESS.md                       # the playbook: routing + phase gate
.claude/skills/
  eng-flow-spec/                 # Stage 1 — requirements
  eng-flow-domain-model/         # Stage 2 — domain model
  eng-flow-architecture/         # Stage 3 — technical design
  eng-flow-eng-review/           # Stage 3.5 — architecture review
  eng-flow-epics-stories-tasks/  # Stage 4 — backlog
  eng-flow-implement/            # Stage 5 — implementation
docs/
  DECISIONS.md                   # design rationale, comparisons, external validation
```
