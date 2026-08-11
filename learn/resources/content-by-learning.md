claude file
Shell commands
Command What it does Example
claude Start interactive mode claude
claude "task" Run a one-time task claude "fix the build error"
claude -p "query" Run one-off query, then exit claude -p "explain this function"
claude -c Continue most recent conversation in current directory claude -c
claude -r Resume a previous conversation claude -r
Session commands
Command What it does Example
/clear Clear conversation history /clear
/help Show available commands /help
/exit or Ctrl+D twice Exit Claude Code /exit

How Claude Code works

The agentic loop
When you run claude in a directory, Claude Code gains access to:
Your project. Files in your directory and subdirectories, plus files elsewhere with your permission.
Your terminal. Any command you could run: build tools, git, package managers, system utilities, scripts. If you can do it from the command line, Claude can too.
Your git state. Current branch, uncommitted changes, and recent commit history.
Your CLAUDE.md. A markdown file where you store project-specific instructions, conventions, and context that Claude should know every session.
Auto memory. Learnings Claude saves automatically as you work, like project patterns and your preferences. The first 200 lines or 25KB of MEMORY.md, whichever comes first, load at the start of each session.
Extensions you configure. MCP servers for external services, skills for workflows, subagents for delegated work, and Claude in Chrome for browser interaction.
Work with sessions
Claude Code saves your conversation locally as you work. Each message, tool use, and result is written to a plaintext JSONL file under ~/.claude/projects/, which enables rewinding, resuming, and forking sessions. Before Claude makes code changes, it also snapshots the affected files so you can revert if needed. For paths, retention, and how to clear this data, see application data in ~/.claude.
Sessions are independent. Each new session starts with a fresh context window, without the conversation history from previous sessions. Claude can persist learnings across sessions using auto memory, and you can add your own persistent instructions in CLAUDE.md.
Since sessions are tied to directories, you can run parallel Claude sessions by using git worktrees, which create separate directories for individual branches.
Resume or fork sessions
Resuming a session with claude --continue or claude --resume reopens it under the same session ID and appends new messages to the existing conversation. Forking with --fork-session or /branch copies the history into a new session ID, leaving the original unchanged.

The context window

anage context with skills and subagents
Beyond compaction, you can use other features to control what loads into context.
Skills load on demand. Claude sees skill descriptions at session start, but the full content only loads when a skill is used. For skills you invoke manually, set disable-model-invocation: true to keep descriptions out of context until you need them. For skills you didn’t write, use skillOverrides to do the same from settings.
Subagents get their own fresh context, completely separate from your main conversation. Their work doesn’t bloat your context. When done, they return a summary. This isolation is why subagents help with long sessions.
See context costs for what each feature costs, and reduce token usage for tips on managing context.

Stay safe with checkpoints and permissions
Claude has two safety mechanisms: checkpoints let you undo file changes, and permissions control what Claude can do without asking.

You can also allow specific commands in .claude/settings.json so Claude doesn’t ask each time. This is useful for trusted commands like npm test or git status. Settings can be scoped from organization-wide policies down to personal preferences. See Permissions for details.

Extend Claude Code
Overview
Extensions plug into different parts of the agentic loop:
CLAUDE.md adds persistent context Claude sees every session
Skills add reusable knowledge and invocable workflows
Code intelligence connects Claude to a language server for symbol-level navigation and live type errors
MCP connects Claude to external services and tools
Subagents run their own loops in isolated context, returning summaries
Agent teams coordinate multiple independent sessions with shared tasks and peer-to-peer messaging
Hooks run your script, HTTP request, prompt, or subagent when Claude Code reaches a lifecycle event
Plugins and marketplaces package and distribute these features

Use Claude Code
How Claude remembers your project

Copy page

Give Claude persistent instructions with CLAUDE.md files, and let Claude accumulate learnings automatically with auto memory.

Each Claude Code session begins with a fresh context window. Two mechanisms carry knowledge across sessions:
CLAUDE.md files: instructions you write to give Claude persistent context
Auto memory: notes Claude writes itself based on your corrections and preferences

CLAUDE.md files Auto memory
Who writes it You Claude
What it contains Instructions and rules Learnings and patterns
Scope Project, user, or org Per repository, shared across worktrees
Loaded into Every session Every session (first 200 lines or 25KB)
Use for Coding standards, workflows, project architecture Build commands, debugging insights, preferences Claude discovers

