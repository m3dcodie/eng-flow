# General user-level CLAUDE.md template

Not a real CLAUDE.md — a reusable skeleton for `~/.claude/CLAUDE.md`. Same naming caution as the
project template: don't name a copy of this file exactly `CLAUDE.md` inside `learnings/`, or Claude
Code's nested-load mechanic will read it as real instructions.

**The one thing that makes this level different from project-level**: everything here should be true
of *you*, regardless of which repo you're in. If a line would only make sense while working on one
specific project, it belongs in that project's `CLAUDE.md`, not here. Test each line with: "would this
still be true if I opened a completely unrelated repo tomorrow?" If no, move it.

Target: short. This file loads on *every single session, in every project* — it's the one CLAUDE.md
guaranteed to compete for attention with the largest number of unrelated tasks over time, so keep it
to durable preferences, not situational advice.

---

```markdown
## About Me

<!-- Grounds the model in who it's talking to — role, environment, how you want to be talked to. -->
- **Role**: <your role / seniority — changes how much explanation vs. terseness is useful>
- **Primary OS**: <OS — affects which shell commands/paths are safe to assume>
- **Primary IDE**: <IDE — relevant if the model ever suggests IDE-specific steps>
- **Preferred Communication**: <tone/length preference, stated as a directive, e.g. "avoid fluff">

## System-Wide Behaviors

<!-- Defaults that apply before any project-level file overrides them. Keep these about YOUR working
     style, not about any one codebase's conventions. -->
- **Response Style**: <when to plan-first vs. just act — the threshold, not a blanket rule>
- **Explanation Layer**: <comment/explanation density you want in generated code>
- **Error Handling**: <what to do automatically before asking you for input on a failure>
- **Code Generation**: <where project-specific style should take precedence over these defaults>

## Workflow & Safety Guardrails

<!-- The irreversible/high-blast-radius stuff. Note from experience: CLAUDE.md can only ask nicely
     here — it is NOT enforcement (see claude-md-guide.md §7). If a rule below is load-bearing enough
     that it must never be skipped, it needs a hook too, not just a line here. -->
- **Git Automation**: <what can happen without asking vs. what always needs explicit confirmation>
- **Destructive Commands**: <warn-before-running list, e.g. commands that delete/overwrite>
- **Dependencies**: <version-pinning policy — exact vs. loose ranges>

## Common Fallback Commands

<!-- Only used when a project doesn't define its own — project-level always wins if present. -->
- **Testing**: <default test command per language, when none is discovered>
- **Linting/Formatting**: <default formatter, when no project config exists>
```

---

## Notes on filling this in

- This file is the *widest-reaching* CLAUDE.md you own — every project you touch loads it. That's a
  reason to keep it short and durable, not a reason to pile on more rules; the more that's here, the
  more competes with project-specific context on every session.
- Don't duplicate something a project-level CLAUDE.md already states more specifically — project wins
  on conflict anyway (load-order recency, see `claude-md-guide.md` §2), so a duplicate here just adds
  tokens without changing behavior.
- Anything phrased as "always/never" here should be checked against the deterministic-actions ladder
  (`claude-md-guide.md` §7) — if it truly must never be violated, it belongs in a hook as well.
