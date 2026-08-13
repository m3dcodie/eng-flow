# `.claude/settings.json` — Deep Dive

Companion to `mastering-ai-assisted-development-checklist.md`'s "Configure `settings.json`" item, which was checked off on the strength of a `settings.local.json` that grew to 38 (eng-flow) / 99 (color-app) allow entries organically. This doc answers the follow-up: why the mechanism exists, how Claude Code actually resolves it, what belongs in it, and what this repo is still missing. Sourced from `code.claude.com/docs/en/settings` (fetched 2026-08-13) plus this repo's own `.claude/settings.local.json`.

## 1. Why it's needed

Claude Code's default posture is to ask before every tool call that isn't trivially safe (`Read` inside the project is auto-allowed; `Bash`, `Write`, `WebSearch`, MCP calls, etc. are not). That's correct for a first session with a new agent and unbearable for a 39th session in a repo whose command set you already trust. `settings.json` is the escape valve: a declarative allow/deny/ask list so the *repeated* judgment call ("is `git commit` safe here?") gets made once, in a file, instead of once per tool call for the life of the project.

The problem it does *not* solve, and that's worth being explicit about: it's a **permission mechanism**, not a **behavior mechanism**. It controls whether Claude is *allowed* to run something, not *what* Claude decides to do. Conflating the two is the most common misuse — see §4.

## 2. How Claude Code actually uses it

### 2.1 Four scopes, resolved in one merged view

| Scope | File | Shared? | Precedence |
|---|---|---|---|
| Managed | `/etc/claude-code/managed-settings.json` (or MDM) | org-wide | Highest — cannot be overridden |
| CLI | `--settings` flag | session-only | 2nd |
| Local | `.claude/settings.local.json` | no — gitignored, personal | 3rd |
| Project | `.claude/settings.json` | yes — committed, team | 4th |
| User | `~/.claude/settings.json` | no — all projects | Lowest |

For most keys, highest precedence wins outright. **Permission rules are the documented exception: they merge across every scope, and the most restrictive rule always applies.** A user-level `Bash(*)` allow does not override a project-level `deny` on `Bash(curl *)` — the deny still wins. This is the one place in the settings system where "lower scope" doesn't mean "loses"; it means "can only tighten, never loosen, what a higher scope already restricted."

### 2.2 Resolution order for a single tool call

1. Any `deny` match → blocked, full stop, no further checks.
2. Any `allow` match → runs without prompting.
3. Otherwise → falls to `ask` (or the default ask-every-time behavior if no `ask` rule exists either).

Patterns are intentionally not regex — `*` for one segment, `**` for path depth, otherwise literal (`Bash(git commit *)`, `Read(./secrets/**)`). This caps expressiveness on purpose: an allow list you can't audit by eye is worse than a shorter one you can.

### 2.3 What's live vs. what needs a restart

