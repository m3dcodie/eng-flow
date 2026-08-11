# Agentic Engineering Knowledge Base / Course — Requirements & Positioning

Status: draft requirements, not yet actioned. Captures the brief as given, plus a first pass of
market research to ground the angle decision. This is a *product* question (what to build, for
whom), separate from the `unicornle-content` skill's per-deck research process — if a course/KB
gets greenlit, individual pieces of content still go through that skill's gated stages.

## 1. The ask

Turn accumulated learnings into a knowledge base — course, guide, or similar — about how AI is
changing software engineering. Core questions the content needs to answer:

- How much of the "AI replaces engineers" narrative is true vs. hype?
- What is actually being replaced, concretely?
- Where is a human still required, and why?
- Where does AI collapse or hallucinate, and what are the failure modes?
- How do you guide AI, and how do you restrict it (guardrails, permissions, budgets)?
- What's actually available today (tools, patterns, platforms) — not a snapshot that decays in a
  month.

## 2. What already exists to draw on

Three tiers of source material, already on disk:

**Primary evidence — the user's own running systems**, not secondhand claims:
- `eng-flow` (`~/projects/eng-flow`) — the agentic-engineering-flow framework itself
  (`PROCESS.md`, `CLAUDE.md`, `docs/DECISIONS.md`). This is the "first attempt" at productizing a
  guided, skill-based, analytics-instrumented dev process — self-described as having room for
  improvement, not a finished product.
- `color-app` (`~/projects/color-app`) — a project built *using* eng-flow, with a live
  `eng-flow/` working directory containing `decisions.jsonl` (~55KB, every judgment call logged),
  `analytics.jsonl` (~64KB, time/token cost log), a `backlog/`, and design artifacts. This is the
  closest thing to a real "black box flight recorder" of an AI-assisted build — two cost baselines
  are already captured in that project's own memory (SVG-import spike: ~64min/~13M tokens; whale
  fixture spike: ~15min/~9M tokens).
- `content-gen` (this repo) — a second, independent proof point: a production pipeline built
  under different constraints (hard budgets, no-silent-retry, fixed agent permission profiles),
  including one documented failure (`unicornle/research/drafts/agentic-pipeline-postmortem.md` —
  the automated Research→Verify pipeline burned ~745k tokens/~$4 for one verified fact, which is
  *exactly* the kind of "where AI collapses" evidence most course content only gestures at
  abstractly).
- `gstack` — described as "created by top people," i.e. an external reference point for a more
  mature/prescriptive productized version of the same idea (skills, guided review pipelines,
  ship/land workflows) — useful as a comparison, not something to imitate wholesale.

**Secondary — already-done market analysis** (both already fully researched and fact-checked
against outside sources as of August 2026, sitting in `unicornle/resources/`):
- `mastering-ai-assisted-development-summary.md` — Addy Osmani's LinkedIn Learning course on
  CLAUDE.md/skills/MCP/subagents/RALPH loops, independently fact-checked line by line.
- `ai-native-engineering-foundations/analysis.md` — Osmani's other course (the "70/30 split,"
  the "agentic AI vs. agentic engineering" definitional split, a 4-course competitive table, and
  a genuinely sharp tension already surfaced: the individual-amplification narrative running
  simultaneously against company-level AI-cited layoffs — see its §6 layoffs table and §8 content
  ideas).
- Neither file has picked a final angle yet — `analysis.md` §9 explicitly flags this as the open
  decision blocking the `content/agentic-engineering` branch.

**This session's market pass** (new, see §4) — confirms where the existing two files' analysis
still holds and adds the competitive-landscape check specific to "should this be a course."

## 3. Reframing the ask: course vs. knowledge base vs. content pillar

