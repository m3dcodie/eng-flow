> **Spec library entry** — copied verbatim from
> `color-app/eng-flow/specs/2026-08-11-convert-artwork-into-a-template/spec.md` (2026-08-11).
> Kept as the reference shape for a *narrow internal dev-tooling spec*: single journey, the
> developer is the end user, success criteria are structural/visual fidelity rather than a
> product metric. Shows the same spec discipline (specific on constraints, open on
> implementation) scales down cleanly to small utility work. Produced a full downstream chain:
> `domain-model.md`, `architecture.md`, 1 ADR. See `README.md` in this folder for why this one
> was kept.

---

# Spec: Convert Artwork into a Template

## Decision
Approved: build an offline dev tool that converts external SVG artwork into this app's Template fixture format (regions + palette matching `src/types.ts`), so new templates can be added to the library without hand-authoring every region's SVG path data by hand.

- **Approved by:** the user — solo project, sole stakeholder.
- **Written source:** this session's decision capture, corroborated by the existing stub docs (`eng-flow/backlog/epics/template-content-pipeline.md`, `eng-flow/backlog/stories/convert-artwork-into-a-template.md`) and the two prior spikes on `spike/svg-template-conversion` (commits `7396bf7`, `ee89521`).
- **Business outcome:** the template library can grow beyond the current one-off hand-built placeholders (dolphin, whale) without per-template hand-authoring cost.
- **Success measured by:** visual/structural fidelity — a converted template's regions visually match the source artwork's intended paint-by-number areas, and the app renders/fills them correctly when the output is loaded in `ColoringScreen` and a few regions are colored.
- **Constraints:** stays a client-only, offline dev tool (in the spirit of ADR-0001 — no backend/service, even though this is dev tooling rather than shipped runtime code); output must exactly match the existing `Template` / `Region` / `Palette` shape in `src/types.ts` — no schema drift.

## Problem & Audience
- **End user of this capability:** the developer/content author (you) — not app end users. This is not an in-app, user-facing upload feature.
- **Problem:** every new template today means hand-authoring SVG path data region-by-region, which is slow and doesn't scale the template library.
- **Status quo:** two spikes proved the concept works (source-SVG conversion via `svgelements`, and from-scratch hand-authoring as a fallback) but neither is a reusable, real tool — spike 1's script is hardcoded to one source file's exact colors/font-sizes and a source that was never license-cleared for shipped content.

## User Journeys
- As the developer building out the template library, I want to run an offline conversion tool against a source SVG file, so that I get a `Template` + `Palette` fixture pair I can drop into `src/fixtures/` and load in the app for coloring — without hand-authoring each region's path data.

## Functional Requirements
- Takes a source SVG file and produces a valid `Template` fixture (`id`, `categoryIds`, `regions: Region[]`, `paletteId`, `viewBox`) matching `src/types.ts`.
- Each output region is its own closed, filled path with a stable `id` — not a shared outline / line art, matching both spikes' approach.
- Produces a matching `Palette` (curated color list) alongside the Template.
- Surfaces source-licensing status (cleared for shipped content vs. spike/local-only) rather than silently treating any converted output as shippable — spike 1's source was never cleared, and that risk needs to be structural, not a one-off note.
- Output is verified against the running app — regions render, are individually clickable/colorable, and don't collide with `ColoringScreen`'s reserved bottom-overlay space — before being treated as usable.
- Hand-authoring (spike 2's approach) remains a documented fallback path for licensing-constrained or non-existent sources, not something the tool forecloses.

## Non-Functional Requirements
- Client-only / no backend: the conversion tool is a local script/CLI, run offline, not a shipped service.
- Output format stability: converted fixtures must exactly match the existing `src/fixtures/` `Template`/`Region`/`Palette` TypeScript shape — no schema drift for the coloring mechanic to consume.

## Domains Touched
- Template authoring/conversion (new)
- Template fixtures (existing)
- Coloring mechanic (structural compatibility target, not modified at runtime)
- Persistence/IndexedDB (only if converted templates need to be registered into the app's existing template list)

## Scope
- **MVP cut:** convert a single, well-formed source SVG into a valid Template + Palette fixture via a CLI/script-driven tool (not an in-app UI); output loads and is colorable in `ColoringScreen`.
- **Out of scope (this cut):**
  - In-app upload/convert UI
  - Robust handling of malformed/complex SVGs (nested groups, gradients, clipping)
  - Batch conversion of multiple files at once
  - Any end-user-facing (non-developer) conversion feature

## Success Criteria
- A source SVG can be converted into a `Template` + `Palette` fixture pair matching `src/types.ts`.
- The converted template loads in `ColoringScreen` and each region is individually colorable.
- The source artwork's licensing status is explicitly recorded on the output, not assumed.

## Open Questions
- Exact supported input format(s) — this spec scopes to SVG; spike 1 actually started from EPS via `svgelements`. Whether EPS (or other formats) stay in/out is deferred to Stage 3 (architecture).
- The region-extraction/number-color-matching heuristic (spike 1's approach was hardcoded to one source file's specific fill colors and font sizes) needs a general, non-hardcoded design — deferred to Stage 3.
