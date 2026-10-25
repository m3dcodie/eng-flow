# eng-flow and the AI capability levels — L1 → L4

**Status:** first draft — content and framing to be polished later.
**Purpose:** explain how `eng-flow` (this repo) maps onto the AI-capability-level framework, and how someone adopting it gets carried from ad hoc AI use up to Level 4 ("Harnessed delivery") — not as a personal evidence log, but as a resource anyone can pick up.

---

## 1. The levels, recapped

| Level | Name | Definition |
|---|---|---|
| 0 | Autocomplete | AI suggests small snippets, phrases, ideas or completions. You still create the work yourself. |
| 1 | Assistant | AI handles small, well-defined tasks. You review, adapt and decide what to use. |
| 2 | Pair collaborator | AI helps you plan, create, test, analyse, design, document or improve work. You still drive the process. |
| 3 | Agent as collaborator | AI produces substantial outputs. You review, steer and integrate the work. |
| 4 | **Harnessed delivery** | Humans define outcomes, constraints and review points. AI or agents work through defined workflows, harnesses, patterns or guardrails. |
| 5 | Software / delivery factory | Specs, goals or briefs go in, and working outputs come out with minimal human review of the implementation details. |

Levels 0–3 are about how *capable* the AI use is. Level 4 is a different axis entirely — it's not "the AI got better," it's "the work now runs inside a defined harness instead of inside one person's judgment call by call." That's the level eng-flow is built to produce.

---

## 2. How eng-flow carries someone from L1 to L4

Read this as a progression, not a menu — each level's mechanisms are still present at the next level, just wrapped in more structure.

### Level 1 — Assistant

Small, well-defined, human-reviewed tasks. **eng-flow's Stage 5** (`eng-flow-implement`) is built around exactly this unit: one pending task from a story's `tasks.md`, implemented with strict scope discipline ("touch only what the task requires, log anything else noticed rather than fixing it inline"). Nothing here requires the rest of the track — this stage alone is L1 if that's all someone runs.

### Level 2 — Pair collaborator

AI helps plan, create, test, analyse, design, document, improve — human still drives. This is most of the production track *as individual stages*: `eng-flow-spec` (plan), `eng-flow-domain-model`/`eng-flow-architecture` (design), `eng-flow-code-review` (improve), `eng-flow-qa` (test), `eng-flow-retro` (document). Each stage is interactive — findings and decisions are surfaced one at a time via `AskUserQuestion`, with explicit accept/change/reject before moving on. The human is never out of the loop at this level; eng-flow just gives that collaboration a repeatable shape instead of starting from a blank page each time.

### Level 3 — Agent as collaborator

AI produces substantial output; human reviews, steers, integrates. **Stage 5's TDD loop** (implement → test → verify → commit, one task per run) is this in practice — a full working change per task, autonomously produced, but every commit is a checkpoint the human can inspect. The **Decision Ledger**'s `owner` field (`user_confirmed` / `user_revised` / `ai_inferred` / `ai_default`) is literally instrumenting the boundary this level describes: how much of each judgment call the AI made vs. the human steered.

### Level 4 — Harnessed delivery

*"Humans define outcomes, constraints and review points. AI or agents work through defined workflows, harnesses, patterns or guardrails."* This is a description of eng-flow's own architecture, not a stretch mapping onto it:

| L4 requirement | eng-flow mechanism |
|---|---|
| Humans define **outcomes** | Stage 1 (`eng-flow-spec`) — plain-language requirements both business and engineering sign off on, before any technical framing happens |
| Humans define **constraints** | Stage 3 (`eng-flow-architecture`) — tech stack, NFRs, and the optional `eng-flow/security-policy.md` (standing project-level rules, established once, checked by every later stage) |
| Humans define **review points** | Stage 3.5 (design review, before code exists) and Stage 6/7 (code review + QA, after code exists) — both interactive, both gate on explicit accept/reject |
| AI works through a **defined workflow** | The ten numbered stages themselves — strictly staged, no stage asks what a later stage owns |
| ...with **harnesses** | Stage 8 (`eng-flow-ship`) refuses to ship unless Stage 6/7's saved verdicts say so — a gate, not a re-review |
| ...**patterns** | Skills reuse named, attributed patterns rather than inventing fresh each time (see `docs/DECISIONS.md`) |
| ...**guardrails** | Hard stops before high-risk/irreversible work (auth, destructive migrations, payments, deletions, deploys, secrets) — explicit sign-off required, not a silent default |

The mechanism that makes this durable rather than a one-off checklist is the **Decision Ledger** + **Analytics** pair running underneath all ten stages: every judgment call is logged with *why* it was routed the way it was, and Stage 9 (`eng-flow-retro`) reads that log back to find where `ai_default`/`ai_inferred` decisions correlated with rework or bugs — promoting exactly those decision points to a mandatory human checkpoint next time. The harness gets stricter where the evidence says it should, not where intuition says it should.

---

## 3. The harness-engineering checklist, mapped

The specific practices that define "Harness engineering" (the make-or-break area for Level 4) and where eng-flow builds each one:

- [x] **Prompt libraries and reusable patterns** — the skills themselves (`SKILL.md` files) are versioned, reusable process definitions, each with its rationale documented in `docs/DECISIONS.md`
- [x] **Test harnesses for AI-generated code** — Stage 5's incremental TDD loop; every task ships with a RED/GREEN test before it counts as done
- [x] **Evaluation harnesses for AI-assisted outputs** — Stage 6/7's severity-tagged review and QA, run against a fixed five-axis checklist (correctness, readability, architecture, security, performance) rather than ad hoc
- [x] **Quality gates or review processes** — Stage 8 hard-gates on Stage 6/7's saved verdicts; Critical/Required findings block merge and get an individual sign-off gate
- [ ] **CI/CD integration for AI-assisted workflows** — not yet wired into eng-flow's own template; the gate logic exists (Stage 8) but nothing runs it automatically on a PR yet — see §6
- [x] **Internal tools and scripts around AI** — `.claude/skills/lib/bin/`: decision-log, analytics-checkpoint/-finish/-report, findings-log, all shared across every stage
- [x] **Context engineering** — `learnings.md` checked before each task, `security-policy.md`/`design-system.md` reused across specs instead of rebuilt, ADRs as the record of *why*, not just *what*
- [x] **Measurement of delivery impact** — Stage 10 (`eng-flow-analytics`): cycle time (elapsed vs. active), rework cycles, bug rate, time/token cost — all derived from data every other stage already logs, not a separate instrumentation effort

Seven of eight are built into the process itself. CI/CD is the one gap in eng-flow's own template today — see the path section below.

---

## 4. Other public resources worth pointing to

These are separate, standalone public repos — each demonstrates a specific capability that sits *alongside* eng-flow rather than inside it. Useful as concrete, inspectable examples for anyone building their own case toward Level 4, not just as citations.

### `eng-flow` (this repo)
The harness itself — everything in §2–3 above. Also the clearest example of "don't build from scratch, reuse and attribute": `docs/DECISIONS.md` documents exactly what was adapted from prior art (gstack, agent-skills, truth-engine) and what was deliberately built fresh, with the reasoning for each call.

### `LLM-Capability-Framework-LCF`
A structured, cross-provider model-and-prompt comparison methodology — six layers (extraction → translation → interpretation → expansion → composition → agency/verification), each with a defined, scored metric (Schema Adherence Rate, Entity Extraction Rate, Hallucination-Free Alignment, Artifact Conversion Rate) and raw model outputs kept for inspection. Comparisons span Anthropic, Google, OpenAI, Alibaba (Qwen), Mistral, and DeepSeek models — genuinely cross-provider, not single-vendor. Good example of what "comparing models, prompts and providers" looks like as a documented practice rather than a one-off "I tried GPT once."