Three different products were named loosely in the same breath ("knowledge base... course
etc."). They have different bars:

| Format | Bar to clear | Fits this material? |
|---|---|---|
| Single deck/post (existing `unicornle-content` skill) | One sourced claim or small cluster, ships in a day | Yes — several are already outlined in `analysis.md` §8, unstarted |
| Ongoing content pillar (recurring decks/posts) | Needs a cadence of new primary sources, not one-time synthesis | Partially — the eng-flow/color-app logs are a renewable source as new stories land |
| Standalone course/knowledge base | Needs enough original material to sustain 5-10+ modules, a reason to pay/subscribe vs. read a LinkedIn post, and a maintenance plan as tools churn | Not yet evidenced — see §5 gap analysis |

Recommendation: don't commit to "build a course" as the first move. The evidence-backed, low-risk
next step is the content-pillar path already half-built (`content/agentic-engineering` branch),
and treat "does this become a course" as a decision to revisit *after* 3-5 pieces of pillar
content ship and get real engagement data in `unicornle/history/metrics.md`. This mirrors the
project's own working style (ship small, verified units; don't build ahead of evidence).

## 4. Market pass (WebSearch, 2026-08-11)

Four queries run: production-grounded-course landscape, "agentic engineering" productized
playbooks, enterprise AI-guardrails content, and CTO/engineering-leader cohort demand.

**Findings:**
- The market is dense with *generic* agentic-AI/prompt-engineering courses (Udemy "AI Agentic
  Engineering: Zero to Hero," Maven's "Agentic AI Engineering Bootcamp," Coursera's practitioner
  courses) — high volume, low differentiation, priced for individual ICs.
- One close conceptual competitor: **Domino.ai's "Agentic Engineering: A Practitioner's
  Playbook"** — a methodology + prompt-framework writeup for data science/ML practitioners. Same
  "playbook" framing this brief is reaching for. Worth reading before finalizing an angle, to
  avoid overlap.
- The CTO/leadership tier is real and paying at premium price points: MIT Sloan's "Implementing
  Agentic AI" ($1,900, cohort), an "AI-Native Engineering Leadership Cohort" (6 weeks, cohort 4
  running Aug 17 2026), InfoQ's AI engineering cohort/certification, Snowflake's CTO Circle
  events. This matches `unicornle/README.md`'s existing stated audience (CTOs/engineering
  leaders) — so audience fit is not the open question; differentiation is.
- **The gap these don't fill:** every competitor product found is *consultant-framework-driven*
  (a methodology taught top-down) or *vendor-demo-driven* (a tool walkthrough). None surfaced are
  *artifact-driven* — built from a named person's own running systems' decision logs, cost data,
  and one documented, admitted failure. That's the one clear differentiation angle this user is
  actually positioned to take that the competitive set is not: "here is what actually happened
  when I ran this, including where it broke," backed by `decisions.jsonl`/`analytics.jsonl`/the
  pipeline postmortem — not a reconstructed case study.

## 5. Gap analysis — is there enough original material yet?

Honest current state, not a sales pitch:

- **Enough for 3-5 pillar pieces now**, drawing on: the pipeline postmortem (a real "where AI
  collapses" story with a dollar figure), the two spike cost baselines from color-app, the
  layoffs-vs-amplification tension already sourced in `analysis.md` §6, and the "agentic AI vs.
  agentic engineering" definitional split.
- **Not yet enough for a full course.** `eng-flow` is explicitly self-described as a first
  attempt with room for improvement, and only one project (`color-app`) has run through it far
  enough to produce real logs. A course claiming "this is the guided process" needs either (a)
  eng-flow to mature through 2-3 more projects first, or (b) the course framed honestly as "field
  notes from building this, in public" rather than "the finished method" — the latter is a
  legitimate and arguably more credible framing given the audience (CTOs can tell polished
  vendor-speak from real learnings, per the course-comparison research above valuing "genuinely
  useful, tool-agnostic mental models" over demos).

## 6. Candidate angle (recommended)

**"Field notes from running agentic engineering in production — what held, what broke, what it
cost."** Practitioner-authored, artifact-backed (real logs, real dollar figures, one real
documented failure), explicitly positioned against the vendor-demo and consultant-framework
courses already crowding the market. Speaks to the brief's core questions directly:
- Truth vs. hype → answered with the pipeline postmortem's actual token/dollar burn, not a general claim.
- What's replaced / where humans still needed → the 70/30 framing (already sourced in
  `ai-native-engineering-foundations/analysis.md`) plus this user's own budget-enforcement and
  no-silent-retry design decisions as concrete "here's where we didn't trust the model" examples.
- Where AI collapses/hallucinates → the postmortem, plus any color-app `decisions.jsonl` entries
  that show a caught bad AI output (needs a pass through that file to extract 2-3 concrete
  examples — not yet done).
