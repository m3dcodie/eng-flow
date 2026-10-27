# Research — session persistence best practices (beyond eng-flow and ECC)

**Status:** D4 (draft/resume) implemented for Stages 1–4 and live-tested (T6, R8); D1 (hooks: §1.1 `SessionStart`, §1.2 `PreCompact`) still open
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

## D4 — shared draft/resume mechanism (in progress, 2026-10-26)

The goal is to save as the work goes, so a crash or restart loses at most the step in progress.

### Decided by the user

- **Format: JSON.** Anthropic found the model is less likely to overwrite or reshape JSON than Markdown. Proposed shape:

  ```json
  {
    "skill": "eng-flow-architecture",
    "topic": "csv export",
    "dated_slug": "2026-08-06-csv-export",
    "last_completed_step": 4,
    "updated": "2026-10-26T14:02:11Z",
    "steps": {
      "1": { "summary": "…", "answers": { "…": "…" } }
    }
  }
  ```

- **Trigger: the end of every numbered step.** A shared helper (`eng-flow-draft save`, under `lib/bin/` next to the analytics scripts) is the last action of each step. It's the same kind of fixed call site as `eng-flow-analytics-checkpoint`, so it always runs. No extra writes inside a step.
- **Resume:** each interview stage's Step 0 looks for a draft and asks "resume from step N?", as `eng-flow-mvp` already does with its checklist.
- **Location: one project-level `eng-flow/.drafts/`**, one file per spec run (`spec--<dated-slug>.json`), not per session. Step 0 lists any drafts there. Stage 1 has no spec folder until Step 7 and Stage 4's backlog isn't tied to a spec, so per-spec folders don't work.
- **Step 0 start:** if a topic was passed in, use it and check for a draft with the same topic. If not, list any drafts plus "Something new". With no drafts, ask "What are we speccing?" straight away.
- **Cleanup:** delete the draft only after the stage's final file saves. An abandoned draft is never deleted automatically. Step 0 offers Resume or Discard.

Full walkthrough of the scenarios (start, crash, resume, finish, abandon, same topic again): `.claude/skills/lib/draft-resume.md` (moved from `eng-flow-spec/` at rollout, 2026-10-27).

### Still open

- None. Git policy decided 2026-10-27: gitignored (T8).

---

## Open decisions (owner: user)

| # | Decision | Reason | Notes |
|---|---|---|---|
| D1 | Should eng-flow ship hooks at all? | `risk` | Carried over from the ECC borrowing proposal (removed from the repo 2026-10-27). Gates 1.1 (`SessionStart` reader) and the rescoped 1.2 (`PreCompact` marker), both described in "What this means for eng-flow" above. |
| D4 | Design of the shared draft/resume mechanism — file location and format, what triggers a write, cleanup after the final save, whether drafts are committed or gitignored | `risk` | In progress, see the D4 section. Format, trigger, location and Step 0 flow decided. Cleanup agreed. Gitignored (T8, 2026-10-27). Rolled out to Stages 2–4 on 2026-10-27 (R1–R7). Shared design in `.claude/skills/lib/draft-resume.md`. |

---

## D4 — implementation tasks (2026-10-26)

Scope: build it for `eng-flow-spec` first, prove it end to end, then roll it out to the other interview stages. Design: `.claude/skills/lib/draft-resume.md`.