When to add to CLAUDE.md
Treat CLAUDE.md as the place you write down what you’d otherwise re-explain. Add to it when:
Claude makes the same mistake a second time
A code review catches something Claude should have known about this codebase
You type the same correction or clarification into chat that you typed last session
A new teammate would need the same context to be productive
Keep it to facts Claude should hold in every session: build commands, conventions, project layout, “always do X” rules. If an entry is a multi-step procedure or only matters for one part of the codebase, move it to a skill or a path-scoped rule instead. The extension overview covers when to use each mechanism.

Choose where to put CLAUDE.md files
CLAUDE.md files can live in several locations, each with a different scope. The table below lists them in load order, from broadest scope to most specific, so a project instruction appears in context after a user instruction.
Scope Location Purpose Use case examples Shared with
Managed policy • macOS: /Library/Application Support/ClaudeCode/CLAUDE.md
• Linux and WSL: /etc/claude-code/CLAUDE.md
• Windows: C:\Program Files\ClaudeCode\CLAUDE.md Organization-wide instructions managed by IT/DevOps Company coding standards, security policies, compliance requirements All users in organization
User instructions ~/.claude/CLAUDE.md Personal preferences for all projects Code styling preferences, personal tooling shortcuts Just you (all projects)
Project instructions ./CLAUDE.md or ./.claude/CLAUDE.md Team-shared instructions for the project Project architecture, coding standards, common workflows Team members via source control
Local instructions ./CLAUDE.local.md Personal project-specific preferences; add to .gitignore Your sandbox URLs, preferred test data Just you (current project)

examples

# Project Name: [e.g., Nexus E-Commerce Platform]

## Project Overview

- **Description**: [e.g., A multi-tenant SaaS application managing custom digital storefronts.]
- **Target Audience**: Business owners and localized customer storefronts.
- **Business Goals**: Achieve page loads under 1.2s to reduce cart bounce rates.
- **Key Decisions**: Database reads prioritize cached Redis items over direct Postgres queries.

## Core Tech Stack

- **Languages**: TypeScript (Strict Mode), SQL
- **Frameworks**: Next.js 15 (App Router), Prisma ORM
- **State / Database**: PostgreSQL, Redis (Caching Layer)
- **Styling / Components**: Tailwind CSS, Shadcn UI
- **DO NOT USE**: Redux (use Context API), Axios (use native Fetch API), or inline styles.

## Build, Run, & Test Commands

- **Install Dependencies**: `npm ci`
- **Development Server**: `npm run dev`
- **Build Application**: `npm run build`
- **Run Unit Tests**: `npm run test:unit`
- **Run Integration Tests**: `npm run test:integration`
- **Linting / Formatting**: `npm run lint` / `npm run format`

## Architecture & Codebase Layout

- `src/app/` - Next.js routing pages and layout endpoints.
- `src/components/` - Global, reusable visual atoms and primitives.
- `src/features/` - Feature-scoped folders containing specific hooks, components, and logic.
- `src/lib/` - Shared utility layers, third-party clients, and configurations.
- `prisma/` - Schema structures and migration records.

## Guardrails & Constraints

- **Branch Protection**: Never commit directly to `main` or `production`. Generate feature branches.
- **Secrets Policy**: Never plaintext hardcode keys. Use environment variables via `process.env`.
- **Data Mutation**: Never drop raw production database indexes or columns without separate mock dry-runs.
- **Surgical Scope**: Touch only the scope targeted by the user query. Do not perform accidental sweep cleanups.

## Coding Standards & Preferences

- **Type System**: Explicit types required for all function arguments. Avoid using `any` at all costs.
- **Component Rules**: Prefer React server components (`RSC`) over client components unless interactivity is required.
- **Simplicity**: Write straightforward, minimal code targeting execution over clever abstractions.
- **Documentation**: Exported utilities require explicit JSDoc commenting structures.

## Typical Feature Workflow

1. **Explore**: Scan for architectural conflicts across related routes.
2. **Draft Plan**: Print a raw step outline detailing codebase impact before touching code.
3. **Draft Tests**: Build standard failing testing boundaries for the request where applicable.
4. **Implement**: Perform target surgical adjustments.
5. # **Verify**: Execute verification checks via `npm run test:unit` before closing the interaction.

