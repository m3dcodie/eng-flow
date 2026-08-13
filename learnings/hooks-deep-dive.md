# `.claude/hooks` — Deep Dive

Companion to `mastering-ai-assisted-development-checklist.md`'s unchecked item: "Write hooks (`.claude/hooks.json`): at least one `PreToolUse` guard (e.g. block `rm -rf`, require approval on `git push`) and one `PostToolUse` automation (lint/format/test on write)." That item's own annotation already names the real gap — this repo's `CLAUDE.md` explains why hooks *aren't* used for its core safety mechanism (Decision Ledger logging: "hooks only fire on tool events, not on 'a decision happened'"), but that reasoning only rules hooks out for *that one job*. It says nothing about safety guards or write-time automation, which are still just prose rules in global `CLAUDE.md` today. This doc covers what hooks actually are, how Claude Code resolves and runs them, and what belongs in this repo. Sourced from `code.claude.com/docs/en/hooks` (fetched 2026-08-13).

## 1. Why it's needed

`settings.json` permissions (see [[settings-json-deep-dive]]) answer *is Claude allowed to run this*. Hooks answer a different question: *what else should happen, deterministically, when it does*. Two jobs permissions can't do:

- **Conditional blocking** — not "never allow `rm -rf`" (a blunt permission deny) but "block this specific `rm -rf` invocation, explain why, and let Claude retry differently." A permission `deny` is a wall; a `PreToolUse` hook is a checkpoint that can inspect the actual command and decide.
- **Deterministic side effects** — "every time a file is written, run the linter" isn't a permission at all, it's an automation. Nothing in `settings.json`'s permission model can express "and then also do X."

The eng-flow skills already use this second capability, just not through `.claude/hooks` — the Decision Ledger's `eng-flow-decision-log` calls are hand-placed at deterministic call sites *inside* each skill's numbered steps. That's a valid alternative to a hook for anything a skill controls step-by-step. Hooks earn their place for the things skills *don't* control: raw `Bash`/`Edit`/`Write` calls typed in plain conversation, or made by any skill, with no per-call-site discipline to lean on.

## 2. How Claude Code actually runs them

### 2.1 The event catalog is much bigger than "Pre/Post ToolUse"

Three cadences:

| Cadence | Events |
|---|---|
| Once per session | `SessionStart`, `SessionEnd`, `Setup` |
| Once per turn | `UserPromptSubmit`, `UserPromptExpansion`, `Stop`, `StopFailure` |
| Every tool call | `PreToolUse`, `PermissionRequest`, `PermissionDenied`, `PostToolUse`, `PostToolUseFailure`, `PostToolBatch` |

Plus lifecycle events most projects will never touch (`SubagentStart/Stop`, `TaskCreated/Completed`, `PreCompact`/`PostCompact`, `FileChanged`, `WorktreeCreate/Remove`, `ConfigChange`, `Elicitation`, …). For this repo's checklist item, only two matter: `PreToolUse` (the guard) and `PostToolUse` (the automation).

### 2.2 Four scopes, same precedence shape as `settings.json`

Hooks live inside the same settings files permissions do — `~/.claude/settings.json` (personal, all projects), `.claude/settings.json` (committed, team), `.claude/settings.local.json` (gitignored, personal-to-this-repo), managed policy settings (org-wide), plus two scopes permissions don't have: plugin `hooks/hooks.json` (bundled with a plugin) and skill/agent YAML frontmatter (active only while that component runs). Unlike permission rules, hooks don't merge-to-most-restrictive across scopes — each configured hook just runs when its event/matcher fires, from wherever it's defined.

### 2.3 Five handler types, not just shell scripts

