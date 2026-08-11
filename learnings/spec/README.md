# spec.md learnings

From a working session going deep on the "Practice writing a `spec.md` per feature" item in
`learn/resources/mastering-ai-assisted-development-checklist.md`, verified against this repo's own
`eng-flow-spec` skill and a real worked example in `color-app`.

- **`spec-guide.md`** — the deep dive: two different artifacts share the name "spec.md" (a
  lightweight per-feature vibe-coding spec vs. eng-flow's heavier Stage 1 spec), the format for
  each, why spec-first work reduces downstream rework (with the course's "2-3x" claim checked and
  found plausible-but-unverified), exactly what the AI model does with it at each stage (routing →
  interrogation → three-stage downstream consumption for eng-flow's format; single-shot generation
  for the vibe-coding format), the effects of doing it well vs. skipping it (grounded in
  color-app's real spec, including a live example of a flagged mid-stream amendment), and an
  evaluation rubric.
- **`vibe-coding-spec-template.md`** — a blank template for the lightweight, per-feature format.
  No real worked example of this format exists in this repo yet (everything here has gone through
  the heavier eng-flow pipeline instead) — see the real eng-flow-format example at
  `color-app/eng-flow/specs/2026-08-09-mvp-coloring-app-color-once-export-everywhere/spec.md`.

## Still open

- No personal spec library yet (specs that produced good output, curated across projects) — only
  one project has been through the full Stage 1 pipeline so far.
- No template repo with a pre-baked `.claude/` setup for new projects.
- The vibe-coding format template hasn't been used standalone in this repo — untested here.
