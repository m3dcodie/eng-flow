# Spec: Plugin reviewer agents

## Context
eng-flow sends independent work to subagents at four places. All four use the built-in `general-purpose` agent with a prompt written inline in the skill. `general-purpose` has every tool, so a reviewer that is meant to be blind and read-only can still Write/Edit the code or docs it is reviewing. Each prompt also exists only inside its own SKILL.md.

## Current State
- `eng-flow-code-review` Step 3.5: an adversarial reviewer (always runs) plus a security reviewer (only when the diff calls for it), sent in parallel, inline prompts, general-purpose.
- `eng-flow-eng-review` Step 3: a blind architecture reviewer, inline prompt, general-purpose.
- `eng-flow-domain-model` Step 7: a blind consistency reviewer, inline prompt, general-purpose. Supports `--review-model`.
- `eng-flow-implement` Step 2.4: on bug-fix tasks, a subagent writes the reproduction test from the bug description alone.
- `plugin.json` declares `skills` and `mcpServers`. There is no `agents/` directory.

## Proposed Change
Add plugin-level agent definitions under `agents/`. Each one gets only the tools its job needs:

| Agent | Used by | Tools |
|---|---|---|
| `eng-flow-adversarial-reviewer` | code-review 3.5 | Read, Grep, Glob, Bash (git diff/log) |
| `eng-flow-security-reviewer` | code-review 3.5 | Read, Grep, Glob, Bash (git diff/log) |
| `eng-flow-doc-reviewer` | eng-review 3, domain-model 7 | Read, Grep, Glob |
| `eng-flow-repro-test-writer` | implement 2.4 | Read, Grep, Glob, Write, Bash (no Edit) |

- No reviewer agent gets Write or Edit.
- The stable part of each prompt moves into the agent file. The skill passes only the run-specific inputs (file paths, diff scope, security-policy path), so blindness is unchanged: the agent never sees the conversation or earlier findings.
- The skills call these agents by `subagent_type` name. Code-review still sends both of its agents in the same turn.
- Model: the agent files leave `model` unset, so they inherit the caller's model. A `--review-model` override is still passed on the call and still wins.
- Fallback: if a named agent isn't available (the plugin agents didn't load), use `general-purpose` with the same prompt inline, log `risk silent_decide ai_default "agent <name> unavailable: general-purpose fallback"`, and keep going. Never skip the review.
- Register `agents/` in `plugin.json` if the plugin format needs it.

## Acceptance Criteria
1. `agents/` holds the four agent files above, each with frontmatter `name`, `description` and `tools` matching the table.
2. None of the reviewer agents (the first three) lists Write or Edit. The repro-test writer lists Write and does not list Edit.
3. The four skill steps call their agent by name. Their inline prompts are gone, except for the fallback.
4. `--review-model` still decides the model where it's supported, and the decision log still records it.
5. The fallback path is written into each of the four steps, with its decision-log call.
6. Installing the plugin fresh makes the four agents show up for the Agent tool.
7. PROCESS.md, README and CHANGELOG mention the agents. `docs/DECISIONS.md` records why we didn't adopt ECC's persona/PROACTIVELY agents.

## Out of Scope
- Moving token-heavy work into subagents (spec `2026-10-29-subagent-context-offload`).
- New review axes or rewording review criteria beyond moving them.
- ECC-style persona agents (planner, architect, tdd-guide) and PROACTIVELY auto-delegation.

## Files Reference
- `.claude/skills/eng-flow-code-review/SKILL.md` (Step 3.5)
- `.claude/skills/eng-flow-eng-review/SKILL.md` (Step 3)
- `.claude/skills/eng-flow-domain-model/SKILL.md` (Step 7)
- `.claude/skills/eng-flow-implement/SKILL.md` (Step 2, item 4)
- `.claude-plugin/plugin.json`, `plugin.json`
- new: `agents/*.md`
