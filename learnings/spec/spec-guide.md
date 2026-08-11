# spec.md — format, why, what the model does with it, effects

Deep dive from a working session going through `mastering-ai-assisted-development-checklist.md`'s
"Practice writing a `spec.md` per feature" item, verified against this repo's own `eng-flow-spec`
skill (`.claude/skills/eng-flow-spec/SKILL.md`, `PROCESS.md` Stage 1) and a real worked example in
`color-app/eng-flow/specs/2026-08-09-mvp-coloring-app-color-once-export-everywhere/spec.md`.

## 1. Two artifacts share one filename — they are not the same thing

The checklist conflates them because the source course and this repo both call the output
`spec.md`, but they're different tools for different jobs:

| | Vibe-coding spec (Osmani course, Ch.1) | eng-flow Stage 1 spec |
|---|---|---|
| **Scope** | One feature/prototype | A whole product/MVP's domains and journeys |
| **Lifespan** | Temporary — task-scoped, throwaway once built | Persistent artifact, checked into `eng-flow/specs/<dated-slug>/` |
| **Consumer** | The model, directly, in the same turn | Three later pipeline stages (domain model → architecture → epics/stories), not just the immediate response |
| **Produced by** | You, freehand, in one prompt | An interrogation loop (`eng-flow-spec` Steps 0–6), routed by decision type first |
| **Best for** | Prototypes, demos, portfolio pieces, internal tools | Anything that has (or will have) more than one build stage after it |

Don't reach for the heavier eng-flow format for a one-off script, and don't reach for the
lightweight 4-ingredient format for anything that needs a domain model or architecture pass after
it — the checklist's own annotation on this item already caught this: color-app's `spec.md` "is
heavier than a per-feature spec... produced by the `eng-flow-spec` stage," but still honors the
same underlying discipline both formats share (§2).

## 2. Format

### 2a. Vibe-coding spec — 4 ingredients, specific on constraints, open on implementation

1. **Technology constraints** — what stack/library/platform this must use or must avoid
2. **Visual requirements** — what it should look like
3. **Interaction model** — what the user does, what happens in response
4. **Performance targets** — what "fast enough" means, concretely

The failure mode on this format runs in both directions: too vague and the model fills every gap
with its own defaults (generic "AI slop" — see `learnings/claude/` for the same failure pattern in
CLAUDE.md); too detailed on *implementation* and it stops being a spec — the course calls this
"code with extra steps," because you've pre-decided things a spec is supposed to leave open.
There's no template file for this format in this repo (nothing here has used it standalone) — see
`vibe-coding-spec-template.md` in this folder for a blank one.

### 2b. eng-flow Stage 1 spec — 8 sections, interrogated not freehand

```markdown
# Spec: [Name]
## Decision            # what was approved, by whom, business outcome, constraints
## Problem & Audience  # problem, end user, stakeholders
## User Journeys       # narrative scenarios: "As a [user], I want to [action], so that [benefit]"
## Functional Requirements
## Non-Functional Requirements
## Domains Touched     # flat list — seeds Stage 2's domain model
## Scope                # MVP cut / explicitly out of scope
## Success Criteria     # specific, testable
## Open Questions
```

(A lighter `Context / Current State / Proposed Change / Acceptance Criteria / Out of Scope / Files
Reference` shape exists for the "feature/story in an existing product" path — same discipline,
smaller because a domain model and architecture already exist to extend rather than create.)

You do not write this freehand. `eng-flow-spec` Step 0 asks a routing question first — *not
decided yet* / *already approved* / *feature-story* — because each path skips different sections
(idea validation only runs on the "not decided" path; technical grounding only on "feature/story").
Getting that routing question wrong cascades: it "determines everything downstream," per the
skill's own Step 0 text, which is why routing gets an explicit `AskUserQuestion` even outside guide
mode.

Real worked example: `color-app/eng-flow/specs/2026-08-09-mvp-coloring-app-color-once-export-everywhere/spec.md`.

## 3. Why we need this

**The core mechanism both formats share:** fence the constraints tightly, leave the implementation
open. A spec that does this lets the model make good local decisions (file structure, component
boundaries, algorithm choice) without re-litigating things you've already decided (stack, scope,
what "done" means). The course frames this as "spec-driven development" being "the highest-impact
practice" it covers, citing a "2-3x better output" figure — checked independently (see
`learn/resources/mastering-ai-assisted-development-summary.md` §4) and found **plausible but
unverified as stated**: outside sources broadly agree spec-first workflows cut rework/iteration
count, some citing "3x fewer iterations," but no source matches this course's specific "2-3x
better output" framing. Treat the number as directional, not a verified stat — the qualitative
claim (less rework, less thrash) is the well-grounded part.

**Second reason, specific to the eng-flow format:** downstream stages have no other input. Stage 2
(domain model) takes "Stage 1 as input" per `PROCESS.md` — it doesn't re-ask the audience/scope
questions, it builds structure on top of what Stage 1 already decided. A vague or skipped section in
Stage 1 becomes a hole three stages downstream, at a point where fixing it costs a re-run of
whichever later stage already built on the gap, not a five-minute edit. `eng-flow-spec` Step 1 says
this explicitly: if the idea validation questions "can't be answered without hand-waving... stop —
offer to keep refining the idea rather than proceeding to a spec that will just encode the
vagueness."

