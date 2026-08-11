# CLAUDE.md learnings

From a working session on eng-flow/color-app understanding how Claude Code's `CLAUDE.md` memory
system actually works, going from first principles rather than assumption.

- **`claude-md-guide.md`** — the findings: full hierarchy (managed/user/project/local/nested/rules),
  load order and precedence, `@import` behavior, size/lazy-loading rules, an evaluation rubric, and
  where deterministic actions actually belong (hooks, not CLAUDE.md).
- **`project-claude-md-template.md`** — a general, reusable project-level CLAUDE.md skeleton.
- **`color-app-claude-md-example.md`** — that template filled in for a real project (color-app),
  including what the file looked like *before* this review (a copy of eng-flow's own CLAUDE.md with
  nothing project-specific in it) for contrast.
- **`user-claude-md-template.md`** — a general, reusable user-level CLAUDE.md skeleton. Note: "user"
  and "global" name the same file (`~/.claude/CLAUDE.md`) — there's one level here, not two.
- **`global-claude-md-example.md`** — that template filled in for real: a snapshot of the actual
  `~/.claude/CLAUDE.md`, annotated against the rubric.
- **`org-managed-claude-md-template.md`** — the third level (org/enterprise-managed, e.g.
  `/etc/claude-code/CLAUDE.md`), confirmed absent on this machine since it's a personal, not
  enterprise-managed, setup. Template + an explicitly-hypothetical illustration only — no real worked
  example exists here, unlike the other two levels.
- **`deterministic-guardrails.md`** — depth on `claude-md-guide.md` §7: the full hook event list,
  exactly how `PreToolUse` hard-blocks a tool call vs. `PostToolUse`'s after-the-fact-only guarantee,
  a longer candidate list (destructive git/SQL ops, loose dependency versions, secret-shaped writes,
  typecheck/format/test-after-write), and the loading-vs-compliance distinction for CLAUDE.md itself.
  Also corrects an error in the other files: hooks live under the `hooks` key in
  `.claude/settings.json`, not a separate `.claude/hooks.json` file.

None of these five files are named `CLAUDE.md` on purpose — Claude Code loads nested `CLAUDE.md`
files on demand when working in that directory, so a template literally named that would get read as
real instructions rather than reference material.

## Still open

- The concrete gap `claude-md-guide.md` §7 flags: no `hooks` key configured in either repo's
  `.claude/settings.json`, so "never push to main" (global CLAUDE.md) is prose-enforced, not
  hook-enforced. Not yet built.
