# Design rationale

Why eng-flow looks the way it does, what was deliberately reused from prior art, and what's genuinely different — so the divergences read as decisions, not oversights.

## Starting question: build our own, or use gstack / agent-skills as-is?

Two mature tools were already available locally:

- **gstack** (`~/projects/gstack`) — a large, opinionated "virtual eng team": 31 Claude Code skills plus a browser-automation daemon. High-rigor by default (its own docs call testing "NEVER SKIP OR COMPRESS" and frame full test/edge-case/error-path coverage as the default goal, not opt-in). Already active as installed skills in this environment (`spec`, `autoplan`, `plan-eng-review`, `ship`, `review`, `office-hours`, etc.).
- **agent-skills** (`~/projects/agent-skills`) — lighter: 8 commands (`/spec /plan /build /test /review /ship`) across 24 checklist-style skills, closer to a curated playbook than gstack's infra+personas package.

Neither repo has a project-level MVP-vs-production gate, an epics/stories/domain-modeling hierarchy, or a staged separation between business requirements and technical design (see below). Both are also actively-developed third-party tools eng-flow shouldn't depend on at runtime — this repo needs to work for engineers who don't have either installed.

**Decision:** don't build a competing automation framework, and don't install both as active tooling (their command names collide: both effectively own `/spec`, `/review`, `/ship`-equivalents). Instead: build a thin, self-contained, tool-agnostic process layer (templates + a playbook) that fills the actual gap — the MVP↔production gate and the staged requirements→domain→architecture→execution breakdown — and copy specific proven mechanics out of prior art by hand where they're good, rather than depending on the source repos.

## Reused directly (with attribution)

From **truth-engine** (`~/projects/truth-engine/.claude/skills/`), a project the user already built and uses:

- **`spec-mvp`** — the five-phase interrogation (why → scope → technical → draft → file) is the direct ancestor of eng-flow's Stage 1 spec phases. Its explicit framing — "the five-questions interrogation... is what makes a spec good, not ceremony" — is the operating principle for keeping eng-flow's spec light without becoming useless.
- **`verify-claims`** and **`idea-evaluator`** — the bounded-research pattern (registry/seed-first, one-fallback-search-max, tiered opt-in depth, "propose, don't run" further research) is copied near-verbatim into eng-flow's optional competitor-analysis sub-step. This exists because unbounded research is a real, previously-hit failure mode: verify-claims' own docs note deep-verifying every claim "cost ~20+ minutes and hundreds of thousands of tokens per transcript in testing" before the bounded version replaced it.

From **gstack**:
- **`office-hours`**'s six forcing questions (demand reality, status quo, desperate specificity, narrowest wedge, observation, future-fit) — trimmed into eng-flow's idea-validation step, used *only* on the "not yet decided" routing path (see below for why not always).
- **`gstack-slug`**'s sanitization approach (strip to `[a-zA-Z0-9._-]`, lowercase) — reused for eng-flow's per-spec slug, adapted from gstack's repo-level slug (keyed to git remote, for a global cross-project store) to a per-spec-topic slug (since eng-flow saves inside each project's own repo, not a global store).
- **`plan-eng-review`**'s "Cognitive Patterns — How Great Eng Managers Think" (7 of its 15, the ones that apply before any code exists: boring by default, blast radius, Conway's Law, essential vs. accidental complexity, two-week smell test, incremental over revolutionary, reversibility preference) and its architecture-review checklist (component boundaries/coupling, data flow bottlenecks, scaling/SPOF, security architecture, per-integration-point failure scenarios, distribution path) — reused for eng-flow's Stage 3.5. Also reused its one-issue-per-`AskUserQuestion`, stop-and-wait discipline, and its line-grounding requirement (a finding must cite the specific text that motivated it), simplified from gstack's numeric confidence-score/false-positive-suppression apparatus since that machinery is built for noisy code-diff review, not a single architecture doc.

## Stage 3.5 — Engineering Review: validated the same way as Stages 1-4

- **gstack** has a direct, well-developed equivalent (`plan-eng-review`): a design-review gate that runs *before* implementation, reviewing a design doc or plan against 15 named engineering-leadership heuristics plus a 4-section checklist (Architecture, Code Quality, Tests, Performance). Its own framing is explicit — this runs "before you start coding — to catch architecture issues before implementation."
- **agent-skills** has no equivalent — only `code-review-and-quality`, which reviews committed code. A grep across every agent-skills `SKILL.md` for "design review," "design doc review," or "architecture review" returned zero hits.
- **External validation** (web search, 2026-08-07): formal review gates before implementation (PDR/CDR-style in systems engineering; architecture review boards per AWS/Mozilla/Microsoft playbooks) are established practice specifically because architectural decisions are far less costly to fix at the design stage than after implementation has started.

**Decision:** add Stage 3.5, positioned after Architecture and before Epics/Stories/Tasks — reviewing `architecture.md` before effort is spent breaking a potentially-flawed design into a backlog. Only the *architecture-relevant* subset of `plan-eng-review` applies, since no code exists yet at this point in the track — its Code Quality/Test/Performance sections, brain-context integration, and telemetry are gstack-infra-specific and don't carry over.

## Where eng-flow deliberately diverges

**Neither gstack nor agent-skills separates "pure requirements" from "technical shape."** Checked directly against source:

- gstack's `/spec` Phase 3 ("Technical Interrogation") is a stated **"HARD requirement"** — even for "truly novel greenfield" work, it still asks about data model, API, background processing, infrastructure; it only skips the code-citation step when there's no code to cite.
- agent-skills' `spec-driven-development` template puts **Tech Stack** and **Project Structure** as fields in the same document as **Objective** — decided in the same pass, no staging between them.

Both make sense for what they're built for: one person going from idea to shipped code in a single sitting, where there's no organizational handoff to protect. eng-flow's actual target (per the user: companies with separate non-technical stakeholders and technical staff who need common ground before technical framing starts) is a different problem.

**External validation** (web search, 2026-08-06) confirms the staged separation matches established practice, not just this user's past employers:

- "The most effective product development processes use PRDs and specs sequentially, defining and validating the problem space before committing to solutions." — [PRD vs TRD](https://medium.com/@kokoproduct/decoding-the-dichotomy-prd-vs-trd-67463a29aa84)
- "The BRD is developed first, and the FRD is created after the BRD is approved... before technical design begins." — [BRD vs FRD](https://deeprojectmanager.com/brd-vs-frd/)
- The BRD→FRD→technical-architecture layering documented across multiple business-analysis sources maps directly onto eng-flow's Stage 1 (requirements) → Stage 2 (domain model) → Stage 3 (architecture).

So eng-flow's four-stage production track (requirements → domain model → architecture → epics/stories/tasks) is a deliberate return to that industry-standard layering, chosen because this repo targets multi-stakeholder orgs, not the solo/small-team velocity context gstack and agent-skills optimize for.

## Routing: why idea-validation isn't always run

Early design had one "spec" skill silently branching between "ask forcing questions" and "take the spec directly" based on whether the project looked like a startup MVP. That's wrong: the trigger isn't "is this an MVP," it's **"has a decision-maker with the authority to approve this already done so?"**

- If nobody's decided yet (solo idea, side project) → idea validation is the right gate; skipping it risks speccing something nobody wants.
- If stakeholders already greenlit it → re-running demand-validation questions on the engineering side re-litigates a decision that isn't the engineer's to make, and is pure friction.
- If it's a feature/story in an existing product → the demand question was answered when the product shipped; go straight to spec.

This is why Phase 0 (routing) checks decision-authority first, not project size or "startup-ness."
