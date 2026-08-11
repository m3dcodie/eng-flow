# Worked example: color-app's project-level CLAUDE.md

The template in `project-claude-md-template.md`, filled in for a real project. This is a snapshot copy
of `color-app/CLAUDE.md` as of this review — the live file is the source of truth if it changes later.

What to notice against the template:
- **Stack & architecture** links to 5 different ADRs instead of re-explaining any of them — if an ADR
  changes, this file doesn't need to.
- **"No ESLint/Prettier configured"** is stated as a known gap, not phrased as an invitation to add
  one unasked.
- **Testing & conventions** calls out which strict-mode flags must *not* be relaxed — a directive, not
  trivia.
- The **eng-flow process** section below the app-specific sections is what this file looked like
  *before* this review (see `claude-md-guide.md` §5) — it was a verbatim copy of eng-flow's own
  project CLAUDE.md, describing the framework's process but nothing about color-app itself. Everything
  above it is what was missing and got added.

---

```markdown
# Color App — project conventions

## Stack & architecture

- React 19 + Vite 8 + TypeScript 7 (strict), client-only PWA — no backend service (ADR-0001).
- Persistence: IndexedDB via `idb`, local-only, no accounts (ADR-0004). `navigator.storage.persist()` +
  an Add-to-Home-Screen prompt mitigate iOS Safari storage eviction — a known, accepted residual risk,
  not fully solved.
- Coloring mechanic: SVG per-region fill (ADR-0003), extended with gradient + textured-brush fills
  (ADR-0006).
- Export: Web Share API with download fallback (ADR-0005); PDF/book generation via `pdf-lib`.
- Node **24.19.0** pinned via `.nvmrc` — required, not a suggestion: older majors hit a Vite `rolldown`
  native-binding failure plus a bad `@testing-library/jest-dom` release (see `README.md`).

## Code layout

- `src/components/` — presentational components: one `.tsx` + a co-located plain `.css` (side-effect
  import, e.g. `import './TopBar.css'` — not CSS modules) + a co-located `.test.tsx`, per component.
- `src/screens/` — screen-level composition (e.g. `ColoringScreen`).
- `src/state/` — React Context + hooks for cross-component state (e.g. `ColoredPageContext`). No
  Redux/Zustand — deliberate, per ADR-0002: no backend/auth means no complex app-wide state to
  coordinate.
- `src/utils/`, `src/fixtures/` — pure helpers and static template data.

## Testing & conventions

- Vitest + React Testing Library. `npm test` runs the suite; `npm run lint` is typecheck-only
  (`tsc -b --noEmit`) — **no ESLint/Prettier configured**, so formatting isn't enforced automatically.
- `tsconfig.app.json` strict mode has `noUnusedLocals`, `noUnusedParameters`,
  `noFallthroughCasesInSwitch` all on — respect these, don't relax them to silence errors.
- Full architecture rationale lives in `eng-flow/specs/.../architecture.md` and its `adr/` folder —
  read before assuming a technical constraint, same as `PROCESS.md` for process.

## eng-flow process

This repo runs the process documented in `PROCESS.md`, implemented as the `eng-flow-*` skills under
`.claude/skills/`. Read `PROCESS.md` before assuming how a stage works — it's the source of truth,
this file only covers behavior for work that falls *outside* a formal skill run.

[... Decision ownership / Scope sections, unchanged from eng-flow's own CLAUDE.md — see the live file]
```
