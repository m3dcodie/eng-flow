# Spec: Subagent context offload

## Context
Two eng-flow runs fill the main conversation context. In `eng-flow-qa`, every Playwright snapshot and screenshot lands in the main thread. In `eng-flow-mvp`, the auto-continuous loop implements every checklist item in a single context. Moving that work into subagents keeps the main thread small. The catch is that `eng-flow-analytics` only counts tokens from the main session transcript, so subagent work would vanish from the numbers without any warning.

## Current State
- `eng-flow-qa` Steps 3–4: explores each page through `eng-flow-browse` (Playwright MCP) inline and records issues as they are found. If the browser is unavailable, it falls back to guided-manual mode.
- `eng-flow-mvp` Step 4: loops over every unchecked checklist item inline (implement, test, commit, optional push), with hard stops for irreversible or high-risk work. A resume picks up from the checklist checkboxes.
- `lib/bin/eng-flow-analytics-checkpoint`: `transcript_path()` reads `~/.claude/projects/<cwd>/<session>.jsonl`, and `extract_usage()` adds up `message.usage` from that file alone. Subagent usage isn't counted. Where Claude Code stores subagent usage hasn't been checked yet: this machine has no subagent transcripts, only `tool-results/`.

## Proposed Change
1. **QA:** Step 3 runs the exploration for each page (or batch of pages) in a foreground subagent that has the `eng-flow-browse`/Playwright tools. It returns a short, structured list of findings: issue, severity, repro steps, path to evidence. The screenshot files stay on disk. The main thread keeps Step 1 mode, Step 2 orientation, the findings ledger, the report and every question to the user. Guided-manual fallback stays inline.
2. **MVP:** Step 4 runs one foreground subagent per checklist item (implement → test → verify → commit). Items run one after another, never in parallel. The main thread keeps the push decision, the hard-stop checks, checking off items and the Decision Ledger. If a subagent hits a hard stop or needs the user, it returns that to the main thread and doesn't decide it.
3. **Analytics:** checkpoint/finish add subagent token usage to the segment that spawned it, wherever that usage can be found (subagent transcripts, or usage totals in the Agent tool result). If it can't be found, the row records `subagent_tokens: "untracked"` and how many subagent calls ran. The report shows that flag next to the totals. It must never silently undercount.
4. **Opt-out:** the `--inline` argument on both skills restores the current inline behaviour.

## Acceptance Criteria
1. In a QA run, raw snapshot/screenshot output stays out of the main thread. Only the structured findings come back, and `qa-report.md` has the same shape as before.
2. In an MVP run, each checklist item is one subagent call followed by one commit. Push, hard-stop and resume behaviour are unchanged.
3. Subagents never ask the user anything or decide anything the Decision Ledger would log as `open_question`/`must_escalate`. They return it to the main thread.
4. An analytics row for a segment that spawned subagents either includes their tokens or carries `subagent_tokens: "untracked"` plus a call count, and `eng-flow-analytics` shows that flag.
5. A spike checks where subagent usage is stored, and the result is written in `docs/DECISIONS.md`.
6. `--inline` restores the current behaviour on both skills.
7. Guided-manual QA and the "browser unavailable" path work as they do today.

## Out of Scope
- Running MVP items in parallel.
- Moving work into subagents for any other stage.
- Redesigning the analytics report beyond the subagent/untracked flag.
- The reviewer agents themselves (spec `2026-10-29-plugin-reviewer-agents`). This spec can reuse its agents/fallback pattern.

## Files Reference
- `.claude/skills/eng-flow-qa/SKILL.md` (Steps 3–4)
- `.claude/skills/eng-flow-mvp/SKILL.md` (Step 4)
- `.claude/skills/eng-flow-browse/SKILL.md`
- `.claude/skills/lib/bin/eng-flow-analytics-checkpoint`, `eng-flow-analytics-finish`, `eng-flow-analytics-report`
- `PROCESS.md` (Analytics section)
