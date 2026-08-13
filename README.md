# eng-flow

A lightweight, tool-agnostic engineering process — from rapid MVP to production-grade — for one engineer or a shared team.

It's a playbook (`PROCESS.md`) plus a set of live Claude Code skills that implement it, distributed as a Claude Code plugin — install once, use in any project.

## The model

Two phases, one gate:

- **MVP mode** — move fast, autonomously, minimal ceremony. A spec if you have one; a one-page brief if you don't. No mandated testing/security process.
- **Production mode** — entered deliberately, as a judgment call (see "Graduation gate" in `PROCESS.md`), not a checklist artifact. Ten staged, non-overlapping stages: pure requirements → domain model → architecture → engineering review → UI design → epics/stories/tasks → implementation → code review → QA → ship → retro → analytics. Each stage only asks what it owns — requirements never touches tech stack, architecture never re-litigates requirements.

See `PROCESS.md` for the full routing logic and what each stage does.

## Install

```
/plugin marketplace add m3dcodie/eng-flow
/plugin install eng-flow@m3dcodie-eng-flow
```

> **SSH errors?** The marketplace clones over SSH by default. If you don't have SSH keys set up
> on GitHub, use the full HTTPS URL instead: `/plugin marketplace add https://github.com/m3dcodie/eng-flow.git`.

**Local / development** (no push required, edits take effect immediately):

```bash
git clone https://github.com/m3dcodie/eng-flow.git
claude --plugin-dir /path/to/eng-flow
```

Updating: `/plugin update` picks up new skill versions in every project that installed it — no
per-project copy or re-scaffold step.

## Why this exists

This isn't a reinvention of existing SDLC tooling — see `docs/DECISIONS.md` for what was deliberately reused from prior art (gstack, agent-skills, a personal project's own battle-tested skills) versus what's genuinely new here and why.

## Layout

```
PROCESS.md                       # the playbook: routing + phase gate
.claude-plugin/
  plugin.json                    # plugin manifest — skills field points at .claude/skills/
  marketplace.json               # marketplace listing, installed via /plugin marketplace add
.claude/skills/
  eng-flow-spec/                 # Stage 1 — requirements
  eng-flow-domain-model/         # Stage 2 — domain model
  eng-flow-architecture/         # Stage 3 — technical design
  eng-flow-eng-review/           # Stage 3.5 — architecture review
  eng-flow-ui-design/            # Stage 3.6 — UI/UX design
  eng-flow-epics-stories-tasks/  # Stage 4 — backlog
  eng-flow-implement/            # Stage 5 — implementation
  eng-flow-code-review/          # Stage 6 — code review
  eng-flow-qa/                   # Stage 7 — QA
  eng-flow-ship/                 # Stage 8 — ship
  eng-flow-retro/                # Stage 9 — retro
  eng-flow-analytics/            # Stage 10 — analytics rollup
docs/
  DECISIONS.md                   # design rationale, comparisons, external validation
```
