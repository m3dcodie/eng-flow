# Spec — `SessionStart` context hook (D1)

**Status:** spec agreed, not implemented (2026-10-28)
**Branch:** `feature/session-start-hook`
**Origin:** D1 in `docs/improvements/session-mgmgt.md` (§1.1 `SessionStart` reader). Compared against ECC's `hooks/memory-persistence/` on 2026-10-28; see "Why not ECC's hooks" below.

---

## Problem

Drafts (D4), `tasks.md` and `checklist.md` already survive a crash or restart, because each skill's Step 0 finds them when the skill is run again. **Mid-skill compaction** is the gap: the session carries on, Step 0 doesn't run again, and the compaction summary may drop details of the draft, or the instruction to keep saving. Nothing tells the model "a draft exists, re-read it."

The same applies to `/resume` of an old session: the transcript comes back, but the files on disk may have moved on since.

## Decided

| Decision | Owner |
|---|---|
| eng-flow ships one plugin hook (D1 = yes) | user (approved the recommended plan, 2026-10-28) |
| Matcher is `compact\|resume` only, not `startup` or `clear`. Step 0 already lists drafts when a stage starts, and `/clear` between implement tasks is meant to start fresh. | user (approved the recommended plan, 2026-10-28) |
| No `PreCompact`, `Stop` or `SessionEnd` hooks. They can't save what's only in the model's head (research §3), and per-step saves already cover it. | user (approved the recommended plan, 2026-10-28) |

## Behavior

A plugin hook, `SessionStart` with matcher `compact|resume`, runs a read-only script and prints a short context block to **stdout**. Claude Code adds a `SessionStart` hook's stdout to the model's context; stderr is not, which is why ECC's `session-start` reaches the model with nothing.

### What it prints

Only when `eng-flow/` exists in the project root (`$CLAUDE_PROJECT_DIR`, falling back to the cwd) **and** there is something in progress. Otherwise it prints nothing.

```
[eng-flow] Work in progress in this project (context was compacted or resumed):
- Draft: architecture--2026-10-26-csv-export.json — stopped after Step 4 (API contracts), Step 5 (NFRs) in progress, 2h ago
- Story: csv-export-download — tasks.md 3/7 done, next: "Add streaming CSV writer"
- MVP: 2026-10-20-invoice-tool — checklist.md 5/9 done
If you were mid-stage: re-read the draft and ${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/draft-resume.md before continuing. Saved steps are the record; keep saving at the end of every step.
```

Sources:

- **Drafts:** every `eng-flow/.drafts/*.json`, via the same parsing as `eng-flow-draft list` (shared code, not a copy).
- **Stories:** `eng-flow/backlog/stories/*/tasks.md` with at least one checked **and** one unchecked box. A story with nothing checked hasn't started, so it isn't shown.
- **MVP:** `eng-flow/mvp/*/checklist.md` with at least one unchecked box.

The final instruction line appears only when at least one draft is listed.

### Limits

- **At most 20 lines.** Newest first within each section; past the cap, print `… and N more`.
- **Under 100 ms** on a typical project. Reads and parses only; never runs git, a subprocess or the network.
- **Writes nothing, calls no model.**
- **Never blocks the session:** on any error, a one-line warning goes to stderr and the script exits 0 with no stdout.
- **Opt-out:** `ENG_FLOW_NO_HOOKS=1` makes it exit 0 straight away.
- Standard-library Python 3 only, like the other `lib/bin` scripts.

### Files

| File | Change |
|---|---|
| `hooks/hooks.json` (plugin root) | New. Registers `SessionStart` / `compact\|resume` → `python3 "${CLAUDE_PLUGIN_ROOT}/.claude/skills/lib/bin/eng-flow-session-context"` |
| `.claude-plugin/plugin.json` | Add `"hooks"` only if the default `hooks/hooks.json` location isn't picked up (check in T1) |
| `.claude/skills/lib/bin/eng-flow-session-context` | New script |
| `.claude/skills/lib/bin/eng-flow-draft` | Move the draft-summary parsing used by `list` into a function the new script can import. `list` output stays exactly the same. |
| `PROCESS.md`, `README.md`, `CHANGELOG.md` | Document the hook, what it prints, and the opt-out |
| `docs/improvements/session-mgmgt.md` | Close D1, link here |

## Out of scope

- A `startup` matcher (could be added later if users miss drafts on session open).
- A `PreCompact` or `PostCompact` marker for analytics.
- Session-end summaries and pattern extraction (overlaps with `eng-flow-retro` and `learnings.md`).
- Loading `learnings.md` entries. Stage 5 already reads them at Step 0; revisit if a live run shows they're lost after compaction.

## Why not ECC's hooks

| ECC hook | Why not |
|---|---|
| `session-start` | Logs to stderr, so the model never sees it, and only points at a file path without loading it |
| `session-end` | Writes an empty template nobody fills in, one file per date shared by every project, and runs on `Stop` (every turn) |
| `pre-compact` | A timestamp marker; saves no state |
| `evaluate-session` | A stderr hint; overlaps with `eng-flow-retro` |

