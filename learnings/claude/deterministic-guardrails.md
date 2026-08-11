# Deterministic guardrails: hooks in depth

Expands `claude-md-guide.md` §7. Verified against Claude Code's hooks documentation — corrects one
thing that guide got wrong: hooks are **not** a separate `.claude/hooks.json` file. They live under a
`"hooks"` key inside `.claude/settings.json` (project), `.claude/settings.local.json` (project,
gitignored), or `~/.claude/settings.json` (global). Fixed everywhere else in this folder too.

## 1. The distinction that actually matters: loading vs. compliance

Two different things get called "CLAUDE.md is loaded," and conflating them is where the "just put it
in CLAUDE.md" instinct goes wrong:

- **Loading is deterministic.** The harness reads `CLAUDE.md` at the applicable levels and injects it
  into context every session, mechanically, with no model judgment involved in whether that happens.
  You cannot forget to load it; it's not a choice the model makes.
- **Compliance is not.** Once injected, a CLAUDE.md instruction is just more text in context. The
  model reads it and tries to act on it, but nothing mechanically stops a tool call that contradicts
  it. A long session, a compacted context window, or the model simply misjudging a situation can all
  result in the instruction being present but not followed.

Hooks close exactly that second gap. A `PreToolUse` hook is deterministic at **both** steps: it fires
on every matching tool-call event (no model discretion over whether it runs), and for `PreToolUse`
specifically, it can produce a genuine **hard block** — exit code 2, or JSON output with
`hookSpecificOutput.permissionDecision: "deny"` — that prevents the tool call from executing at all.
This isn't the model "choosing" to comply; the call never happens. That's categorically stronger than
anything a CLAUDE.md line can provide, no matter how emphatically it's worded.

`PostToolUse` is deterministic in *firing* (runs after every matching tool call, unconditionally) but
not in blocking — it's strictly after-the-fact. It can inject additional context or modify recorded
tool output, but the action already happened; there's no rollback. Useful for "make sure X always
runs after Y," not for "prevent Y."

## 1.5. Hooks are not the only forcing mechanism

Corrected from an earlier pass in this session, which undersold this: the `permissions` key in
`.claude/settings.json` (`allow`/`ask`/`deny` lists for tool patterns) is **also genuinely forcing**,
not just a friction-reducer. "Permission rules are enforced by Claude Code, not by the model." A bare
`deny` entry (e.g. `Edit`) removes the tool from the model's context entirely — it never even sees the
tool exists. A scoped deny (`Bash(rm -rf *)`) blocks that specific pattern mechanically at runtime.
**No hook script required.**

The full forcing-power stack, strongest/most primitive first:

| Layer | Forcing? | How |
|---|---|---|
| Sandboxing (OS-level: macOS Seatbelt / Linux seccomp) | Yes — outside the app entirely | Kernel-level filesystem/network isolation on the Bash tool |
| Managed/enterprise settings | Yes — highest authority within the app, non-overridable by any lower level | Same immunity as the managed CLAUDE.md's `claudeMdExcludes`-exemption (`org-managed-claude-md-template.md`) |
| `permissions.deny` (`.claude/settings.json`) | Yes — **the actual ceiling of authority** | Static pattern match → mechanical block, enforced by Claude Code itself, no custom logic. A deny here cannot be bypassed by a hook's `"allow"`, at any tier |
| Hooks (`PreToolUse`/`PostToolUse`) | Yes, but **only additive** | Runs before the permission check and can add a deny beyond what `permissions` specifies; cannot remove one. Silent hook → falls through to `permissions`. Needed only when the block condition requires logic a static pattern can't express |
| CLAUDE.md / `SKILL.md` / system prompt | **No** | Advisory — read and (hopefully) followed, nothing mechanically stops deviation |

**Practical implication for the `git push origin main` gap already flagged**: this doesn't need a hook
at all. A `permissions.deny` entry blocking that Bash pattern is the cheaper, no-code answer sitting
right there. Reach for a hook only when the condition needs real logic — e.g. the loose-dependency-
version check (§4) needs to read the actual diff content, which a static permission pattern can't do,
so that one genuinely needs a hook script.

