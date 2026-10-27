# Spec — `SessionStart` context hook (D1)

**Status:** T1–T4, T6 and T7 (code) done (2026-10-28). T5 passed: the compaction case in run 2, and resume with drift after the T7 fix
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
| T5 | **Live check, like R8.** In the sandbox with `--plugin-dir`: start `eng-flow-architecture`, run `/compact` mid-Step 4, and confirm the model re-reads the draft and `draft-resume.md`, carries on at the right step, and keeps saving. Repeat with `/resume` of a closed session after editing `spec.md`, and confirm the drift flag. This run also confirms R8's two unconfirmed `draft-resume.md` wording fixes (resume-line hints, no rewriting saved steps). | Finished `architecture.md` matches the draft; no step asked twice; decisions logged | ✅ **Run 1 (2026-10-28, session `54e74033`): compaction at Step 8, inconclusive.** Interactive, `--plugin-dir`, a full rebuild of the sandbox's `architecture.md`. The model went through Steps 0–7 quickly, so `/compact` hit at the Step 8 "does this match?" check, with Steps 0–7 saved (6.6 KB). The hook fired (`SessionStart:compact`, exit 0) and delivered the right block. **The model did not Read the draft or `draft-resume.md`.** It went straight to saving Step 8, then `cp` of `architecture.md` and the ADR from its scratchpad (checked with `cmp`), then `clear`. The output was correct, with no step asked again and the decision logged. It got away with it because it had rendered the full draft to its scratchpad before compacting; the compaction summary named that path and Claude Code re-attached the file. **Takeaway:** the hook text reaches the model but is treated as optional ("If you were…"). Run 1 can't test the case the hook exists for, where the draft JSON is the only full record. Part A (resume + drift) wasn't run. **Next, run 2:** compact during Step 4 (API contracts) after at least one answer, before anything has been rendered, then do part A. If the model ignores the instruction again, make the wording unconditional when a draft was updated recently. The sandbox was reset to its baseline commit `9ee3c63` after run 1.<br>**Run 2 (session `5bf0611d`).** *Compaction at Step 4 passed:* the hook fired, and the model's first action after `continue` was to read the draft JSON. Before Step 5 it pulled Step 4's contracts from the draft, built `architecture.md` from the draft's steps, and saved Steps 5–7. Nothing was asked twice. It didn't read `draft-resume.md`, which didn't matter here. A Step 5 question was dismissed, so the model went on with the recommended NFR and logged it as an assumption, which is correct. No user-only Step 4 answer was planted, so carry-over of one wasn't checked. *Resume after a drift failed:* at the Step 8 confirmation, `/exit`, a `- Must support files up to 200 MB` line added to `spec.md`, then `--continue` and `continue`. The hook fired. The model took `continue` as approval, logged "draft accepted", then saved, wrote `architecture.md` and both ADRs, and cleared the draft in one command. It noticed the drift only afterwards, from that command's `git diff --stat`. It never ran `check`. **Cause:** a resumed session still has the whole conversation, so it has no reason to re-read anything. The drift check only lives in Step 0, which a resume doesn't run again, and the hook didn't mention the drift. **Fix: T7.** |
| T7 | **Drift check in the hook** (from T5 run 2). `eng-flow-draft` gets an importable `drift(inputs, root)` that `check` also uses, and `summaries()` carries each draft's `inputs`. The hook hashes each listed draft's inputs. A draft with changed or missing inputs is listed first, with the files named (`INPUT CHANGED since this draft started: <file> (changed)`). A final line tells the model to flag the change and offer *review* / *discard* before it saves a step or writes a file, and says a bare "continue" or "yes" isn't approval of a step shown before the change. | `list` and `check` output unchanged; T3 checks still pass; new checks for unchanged, changed and missing inputs, ordering, and the 20-line cap with both instruction lines; live: run 2's resume case flags `spec.md` before anything is written | ✅ Code and scripted checks done 2026-10-28: `list` and `check` byte-identical on the T2 and T7 fixtures, 36/36 checks pass, about 27 ms. ✅ Live re-test passed 2026-10-28 (session `731257a8`). Architecture was redone up to Step 2, then `/exit`, the 200 MB line added to `spec.md`, `--continue` and a bare `continue`. The hook listed the draft with `INPUT CHANGED … spec.md (changed)` and the stop line. The model read `draft-resume.md` and the draft, diffed the specs, and ran `eng-flow-draft check`, then saved nothing: the draft stayed at Step 1 with an unchanged timestamp. It asked the user which scale target to use, naming the added line and its conflict with the saved Step 1 answer (about 1 GB streaming), with *discard* as one of the options. |
| T6 | **Docs.** `PROCESS.md` (hooks section), `README.md` (what runs automatically, opt-out), `CHANGELOG.md`; close D1 in `session-mgmgt.md`. | D1 row marked decided with a link here | ✅ 2026-10-28. `PROCESS.md` gets a "Session hook" cross-cutting section, `README.md` an Install note and a Layout entry, and `CHANGELOG.md` an Unreleased entry. D1 was already marked decided in `session-mgmgt.md` when the spec was written. The docs describe what's verified so far; T5 (behavior after compaction) is still open. |