eng-flow's per-step drafts, `tasks.md`, `decisions.jsonl` and per-task commits already are the durable state. The hook only needs to point the model back at them.

---

## Tasks

| # | Task | Acceptance | Status |
|---|---|---|---|
| T1 | **Hook registration.** Add `hooks/hooks.json`; confirm Claude Code loads plugin hooks from there under `--plugin-dir` (if not, add `"hooks"` to `plugin.json`). Start with a stub script that prints one fixed line. | `/compact` in the sandbox shows the stub line in context; a fresh `startup` doesn't | ✅ 2026-10-28, Claude Code 2.1.312, `--plugin-dir`, `claude -p` runs in the sandbox. Transcript shows `hook_success` for `SessionStart:resume` and `SessionStart:compact`, none on startup. The output reaches the model as a system message (`SessionStart:compact hook success: <stdout>`), including in the first request after the compaction summary. `hooks/hooks.json` loads without any `plugin.json` change. **For T5:** Haiku answered "NONE" when asked whether the line was in its context, although the transcript shows it was sent. Getting the model to act on it depends on the wording, so T3's instruction line needs to be direct, and T5 has to check behavior, not just delivery. |
| T2 | **Shared draft parsing.** Refactor `eng-flow-draft` so the summary behind `list` is an importable function. | `eng-flow-draft list` output byte-identical before and after on the R1 temp-dir fixtures | ✅ 2026-10-28. `summaries(d, want="")` returns one dict per readable draft (`file`, `slug`, `topic`, `stopped`, `updated`); `cmd_list` formats them as before. Fixture run (spec ×2, eng-review in-progress, domain-model in-progress with no completed step, a corrupt draft, a non-object draft, a multi-line topic, an unsafe skill, a usage error): `list` output, stderr included, identical before and after; drafts written by both versions identical apart from timestamps. Import works with `want=""` across all stages. **For T3:** load it with `importlib.util.spec_from_loader` and `SourceFileLoader`, not the deprecated `load_module()`. |
| T3 | **`eng-flow-session-context` script.** Drafts, stories and MVP sections; 20-line cap; final instruction line only with drafts; opt-out env var; silent when there's no `eng-flow/` or nothing in progress; exit 0 on every error. | Scripted check in a temp dir covers: no `eng-flow/`, empty `eng-flow/`, one draft, an in-progress step, stories with 0/N, k/N and N/N done, MVP checklist, over 20 items, a corrupt draft JSON, `ENG_FLOW_NO_HOOKS=1`. Runs in under 100 ms on 50 drafts. | ✅ 2026-10-28. 28 checks pass in a temp dir, covering every case listed plus `resume` source, bad stdin, falling back to the stdin `cwd`, ASCII-only output, an unreadable `tasks.md`, and a missing draft helper. Median about 26 ms for 50 drafts, 10 of them 200 KB, including interpreter start. **Beyond the spec, decided by the AI:** (a) a story's progress counts `## Task N:` sections, not checkboxes. A task is done when all its boxes are checked, matching `eng-flow-implement`'s "first unchecked task" rule. A story counts as started once any box is checked, so compaction during its first task still lists it. (b) MVP lines also show the next item. (c) An unreadable story or checklist file is skipped with a warning, like a corrupt draft, so one bad file can't hide the rest. The "no stdout" guard is kept for real failures (helper won't load). (d) Output is ASCII only, so an unusual locale can't make the print fail. (e) The project root is `$CLAUDE_PROJECT_DIR`, then the hook input's `cwd`, then the process cwd. |
| T4 | **Wire up.** Replace the T1 stub with the real script. | `compact` and `resume` both print the block in the sandbox | ✅ 2026-10-28. No stub to replace: `hooks.json` already called this script's path from T1. In the sandbox with `--plugin-dir`, one throwaway architecture draft (Step 2 in progress) and `claude -p` startup → resume → `/compact`: the transcript shows `SessionStart:resume` and `SessionStart:compact` `hook_success` (exit 0, empty stderr), each with the full block (draft line plus re-read instruction), and nothing on startup. The compact block is in the first request after the summary. The `draft-resume.md` path is absolute, filled in from the script's location. The throwaway draft and the helper's `.gitignore` were removed afterwards, so the sandbox is as it was. |
| T5 | **Live check, like R8.** In the sandbox with `--plugin-dir`: start `eng-flow-architecture`, run `/compact` mid-Step 4, and confirm the model re-reads the draft and `draft-resume.md`, carries on at the right step, and keeps saving. Repeat with `/resume` of a closed session after editing `spec.md`, and confirm the drift flag. This run also confirms R8's two unconfirmed `draft-resume.md` wording fixes (resume-line hints, no rewriting saved steps). | Finished `architecture.md` matches the draft; no step asked twice; decisions logged | ⬜ |
| T6 | **Docs.** `PROCESS.md` (hooks section), `README.md` (what runs automatically, opt-out), `CHANGELOG.md`; close D1 in `session-mgmgt.md`. | D1 row marked decided with a link here | ⬜ |