| # | Task | Size | Depends on | Done when | Files |
|---|---|---|---|---|---|
| T1 ✅ | **Shared helper `eng-flow-draft`** (done 2026-10-26) with `save`, `list` and `clear`. `save` reads the step's JSON from stdin, because quoting JSON inside a `SKILL.md` shell line is fragile. It creates the draft on the first call, adds the step, updates `last_completed_step` and `updated`, and writes to a temp file and then renames it, so a crash can't leave half a file. Like the other helpers, it never blocks: on error it prints a warning and exits 0. | S | — | Run by hand in a temp dir: `save` ×3 builds the expected JSON; `list` shows topic, last step and age; `clear` removes the file; a bad argument exits 0 with a warning. | `.claude/skills/lib/bin/eng-flow-draft` |
| T2 ✅ | **Spec Step 0, new run.** (done 2026-10-26) Use `$ARGUMENTS` as the topic if given, otherwise ask "What are we speccing?" Move the slug code from Step 7 to Step 0. After the routing answer, call `save` for step 0. | S | T1 | A fresh run with and without a topic creates `eng-flow/.drafts/spec--<dated-slug>.json` after Step 0. | `.claude/skills/eng-flow-spec/SKILL.md` |
| T3 ✅ | **Spec Steps 1–6 save at the end of each step.** (done 2026-10-26) Skipped steps (1, 4, 5) are saved as `skipped` with a reason. | S | T2 | After each step the draft has that step's agreed answers, and skipped steps are marked. | `.claude/skills/eng-flow-spec/SKILL.md` |
| T4 ✅ | **Spec Step 7 cleanup.** (done 2026-10-26) Reuse the slug from Step 0 for the spec folder. Call `clear` only after `spec.md` is written. | XS | T2 | After a full run, `spec.md` exists and the draft is gone. If the write fails, the draft stays. | `.claude/skills/eng-flow-spec/SKILL.md` |
| T5 ✅ | **Spec Step 0, resume.** (done 2026-10-26) Call `list`. With no topic: one question listing the drafts plus "Something new". With a topic: if a draft has that topic, ask whether to resume or start a separate spec. On resume, play back the saved answers for confirm or correct, then continue at `last_completed_step + 1`. Offer Discard (asks before deleting). | M | T1, T3 | Stop after Step 3, start a new session, run the skill: it offers the draft, plays back the answers and continues at Step 4. Discard removes the file. | `.claude/skills/eng-flow-spec/SKILL.md` |
| T6 ✅ | **End-to-end check.** Run a real spec, close the terminal partway through a step, resume in a new session, finish. Also try two drafts at once and the same topic twice. **Scripted part done 2026-10-26:** a harness drove the helper through every `draft-resume.md` scenario (31 checks, all pass). It found three gaps, now fixed: (1) and (2) a resume correction that makes later steps stale (changed routing path, or an answer changed after Step 6 was saved) left those steps counted as finished if the session crashed before they were redone, so a resume could jump to Step 7 and write an outdated `spec.md`; fixed with a new `truncate` subcommand that `SKILL.md` runs before re-saving. (3) A tab or newline in a topic broke `list`'s tab-separated output; now flattened. The trace of `SKILL.md` also found that two drafts with the same topic (after a "separate spec") had no defined Step 0 behaviour; it now lists them by slug, and the no-topic draft list adds the slug to the label when two drafts share a topic. **Live run, 2026-10-26 (sandbox via `--plugin-dir`, three sessions):** fresh start, step-by-step saves with skips, a crash during Step 6, the new session's draft list, resume at Step 6, and the finish (`spec.md` written under the Step 0 slug, draft deleted) all behaved as written. The run changed the resume design at the user's request: picking a draft now resumes at once, with *review* / *discard* on request instead of two more questions; the resumed step's analytics checkpoint runs before the "Resuming…" line. **Logged, not fixed:** (a) the model wrote `spec.md` first and then built Step 6's `spec_markdown` from it, because JSON-escaping a long spec inline is awkward; harmless (a crash in between just redoes Step 6), but `save` could take a file for a field to make the documented order easy; (b) Step 6's "is this accurate?" went out as plain text, and a reply was lost once; an `AskUserQuestion` would be harder to miss; (c) analytics tool-output noise, now T9. Not run live: the same topic twice (the scripted check covers the helper side). | S | T2–T5 | Every scenario in `draft-resume.md` behaves as written. Any gaps are fixed or logged. | `eng-flow-draft`, `SKILL.md`, `draft-resume.md` |
| T7 ✅ | **Docs.** (done 2026-10-26) Add `.drafts/` to PROCESS.md's save conventions, change `draft-resume.md` to "implemented for Stage 1", and add a CHANGELOG entry. | XS | T6 | Docs match the code. | `PROCESS.md`, `draft-resume.md`, `CHANGELOG.md` |
| T8 ✅ | **Git policy for drafts.** (done 2026-10-27) User decided: gitignored. A root `.gitignore` entry in this repo wouldn't reach plugin users' projects, so `eng-flow-draft` writes a self-ignoring `eng-flow/.drafts/.gitignore` (`*`) on first write. | XS | — | In a fresh git repo, `save` creates the draft and `git status --ignored` reports `eng-flow/` as ignored; `list` and `clear` unaffected (checked 2026-10-27). | `eng-flow-draft` |
| T9 | **Tool-output noise from analytics writes.** Each step's analytics checkpoint writes `eng-flow/analytics.jsonl` and `.analytics-tmp/` inside the project, so Claude Code shows a "files changed" block after every checkpoint (seen in the T6 live run). A skill can't hide that view. Option: move the in-flight marker (or the whole log) to `${CLAUDE_PLUGIN_DATA}`; trade-off is Stage 10 and repo readers losing `eng-flow/analytics.jsonl`. | S | **Blocked: user decision** | Decided, then applied or closed. | `eng-flow-analytics-*`, `PROCESS.md` |