**Ordering vs. authority — these are different axes, don't conflate them.** Hooks are evaluated first
in execution order, but `permissions.deny` is the higher authority, not the other way around. Verified
directly from the docs: *"Hook decisions don't bypass permission rules... a matching deny rule blocks
the call... This preserves the deny-first precedence... including deny rules set in managed settings."*
Concretely:

- A hook can **add** restriction: it can deny something `permissions` would otherwise have allowed
  (exit 2 / `permissionDecision: "deny"` before the permission list is even consulted).
- A hook **cannot remove** restriction: an explicit `permissionDecision: "allow"` from a hook does
  **not** bypass a `permissions.deny` entry — the deny rule still wins, evaluated regardless of what
  the hook returned. This holds at every tier, including a managed-settings deny overriding a
  project-level hook's allow.
- If the hook stays silent (defers), normal permission-list evaluation applies as if no hook existed.

So the mental model isn't "hooks then permissions" as a priority chain — it's "deny always wins,
hooks are just an earlier opportunity to add a deny, never to remove one." Not confirmed: whether
managed-level hooks take precedence over project/user-level hooks specifically (a hook-vs-hook
question, distinct from the hook-vs-permissions question just answered) — the docs didn't state that
precedence explicitly.

## 2. Hook events (not just PreToolUse/PostToolUse)

| Event | Fires | Typical use |
|---|---|---|
| `SessionStart` | Start of a session (`startup`/`resume`/`clear`/`compact`/`fork` — matcher-selectable) | Inject **live, computed** context every session — current branch, uncommitted-changes count, whether tests currently pass. Unlike CLAUDE.md (static text), this runs a real script and its output is fresh every time. Directly answers "how does the model load things each session" for anything that needs to be current, not just persistent. |
| `SessionEnd` | Session ending | Cleanup, final logging |
| `UserPromptSubmit` | Every user message, before the model sees it | Inspect/augment/reject a prompt before it reaches the model |
| `PreToolUse` | Before a matching tool call | Hard-block dangerous calls (see §3) |
| `PostToolUse` | After a matching tool call succeeds | Guaranteed follow-up action (see §4) |
| `Stop` / `StopFailure` | End of a turn | Detect/react to a turn ending in a particular state |
| `Notification` | `permission_prompt`, `idle_prompt`, `auth_success` | React to Claude Code's own UI events |
| `FileChanged` | A watched literal filename changes | External-edit detection |
| `PermissionRequest` | Referenced in the hooks overview as related to the permission system | Distinct from `PreToolUse`'s deny mechanism; not fully documented in what was fetched — treat as unconfirmed until you're implementing against it directly |

Not guaranteed exhaustive — the docs didn't present this as a complete enumerated table, so treat this
as "the events found," not "the only events that exist."

## 3. PreToolUse — the actual blocking mechanism

- Hook exits with code **2** → blocking error, tool call genuinely does not execute.
- Or hook emits JSON with `hookSpecificOutput.permissionDecision: "deny"` (+ a
  `permissionDecisionReason` string, fed back to Claude as the error it sees).
- Exit code 0 with no denying JSON → normal permission flow continues (Claude Code's usual
  allow/ask/deny prompt behavior applies as if no hook existed).
- **Only `"deny"` is confirmed in what was verified.** An `"ask"` value is plausible (Claude Code's
  permission system has an ask state generally) but wasn't explicitly confirmed in the hooks reference
  fetched for this doc — check the live docs before building against it if you need that specific
  behavior rather than a hard allow/deny.

Matcher syntax: exact tool name (`Bash`, or `Edit|Write` for multiple), or an unanchored JS regex when
the pattern contains special characters — so a hook can match `Bash` calls generally, or specifically
match on the **command string** for tools like `Bash`, not just the tool name. MCP tools match via
`mcp__<server>__<tool>`.

## 4. Candidate deterministic actions — organized by direction

### Block-worthy (PreToolUse, hard deny)

