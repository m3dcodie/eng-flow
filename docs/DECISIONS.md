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

From **agent-skills**:
- **`incremental-implementation`** and **`test-driven-development`** — the per-task loop (read acceptance criteria → load context → simplest-thing-that-works → RED/GREEN test → full regression + build → commit → mark complete), scope discipline ("noticed but not touching" instead of drive-by fixes), and hard stop conditions for high-risk/irreversible work — reused near-verbatim for eng-flow's Stage 5 (`eng-flow-implement`). The `/build` vs `/build auto` mode split, and the clean-baseline preflight that protects auto mode, were noted but not built yet — see Stage 5 section below.
- **`code-review-and-quality`** — the five-axis review (correctness, readability, architecture, security, performance), severity-label gating (Critical/Required block merge and get individual `AskUserQuestion` gates; Nit/Optional/FYI are listed, not gated), "review tests first," structural-remedy-not-just-complaint, and dead-code-ask-before-deleting — reused for eng-flow's Stage 6 (`eng-flow-code-review`), near-verbatim; this skill was already noticeably more complete than gstack's equivalent checklist (see Stage 6 section below).

## Stage 6 — Code Review: validated the same way as Stages 1-5

- **gstack**'s `review` (pre-landing PR review) has a code-quality/test/performance checklist, but thin (4 bullets each) — most of its bulk is gstack-infra machinery (specialist subagent dispatch, cross-model synthesis, Greptile comment resolution, gbrain persistence) that has no eng-flow equivalent and shouldn't.
- **agent-skills**' `code-review-and-quality` is comprehensive and directly reusable — see above.
- **External validation** (web search, 2026-08-07): independent code-review-checklist sources converge on the same five/six review dimensions and the same severity-labeling practice, confirming this isn't just one repo's house style.

**Decision:** add Stage 6 (`eng-flow-code-review`), the post-implementation counterpart to Stage 3.5 — Stage 3.5 reviews the design before code exists, Stage 6 reviews the code once Stage 5 has produced it. Scoped to a branch diff or a story's commits, output saved alongside the story next to its `tasks.md`.

## Stage 7 — QA: validated the same way as Stages 1-6

- **gstack**'s `qa` is capable but built entirely on gstack's own proprietary browser daemon (`$B`/`browse`) — confirmed to be a wrapper around Playwright, not a bespoke automation layer. eng-flow can't depend on gstack's wrapper at runtime, so the reusable part is the *methodology*, not the commands: mode structure (diff-aware default / full / quick), phased workflow (orient → explore → document), the per-page checklist (visual scan, interactive elements, forms, navigation, states, console errors, responsiveness), and evidence-immediately-not-batched documentation discipline.
- **agent-skills** has nothing here — it's a coding-agent toolkit, not concerned with browser-based behavioral QA.
- This environment itself has no browser-automation tool registered (checked directly via tool search) — confirming the skill must degrade gracefully (Playwright if available, MCP or project-local; guided-manual checklist if not) rather than assume one exists, unlike gstack which assumes its own daemon is always present.

**Decision:** add Stage 7 (`eng-flow-qa`), browser-based and front-end-only per explicit scoping — skip entirely for backend/CLI/library work. One structural change from gstack: split finding bugs from fixing them. gstack's `qa` does both in one skill; eng-flow's QA only documents (with evidence, severity-tagged using the same vocabulary as Stage 6), then hands fixes off to the already-built `eng-flow-implement` — keeps every stage's responsibility non-overlapping, consistent with the rest of the track.

From **agent-skills**, additionally:
- **`git-workflow-and-versioning`**'s branch-naming convention (`feature/<slug>`, `fix/<slug>`) — reused for `eng-flow-implement`'s new branch-setup step (pull latest base, create a branch, before starting a task) and referenced by `eng-flow-ship`'s pre-flight.
- **`/ship`**'s GO/NO-GO decision format with a mandatory rollback plan — reused for `eng-flow-ship`'s final report, without its parallel-subagent fan-out mechanism (no other eng-flow skill uses subagent dispatch, so this stayed single-agent for consistency).

## Stage 8 — Ship: validated the same way as Stages 1-7

- **gstack**'s `ship` is a large automated pipeline (review-readiness dashboard, specialist review army, Greptile comment resolution, adversarial review, its own version-bump CLI) — almost entirely built on gstack's proprietary infra (gbrain, review-log tooling) that eng-flow has no equivalent for and shouldn't build one just to mirror it. Its *sequence* is sound, though: pre-flight → review-readiness check → merge base before final tests → version/changelog → commit → push → PR.
- **agent-skills** splits this across `git-workflow-and-versioning` (semver, changelog, atomic commits, trunk-based development, branch naming) and a `/ship` fan-out orchestrator (parallel code-review/security/test-coverage subagents merged into a GO/NO-GO decision with a mandatory rollback plan).

**Decision:** add Stage 8 (`eng-flow-ship`), terminal. The key simplification versus both: eng-flow doesn't need a "review readiness" mechanism built from scratch, because Stages 6 and 7 already produce exactly that evidence as saved files (`code-review.md`, `qa-report.md`) — Ship's gate check just reads them instead of re-reviewing or dispatching parallel reviewers. Version/changelog bumping is conditional on the project already tracking them, not imposed. No cloud-deploy step — matches the user's actual local/container/GitHub practice, same as Stage 3's deployment section.