**Rollout, after T6 passes** (one task each, same pattern, own file prefix): Stage 2 `eng-flow-domain-model`, Stage 3 `eng-flow-architecture`, Stage 3.5 `eng-flow-eng-review`, Stage 3.6 `eng-flow-ui-design`, Stage 4 `eng-flow-epics-stories-tasks`. Their Step 0 `list` only shows that stage's own drafts.

### Rollout to Stages 2–4 (2026-10-27)

**Decided by the user:**

- **Per-item loops save after each item** (status `in_progress`, which doesn't count toward `last_completed_step`): domain-model Step 8 (findings), eng-review Step 4 (issues), ui-design Step 3 (gate dimensions). Domain-model Step 1 (per domain) does the same, for the same reason.
- **Large content stays inline JSON** (full drafts, ADRs, HTML mockups). No sidecar folder. T6 follow-up (a) stays open.
- **Input drift is detected and flagged:** Step 0 hashes the input files (`inputs`), and a resume runs `check`, names any changed file, suggests *review*, and still resumes.
- **Shared rules live in one file**, `.claude/skills/lib/draft-resume.md`. Each `SKILL.md` keeps its own Step 0 trigger and per-step saves inline.

**Decided by the AI (logged, `ai_inferred`):** downstream drafts are keyed on the spec folder's existing slug, with one draft per stage per spec and no "separate run" option. Picking a spec that has a draft resumes it at once (the T6 resume preference). Stage 4's Step 7 task breakdown gets no draft (audit: "No").

| # | Task | Status |
|---|---|---|
| R1 | Helper: `in_progress` status, `inputs` / `check`, and `list` shows the in-progress step | ✅ scripted check in a temp dir (save/in_progress/done, truncate, changed + missing inputs, usage errors exit 0) |
| R2 | Shared `lib/draft-resume.md`; spec skill points to it | ✅ |
| R3 | `eng-flow-domain-model`: Steps 0–9 save; Steps 1 and 8 save per item; Step 9 clears | ✅ written, not live-tested |
| R4 | `eng-flow-architecture`: Steps 0–8 save (ADRs and the full draft inline); Step 9 writes from the draft and clears | ✅ written, not live-tested |
| R5 | `eng-flow-eng-review`: Steps 0–3 save (subagent findings kept); Step 4 per issue; Step 5 skips edits already applied, then clears | ✅ written, not live-tested |
| R6 | `eng-flow-ui-design`: Steps 0–5 save (design system, wireframes, mockup HTML inline); Step 3 per dimension; Step 6 clears | ✅ written, not live-tested |
| R7 | `eng-flow-epics-stories-tasks`: Steps 0–4 save (rendered backlog files at Step 4); Step 5 skips files this run already wrote, then clears | ✅ written, not live-tested |
| R8 ✅ | **Live check**, as T6: crash mid-loop in eng-review Step 4 and resume, then edit `spec.md` between sessions and confirm the drift flag | Done (2026-10-27, sandbox, `--plugin-dir`, five sessions). Session 1 closed at issue 3/13: steps 0–3 `done` (13 merged issues saved), step 4 `in_progress` with 2 resolved, `last_completed_step` 3, no `architecture.md` edits. Session 2 resumed at issue 3/13 with no question, logged the resume and the drift, and didn't run the reviewer again. **Gap found and fixed:** the resume line noticed the spec change but didn't name `spec.md` or offer *review* / *discard*; the model decided by itself that the change was harmless. `draft-resume.md` now requires the file name and both hints, with an example. **Path check:** Claude Code fills in `${CLAUDE_PLUGIN_ROOT}` in the skill text (commands showed the full path), and the model read `lib/draft-resume.md` on the first try, so there's no need to load it through Bash instead. Seen with `--plugin-dir` only, not a marketplace install. **Second gap, found and fixed:** to shorten the test, the AI trimmed step 3's merged list from 13 to 4 issues, but left steps 1–2's 7 raw findings, so the draft no longer agreed with itself. The resumed session silently re-merged them and re-saved step 3 with 7 issues. That was a sensible repair, but a resume shouldn't rewrite a `done` step without asking. `draft-resume.md` now says saved steps are the record: flag a mismatch, and let the user fix it through *review*. **Finish passed:** `eng-review.md` lists all 7 issues; each accepted resolution is in `architecture.md` once, with no section duplicated; all decisions are logged; the draft is deleted. **Still to confirm in a later live run:** the two `draft-resume.md` wording fixes (resume line hints, no rewriting saved steps), since no session loaded them. |
