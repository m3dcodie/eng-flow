# Mastering AI-Assisted Development — Personal Action Checklist

Derived from `mastering-ai-assisted-development-summary.md`. Ordered roughly by dependency (later items assume earlier ones are in place), not by time.

## Foundation: spec-first habits
- [x] Write a `CLAUDE.md` for each active project (stack, standards, architecture rules — persistent)
  - Done in both `eng-flow` and `color-app`. Learning: the discipline that kept it useful was *persistent-rules-only* — task/story state lives in `eng-flow/` (specs, backlog, decisions.jsonl), not CLAUDE.md, so it never rotted into a stale to-do list.
- [x] Practice writing a `spec.md` per feature: technology constraints, visual requirements, interaction model, performance targets — specific on constraints, open on implementation
  - `color-app/eng-flow/specs/.../spec.md` is heavier than a per-feature spec (domains/journeys for the whole MVP), produced by the `eng-flow-spec` stage. It still nails "specific on constraints, open on implementation" — e.g. the hard "no per-user AI call in v1" rule and the export/format matrix in `specs/requirement.md` fenced the *what's allowed* tightly while leaving component structure fully open to Stage 3.
- [x] Start a personal spec library — save specs that produced good output
  - Started: `learn/spec-library/` with 2 entries pulled from `color-app` (the only project through the full spec stage so far) — an end-user MVP feature spec and a narrow internal dev-tool spec, both chosen because they produced a full downstream chain (domain-model.md, architecture.md, ADRs). Deliberately kept unabstracted (copied verbatim, not stripped of project specifics) — genericizing with zero cross-project data points yet would be guessing at a taxonomy. Revisit once a second project produces specs to compare against.
- [x] Build a template repo with a pre-baked `.claude/` setup for new projects
  - Done, but not as a template repo — as a Claude Code plugin (`feature/eng-flow-plugin`, commit `aef1a36`). A first pass (`feature/claude-project-template`, `49bf374`) built the copy/scaffold-script version this item literally describes, then got superseded once `docs/DECISIONS.md`'s "Distribution: plugin, not copy or symlink" comparison showed the plugin route (`.claude-plugin/plugin.json` + `marketplace.json`, installed via `/plugin marketplace add`) gets automatic update propagation (`/plugin update`) with zero custom install-script code, vs. a template repo's silent version drift. Learning: the checklist item's literal shape (a repo to copy) wasn't actually the goal — "new projects get a pre-baked, staying-current `.claude/` setup" was, and Claude Code's own plugin mechanism satisfied that better than the mechanism the item named. Not yet merged to main.

## Claude Code power-user setup
- [x] Configure `settings.json` permission allow/deny lists so trusted commands don't stop every 10s
  - Not an eng-flow feature — plain Claude Code mechanics. `color-app/.claude/settings.local.json` grew to 99 allow entries organically (approve-as-you-go), vs. eng-flow's own 38. Learning: this works but is reactive, not designed — nobody pre-authored a "safe commands for a Vite/React project" list, and a **deny list was never used once, in either repo**. Deep dive on the mechanism itself (precedence, merge semantics, what belongs in committed vs. local scope): `settings-json-deep-dive.md`. Its §6 next step (extract generic entries into a committed `.claude/settings.json` + deny list) is now done for eng-flow — `color-app` still only has the local file.