===
awesome-claude-md CLAUDE.md

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is **awesome-claude-md** - a curated collection of high-quality `CLAUDE.md` files from public GitHub repositories, plus a curated list of tools that generate, sync, or manage `CLAUDE.md` files. The goal is to showcase best practices for using `CLAUDE.md` files to onboard AI assistants to codebases.

## Repository Structure

The repository follows this directory structure:

```
awesome-claude-md/
├── README.md                    # Main landing page with table of contents
├── CLAUDE.md                    # Project guidance for Claude Code
├── .github/
│   └── copilot-instructions.md  # GitHub Copilot instructions
├── docs/                        # GitHub Pages static site
│   ├── _config.yml              # Jekyll configuration
│   ├── _layouts/                # HTML layouts
│   ├── assets/                  # CSS and JavaScript
│   │   ├── css/style.css        # Dark-themed responsive styles
│   │   └── js/main.js           # Search and filter functionality
│   └── index.html               # Main browsable page
└── scenarios/                   # Categorized examples
    ├── [category]/
    │   └── [owner]_[repo]/
    │       └── README.md        # Analysis with links to original files
```

## Core Categories

When adding examples, use these primary categories:

- **complex-projects**: Multi-service projects with detailed architecture
- **libraries-frameworks**: Core concepts, APIs, and usage patterns
- **developer-tooling**: CLI tools with commands and configuration
- **project-handoffs**: Current state with blocking issues and next steps
- **getting-started**: Development environment setup focused
- **infrastructure-projects**: Large-scale systems and runtime environments

## Tools & Ecosystem

The repository also maintains a curated table of tools directly related to `CLAUDE.md` workflows (e.g., tools that generate, sync, or manage `CLAUDE.md` files). These are listed in the `README.md` under the "Tools & Ecosystem" section and are NOT counted as examples. Inclusion criteria: must be directly related to `CLAUDE.md` workflows. License type is shown for reference.

## Repository Maintenance Tasks

### Automated Discovery System

The repository includes an automated discovery system for finding new CLAUDE.md files:

- **GitHub Action**: `.github/workflows/discover-claude-files.yml` runs weekly
- **Discovery Script**: `scripts/discover_claude_files.py` orchestrates the discovery workflow
- **Modular Architecture**: Discovery system is split into focused modules:
  - `scripts/discovery/loader.py`: Loads existing repositories to avoid duplicates
  - `scripts/discovery/searcher.py`: Searches GitHub for CLAUDE.md files
  - `scripts/discovery/evaluator.py`: Evaluates and scores repository candidates
  - `scripts/discovery/reporter.py`: Creates issues and reports
  - `scripts/discovery/reporters/`: Specialized reporter components for formatting
  - `scripts/discovery/utils.py`: Shared utilities (retry logic, logging)
- **Community Review**: Creates issues with ranked candidates for manual review
- **Documentation**: See `AUTOMATED_DISCOVERY.md` for complete details

### Adding New Examples

1. **Automated Path**: Review discovery issues created by the automation system
2. **Manual Search**: Use GitHub search (`filename:CLAUDE.md`) to find examples
3. **Create Directory Structure**: `scenarios/[category]/[owner]_[repo]/`
4. **Write Analysis**: Create `analysis.md` with:
   - Category assignment and rationale
   - Source repository link and original CLAUDE.md link
   - License information and proper attribution
   - Specific features that make it exemplary
   - 2-3 key takeaways for developers

### Ethical Guidelines

- **Never copy** `CLAUDE.md` files directly into this repository
- **Always link** to the original source repository
- **Include attribution** with source links, licensing information, and proper credit
- **Respect copyright** and only reference publicly available files under permissive licenses

### Quality Standards

Our selection prioritizes **content quality and educational value over popularity metrics**:

#### Primary Criteria (70% weight)

1. **Content Depth** - Comprehensive architecture, workflows, and context
2. **Educational Value** - Demonstrates unique patterns and best practices
3. **AI Effectiveness** - Well-structured for AI assistant consumption

#### Secondary Criteria (30% weight)

4. **Project Maturity** - Active maintenance and production usage
5. **Community Recognition** - Industry validation and engagement

#### Scoring Framework