## Stage 9 — Retro: validated the same way as Stages 1-8

- **gstack**'s actual `retro` skill is a weekly team-velocity dashboard (shipping streak, session/context-switching patterns, per-author commit metrics) — not what "reflect learnings, don't repeat mistakes" describes. The real match is embedded *inside* `plan-eng-review` and `ship`: a "Capture Learnings" mechanism — typed entries (pattern/pitfall/preference/architecture/tool/operational), confidence-scored, gated on "would this save time in a future session?" — built on `gstack-learnings-log` + gbrain, both proprietary infra with no eng-flow equivalent.
- **agent-skills** has no retro concept, but `debugging-and-error-recovery`'s "Fix the Root Cause, not the symptom" (ask why until reaching the actual cause) and "Guard Against Recurrence" (a regression test that would have caught it) carry the right instinct at a smaller scale.
- **External validation** (web search, 2026-08-07): the standard industry shape for this is the blameless postmortem — system-focused language ("what process/check was missing," not "who"), root cause analysis, action items with owners. Organizations that run these consistently resolve repeat failures 2.5× faster than blame-first review cultures.

**Decision:** add Stage 9 (`eng-flow-retro`) — per-story, not weekly; root-cause-focused, not a metrics dashboard. Reuses gstack's learnings-entry taxonomy and confidence scoring as a plain project-level `eng-flow/learnings.md` file instead of `gstack-learnings-log`/gbrain. The mechanism only works if something reads the log later — `eng-flow-implement`'s Step 0 now checks `learnings.md` for entries relevant to the files/domain about to be touched, before starting a task.

## Stage 10 — Analytics (time + token logging): validated the same way as Stages 1-9

- **gstack**'s own telemetry (`ship`'s "Telemetry" section, directly inspected) logs skill name, wall-clock duration, outcome, and a synthetic per-invocation id (`$$-$(date +%s)` — not a real session identifier) to local JSONL files. It does **not** track tokens anywhere — confirmed by grepping the whole telemetry pipeline for "token" and finding nothing usage-related. This environment's own Claude Code harness-level session logs (`~/.claude/analytics/session-ends.jsonl`) have a `total_cost_usd` field in the schema that is `null` in every entry checked directly — token/cost tracking isn't something even the harness reliably surfaces by default.
- **The actual mechanism**: Claude Code writes a full session transcript to `~/.claude/projects/<sanitized-cwd>/$CLAUDE_CODE_SESSION_ID.jsonl` (confirmed live in this environment), where every assistant turn carries a `message.usage` block (`input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`). `$CLAUDE_CODE_SESSION_ID` is a real, live env var. This is genuinely new — neither gstack nor agent-skills does this. It's Claude-Code-specific (depends on an undocumented transcript format that could change) and approximate (line-range-based, doesn't isolate subagent/Task-tool transcripts), so it must degrade to time-only rather than fail when unavailable.
- **agent-skills** has no telemetry/analytics mechanism at all.

**Decision:** add Stage 10 (`eng-flow-analytics`), read-only, plus two shared scripts (`eng-flow-analytics-checkpoint`, `eng-flow-analytics-finish`) under `.claude/skills/lib/bin/` that every other skill (Stages 1-9) now calls at the start of every step and the end of its final step. Per user instruction ("log as we go... if terminal killed or system restart we might lose [it]"), logging is incremental per-step, not just start/end of a whole skill run — a marker file is flushed to `eng-flow/analytics.jsonl` and reopened at every step boundary, so a crash loses at most the current in-flight step, not the whole run; an orphaned marker from a genuine crash gets recovered and flushed (tagged `"outcome":"recovered"`) the next time that skill starts, rather than silently lost. Token extraction tries python3 first (proper JSON parsing, tested against this session's real transcript and confirmed correct), with graceful no-op degradation, not a hard failure, if the session id or transcript aren't available. No dollar-cost estimate is computed — pricing isn't this mechanism's job to track; raw token counts by category only.

## Stage 5 — Implementation: validated the same way as Stages 1-4

- **gstack** has no equivalent. `autoplan` is a review-orchestration pipeline (chains its CEO/design/eng/DX review skills), not a code-writing loop; `ship` is post-implementation (merge, test, PR, deploy); `careful` is a destructive-command guardrail. gstack assumes ordinary agentic coding happens between its review gates — there's no prior art here to reuse.
- **agent-skills** has exactly this, cleanly separated from planning: `/build` (single task, then stop) and `/build auto` (whole plan, one upfront approval, still one commit per task) built on `incremental-implementation` + `test-driven-development`.

**Decision:** add Stage 5 (`eng-flow-implement`), terminal, consuming the `tasks.md` that Stage 4's on-demand task breakdown already produces in the exact template agent-skills uses — so the handoff is direct, no translation needed. Built only the default single-task mode this round; whole-story auto mode is explicitly deferred rather than approximated, per the user's call ("auto we will do it later").

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
