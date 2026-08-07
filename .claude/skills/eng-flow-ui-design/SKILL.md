---
name: eng-flow-ui-design
description: Production Stage 3.6 — takes a saved spec/domain-model/architecture and produces the UI design a UI-touching story needs before Stage 4 exists — a reusable project-level design system, per-journey wireframes, and (once gated) static HTML/CSS mockups. Skipped entirely if the spec's journeys have no UI surface. Between Stage 3.5 (Eng Review) and Stage 4 (Epics/Stories/Tasks).
allowed-tools:
  - Read
  - Grep
  - Glob
  - Write
  - Bash
  - AskUserQuestion
  - WebSearch
triggers:
  - ui design
  - design the ui
  - wireframes
  - mockups
  - design system
  - eng-flow ui design
---

# eng-flow UI design

Stage 3.6 of the production track, between Engineering Review and Epics/Stories/Tasks. Needs Stage 3's frontend framework choice to know what it's designing for, and must land before Stage 4 so UI-touching stories get concrete acceptance criteria instead of Stage 5 improvising layout mid-implementation. Skip entirely if the spec's journeys have no UI surface — same skip discipline `eng-flow-qa` applies to backend/CLI/library work.

## Analytics

At the start of every numbered step below (including Step 0), run `python3 .claude/skills/lib/bin/eng-flow-analytics-checkpoint eng-flow-ui-design "<step name>" "<dated-slug>"`. As the last action of Step 7, run `python3 .claude/skills/lib/bin/eng-flow-analytics-finish eng-flow-ui-design "<dated-slug>"`. See `eng-flow-spec`'s Analytics section for what this logs and why; rollup via `eng-flow-analytics` (Stage 10).

## Decision Ledger

Check `$ARGUMENTS` for a `--guide` token; if present, every decision point below gets an explicit `AskUserQuestion` instead of a silent default, and Step 7's report adds a "Decisions I made / decisions you made" summary. Log every decision point via `python3 .claude/skills/lib/bin/eng-flow-decision-log eng-flow-ui-design "<step>" <reason> <mode> <owner> "<description>" "<dated-slug>"`. See `eng-flow-spec`'s Decision Ledger section for the taxonomy and why. Rollup/analysis: `eng-flow-retro` Step 1 (Stage 9).

Step 1's aesthetic-direction decision is this skill's clearest **`knowledge_asymmetry`** case (stakeholder/customer/brand context the AI can't infer) — see that step for the specific mode logic.

## Step 0 — Find the inputs, check the trigger

Look for `eng-flow/specs/*/spec.md`, `domain-model.md`, and `architecture.md` in the same folder. If any are missing, tell the user which stage to run first. If more than one spec folder exists, ask which one this run is for.

Read the spec's **User Journeys**. If none of them involve a UI surface (pure API/backend/CLI/library work), **stop here** — tell the user this stage doesn't apply and that Stage 4 can proceed straight from `architecture.md`. Don't run the remaining steps for a spec with no UI journeys.

If domain-model.md has an optional Step 4 wireframe section, note it — Step 2 below starts from it rather than redrawing from scratch. Read architecture.md's tech-stack and frontend/backend-separation decisions (Steps 1 and 3 there) — this stage designs for that stack, it doesn't relitigate it.

---

## Step 1 — Design system (project-level, reuse if it exists)

Check for `eng-flow/design-system.md` at the project root — this is project-level, like `eng-flow/backlog/`, not per-spec: it should outlive any one spec and get reused, not rebuilt each run.

- **If it exists:** read it, state that this run is reusing it rather than relitigating it (same "boring by default" discipline `eng-flow-architecture` Step 1 applies to tech stack), and skip to Step 2. Only revisit it if the current spec's journeys genuinely can't be served by it — ask the user to confirm before changing a project-level artifact other specs may already depend on. Log it: `risk silent_decide ai_default "design system: reused existing, no relitigation"`.
- **If it doesn't exist:** before asking anything, check for material that's already real and surfaceable — a brand guide, prior mockups, an existing style guide elsewhere in the repo. If something surfaceable exists, present it as candidates rather than asking blind (`knowledge_asymmetry surface_existing user_confirmed`). If nothing exists to surface, ask the user for the aesthetic direction — typography, color, spacing, motion, tone (`knowledge_asymmetry open_question user_confirmed`). If they name a concrete reference (a product, a URL, "make it feel like X"), bounded research applies: one fetch per named reference, propose findings, don't bulk-expand without the user asking (same rules as Stage 1's competitor-analysis sub-step in `PROCESS.md`). Never open-ended "go find inspiration" discovery. Only synthesize novel candidates from scratch (`generate_options`) if the user explicitly asks for options — this is invention, not retrieval, and costs more to get wrong.

