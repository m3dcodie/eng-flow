# Improvement proposal — borrow learning/memory hooks and rules packs from ECC

**Status:** proposed — nothing implemented yet
**Compiled:** 2026-10-23
**Source compared:** [ECC](https://github.com/affaan-m/ECC) v2.2.3 (MIT, cloned locally at `../ECC`), commit `90606c83`
**Scope:** two ECC capabilities only — (1) hooks/memory/continuous learning, (2) rules packs. Everything else in ECC (71 agents, 311 skills, multi-harness adapters, AgentShield) is out of scope here.

---

## Context — how the two repos differ

| | **eng-flow** | **ECC** |
|---|---|---|
| What it is | One opinionated process (`PROCESS.md`) + 14 skills implementing it | A large toolbox: 71 agents, 311 skills, 97 commands, hooks, rules, MCP configs |
| Shape | Linear, gated: Routing → MVP **or** Production Stages 1–10 | À la carte: pick the skills/commands you want |
| Runtime | Small Python helpers in `.claude/skills/lib/bin` | Node.js codebase, test suite, dashboard, Rust subproject |
| Harnesses | Claude Code only | Claude Code first, plus Codex, Kimi, Cursor, Gemini, etc. |

They're complementary: eng-flow is the workflow spine; ECC's strongest *infrastructure* pieces are the two below.

---

## 1. Hooks, memory and continuous learning

### How ECC does it

- **Continuous learning v2** (`skills/continuous-learning-v2/`) — `PreToolUse`/`PostToolUse` hooks fire on every tool call; a background Haiku "observer" agent distills them into atomic **instincts** (one trigger → one action, confidence 0.3–0.9, domain-tagged, project- or global-scoped), stored under `~/.local/share/ecc-homunculus/projects/<hash>/`.
- **`/evolve`** — clusters related instincts and generates new skills / commands / agents from them.
- **`/learn`** — on-demand: extract a reusable pattern from the current session into a new `SKILL.md`, with guarded writes (slug validation, redaction, diff + explicit approval before overwrite).
- **`/skill-create`** — mines git history (commit conventions, co-changed files) into a `SKILL.md`.
- **Session persistence** (`hooks/memory-persistence/`) — `SessionStart` loads bounded prior context (`ECC_SESSION_START_MAX_CHARS`, opt-out via `ECC_SESSION_START_CONTEXT=off`); `PreCompact` snapshots state before compaction; `session-end` writes summaries.
- Hooks are profile-gated (`ECC_HOOK_PROFILE` = minimal / standard / strict, `ECC_DISABLED_HOOKS`).
- Implementation weight: ~15 hook entries in `hooks/hooks.json`, each a large inline `node -e` root-resolver one-liner dispatching into `scripts/hooks/*.js`.

### What eng-flow already has

- Stage 9 (`eng-flow-retro`) appends typed (`pattern` / `pitfall` / `preference` / `architecture` / `tool` / `operational`), confidence-scored entries to `eng-flow/learnings.md`.
- Stage 5 (`eng-flow-implement`) Step 0 reads `learnings.md` before each task.
- Retro also reads `decisions.jsonl` back and can promote decision points to `must_escalate`.

So eng-flow already has the *instinct* concept — it just captures at retro time instead of per tool call, and has a single consumer.

### Proposed — borrow

| # | Item | Rationale |
|---|---|---|
| 1.1 | **`SessionStart` hook** that injects eng-flow state: current stage, active story, next pending task, top-N relevant `learnings.md` entries. Bounded size, opt-out env var (copy ECC's pattern). | Cheap, high value — every new session knows where the work stands. eng-flow currently ships no hooks. |
| 1.2 | **`PreCompact` hook** that snapshots in-flight stage/task state to `eng-flow/` | Long Implement / MVP auto-loops survive context compaction. |
| 1.3 | **`eng-flow-learn` skill** (analogue of `/learn`) — appends a typed entry to `learnings.md` mid-work, not a new skill. Reuse ECC's guarded-write discipline (redaction, approval). | Captures lessons between retros; keeps a single learnings store. |
| 1.4 | **More consumers of `learnings.md`** — `eng-flow-code-review`, `eng-flow-qa`, `eng-flow-architecture` read relevant entries, not just implement. | Same "a log nobody reads is dead weight" principle `PROCESS.md` already states. |
| 1.5 | **Promotion path** (lightweight `/evolve`) — a learning seen repeatedly at high confidence gets promoted into a project rule (see §2). | Closes the loop learnings → standards. |

### Proposed — skip

| Item | Why not |
|---|---|
| Always-on per-tool-call observer + background Haiku agent | Token cost on every tool call, Node runtime dependency, transcripts flowing to a background agent — conflicts with eng-flow's lightweight, project-scoped design. |
| Auto-generating skills / commands / agents (`/evolve --generate`, `/skill-create`) | High noise; eng-flow's value is a small, deliberate skill set. |
| ECC's Node root-resolver hook boilerplate | eng-flow can resolve via `${CLAUDE_PLUGIN_ROOT}` and Python/bash, consistent with `lib/bin`. |

---

## 2. Rules packs

### How ECC does it

- `rules/common/` — 10 files, ~564 lines: `agents`, `code-review`, `coding-style`, `development-workflow`, `git-workflow`, `hooks`, `patterns`, `performance`, `security`, `testing`.
- 21 language/framework packs: `typescript`, `python`, `golang`, `rust`, `java`, `kotlin`, `react`, `vue`, `swift`, `cpp`, `csharp`, etc.
- **Claude Code plugins cannot distribute rules**, so ECC instructs a manual copy: `cp -R rules/common ~/.claude/rules/ecc/` plus one stack pack.

### Proposed fit for eng-flow

| # | Item | Rationale |
|---|---|---|
| 2.1 | **Install at Stage 3** — once `eng-flow-architecture` picks the tech stack, offer to write a common pack + the matching stack pack into the project's `.claude/rules/`. | Stage 3 is where the stack is decided; installing earlier would be guessing. Works around the plugin limitation with a deliberate, gated step. |
| 2.2 | **Enforce at Stage 6** — `eng-flow-code-review` checks the diff against the installed rules. | Gives the rules a consumer. |
| 2.3 | **Grow from retros** — promoted learnings (1.5) land in the same project rules directory. | Rules become project-specific over time, not just generic opinions. |
| 2.4 | **Language packs, starting with Python** — see detail below. | First concrete pack; proves the mechanism while keeping eng-flow language-agnostic. |

### 2.4 Detail — Python rules pack, detected and installed per project

**Why it doesn't break language-agnosticism:** ECC's language rules carry `paths:` frontmatter (e.g. `paths: ["**/*.py", "**/*.pyi"]`), so Claude Code only loads them when working on matching files. eng-flow stays agnostic; the *project* gets a pack only once its stack is known, and the pack costs nothing outside Python files.

**ECC's Python pack:** `rules/python/` — 6 files, ~226 lines: `coding-style` (PEP 8, type annotations, immutability), `patterns`, `security`, `testing`, `hooks`, `fastapi`. Each extends its `rules/common/` counterpart.

**Mechanism:**

1. **Ship packs as plain files** in the plugin, e.g. `.claude/skills/lib/rules/common/` and `.claude/skills/lib/rules/python/`. Plugins can't *register* rules, but they can carry files to copy.
2. **Detect the stack at whichever entry point knows it first:**
   - **Production, greenfield** — Stage 3 (`eng-flow-architecture`), the moment the stack is decided.
   - **Feature/story on existing code** — `eng-flow-spec` already inspects code on that path; detect `pyproject.toml` / `requirements.txt` / `setup.py`.
   - **MVP** — when the one-pager or first checklist item settles the stack, or the same file detection if code already exists.
3. **Offer once, skill does the copy:** "Python project detected — install Python rules into `.claude/rules/eng-flow/`?" On yes, the skill copies; if rules already exist, show the diff before replacing. Log the answer to `decisions.jsonl` (`stated_preference`, owner: user).
4. **Consume at Stage 6** — `eng-flow-code-review` checks the diff against the installed rules.

**Skill copies vs. asking the user to `cp`:** skill copies, after one confirm. It's a reversible, in-repo write visible in `git diff`; manual copy is friction and error-prone (ECC's own docs warn against flattening directories); and it matches how every eng-flow stage already writes its artifacts after a gate.

**Open points:** the common pack has no `paths:` scope and loads every turn — keep it short (ECC's is ~564 lines) or write a leaner eng-flow version. Vendoring vs. writing our own is D2 below.

### Cautions

- Rules are always loaded into context — keep them short; context cost scales with every turn.
- ECC is MIT — copying files verbatim requires retaining its copyright notice. Writing eng-flow's own (ECC as inspiration) avoids that.

---

## Open decisions (owner: user)

Surfaced, not decided — per the Decision Ledger taxonomy in `PROCESS.md`.

| # | Decision | Reason | Notes |
|---|---|---|---|
| D1 | Should eng-flow ship hooks at all? | `risk` | First runtime behavior beyond skills; runs in every project with the plugin installed. Python/bash keeps it consistent with `lib/bin`. |
| D2 | Rules — vendor ECC's files or write our own? | `knowledge_asymmetry` | Licensing attribution vs. how opinionated eng-flow should be. |
| D3 | Rules install scope — project `.claude/rules/` or global `~/.claude/rules/`? | `risk` | Recommendation: project-level, matching eng-flow's "everything lives in the project repo" save convention. |

---

## Suggested order

1. **1.1** `SessionStart` hook and **1.3** `eng-flow-learn` — smallest, highest value (pending D1 for 1.1).
2. **1.4** extra `learnings.md` consumers — skill text changes only, no new runtime.
3. **2.1 / 2.2 / 2.4** rules at Stage 3 / Stage 6, starting with the Python pack (pending D2, D3).
4. **1.2** `PreCompact` and **1.5 / 2.3** promotion — once the above have been used on a real story.