| Type | What runs | When it's the right choice |
|---|---|---|
| `command` | A local shell command/script | Default choice — fast, no network dependency, full shell/`jq` access to the JSON payload |
| `http` | POST to a URL | Centralized policy shared across machines/team, or logging to an external system |
| `mcp_tool` | A tool on an already-connected MCP server | Reusing security scanning or validation logic that already lives behind MCP |
| `prompt` | A yes/no question sent to Claude | Judgment calls too fuzzy for a regex (e.g. "does this diff look like it's touching prod config") |
| `agent` | A subagent spawned to verify a condition (experimental) | Multi-step verification before allowing an action — heaviest-weight option |

For this repo's two target hooks (block-`rm -rf`, lint-on-write), `command` is the only type that makes sense — both are pattern-matchable, deterministic, and shouldn't cost a model call or network round-trip.

### 2.4 Input/output contract for `command` hooks

**In:** JSON on stdin. Common fields on every call (`session_id`, `cwd`, `hook_event_name`, `permission_mode`, …) plus event-specific ones — for tool events, `tool_name`, `tool_input` (e.g. `tool_input.command` for `Bash`), `tool_use_id`.

**Out:** exit code + optional JSON on stdout.
- Exit `0` → no opinion, normal flow continues.
- Exit `2` → blocking error, on events that support blocking.
- JSON on stdout with `hookSpecificOutput.permissionDecision` (`"allow"` / `"deny"` / `"prompt"`) + `permissionDecisionReason` is the structured way to block-with-explanation, which is what the docs' own `rm -rf` example uses instead of a bare exit 2.
- `PreToolUse` can also return `updatedInput` to *rewrite* the tool call rather than just allow/deny it — not needed for either of this repo's two target hooks, but worth knowing it exists for later (e.g. auto-injecting a `--dry-run` flag instead of blocking outright).

Only a subset of events can actually block (`PreToolUse`, `UserPromptSubmit`, `Stop`/`SubagentStop`, `PreCompact`, a handful of others). `PostToolUse` is not one of them — it runs *after* the tool already succeeded, so it's automation-only, never a gate. That maps exactly onto the checklist item's own split: `PreToolUse` = guard (can say no), `PostToolUse` = automation (can only react).

### 2.5 Matchers

Filters which hook fires, evaluated against `tool_name` for tool events: `"*"`/omitted matches everything, exact tool names (`Bash`, `Edit|Write`) match exactly, anything else is treated as an unanchored regex (`mcp__.*` for any MCP tool). The `if` field is a second, finer filter *inside* a matched hook — a permission-rule-shaped string like `"Bash(rm *)"` that scopes the hook to only that command shape, so the guard script itself doesn't have to reimplement command parsing for cases the matcher could've excluded already.

### 2.6 Fail-open by design

Two details worth internalizing before writing a guard: hooks fail open if command parsing fails (a broken hook script doesn't accidentally become a hard block on everything), and `disableAllHooks: true` exists as an escape hatch — but only managed-policy settings can disable managed-policy hooks, so a team-mandated guard can't be locally switched off by a project's own settings.

## 3. Priority — what's worth setting up first here

1. **A `PreToolUse` guard on `Bash(rm -rf *)`** — directly closes the gap `CLAUDE.md`'s "Destructive Commands" line only half-covers today (it says to *warn* before such commands, which relies on the model remembering to ask; a hook makes the block mechanical regardless of what the model decides to do). Cheapest possible script: `jq -r '.tool_input.command'`, grep for the pattern, emit the deny JSON.
2. **A `PostToolUse` automation on `Edit|Write`** — this repo is Python-scripted (`.claude/skills/lib/bin/*`) with no lint/format config or test suite discovered yet (see below), so there's currently nothing for a lint-on-write hook to *call*. This is a prerequisite gap, not a hooks gap: adopting `black`/`ruff` (per the global CLAUDE.md fallback — "default to `black .` if no local configuration file exists") has to happen before a `PostToolUse` hook has anything to invoke.
3. Everything else in the event catalog (`SessionStart` banners, `Notification` hooks, MCP-write validators) — real, but speculative for a repo this size; not worth setting up without a concrete pain point driving it, same "reactive vs. pre-authored" caution as §5 of the settings.json deep dive.

## 4. What to include vs. what not to

**Include:**
- A narrowly-matched `PreToolUse` deny for the two or three commands that would actually hurt if run unattended (`rm -rf`, force-push to `main`, `git reset --hard`) — mirrors the `deny` list already in `.claude/settings.json`, but catches shapes a static pattern can't (a permission `deny` on `Bash(rm -rf *)` already exists in this repo's committed settings — a hook only adds value here if it needs to reason about the command beyond what a permission pattern can express, e.g. distinguishing `rm -rf ./tmp` from `rm -rf /`).
- `PostToolUse` automation that's idempotent and fast — formatters, linters, nothing that mutates state in a way a re-run would double-apply.
- Hook scripts checked into the committed `.claude/settings.json` + a real script file (not inlined JSON strings) once the guard is generic enough to be team-safe — same personal-vs-team split as `settings.local.json` vs `settings.json`.

