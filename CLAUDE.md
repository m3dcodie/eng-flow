# eng-flow — project conventions

This repo runs the process documented in `PROCESS.md`, implemented as the `eng-flow-*` skills under `.claude/skills/`. Read `PROCESS.md` before assuming how a stage works — it's the source of truth, this file only covers behavior for work that falls *outside* a formal skill run.

## Decision ownership, outside a skill run

The `eng-flow-*` skills each log every judgment call they make to `eng-flow/decisions.jsonl` (see `PROCESS.md`'s "Decision Ledger" section for the full taxonomy: `reason` — `risk` / `knowledge_asymmetry` / `stated_preference`; `mode` — `silent_decide` / `surface_existing` / `generate_options` / `open_question` / `must_escalate`; `owner` — who actually decided). That discipline should carry into plain conversation in this repo too, not just formal skill runs:

- If a judgment call here carries **knowledge asymmetry** (stakeholder/customer/brand context that can't be inferred from the repo) or **meaningful risk** (costly or hard to reverse), surface it explicitly rather than deciding silently — same taxonomy the skills use.
- If the user says "guide mode" (or similar) for a given request, apply the same heightened-clarification behavior a skill's `--guide` token triggers: ask rather than default, and summarize what was decided by whom before finishing.
- When a notable judgment call happens outside a skill's numbered steps, log it the same way: `python3 .claude/skills/lib/bin/eng-flow-decision-log <context> "<what>" <reason> <mode> <owner> "<description>"` — best-effort, not guaranteed.

**Reliability caveat, stated plainly:** a skill's Decision Ledger logging is reliable because every numbered step is a deterministic call site — the checkpoint always fires. Plain conversation has no such structure, and hooks only fire on tool events, not on "a decision happened." So logging here is self-enforced per turn, not checkpoint-guaranteed. Don't treat `eng-flow/decisions.jsonl` as a complete record if the conversation included work outside a skill.

## Scope

This applies to the `eng-flow-*` skills and conventions in this repo only. Other installed skills (gstack's `/ship`, `/qa`, etc., claude-seo's) are not part of this repo and aren't covered by any of the above.
