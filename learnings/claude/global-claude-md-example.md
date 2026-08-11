# Worked example: the actual `~/.claude/CLAUDE.md`

The template in `user-claude-md-template.md`, filled in for real. This is a snapshot copy of the live
file as of this review — `~/.claude/CLAUDE.md` itself is the source of truth if it changes later.

What to notice against the template and the rubric (`claude-md-guide.md` §6):

- Every line passes the "would this still be true in an unrelated repo tomorrow?" test — role, OS,
  IDE, and communication style don't change per project, so they belong here and nowhere else.
- **"Always follow the coding style mentioned in project claude.md file or README.md file"** is the
  explicit hand-off point: it tells the model project-level wins on style, rather than silently
  letting recency-based precedence (`claude-md-guide.md` §2) do that job implicitly. Worth keeping —
  makes an already-true mechanic legible instead of relying on it being inferred correctly every time.
- **"Never push, merge, or force-push directly to `main`"** is the one line in this file that reads
  as a hard rule but is only prose-enforced today — see `claude-md-guide.md` §7. It's a good candidate
  for a `PreToolUse` hook precisely because it's already been identified as important enough to state
  explicitly here.
- Nothing project-specific leaks in (no stack, no repo paths) — this file stays valid regardless of
  which of this user's projects is currently open, which is the whole point of the level.

---

```markdown
## About Me

- **Role**: Senior Software Engineer
- **Primary OS**: Ubuntu-22.04 WSL
- **Primary IDE**: VS Code
- **Preferred Communication**: Technical, direct, and concise. Avoid conversational summaries or fluff.

## System-Wide Behaviors

- **Response Style**: For non-trivial work (multi-file changes, ambiguous scope, architecture/design
  decisions), propose a plan first — list what files/libraries need to be created or updated,
  formatted in markdown for readability. For trivial, well-defined changes (typos, one-line fixes, a
  single obvious edit), just make the change directly — don't force a plan step.
- **Explanation Layer**: Default to no comments. Only add one when it explains a non-obvious WHY — a
  hidden constraint, a workaround, a subtle invariant — not what the code does. Self-explanatory
  function/class names should cover the "what."
- **Error Handling**: When a command fails, evaluate log files before asking me for input.
- **Code Generation**: Always follow the coding style mentioned in project claude.md file or README.md
  file.

## Workflow & Safety Guardrails

- **Git Automation**: Committing, pushing to feature/story branches, and merging/pushing to `develop`
  (or other integration branches) can happen without asking — no need to stop for confirmation on
  routine git ops. The one hard exception: never push, merge, or force-push directly to `main` — that
  always requires explicit confirmation first.
- **Destructive Commands**: Warn me before executing commands that overwrite files or delete
  directories (e.g., `rm -rf`).
- **Dependencies**: Use precise package versions instead of loose tags (e.g., use `^1.2.3` or exact
  locking).

## Common Fallback Commands

- **Testing**: If no local test framework is discovered, fall back to language defaults (e.g.,
  `npm test`, `pytest`).
- **Linting/Formatting**: Default to `prettier --write .` or `black .` if no local configuration file
  exists.
```