**Never include:**
- Secrets or credentials referenced directly in a hook's `command` string (visible in settings JSON, same rule as §4 of the settings.json deep dive) — use `allowedEnvVars` on `http` hooks or read from an untracked file at runtime instead.
- A hook that blocks on a *fuzzy* judgment call better served by asking the model or the user — that's what `prompt`/`agent` hook types or a plain `AskUserQuestion` are for; a `command` hook that tries to regex its way through "does this look risky" will be wrong in both directions.
- Any hook that assumes it can't fail open — the docs' fail-open default is a feature, not something to defeat with `set -e` and no error handling; a guard script that crashes should not become a silent allow *or* a silent full block, so exit-code discipline (`0` = no opinion, `2` = deliberate block) matters more here than in normal scripting.

## 5. Where this repo actually stands

- **Zero hooks configured** — confirmed via `.claude/settings.json` and `.claude/settings.local.json` (both permissions-only, no `hooks` key).
- The one hook-shaped mechanism that *does* exist is the Decision Ledger (`eng-flow-decision-log`), and it's deliberately *not* implemented as a hook — `CLAUDE.md` states the reason explicitly (event-based hooks can't fire on "a decision happened," only on tool calls), and substitutes structural discipline (deterministic call sites inside each skill's numbered steps) instead. That's a considered trade-off, not a gap.
- The safety-guard and write-time-automation half of the checklist item, by contrast, **is** a real gap: no `rm -rf` guard beyond the existing permission `deny`, and no formatter/linter/test runner discovered in the repo at all — `find .claude/skills -name "*.py" | wc -l` shows 4 Python scripts under `.claude/skills/lib/bin/` with no `pyproject.toml`/`setup.cfg` and nothing under version control governing their style.

## 6. Concrete next step

Not yet done — recommended, in order:

1. Adopt a formatter for the repo's Python scripts first (`black` per the global CLAUDE.md fallback) so a `PostToolUse` lint hook has something to call; committing to a formatter is itself a `stated_preference`-class decision worth a quick confirm, not a silent pick, since it changes every future diff's shape.
2. Add a `PreToolUse` guard for `Bash(rm -rf *)` to committed `.claude/settings.json`, using the docs' own pattern (`jq` parse `tool_input.command`, emit `permissionDecision: deny` with a reason) — this is additive to the existing permission `deny` on the same pattern, not a replacement; keep both, since the permission layer is the cheap first line and the hook is only needed if/when the pattern needs to get smarter than a glob.
3. Add a `PostToolUse` hook on `Edit|Write` running the formatter from step 1, matched broadly at first (`*.py`) and narrowed if it proves noisy.
4. Log the formatter choice and the guard's exact blocked-pattern list to `eng-flow/decisions.jsonl` per this repo's own `CLAUDE.md` convention — both are default-behavior decisions with real (if small) blast radius, not private preference.
