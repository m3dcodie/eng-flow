# CLAUDE.md — how the memory hierarchy actually works

Findings from a working session going through eng-flow's and color-app's real `CLAUDE.md` files, verified against Claude Code's official docs (not recalled from memory alone).

## 1. The full hierarchy

| Level | Location | Loaded when | Purpose |
|---|---|---|---|
| Managed policy | `/etc/claude-code/CLAUDE.md` (or managed settings `claudeMd` key) | Always, org-controlled, cannot be excluded | Org-wide standards — every user, no opt-out |
| **User** | `~/.claude/CLAUDE.md` | Always, every session | Personal preferences — you, across *all* your projects |
| **Project** | `./CLAUDE.md` or `./.claude/CLAUDE.md` | Always, when in that repo | Team-shared architecture/standards/workflow — checked into git |
| Local | `./CLAUDE.local.md` | Always, when in that repo | Personal-but-project-specific (sandbox URLs, scratch notes) — gitignored |
| Nested | `subdir/CLAUDE.md` + `CLAUDE.local.md` | Only when Claude reads files in that subdir | Directory-specific guidance, lazy-loaded |
| Rules | `.claude/rules/*.md` | At launch, or path-scoped via YAML frontmatter | Topic-specific slices |

**Correction to a common assumption**: there is no separate "user" level vs. "global" level — `~/.claude/CLAUDE.md` *is* the global file. "User" and "global" name the same file.

## 2. Load order & precedence

All applicable levels are **concatenated into context, not overridden**. Load order (oldest to newest):

1. Managed policy
2. `~/.claude/CLAUDE.md` (user)
3. Project ancestor `CLAUDE.md` files, filesystem root down to working dir
4. Within each dir: `CLAUDE.md` then `CLAUDE.local.md`
5. Subdirectory files, loaded on demand when Claude reads those paths

Because later-loaded content is read last, **more specific levels effectively win on conflict** — project beats user, nested beats project. This is implicit priority through recency, not an explicit override mechanism, so explicitly-conflicting rules should be called out in the more specific file rather than left to silent arbitration.

## 3. `@path/to/file` imports

Works, but doesn't reduce context — imported files expand and load fully at launch, same as if inlined. Relative or absolute paths, max 4 hops of recursion. External imports (e.g. `@~/.claude/...`) trigger an approval dialog on first encounter; user-scope imports don't. Backtick-wrapped `` `@path` `` stays literal (not imported). Imports inside code spans/fenced blocks are skipped.

## 4. Size and lazy-loading

- Target **under ~200 lines** per `CLAUDE.md` — longer files measurably reduce instruction adherence.
- Nested `CLAUDE.md` and path-scoped rules are genuinely lazy (loaded only when Claude touches that directory) — imports are not (always loaded at launch regardless of whether the imported content is ever relevant).
- HTML comments (`<!-- ... -->`) are stripped before injection — free to annotate a template with them, no token cost.
- One line per entry; point to a source-of-truth doc (ADR, `PROCESS.md`) instead of duplicating it, so the CLAUDE.md doesn't drift out of sync with the thing it's describing.

## 5. Real finding: color-app's CLAUDE.md was a copy, not an original

`color-app/CLAUDE.md` was a verbatim copy of `eng-flow/CLAUDE.md` (the framework's own project-level file) — it documented eng-flow's Decision Ledger and skill scope, but said nothing about color-app itself: no stack, no code layout, no testing conventions. Per the table above, that's exactly the content project-level is *for* — it was simply never written. Fixed by adding Stack & architecture / Code layout / Testing & conventions sections, each linking to the project's actual ADRs instead of re-explaining them (see `color-app-claude-md-example.md` in this folder for the result).

## 6. Rubric for evaluating any CLAUDE.md

- [ ] **Persistent rule, not a task** — holds indefinitely; current work belongs in the backlog, not here
- [ ] **Actually behavior-changing** — changes a default choice I'd otherwise make, not just trivia
- [ ] **Correctly scoped to this level** — project facts in project CLAUDE.md, personal prefs in user CLAUDE.md, no duplication across levels
- [ ] **Points to source-of-truth instead of duplicating it** — links to ADRs/process docs rather than re-explaining them
- [ ] **Phrased as a directive, not a description** — "respect X, don't do Y" acts as instruction; "X is true" is easy to skim past
- [ ] **Self-contained** — makes sense to a reader who's never seen the codebase
- [ ] **Doesn't try to guarantee what it can't** — flags things that need to be deterministic; doesn't pretend prose can enforce them (see §7)

## 7. Where deterministic actions actually belong

CLAUDE.md is never deterministic — it's prose loaded into context that the model reads and tries to follow, with nothing mechanically enforcing compliance. eng-flow's own `CLAUDE.md` says as much about itself: its Decision Ledger logging is reliable *inside* a skill (numbered steps are deterministic call sites) but only "self-enforced per turn, not checkpoint-guaranteed" in plain conversation, because hooks — the actually-deterministic mechanism — "only fire on tool events, not on 'a decision happened.'"

The real determinism ladder:

| Mechanism | Determinism | Use for |
|---|---|---|
| **Hook** (`PreToolUse`/`PostToolUse`, configured under the `hooks` key in `.claude/settings.json`) | True — fires on every matching tool event, no model discretion involved | "Must never happen" (block `rm -rf`, gate `git push`) or "must always happen" (run typecheck/lint after every write) |
| **Script/binary** | Deterministic once invoked | The mechanical work itself — reliability depends entirely on what invokes it |
| **Skill step** (numbered step in a `SKILL.md`) | Structurally likely, not guaranteed | Repeatable multi-step procedures where the model is already mid-flow |
| **CLAUDE.md** | Probabilistic | Preferences, context, "prefer X" — never "must" |

**Concrete gap found in this repo**: the global `CLAUDE.md`'s "never push, merge, or force-push directly to `main`" reads like a hard rule but isn't one — no `hooks` are configured in either `eng-flow` or `color-app`'s settings (confirmed by search), so today that rule is exactly as strong as compliance on any given turn. A `PreToolUse` hook blocking `git push origin main` would make it actually deterministic instead of just well-intentioned. Not yet built — flagged as the highest-leverage next step from this review.

**Full depth on this** (hook event list, exact blocking mechanism, a longer candidate list beyond the
`main`-push example, and the loading-vs-compliance distinction for CLAUDE.md itself) — see
`deterministic-guardrails.md` in this same folder.

## 8. Where instructions realistically get missed

Not from file length here (both files reviewed are well under 200 lines) — the real risk is **long sessions**: Claude Code auto-compacts as context fills, and a CLAUDE.md read at session start competes for attention with everything read since. eng-flow's mitigation is structural, not a wording fix: `/clear` between tasks (see `eng-flow-implement/SKILL.md:106`) means CLAUDE.md gets re-read fresh every task instead of drifting further from "recently seen" as one session grows long.