Beyond `git push origin main` / `git push --force` to a protected branch (the example already flagged
in `claude-md-guide.md` §7 and directly mirroring global CLAUDE.md's existing prose rule):

- `git reset --hard`, `git clean -fd`, `git branch -D` — destructive local history/file operations,
  same "warn before destructive commands" intent already stated in global CLAUDE.md, currently
  prose-only.
- Destructive SQL — `DROP TABLE`, `TRUNCATE`, `DELETE`/`UPDATE` without a `WHERE` clause — for any
  project with a real database, a much higher-stakes analog of the git cases.
- `rm -rf` outside an explicitly allowed scratch/temp path — narrower and safer than a blanket warning,
  since it can allow-list the actual scratch directory and deny everywhere else.
- Writing a value that pattern-matches a credential/API-key shape into a file that isn't gitignored —
  catches a secret-into-source mistake before it's ever written, not just before it's committed.
- Adding a new dependency with a loose version range (`^`/`~`) instead of an exact pin — this one maps
  **directly** onto an existing prose-only rule: global CLAUDE.md already says "use precise package
  versions instead of loose tags," and today nothing checks that a `package.json` edit actually
  complies. A hook matching `Edit`/`Write` on `package.json` that greps the diff for `^`/`~` on any
  newly-added line is a very concrete, buildable next step.
- `gh pr merge`, or pushing directly to a release/deploy branch — same shape as the `main`-push case,
  generalized to whatever branch actually triggers deployment in a given project.
- Editing files outside a declared scope for the current task — this is what gstack's own `/freeze`
  skill is aimed at (restrict edits to one directory for a session); worth checking directly whether
  that skill is hook-backed or just an instruction the skill loads, since that distinction is exactly
  what this document is about — not confirmed here either way.

### Always-should-happen (PostToolUse, guaranteed follow-up)

- Run `tsc --noEmit` (or the project's typecheck) after every `Edit`/`Write` to a source file — this is
  the direct next step for color-app specifically: its `CLAUDE.md` already flags "no ESLint/Prettier
  configured" and "don't relax these strict-mode flags" as known, currently prose-only constraints.
- Auto-run the paired test file after an edit to its implementation file — would make eng-flow's
  TDD loop (implement → test → verify → commit, per `eng-flow-implement`) hook-guaranteed instead of
  skill-instruction-guaranteed, closing the same gap `claude-md-guide.md` §8 already named for `/clear`
  discipline: structural enforcement beats hoping the flow gets followed under context pressure.
- Auto-format after write — maps onto global CLAUDE.md's existing fallback rule ("default to
  `prettier --write .` ... if no local configuration exists"), currently prose only.

**Real evidence this pattern already works, from tooling already installed here**: gstack's own
`learn` skill states, for its AskUserQuestion logging, "PostToolUse hook also captures deterministically
when installed" — an already-built example of turning a "hopefully logged" habit into a guaranteed one.

**A useful counter-example, from the same toolset**: gstack's "Continuous Checkpoint Mode" (auto-commit
WIP after each logical unit) reads like it should be a hook, but per its own skill text it's actually
just a model-followed instruction gated by a config flag ("If `CHECKPOINT_MODE` is `continuous`:
auto-commit completed logical units..."). It is **not** hook-backed. Worth sitting with this one
specifically — it's something that sounds automatic and is not, which is exactly the trap this whole
document is about avoiding.

## 5. Priority order for this repo/color-app, if building any of these next

1. `permissions.deny` entry blocking `Bash(git push * main)`/`Bash(git push * --force *)` patterns —
   **not a hook**, a plain `.claude/settings.json` deny rule. Closes the most explicit existing gap
   between a stated CLAUDE.md rule and actual enforcement, with no script to write.
2. `PostToolUse` hook running `tsc --noEmit` after writes in color-app — this one genuinely needs a
   hook (has to run a command and inspect its result, not just pattern-match an invocation), and
   directly backs the strict-mode rule its own `CLAUDE.md` already states but can't enforce on its own.
3. `PreToolUse` hook denying loose-version-range dependency edits — needs a hook, not a permission
   pattern, because it has to read the diff content (is this line adding `^`/`~`?), not just match the
   tool invocation. Backs an existing global CLAUDE.md rule instead of inventing a new one.
4. Everything else in §4's lists — sort each one into "static pattern → `permissions.deny`" or "needs
   real logic → hook" using the same test as #1 vs. #2/#3 above before building it.
