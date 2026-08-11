# spec.md — vibe-coding format (per-feature, one-shot)

Blank template for the lightweight format described in `spec-guide.md` §2a — a single feature or
prototype, handed to the agent in one prompt. Not the eng-flow Stage 1 format (see
`color-app/eng-flow/specs/2026-08-09-mvp-coloring-app-color-once-export-everywhere/spec.md` for
that one) — use this only when there's no domain-model/architecture stage coming after it.

Fill in constraints and targets. Leave implementation out on purpose — that's the model's job.

```markdown
# Spec: [Feature/prototype name]

## Technology constraints
- Must use: [stack/library/platform, if fixed]
- Must avoid: [anything explicitly ruled out]
- Integrates with: [existing system this plugs into, if any]

## Visual requirements
- [What it should look like — style references, layout expectations, brand constraints]

## Interaction model
- [What the user does, step by step]
- [What happens in response to each action]
- [Any states: empty, loading, error, success]

## Performance targets
- [What "fast enough" means, concretely — e.g. "loads under 2s on a mid-tier phone," not "fast"]
```

## Self-check before handing this to the model

- [ ] Every constraint is something the model *must* honor, not a preference
- [ ] Nothing here describes component structure, file layout, or algorithm choice — that's
      implementation, and belongs to the model, not the spec
- [ ] Performance targets are numbers or concrete conditions, not adjectives
- [ ] If this spec is good enough to reuse the pattern elsewhere, save a copy — see `spec-guide.md`
      §5's "spec library" note (still unchecked in this repo as of 2026-08-11)