- **100-point scale** emphasizing content quality
- **60+ points required** for inclusion
- **Stars contribute only 10%** of total score
- **No hard star minimums** - quality content from any repository size

#### Selection Process

1. **Automated Discovery** finds candidates using enhanced content analysis
2. **Community Review** evaluates educational value and uniqueness
3. **Manual Curation** ensures alignment with quality standards

### README Maintenance

After adding examples, update main `README.md` with table of contents linking to each `README.md`, organized by category.

## GitHub Pages Static Site

The repository includes a browsable static site at `https://josix.github.io/awesome-claude-md/` for easy example navigation.

### Site Structure

- **Location**: `docs/` folder (served via GitHub Pages)
- **Technology**: Jekyll with custom dark theme
- **Features**: Search, category filters, language filters, responsive design

### Key Files

- `docs/_config.yml`: Jekyll configuration (baseurl, title, theme settings)
- `docs/_layouts/default.html`: Base HTML layout with header, footer, navigation
- `docs/assets/css/style.css`: Dark-themed responsive CSS with CSS variables
- `docs/assets/js/main.js`: Client-side search and filter functionality
- `docs/index.html`: Main page with all examples as filterable cards

### Adding New Examples to the Site

When adding new examples to `scenarios/`, also update `docs/index.html`:

1. Add a new `<div class="example-card">` in the appropriate category section
2. Include required data attributes: `data-category`, `data-language`, `data-title`, `data-repo`, `data-description`
3. Follow the existing card structure with icon, title, description, tags, and links

### Site Features

- **Search**: Real-time filtering by title, repo name, or description (Ctrl+K shortcut)
- **Category Filters**: Filter by 6 categories (complex-projects, developer-tooling, etc.)
- **Language Filters**: Filter by programming language (TypeScript, Python, Rust, Go, Swift, Java)
- **Responsive Design**: Mobile-friendly layout with dark theme
- **Direct Links**: Each card links to both the analysis page and original repository

### Local Development

To preview the site locally:

```bash
cd docs
bundle install  # First time only
bundle exec jekyll serve
# Visit http://localhost:4000/awesome-claude-md/
```

## GitHub Copilot Integration

This repository includes `.github/copilot-instructions.md` for GitHub Copilot users. Both CLAUDE.md and copilot-instructions.md are kept in sync to ensure consistent AI assistant behavior across different tools.

## Search Strategies

### Manual Search Queries

Use these GitHub search queries to find quality examples:

- `filename:CLAUDE.md "## Architecture"`
- `filename:CLAUDE.md "## Development Commands"`
- `"## Testing" filename:CLAUDE.md`
- `"## Deployment" filename:CLAUDE.md`
- `filename:CLAUDE.md language:TypeScript`

### KOL and Expert Organization Search

Target repositories from key opinion leaders and expert organizations:

- `filename:CLAUDE.md user:anthropics` - AI experts and Claude creators
- `filename:CLAUDE.md user:pydantic` - Python validation library experts
- `filename:CLAUDE.md user:microsoft` - Enterprise AI and infrastructure
- `filename:CLAUDE.md user:gaearon` - React co-creator Dan Abramov
- `filename:CLAUDE.md user:openai` - AI research and development
- `filename:CLAUDE.md user:cloudflare` - Infrastructure and runtime systems
- `filename:CLAUDE.md user:pytorch` - Machine learning frameworks

### Domain-Specific Searches

- **Python Ecosystem**: `filename:CLAUDE.md user:fastapi OR user:tiangolo OR user:pydantic`
- **JavaScript/React**: `filename:CLAUDE.md user:vercel OR user:facebook OR user:nextjs`
- **AI/ML**: `filename:CLAUDE.md user:huggingface OR user:langchain-ai`
- **Infrastructure**: `filename:CLAUDE.md user:docker OR user:kubernetes`

### Current Top Examples from Expert Search

Based on embedding-based similarity search for high-quality patterns:

#### Exceptional Quality (Industry Leaders)

- **pydantic/genai-prices**: Expert Python data processing pipeline patterns
- **gaearon/overreacted.io**: React co-creator's advanced Next.js blog architecture
- **anthropics/anthropic-quickstarts**: Official AI development best practices
- **microsoft/semanticworkbench**: Enterprise AI assistant platform

#### High Quality (Established Organizations)

