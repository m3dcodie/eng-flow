# Research — session persistence best practices (beyond eng-flow and ECC)

**Status:** research — feeds §1.1 / §1.2 of `2026-10-23-borrow-from-ecc.md`; nothing implemented
**Compiled:** 2026-10-26
**Scope:** how to carry agent state across sessions and context compaction. ECC's `hooks/memory-persistence/` is the starting point; this looks at what else is known.

---

## Bottom line

The pattern that works is **durable state in plain files the agent updates during the work itself**, with hooks only *reading* that state back in. Using hooks to save state at the last moment doesn't work in Claude Code today.

---

## Findings

### 1. Anthropic's long-running-agent harness (primary source)

[Effective harnesses for long-running agents](https://anthropic.com/engineering/effective-harnesses-for-long-running-agents):

- **State files:** a progress log (`claude-progress.txt`), a feature list in JSON, an `init.sh` that starts the app, and a git commit after every change.
- **Session start:** check the working directory, read the git log and progress file, pick the next unfinished item, run `init.sh` plus a smoke test to catch bugs left by earlier sessions.
- **Session end:** leave the code in a mergeable state, commit, update the progress file.
- **JSON over Markdown, on purpose:** the model is less likely to overwrite or reshape JSON.
- **Failure modes seen:** declaring victory too early, leaving broken state when context ran out mid-feature, marking work done without testing it end to end.

### 2. What Claude Code hooks can and can't do

From the [hooks reference](https://code.claude.com/docs/en/hooks.md):

| Event | Matchers / input | Can it change anything? | Handler types |
|---|---|---|---|
| `SessionStart` | `startup`, `resume`, `clear`, `compact`, `fork` | Injects `additionalContext` (or plain stdout) before the first prompt; can also write env vars to `$CLAUDE_ENV_FILE` | `command`, `mcp_tool` (MCP is skipped at launch) |
| `PreCompact` | `manual` / `auto`; gets `custom_instructions` | Can **block** compaction (exit 2 or `decision: block`); can't ask Claude to write anything | `command`, `http`, `mcp_tool` only — no `prompt` / `agent` |
| `PostCompact` | `manual` / `auto`; gets `compact_summary` | No decision control; good for logging the summary | `command`, `http`, `mcp_tool` only |
| `SessionEnd` | `clear`, `resume`, `logout`, `prompt_input_exit`, `other` | Can't block; **1.5 s** default budget (up to 60 s if a hook sets `timeout`) | `command`, `http`, `mcp_tool` only |

Plugins declare hooks in `hooks/hooks.json`, reference scripts via `${CLAUDE_PLUGIN_ROOT}`, and can keep state that survives updates in `${CLAUDE_PLUGIN_DATA}`.

### 3. Saving state at compaction time has been tried and fails

[anthropics/claude-code#43733](https://github.com/anthropics/claude-code/issues/43733): a hand-maintained `SESSION.md`, a `PreCompact` warning, and a `SessionStart(compact)` reminder to re-read the file. The author's verdict was "hit or miss": Claude often wrote the file only *after* compaction, and post-compact injected context isn't guaranteed to be acted on. Requests for a hook that can prompt Claude before compaction (#43733, #36749, #32881, #17237) were closed automatically as stale, not decided on merit. Don't design around one arriving.

### 4. Compaction loses facts quietly

[Context Engineering in 2026](https://www.louisbouchard.ai/context-engineering-2026/) (single study, treat the numbers loosely):

- Recall of planted facts: 92% with full history vs 38% after summarizing, while a blind judge still rated the summarized answers as good.
- Runs with 5–6 compaction events answered roughly one probe in three.
- Cheap deterministic trimming (capping tool output) cut cost 38% with no recall loss. Summarizing added cost.

The lesson: keep requirements and decisions in files and re-read them after compaction; don't trust the summary.

### 5. Other tools and community kits

- **Codex / Cursor / Aider** have converged on `AGENTS.md` for *stable rules*, not in-progress state. Codex's memory feature is opt-in and off by default ([mem0](https://mem0.ai/library/coding-agents/how-memory-works-in-codex-cli)). Aider's history restore is lossy ([Coder docs](https://coder.com/docs/ai-coder/agent-compatibility)).
- **Community kits** ([ECC](https://cdn.jsdelivr.net/npm/ecc-universal@2.0.0/hooks/memory-persistence/README.md), [claude-mem](https://docs.claude-mem.ai/configuration/hooks), [claudecn](https://claudecn.com/en/docs/claude-code/workflows/session-continuity/), [claudefa.st](https://claudefa.st/blog/tools/hooks/session-lifecycle-hooks)) load only the latest summary, cap its size and age (e.g. 7 days), keep `SessionStart` fast, and log a marker at each compaction so gaps can be traced later.

---

## What this means for eng-flow

eng-flow is already well placed: `tasks.md`, `decisions.jsonl`, `learnings.md` and per-task commits are the "progress file plus git" Anthropic recommends.

| Proposal item | Recommendation |
|---|---|
| **1.1 `SessionStart` hook** | **Keep, stronger.** Fast, deterministic Python reader on `startup\|resume\|compact`: active story, next pending task, last commit, size-capped `learnings.md` entries. Writes nothing, calls no model. Opt-out env var. |
| **1.2 `PreCompact` hook** | **Rescope.** It can't save state that only exists in Claude's head. It can record a compaction marker in `eng-flow/`, and a `PostCompact` hook can append `compact_summary` to a log. The real fix is in the skills: `eng-flow-implement` and `eng-flow-mvp` update their state file at every task boundary, so compaction is safe whenever it happens. That matches PROCESS.md's rule that each numbered step is a fixed point that always runs. |
| **Session-end summaries** | **Skip.** 1.5 s budget, no model access, and per-task-boundary writes already cover it. |
| **New: smoke check at session start** | From Anthropic: `eng-flow-implement` Step 0 runs the project's tests before the next task, to catch broken state left by the previous session. |

---

## Two layers of persistence

| Layer | Owner | What persists | Status |
|---|---|---|---|
| **1. Claude Code's session** | Claude Code | The conversation: transcripts (`--continue` / `/resume`), auto-compaction, `CLAUDE.md`, auto memory | Not ours to change; we can only hook into it |
| **2a. eng-flow, between skills** | eng-flow | Each stage's saved file, read by the next stage | Solved — the handoff survives a restart |
| **2b. eng-flow, within a skill run** | eng-flow | Work done so far *inside* a stage | **Gap** — see audit below |

A `SessionStart` hook (§1.1) is the bridge: it reads layer-2 files into a new or compacted layer-1 session.

---

## Audit — does each production stage save as it goes?

Checked against each `SKILL.md` on 2026-10-26: when the stage writes its output, and whether its Step 0 looks for unfinished work to resume.

| Stage | Writes its file at | Lost on a mid-stage restart | Needs persistence? |
|---|---|---|---|
| **1 Spec** | Step 7 of 8 | Whole interview: problem, audience, scope, domains, research | **Yes, high** — long, and every answer is the user's |
| **2 Domain model** | Step 9 of 10 | Domains, data flows, diagram, refinement Q&A, subagent review | **Yes, high** |
| **3 Architecture** | Step 9 of 10 | Tech stack, API, NFRs, deployment, ADRs — each agreed with the user | **Yes, highest** — most decisions per stage |
| **3.5 Eng review** | Step 5 of 6 | Issues already resolved one by one in Step 4 | **Yes, medium** |
| **3.6 UI design** | Step 6 of 7 | Wireframes and design-gate scoring rounds | **Yes, medium–high** |
| **4 Epics/stories** | Step 5 of 6 | Proposed and agreed epics and stories | **Medium** — shorter, but interactive |
| **4 Tasks** (Step 7) | End | Task breakdown | **No** — short, cheap to re-run |
| **5 Implement** | One commit per task | Only the in-flight task; code edits stay on disk | **No** — already persists per task |
| **6 Code review** | Step 7 (end) | Findings | **Low** — mostly automatic; a re-run costs tokens, not user time |
| **7 QA** | Step 5 (end) | Bugs found while browsing | **Medium** — re-exploring takes time |
| **8 Ship** | Git operations | Little; git keeps the state | **Different problem** — see below |
| **9 Retro** | Step 5 (end) | Reflection | **Low** |

Other findings:

- **No stage has a resume check at Step 0** except `eng-flow-mvp`, which resumes from its `checklist.md` checkboxes. `eng-flow-epics-stories-tasks` checks for an existing backlog, but that's for an update pass, not a resume.
- **`decisions.jsonl` and `analytics.jsonl` are already written step by step**, so they survive a crash. After a restart the decision log still shows what was decided; nothing uses it to resume yet.
- **Ship's risk is idempotency, not lost answers.** A crash can leave git half-done (base merged but not pushed, or pushed with no PR). Open question: does re-running `eng-flow-ship` handle "already merged / already pushed / PR already exists" cleanly? Not yet checked.

**Pattern:** the interview stages (1, 2, 3, 3.5, 3.6, 4) are where a restart costs real time, because what's lost is the user's own answers. The execution stages (5, 6, 9) are fine as they are.

---

## Direction agreed — one shared mechanism (2026-10-26)

**Decided by the user:** fix layer 2b with **one shared mechanism** across the interview stages, not a per-stage fix. It doesn't depend on hooks, so D1 doesn't block it.

Starting sketch, to be designed next:

- Each interview stage writes a draft at the end of every numbered step (e.g. `eng-flow/specs/<slug>/.drafts/<stage>.md`).
- Its Step 0 checks for that draft and asks "resume from step N?", the way `eng-flow-mvp` already does with its checklist.

---

## Open decisions (owner: user)

| # | Decision | Reason | Notes |
|---|---|---|---|
| D1 | Should eng-flow ship hooks at all? | `risk` | Carried over from `2026-10-23-borrow-from-ecc.md`. Gates 1.1 and the rescoped 1.2. |
| D4 | Design of the shared draft/resume mechanism — file location and format, what triggers a write, cleanup after the final save, whether drafts are committed or gitignored | `risk` | Next step. |
