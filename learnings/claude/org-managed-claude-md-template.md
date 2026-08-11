# Organization-managed CLAUDE.md — template + why there's no real example here

The third level in the hierarchy (`claude-md-guide.md` §1), separate from both "project" and
"user/global". **Confirmed absent on this machine**: no `/etc/claude-code/CLAUDE.md`, no
`managed-settings.json` anywhere. That's expected, not a misconfiguration — this level exists for
enterprise-managed devices, pushed by an IT/platform admin (typically via MDM), not written by the
individual developer. A personal WSL dev machine has no admin pushing policy to it, so there is
nothing to show as a real worked example, unlike `color-app-claude-md-example.md` or
`global-claude-md-example.md`. What follows below is illustrative only — clearly marked as
hypothetical, not evidence from this environment.

## Where it lives

| Platform | Path |
|---|---|
| Linux/WSL | `/etc/claude-code/CLAUDE.md` |
| macOS | `/Library/Application Support/ClaudeCode/CLAUDE.md` |
| Windows | `C:\Program Files\ClaudeCode\CLAUDE.md` |
| Alternative | `claudeMd` key inside managed settings JSON |

## What makes this level different from the other two

- **Loads first** (least specific) in the concatenation order — everything else (user, project,
  local, nested) loads after it.
- **Cannot be excluded.** `claudeMdExcludes` in project settings can suppress other levels' content
  from being pulled in on request, but the managed policy file is exempt from that — an individual
  developer or project can't opt out of it.
- **Who writes it**: an org's platform/security team, not the developer using Claude Code day to day.
  Contrast with user-level (you write it, for yourself) and project-level (the team writes it,
  scoped to one repo).
- **Typical content**: security/compliance mandates, restricted tool/command lists, required data
  handling rules, company-wide coding standards that apply regardless of team or repo — things that
  need to hold even if a individual project's CLAUDE.md tries to say otherwise.

## Illustrative example (hypothetical — not a real file, not from this machine)

```markdown
## Security & Compliance (mandatory, org-wide)

- Never write API keys, tokens, or credentials into source files — use the org secrets manager.
- Do not call external/third-party APIs from generated code without a data-handling review; internal
  APIs only unless explicitly approved per project.
- All generated code touching customer PII must go through the standard PII-review checklist before
  merge — link the checklist, don't restate it here.

## Required Tooling

- Use the org's approved dependency-scanning step before adding a new package.
```

## Why this matters even though it's not populated here

Since managed-policy content loads *first* and project/user content loads *after* it, in a session on
an enterprise-managed machine an org rule would be present in context before either of the other two
levels, and (per §2's recency-implies-priority reasoning) a *conflicting* project or user rule would
technically be read more recently — which is exactly why `claudeMdExcludes` being blocked for this
level matters: recency-based priority alone would otherwise let a project override an org security
rule, so the platform enforces the exemption at the loading level instead of relying on the model to
resolve the conflict correctly every time. On this machine none of that is in play — there's simply
nothing at this level to interact with the other two.