- **openai/openai-agents-python**: Multi-agent workflow framework
- **microsoft/recipe-tool**: Automation recipe patterns
- **blueprintui/blueprintui**: UI component library architecture

## Development Commands

### Code Quality Tools

- `ty check`: Run type checking
- `ruff check .`: Lint entire project
- `ruff format .`: Format code using Ruff
- `complexipy scripts/`: Analyze code complexity
- `ty check && ruff check . && ruff format .`: Combined type checking, linting, and formatting
- `pre-commit run --all-files`: Run all pre-commit hooks on all files
- `pre-commit run`: Run pre-commit hooks on staged files only
- **Remember to fix type errors and linting errors after running ty and ruff**

### Development Workflow

- `uv sync`: Install dependencies
- `pre-commit install`: Install pre-commit hooks (run once after cloning)
- `uv run discover-claude-files`: Run the discovery script
- `pytest`: Run tests
- `pytest --cov`: Run tests with coverage

### File Synchronization

- **Sync CLAUDE.md with copilot-instructions.md**: Keep both AI assistant instruction files synchronized when making changes to project structure, guidelines, or development commands

### Code Analysis

- `complexipy scripts/discover_claude_files.py`: Check complexity of main discovery script
- `complexipy scripts/discovery/`: Analyze complexity of discovery modules
- `complexipy scripts/ --max-complexity 10`: Set custom complexity threshold
- `complexipy scripts/ --output json`: Export complexity analysis as JSON

### Discovery System Architecture

The discovery system follows a modular design with single responsibility principle:

- **Main Script** (`discover_claude_files.py`): 45 lines - lightweight orchestrator
- **Individual Modules**: Each module handles one specific concern (loading, searching, evaluating, reporting)
- **Reduced Complexity**: Complex functions split into smaller, focused components
- **Better Testability**: Each module can be tested independently with 70+ comprehensive tests
- **Maintainability**: Changes to one component don't affect others
- **Clean Test Structure**: Test files mirror module structure in `tests/discovery/`

====
openai-agents-python
/AGENTS.md
Welcome to the OpenAI Agents SDK repository. This file contains the main points for new contributors.

## Repository overview

- **Source code**: `src/agents/` contains the implementation.
- **Tests**: `tests/` with a short guide in `tests/README.md`.
- **Examples**: under `examples/`.
- **Documentation**: markdown pages live in `docs/` with `mkdocs.yml` controlling the site.
- **Utilities**: developer commands are defined in the `Makefile`.
- **PR template**: `.github/PULL_REQUEST_TEMPLATE/pull_request_template.md` describes the information every PR must include.

## Local workflow

1. Format, lint and type‑check your changes:

   ```bash
   make format
   make lint
   make mypy
   ```

2. Run the tests:

   ```bash
   make tests
   ```

   To run a single test, use `uv run pytest -s -k <test_name>`.

3. Build the documentation (optional but recommended for docs changes):

   ```bash
   make build-docs
   ```

   Coverage can be generated with `make coverage`.

All python commands should be run via `uv run python ...`

## Snapshot tests

Some tests rely on inline snapshots. See `tests/README.md` for details on updating them:

```bash
make snapshots-fix      # update existing snapshots
make snapshots-create   # create new snapshots
```

Run `make tests` again after updating snapshots to ensure they pass.

## Style notes

- Write comments as full sentences and end them with a period.

## Pull request expectations

PRs should use the template located at `.github/PULL_REQUEST_TEMPLATE/pull_request_template.md`. Provide a summary, test plan and issue number if applicable, then check that:

- New tests are added when needed.
- Documentation is updated.
- `make lint` and `make format` have been run.
- The full test suite passes.

Commit messages should be concise and written in the imperative mood. Small, focused commits are preferred.

## What reviewers look for

- Tests covering new behaviour.
- Consistent style: code formatted with `uv run ruff format`, imports sorted, and type hints passing `uv run mypy .`.
- Clear documentation for any public API changes.
- Clean history and a helpful PR description.

Global CLAUDE.md Template

# Global Developer Preferences

## About Me

- **Role**: Senior Software Engineer / Full-Stack Developer
- **Primary OS**: macOS / Linux (bash/zsh)
- **Primary IDE**: VS Code / Cursor
- **Preferred Communication**: Technical, direct, and concise. Avoid conversational summaries or fluff.