## 4. What the AI model actually does with it

The two formats hand the spec to the model in structurally different ways:

**Vibe-coding spec → single-shot generation.** You write the 4 ingredients, hand the whole thing to
the agent in one prompt, and it produces a working artifact in that turn. The model isn't
"consuming" the spec across multiple stages — it's the entire brief for one generation pass. This
is also why over-specifying kills the format's value: the more of the *how* you pre-decide, the
less the model's generation step is actually doing.

**eng-flow Stage 1 spec → interrogation, then multi-stage consumption.** Two distinct model
behaviors, not one:

1. **Producing it** — `eng-flow-spec` runs a phased interrogation (Steps 0–6: route → idea
   validation → decision capture → audience/scope/domains → optional technical grounding → optional
   competitor research → draft and confirm). The model asks "until answered concretely — not
   restated requests," per Step 1 — it's explicitly instructed not to accept hand-waved answers and
   move on. This is the opposite of the vibe-coding mode's one-shot: the model's job here is
   extraction and refusal-to-proceed-on-vagueness, not generation.
2. **Consuming it** — once saved to `eng-flow/specs/<dated-slug>/spec.md`, three later stages read
   it as their starting input, each pulling a different slice:
   - **Stage 2 (domain model)** reads the `Domains Touched` list and `User Journeys` to build
     entities, relationships, and a conceptual system diagram
   - **Stage 3 (architecture)** reads `Non-Functional Requirements` to quantify deployment/tech
     decisions, and `Functional Requirements`/`Scope` to bound what needs building
   - **Stage 4 (epics/stories)** cross-checks `User Journeys` against Stage 3's system diagram to
     produce backlog items — "epics should map to the domains/functional areas named back in Stage
     1," per `PROCESS.md`

So the eng-flow format's spec.md is closer to a contract between pipeline stages than a prompt —
its audience is as much "the next skill run, days later, possibly a different session" as it is the
model answering you right now.

## 5. Effects / after-effects

**When it's done well:** the color-app spec's own text is a working example of "specific on
constraints, open on implementation" doing real work — its `Non-Functional Requirements` section
locks "no live third-party API integration in v1 (no POD fulfillment, no payment processing...)"
and its `Scope` section locks out user accounts, merch print, and AI style-transfer filters, all
without saying one word about component structure, state management, or file layout. Stage 3
(architecture) was free to make those calls with zero re-litigation of scope.

**When it's skipped or rushed:** `eng-flow-spec` Step 1's stop condition exists because a spec
built on hand-waved answers doesn't fail loudly — it succeeds at being *written*, then fails quietly
three stages later when Stage 2 or 3 hits a gap Stage 1 should have closed. The cost isn't paid at
spec time, it's paid downstream, which is exactly why the Decision Ledger treats spec-stage
routing as `risk`-tagged (`reason: risk` in the taxonomy) — "getting it wrong is costly or hard to
reverse downstream."

**A live example of a spec evolving instead of staying static:** the color-app spec itself was
amended after initial save — journeys 7 and 8 carry `(Added 2026-08-10, ...)` annotations, and
journey 7's amendment explicitly flags that stories were "drafted directly from this spec addition
per explicit user direction... this inverts the normal Stage 1→2→3→4 order," with each affected
story's Architecture Notes marked as pending a Stage 3 pass. That's the cascade risk from §3 made
concrete: skipping a stage (here, deliberately and flagged) leaves a traceable gap rather than a
silent one — the flag is what keeps the shortcut safe.

**Unchecked in this repo (from the checklist):** "Start a personal spec library — save specs that
produced good output" is still unchecked. Only one project has been through the full Stage 1 spec
process end to end, so there's no cross-project curated collection yet of "this spec's shape
produced good downstream output, reuse the pattern."

## 6. Rubric for evaluating any spec before it gets used

- [ ] **Specific on constraints, silent on implementation** — locks *what's allowed*, says nothing
      about component structure, file layout, or algorithm choice
- [ ] **Every answer is concrete, not restated** — no "everyone," no "as needed," no circular
      restatement of the question as the answer
- [ ] **Scope has an explicit out-of-scope list**, not just an in-scope one — the color-app example
      names user accounts, merch print, and AI style-transfer explicitly, not just what's in
- [ ] **Non-functional requirements are asked for, not assumed** — `eng-flow-spec` Step 3 flags
      these get skipped "if not prompted for"; silence here is a gap, not an implicit "none"
- [ ] **Right format for the job** — vibe-coding's 4 ingredients for a one-shot build; eng-flow's 8
      sections for anything with a domain-model/architecture stage after it
- [ ] **Amendments are flagged, not silently merged in** — if scope changes after the spec is saved,
      date-stamp the addition and flag any stage-order inversion it causes (§5's journey 7 example)

## Still open

- No personal spec library exists yet (checklist item, unchecked) — revisit once 2-3 more projects
  have been through `eng-flow-spec` end to end.
- No template repo with a pre-baked `.claude/` setup exists yet either (adjacent checklist item,
  also unchecked) — every new project currently means hand-copying the 13 `eng-flow-*` skills.
- The vibe-coding format has no real worked example in this repo yet, only the template in
  `vibe-coding-spec-template.md` — nothing here has used it standalone outside the eng-flow
  pipeline.
