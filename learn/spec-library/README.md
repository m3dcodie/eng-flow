# Spec Library

Specs that produced good downstream output — saved here so the pattern can be reused on
future work instead of re-derived from scratch. See the checklist item in
`mastering-ai-assisted-development-checklist.md` ("Start a personal spec library").

An entry qualifies once it's actually been run through the pipeline (domain model +
architecture came out the other end), not just written.

## Entries

| Entry | Pattern | Source | What it produced |
|---|---|---|---|
| [`mvp-feature-spec--coloring-app.md`](mvp-feature-spec--coloring-app.md) | End-user MVP feature spec — new product, multiple journeys, tight constraints/open implementation | `color-app/eng-flow/specs/2026-08-09-mvp-coloring-app-color-once-export-everywhere/spec.md` | domain-model.md, architecture.md, 6 ADRs, wireframes, mockups |
| [`dev-tool-spec--artwork-to-template-converter.md`](dev-tool-spec--artwork-to-template-converter.md) | Internal dev-tooling spec — single journey, developer-as-user, strict output-format constraint | `color-app/eng-flow/specs/2026-08-11-convert-artwork-into-a-template/spec.md` | domain-model.md, architecture.md, 1 ADR |

## Why these two

Different shapes, both worth keeping as reusable templates:

- **MVP feature spec** — the "textbook" shape from `mastering-ai-assisted-development-summary.md`:
  specific on constraints (no per-user AI call, no accounts, static/client-side only), open on
  implementation. 8 user journeys, an explicit Non-Functional Requirements section, and an Open
  Questions section that turned out to matter (later stories were drafted directly off an
  amendment to this spec).
- **Dev-tool spec** — same discipline applied to a much narrower, single-journey internal tool.
  Good template for "small utility, one user (you), one success criterion" work — shows the
  pattern scales down without losing structure.

## Not included

`color-app/specs/requirement.md` — the office-hours-style research/ideation doc that fed into
the MVP spec above. Not a spec itself (no journeys, no functional requirements, no domain
boundary) — it's the hypothesis-validation step that precedes writing one. Worth knowing it
exists as the "how do I get from idea to spec-ready" reference, but it belongs in a different
part of the library if one gets built for that stage.