### `rag-pipeline`
A working, tested, multi-service RAG implementation — ingestion, chunking, embedding (`sentence-transformers`), vector storage/query (ChromaDB), context augmentation — with functional tests exercising the full chain end-to-end, not just unit-level stubs. Good example of RAG/embeddings/vector-database practice backed by a runnable system rather than a tutorial follow-along.

### `arch_agent` (ADAG)
A multi-agent LangGraph system that audits infrastructure-as-code against compliance policy and generates auto-remediation patches — five orchestrated nodes (`intake → policy_analyst → auditor → validator → remediation`) with checkpointed state, built across three shipped phases with tests. Good example of agent orchestration *and* guardrails-as-product — the whole system's purpose is enforcing constraints automatically, which is the same instinct Level 4 asks for applied to a different domain (infra policy instead of software delivery).

---

## 5. Case study — why the harness matters, not just that it exists

*(Generalized from a real incident on a separate, private project — details below are accurate but the source project isn't named here.)*

An autonomous, multi-stage AI pipeline — research an input, verify the output, synthesize a final artifact — was given hard token/call budgets, per-stage permission profiles, and full audit logging specifically because an earlier, less-guarded system had already burned significant time and spend failing the same way once before. On its first real run against a live topic, it failed again anyway: one stage blew through its budget on a single oversized call, and the next stage's verification step returned empty responses on 12 of 13 attempts — silently, at close to full token cost each time. Total spend for one usable output: roughly 745,000 tokens.

The root cause wasn't a bad plan. The budgets, permission boundaries, and audit tables were all there and all correctly designed on paper. The failure was in a gap between what the system's guardrails *assumed* they could control and what the underlying model API actually exposed for intervention mid-call — a single API call is opaque and uninterruptible from the outside, no matter how carefully the surrounding harness is built. Nobody could see the problem happening in time to stop it, because the unit of control was too coarse.

Root-causing it well is the part worth generalizing hardest: rather than trusting a secondhand summary of what went wrong, the investigation went straight to the pipeline's own audit-log table and queried it directly. That corrected the original hypothesis (the failure was an empty response, not a truncated one) and surfaced a second, non-obvious finding — the audit table designed to survive exactly this kind of failure did, because it committed per-call, immediately; a separate table meant to hold richer diagnostic detail didn't, because it only committed at a later point the forced-killed process never reached. "Log the moment it happens" and "hold full detail for later" turned out not to be the same guarantee.

**The generalizable lesson for Level 4 work specifically:** a harness's safety controls can only be as precise as the granularity the underlying tooling actually gives you to intervene — and you don't find out whether that granularity is sufficient until you've run the real thing, small and cheap, at least once. This is exactly why eng-flow's review points sit *between* stages rather than only at the very end, and why hard stops exist before irreversible work: the smaller the blast radius of an unsupervised step, the less this failure mode costs when it happens — and it will happen.

---

## 6. Practical path — closing what's left

- [ ] Wire CI (e.g. GitHub Actions) to run the same pass/fail checks Stage 8's gate logic already applies, so the quality bar is enforced automatically on every PR, not only when someone remembers to run the ship stage manually — this is eng-flow's one remaining harness-engineering gap (§3).
- [ ] When adopting eng-flow on a new project, run the full ten-stage track at least once end to end before judging it — several of the L4 mechanisms (Decision Ledger correlation, analytics rollups) only produce useful signal after enough data accumulates.
- [ ] Treat the case study in §5 as a standing argument for *why* review points and hard stops exist, not just process overhead — it's cheap to skip a gate until the one time skipping it costs 745K tokens.

---

*Next pass: tighten the level-by-level mapping with more direct file citations (line numbers), decide whether §5's case study needs even lighter genericizing, and fold in whichever new public repos surface.*