## System-Wide Behaviors

- **Response Style**: Lead with the solution or direct code blocks immediately.
- **Explanation Layer**: Place short explanations below code blocks, not before them.
- **Error Handling**: When a command fails, evaluate log files before asking me for input.
- **Code Generation**: Always provide complete, copy-pasteable files rather than truncated snippet diffs.

## Workflow & Safety Guardrails

- **Git Automation**: NEVER run `git push` or commit changes without explicitly asking me for confirmation.
- **Destructive Commands**: Warn me before executing commands that overwrite files or delete directories (e.g., `rm -rf`).
- **Dependencies**: Use precise package versions instead of loose tags (e.g., use `^1.2.3` or exact locking).

## Common Fallback Commands

- **Testing**: If no local test framework is discovered, fall back to language defaults (e.g., `npm test`, `pytest`).
- # **Linting/Formatting**: Default to `prettier --write .` or `black .` if no local configuration file exists.
  Organize rules with .claude/rules/
  For larger projects, you can organize instructions into multiple files using the .claude/rules/ directory. This keeps instructions modular and easier for teams to maintain. Rules can also be scoped to specific file paths, so they only load into context when Claude works with matching files, reducing noise and saving context space.

=====
https://code.claude.com/docs/en/memory
Set up rules
Place markdown files in your project’s .claude/rules/ directory. Each file should cover one topic, with a descriptive filename like testing.md or api-design.md. All .md files are discovered recursively, so you can organize rules into subdirectories like frontend/ or backend/:
your-project/
├── .claude/
│ ├── CLAUDE.md # Main project instructions
│ └── rules/
│ ├── code-style.md # Code style guidelines
│ ├── testing.md # Testing conventions
│ └── security.md # Security requirements

Path-specific rules
Rules can be scoped to specific files using YAML frontmatter with the paths field. These conditional rules only apply when Claude is working with files matching the specified patterns.

---

paths:

- "src/api/\*_/_.ts"

---

# API Development Rules

- All API endpoints must include input validation
- Use the standard error response format
- Include OpenAPI documentation comments

User-level rules
Personal rules in ~/.claude/rules/ apply to every project on your machine. Use them for preferences that aren’t project-specific:
~/.claude/rules/
├── preferences.md # Your personal coding preferences
└── workflows.md # Your preferred workflows

https://code.claude.com/docs/en/permission-modes#vs-code
Choose a permission mode

https://code.claude.com/docs/en/sessions

# https://code.claude.com/docs/en/common-workflows#run-parallel-sessions-with-worktrees

- Skills, not CLAUDE.md imports, for niche content-integrity rules. This directly answers the AGENT.md question from earlier — the doc confirms imports load eagerly (bloats every session) and child CLAUDE.md only helps if work stays inside one directory (it doesn't here: football/geo spans src/, matches/, downloads/, spec/). Skills load on demand by description match, which is the actual fix. Good candidates: the Gate 1/2 sign-off checklist, the text-overlay confirm-then-screenshot workflow, the geo/football source-verification process — all "only relevant sometimes" domain workflows, exactly what the doc says to move out of CLAUDE.md.
- A Stop or PostToolUse hook for typecheck/lint/test. CLAUDE.md already says "run typecheck+lint+test before treating any change as done" — that's advisory only right now. A hook makes it deterministic instead of relying on me remembering.
- A verification subagent for the Gate sign-off / content-integrity checks. Your memory already has "never call a Short good without stating pass/fail on every Gate 1/2 item" — that's exactly the adversarial-review pattern the doc describes (fresh context, checks the diff/render against stated criteria, not the reasoning that produced it). Worth a dedicated .claude/agents/gate-reviewer.md.
- CLAUDE.md itself checks out against their Include/Exclude table — no bloat to prune right now.

General, for other projects

- Skills > CLAUDE.md for anything project-specific but situational — this is the biggest one I'd carry forward.
- Give every non-trivial task a verifiable check (test/build/screenshot) rather than trusting "looks done."
- /clear between unrelated tasks, and after 2 failed corrections on the same issue — rather than letting context accumulate failed attempts.
- Subagents for investigation to keep exploration out of main context.

Want me to convert the Gate 1/2 + text-overlay workflow into an actual skill now, or add the typecheck/lint Stop hook? Both are small, contained changes.