Permission rules, hooks, `env`, and `apiKeyHelper` hot-reload — edit the file mid-session and the next matching tool call sees the change. `model` and `outputStyle` are read once at startup and need `/clear` or a restart. `/status` shows which setting sources actually loaded (a source with zero applicable keys won't appear even if the file exists); `claude doctor` shows the fully merged, resolved view and flags any managed-settings entries that got silently stripped for being malformed.

### 2.4 Trust gate

`.claude/settings.json` (project, committed) only takes effect after the user approves workspace trust for that directory — it's team-shared and could arrive via `git clone`, so it isn't auto-trusted. `.claude/settings.local.json` is exempt from that gate: it's gitignored and user-authored on this machine, so there's nothing to distrust.

## 3. Priority, in the sense that matters day to day

Not "which file wins" (covered above) but "which key is worth your time first." Ranked by leverage-per-line for a project like this one:

1. **`permissions.allow` for the commands you already run every session** — this is the entire point of the checklist item. Highest ROI, zero risk if scoped to read-only/idempotent commands (`git status`, `git diff`, `Bash(git ls-remote *)`, project-local `Read`).
2. **`permissions.deny` for the handful of genuinely dangerous patterns** — `Bash(rm -rf *)`, `Read(./.env)`, `Read(./secrets/**)`, `Write(/root/**)`. Small list, written once, rarely touched again. This is the item both this repo and color-app skipped entirely (see §5).
3. **`env`** — only if a project genuinely needs a variable set every session (e.g. `NODE_ENV`). Don't use it for anything secret-shaped; see §4.
4. **Everything else** (`model`, `outputStyle`, `hooks`, `attribution`, UI toggles) — real, but personal-preference-tier. Belongs in `~/.claude/settings.json` (this user's global CLAUDE.md already carries the equivalent instructions in prose — OS, IDE, communication style — so there's currently no duplication to worry about) or is fine left at defaults.

## 4. What to include vs. what not to

**Include (project-scoped, committed `.claude/settings.json`):**
- Read-only / idempotent commands your team runs constantly: `git status`, `git diff`, `git log`, test runners, linters.
- A short, explicit `deny` list for the commands that would hurt if approved on autopilot: destructive filesystem ops, credential/secret file reads, `curl`/network calls that exfiltrate rather than fetch.
- Team-wide `env` values that aren't secrets (`NODE_ENV`, feature flags).

**Include (personal, gitignored `.claude/settings.local.json`):**
- Anything approved reactively during your own sessions that isn't safe/generic enough for the whole team — this is what both `eng-flow` and `color-app` currently have, entirely.
- Machine-specific paths (`Read(//home/mst/projects/**)` in this repo's own file is a good example — meaningless to a teammate on a different machine).

**Never include, in any scope:**
- API keys, tokens, passwords — use `apiKeyHelper` to inject them dynamically, or `.env` + a `deny` rule on reading it. `settings.json` files, especially the committed one, are not a secrets store and the docs are explicit about this.
- Blanket allows without a deny backstop — `Bash(*)` alone means "never ask about anything," which defeats the mechanism's purpose rather than using it.
- Personal preferences (editor mode, model choice, UI toggles) in the **project**-scoped file — those belong in user scope; putting them in project scope forces them on every teammate who pulls the repo.
- Complex or compound shell expressions as a single rule — keep patterns to one command shape per line; a rule you can't read at a glance is a rule nobody will audit before approving 99 more like it.

## 5. Where this repo actually stands

Checked against §4 and §3:

- **No committed `.claude/settings.json` exists in either `eng-flow` or `color-app`.** Everything so far is in the personal, gitignored `settings.local.json` — confirmed via `git ls-files` (nothing tracked) and `.gitignore` (the local file is explicitly excluded). The checklist's "done" checkmark is honest for *personal* setup but there is no team-shared permission baseline yet.
- **Zero `deny` rules in either project**, despite that being the highest-leverage 10-minute addition per §3.2. Every one of eng-flow's 38 entries and color-app's 99 is an `allow`, and every one arrived reactively (approve-as-you-go), not from a pre-authored "safe commands for this stack" list.
- The existing allow list mixes genuinely generic, safe-to-share rules (`Bash(git status)`-shaped things, if present) with clearly personal/one-off ones (`Bash(find / -maxdepth 8 -ipath "*unicornle/resources/...")`, absolute machine paths like `Read(//home/mst/projects/**)`) — exactly the kind of list that should be split, not promoted wholesale to a project file.

## 6. Concrete next step

Done, 2026-08-13, via the `update-config` skill:

1. Extracted the subset of `eng-flow/.claude/settings.local.json`'s allow entries that are stack-generic and safe for anyone — git basics (`ls-remote`/`remote`/`checkout`/`add`/`commit`/`push`/`pull`), `env`, `jq --version`, `WebSearch`, and three `Skill(...)` invocations (`update-config`, `learn`, `cso`) — into a new, committed `.claude/settings.json`.
2. Added a short `deny` list alongside it: `Read(./.env)`, `Read(./secrets/**)`, `Bash(rm -rf *)`.
3. Left the machine-specific and one-off entries (absolute `Read(//home/mst/**)` paths, one-off `find`/`awk`/`python3 -c` commands, color-app/template `cp`/`mkdir` commands) in `settings.local.json`, untouched.

Logged to `eng-flow/decisions.jsonl` (context `settings-json`) per this repo's `CLAUDE.md` — it's team-visible default behavior, not pure personal preference.