---

## Step 2 — Wireframes, per journey (markdown/ASCII)

For each UI-touching journey from Step 0: draft a lightweight markdown/ASCII wireframe — layout blocks, content, and flow only. No colors, no styling, no component-library references, same low-fidelity discipline `eng-flow-domain-model` Step 4 already uses for its optional wireframes. If that stage already produced one for this journey, start from it instead of redrawing.

Represent each as labeled boxes and flow, plain text:

```
[Header: logo, nav]
[Hero: headline, CTA button]
[Card grid: 3-col, each card = image + title + price]
[Footer: links]
```

---

## Step 3 — Design gate (score, iterate, per journey)

For each journey's wireframe, call `AskUserQuestion` one dimension at a time — never batched, same discipline `eng-flow-eng-review` Step 3 uses:

1. **Visual hierarchy** — is it obvious what matters most on the screen?
2. **Consistency** — does it match `design-system.md`'s conventions (or, if Step 1 just created it, is it internally consistent)?
3. **Accessibility** — reading order, contrast implied by the design system, obvious focus/interactive targets?
4. **Responsiveness** — does the layout have a stated behavior at narrow widths, or does it only exist at one size?

Score each 0–10, explain what a 10 looks like, revise the wireframe to close the gap, then move to the next dimension. **Stop and wait for the user's answer before raising the next dimension.** A journey is "gated" only once the user explicitly approves it — that's what unlocks Step 4 for that journey specifically; ungated journeys stay markdown-only and are reported as such in Step 6.

Log each dimension: `risk open_question <user_confirmed|user_revised> "journey '<name>' <dimension>: score <N>/10"`.

---

## Step 4 — Promote gated wireframes to HTML/CSS

Only for journeys that passed Step 3. Generate a static (non-interactive) HTML/CSS mockup per gated journey, styled using `design-system.md`'s tokens and targeting the frontend framework/stack `architecture.md` already chose — this is a visual mockup for sign-off and for Stage 5 to build against, not the actual implementation.

Skip this step for any journey still ungated — don't spend HTML-generation effort on a wireframe the user hasn't approved yet.

---

## Step 5 — Draft and confirm

Show a summary: the design system (new or reused), each journey with its gate status (gated + mockup path, or still markdown-only and why), and ask: "Does this match what you'd actually build, or anything to change?"

---

## Step 6 — Save

- `eng-flow/design-system.md` — project root, create or update per Step 1.
- `eng-flow/specs/<dated-slug>/wireframes/<journey-slug>.md` — one per journey from Step 2.
- `eng-flow/specs/<dated-slug>/mockups/<journey-slug>.html` — one per gated journey from Step 4.

---

## Step 7 — Report back

Confirm the saved paths and which journeys are gated vs. still markdown-only. Tell the user this feeds Stage 4 (UI-touching stories get acceptance criteria grounded in the approved wireframe/mockup) and Stage 5 (implementation builds against the mockup instead of improvising layout).

If this run was in guide mode, add a "Decisions I made / decisions you made" summary here, drawn from this run's `eng-flow-decision-log` calls.

Run the Step 7 analytics-finish call (see Analytics section above) before ending.