- How to guide/restrict AI → this repo's own locked architecture decisions (hard budgets, fixed
  permission profiles, no orchestration framework) are a directly reusable case study.

## 7. Audience

Matches the existing unicornle stated audience — CTOs and engineering leaders evaluating AI
adoption, tooling, and how to structure teams/process around it (per `unicornle/README.md`) — not
individual ICs learning to prompt better. The market pass in §4 confirms this tier pays for
cohort-style content; a lighter-weight version (pillar content building toward a possible later
course) is the appropriate entry point rather than launching a paid product cold.

## 8. Recommended structure, if/when a course is greenlit

Not a commitment — a placeholder shape to revisit after §3's evidence-gathering step:

1. The claim vs. the data (truth vs. hype, using real layoffs data + the productivity-paradox research already sourced)
2. What actually gets replaced (the 70/30 split, with this user's own artifacts as the worked example instead of the abstract version)
3. Where it collapses (the pipeline postmortem, walked through as a case study — root cause, cost, the guardrail added afterward)
4. Guiding and restricting AI (budgets, permission profiles, no-silent-retry, human-in-the-loop review gates — drawn directly from this repo's and eng-flow's actual locked decisions)
5. What's available today (a maintained-not-frozen module — likely the fastest-decaying part per the existing course comparison, so keep it thin and dated rather than exhaustive)
6. Running it yourself (a distilled, honest version of eng-flow/PROCESS.md — framed as "field notes," not "the finished method," per §5)

## 9. Open decisions before any content work starts

- Confirm format for the *next* concrete step: is it (a) resume the stalled `unicornle-content`
  skill run on `content/agentic-engineering` for one pillar piece (angle candidates already listed
  in `analysis.md` §8), or (b) something new scoped directly at this course question?
- Decide whether "field notes, in public" (honest, still-evolving) or "the finished method"
  (higher production value, needs eng-flow to mature more first) is the framing — §5 recommends
  the former for now.
- A pass through `color-app/eng-flow/decisions.jsonl` and `agentic-pipeline-postmortem.md` to pull
  2-3 concrete "AI got it wrong, here's what caught it" examples has not been done yet — needed
  before §6's angle can move from claim to sourced content.
- This document itself does not commit token/search budget the way the `unicornle-content` skill's
  Stage 1/2 does — treat it as pre-Stage-0 scoping, not research output.

## Sources consulted (this session's market pass)

- [AI Agentic Engineering: Zero to Hero Masterclass 2026 — Udemy](https://www.udemy.com/course/ai-engineer-course/)
- [The Agentic AI Engineering Bootcamp & Certification — Maven](https://maven.com/stemplicity/become-an-agentic-ai-engineer)
- [Agentic Engineering: A Practitioner's Playbook — Domino.ai](https://domino.ai/blog/agentic-engineering-practitioners-playbook)
- [agentic ai content for practitioners — Coursera](https://www.coursera.org/learn/agentic-ai-content-for-practitioners)
- [Agentic Development Platforms — Platform Engineering University](https://university.platformengineering.org/agentic-development-platforms-october-2026)
- [Implementing Agentic AI: Building Your Organizational Playbook — MIT Sloan / GetSmarter](https://www.getsmarter.com/products/mit-sloan-implementing-agentic-ai-program)
- [AI-Native Engineering Leadership Cohort — END GAME](https://end.game/cohort/)
- [InfoQ Launches Online AI Engineering Cohort and Certification — InfoQ](https://www.infoq.com/news/2026/05/ai-engineering-certification-pro/)
- [CTO Circle: Lessons on Building AI-Native Engineering Teams — Snowflake](https://www.snowflake.com/en/blog/cto-circle-ai-native-engineering/)
- [AI Guardrails: The Foundation of Responsible Enterprise AI — Medium/Zapcom](https://medium.com/@zapcomgroup/ai-guardrails-the-foundation-of-responsible-enterprise-ai-8446f4332e35)
- [AI Coding Assistants in 2026: 4x Faster, 10x Riskier — Kusari](https://www.kusari.dev/blog/ai-coding-assistants-in-2026-4x-faster-10x-riskier-the-hidden-security-cost)
