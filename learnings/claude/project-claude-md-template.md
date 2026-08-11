# General project-level CLAUDE.md template

Not a real CLAUDE.md — a reusable skeleton. Copy this into a new project's `./CLAUDE.md` (never name a
file exactly `CLAUDE.md` inside this `learnings/` folder itself — Claude Code loads nested `CLAUDE.md`
files on demand, so a template literally named that would get read as real instructions the moment
Claude works in this directory).

Target: under ~200 lines once filled in. Every section below should hold *persistent* facts — things
true regardless of what task is being worked on right now. If it's only true for this week's task, it
belongs in a backlog/task file, not here.

---

```markdown
# <Project Name> — project conventions

## Stack & architecture

<!-- One line per fact. Link to ADRs/architecture docs instead of re-explaining the reasoning here —
     duplication drifts out of sync with the source; a link doesn't. -->
- <Language/framework/runtime + version, and the one-line reason if it's non-default>
- <Key architectural constraint that isn't obvious from the code> (link to ADR if one exists)
- <Persistence/data layer choice + why>
- <Anything environment-pinned that would silently break otherwise, e.g. a Node/toolchain version>

## Code layout

<!-- Where things live and why — oriented at someone who has never opened this repo. -->
- `<dir>/` — <what belongs here, one line>
- `<dir>/` — <what belongs here, one line>

## Testing & conventions

<!-- Directives, not trivia. "Respect X" acts as an instruction; "X is true" is easy to skim past. -->
- <Test framework/command, and what "done" looks like for a change>
- <Any strict compiler/linter flags that must not be relaxed to silence an error>
- <Anything explicitly NOT configured yet, stated as a known gap, not an invitation to add it unasked>

## Workflow / process (optional — only if this repo runs a defined process)

<!-- If this project follows a documented process (like eng-flow's own PROCESS.md), point to it here
     and describe only the behavior for work that falls OUTSIDE that process's normal flow. Don't
     duplicate the process doc itself. -->
- <Pointer to the process doc>
- <What to do when a judgment call happens outside that process's normal steps>

## Deterministic guardrails — pointer only, not enforcement

<!-- CLAUDE.md cannot enforce a "must never happen" or "must always happen" rule — it's prose the
     model reads and tries to follow, not a mechanically-guaranteed check. If a rule genuinely needs
     to be deterministic (block a dangerous command, always run a check after writes), it belongs in
     the `hooks` key of `.claude/settings.json` (PreToolUse/PostToolUse), not here. State that pointer
     explicitly so nobody mistakes a CLAUDE.md line for an enforced guarantee. -->
- <"X must never happen" style rules live in hooks, see `.claude/settings.json`'s `hooks` key>
```

---

## Notes on filling this in

- Don't invent a convention that hasn't actually emerged in the code yet (e.g. naming patterns) —
  document what's real, not what you'd prefer were real.
- Every fact here should change a default choice the model would otherwise guess at. If a line
  wouldn't change any behavior, cut it.
- If a fact would also be true in a sibling project, it might belong at the user/global level instead
  — check before duplicating it project by project.