- [ ] Write hooks (`.claude/hooks.json`): at least one `PreToolUse` guard (e.g. block `rm -rf`, require approval on `git push`) and one `PostToolUse` automation (lint/format/test on write)
  - Zero hooks in either repo — confirmed. eng-flow's `CLAUDE.md` explains why it doesn't use hooks for its core safety mechanism (decision logging: "hooks only fire on tool events, not on 'a decision happened'") and substitutes structural discipline (deterministic call sites inside skills' numbered steps) instead. But that reasoning doesn't cover safety guards or write-time automation — those are still just prose rules in global CLAUDE.md today, which a hook enforces mechanically and prose doesn't. Real gap, not a considered trade-off — highest-leverage unchecked item in this section.
- [x] Learn the memory hierarchy (home → project → nested dirs) and where to put what
  - Two levels correctly modeled and used: `~/.claude/CLAUDE.md` (personal workflow prefs, cross-project) vs. project `CLAUDE.md` (eng-flow process conventions, this repo only). Learning: untested at the third level — no nested-dir `CLAUDE.md` needed yet since color-app is still a single small app, so the hierarchy's harder case (subfolder overrides) hasn't been exercised.
- [x] Get comfortable with `/model`, `/compact`, `--continue`, and the verbose flag
  - Model switching rarely needed — Sonnet is the default, Fable only for deep research. `/compact` never invoked manually; Claude Code's auto-compaction handles it. The real habit that emerged wasn't on this list at all: **`/clear` between every `eng-flow-implement` task**, which turned out to matter more — it's now a hard recommendation inside `eng-flow-implement/SKILL.md:106` itself, because each task reads its own context fresh from `tasks.md`/story/`architecture.md`, so carrying prior tasks' conversation forward is pure cost with no benefit. Learning: per-task context hygiene (clean session boundaries) beat manual context management (`/compact`, `--continue`) for this workflow shape.

## Agent Skills
- [ ] Identify one recurring code-review nag (the thing you correct every time) and write a 3-principle skill for it — not a 30-principle one
- [ ] Follow the 5-section skill anatomy: Purpose, Core Principles, Implementation Patterns, Anti-Patterns, Checklist
- [ ] Confirm skills hot-reload mid-session so iteration is cheap

## Custom slash commands
- [ ] Convert any prompt you've typed 3+ times into a command under `.claude/commands/`
- [ ] Start with `/verify` and `/review`
- [ ] Make commands explicit: steps + structured output (tables/severity levels) + clear success criteria

## MCP (Model Context Protocol)
- [ ] Set up Context7 to kill "hallucinated deprecated API" failures
- [ ] Set up Chrome DevTools MCP for screenshots/console/network/perf-trace debugging
- [ ] Browse the public MCP server catalog before building anything custom (GitHub, Postgres, Playwright, Sentry, etc.)
- [ ] If nothing fits: build a minimal custom MCP server (~90 lines, 2–3 tools) and register it in `settings.json`
- [ ] Write specific tool descriptions — that's how the agent decides which tool to call
- [ ] Apply the security model: local servers keep creds local, read-only DB creds, env vars not committed config, confirm before writes

## Autonomous agent patterns
- [ ] Try the RALPH loop on one real backlog: `prd.json` of small self-contained stories → `ralph.sh` that implements, tests, commits, flips status, logs learnings to `progress.txt`
- [ ] Size stories like PRs ("add a DB column + migration", not "build the dashboard")
- [ ] Use Claude Code Tasks for anything with a dependency chain — let it parallelize independent tasks automatically
- [ ] Give tasks tight, binary acceptance criteria (`npm test` passes) so it can self-verify
- [ ] For any monolith refactor: write a committed 5–6 phase migration plan (analyze → scaffold → implement module-by-module → test → integrate) before touching code
- [ ] Commit at every phase boundary as a rollback checkpoint

## Orchestrator paradigm
- [ ] Internalize the four-pattern spectrum: single agent → subagent → swarm → agent team, and match complexity to the task ("don't set up a three-agent team to build a utility function")
- [ ] Try one subagent delegation: split a feature into data-layer/business-logic/API-layer specialists off a shared `types.ts` contract, assign model tiers (Haiku for read-only, Sonnet for implementation)
- [ ] If parallel work is worth the token cost, try agent teams (env-var gated) in separate worktrees, coordinating via shared task list — not chat
- [ ] Apply the golden rules: shared types file first, clear file ownership (no overlap), coordinate via task list/shared files, test agent goes last

## Production workflows
- [ ] Try one generative-media integration (start with image gen — cheapest/fastest/most mature)
- [ ] Run the three AI-testing workflows: generate tests for existing code, TDD-with-AI (write failing tests first, never let the agent touch tests after), Chrome DevTools MCP as a UX-audit slash command
- [ ] Adopt the AI-augmented code review discipline: AI flags mechanical issues first, you own architecture/business logic, verify every AI finding, use a different model than the code's author for review
- [ ] Use the PR Contract framework: what/why, proof it works, risk tier + what AI generated, 1–2 review-focus areas
- [ ] If wiring AI into CI/CD: AI suggests/humans approve, set token budgets, fail open (never block pipeline on AI review), secrets via GitHub Secrets only
- [ ] Track meaningful metrics (cycle time, bug rate, coverage trend, review turnaround) — stop tracking vanity metrics (LOC/day, % AI-written)
- [ ] Run a periodic retro: what worked, what's worth saving as a template/skill/command

## Decision framework (keep as a standing reference, not a one-time task)
- [ ] Visual/describable task → vibe coding
- [ ] Repeated task → skill or slash command
- [ ] Needs live/external data → MCP
- [ ] Too big for one context window → RALPH or Tasks
- [ ] Cleanly separable concerns → subagents or agent team
- [ ] Needs QA → AI testing workflow
