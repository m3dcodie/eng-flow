> **Spec library entry** — copied verbatim from
> `color-app/eng-flow/specs/2026-08-09-mvp-coloring-app-color-once-export-everywhere/spec.md`
> (2026-08-09). Kept as the reference shape for an end-user MVP feature spec: 8 user journeys,
> functional + non-functional requirements, explicit domain boundaries, scope cut, and an Open
> Questions section that later got amended and fed directly into new stories. Produced a full
> downstream chain: `domain-model.md`, `architecture.md`, 6 ADRs, wireframes, mockups. See
> `README.md` in this folder for why this one was kept.

---

# Spec: Coloring App — Color Once, Export Everywhere (MVP)

## Decision
Not yet stakeholder-approved — self-directed idea, validated through office-hours-style exploration (see `specs/requirement.md`) and this spec-intake conversation. No externally imposed business outcome, budget, timeline, or compliance constraint.

## Problem & Audience
**Problem:** Coloring apps are fully commoditized on content/experience (Colorfy, Happy Color, Pigment, etc. all compete there). Finishing a page in an existing app is a dead end — no path to turn it into anything.

**End user:** Not bounded by age or gender — content is organized by interest category (kids, adults, nature, etc.), not an audience label. Closest concrete framing: a "keepsake maker" — someone who colors specifically to produce something lasting (a watch face, a printed page, a growing personal named book), not just for relaxation.

**Status quo:** They color in an existing app (Happy Color, Colorfy, etc.), finish a page, and it just sits there — screenshotted or forgotten. No export/output path exists in any researched competitor.

**Why now:** Direct research this session across 3-4 top coloring apps confirmed real, active demand for the coloring mechanic itself — the audience is real and engaged. The opportunity is a differentiated angle (output, not content) that no researched competitor occupies.

## User Journeys
1. As a keepsake maker, I want to pick from a small curated template library (organized by interest category, not licensed IP) and color it in guided (by-number) or free mode with a hand-curated palette, so I get a finished piece I actually like.
2. As a keepsake maker, I want to export my colored page as a watch-face-sized image, so I can set it as my watch face via my platform's own native "photo as watch face" feature (Apple Watch, Wear OS/Samsung, Huawei Health).
3. As a keepsake maker, I want to export my colored page as a printable PDF (blank/dotted outline or finished), so I can print it myself.
4. As a keepsake maker, I want to apply a B&W or pixel-art filter to a colored page, so I can get a different stylized output from the same work.
5. As a keepsake maker, I want to add colored pages to a personal, named collection book over time, so I build a growing keepsake I'm invested in and don't want to abandon.
6. As a keepsake maker, I want to export my named book as a print-ready file, so I can send it to a print-on-demand service of my choice myself (v1 does not place the order for me).
7. As a keepsake maker, I want a richer color-picking and fill experience than the curated swatch row alone — entering an exact hex code, browsing a wider palette, and giving a region a gradient or textured-brush look — so I can get closer to the finished look I have in mind. (Added 2026-08-10, user-seeded competitor reference: Colorfy's color picker/fill-style set. Still region-based, not free-hand painting — see Domains Touched note.)
8. As a keepsake maker, I want my recently used colors to show up as quick-pick swatches, so I don't have to retype the same hex code every time I want to reuse it. (Added 2026-08-10, deferred — story written, not scheduled yet. Explicitly not a user-designed/saved custom palette — see the story's Explicitly Out of Scope note on why that's a bigger, separately-considered decision.)

## Functional Requirements
- Pre-generated template library: tens of templates (not 200+) at launch, free/general unlicensed categories only (nature, florals, animals, abstract/mandala, etc.), organized by interest, not gated by an audience label.
- Hand-curated, subject-matched color palette per template (not one generic palette reused everywhere).
- Per-template coloring-mode toggle: guided/by-number (region shows a number/hint, taps snap to intended color) vs. free-coloring (any color, any region). Region → intended color/number is static template metadata, no runtime cost.
- Coloring mechanic: SVG paths or canvas flood-fill over static templates.
- Watch-face export: resize/crop the colored page to per-platform presets (Apple, Wear OS/Samsung, Huawei), save to the device photo gallery for the user to set manually. No native watch app, no per-platform SDK.
- Print export: generate a downloadable PDF (blank/dotted outline or finished colored version) from the template asset.
- B&W filter: client-side desaturation.
- Pixel-art filter: client-side downsample + palette-quantize.
- Named collection book: user names a book and adds colored pages to it over time; v1 generates a downloadable, print-ready book-layout file. No live POD API call, no order placed by the app — the user takes the file to a POD service (e.g. Gelato, Lulu xPress) themselves.
- Rich color & fill picker (added 2026-08-10): hex code entry for an exact color; an expanded palette beyond the template's curated swatch row; a per-region fill style — gradient (none/radial/axial) and a textured-brush look (default/pen/crayon/oil) — applied to the tapped region, not free-hand painting (region boundaries stay authoritative, per ADR-0003). Undo/redo for coloring actions.
- Persistence: book and colored-page state stored via local device storage (localStorage/IndexedDB) in the PWA — no accounts, no server-side storage. Data does not survive a cleared browser or a device switch; accepted as a v1 limitation.
- Monetization: one-time purchase, no ads, no subscription (backend decision, not part of the marketing pitch).

## Non-Functional Requirements
- Minimal for v1 — no hard performance/availability targets set; reasonable mobile-web performance expected, standard HTTPS.
- No live third-party API integration in v1 (no POD fulfillment, no payment processing for physical goods) — removes the PII/payment-handling surface that would otherwise apply. Revisit NFRs if a POD API is added in a fast-follow.
- No formal compliance requirements for this hypothesis-stage MVP.

## Domains Touched
- Coloring engine (template rendering, region coloring, guided/free mode)
- Template & palette content (library, categories, curated palettes)
- Export/rendering (watch-face image sizing, PDF generation, book-layout file generation)
- Style filters (B&W, pixel-art)
- Local persistence (book/collection state, no accounts)

## Scope
**MVP cut:** handful-to-tens of templates, both coloring modes, curated palettes, watch-face export, printable PDF export, named collection book (file-generation only, no live POD API), B&W + pixel-art filters, local-storage persistence, no accounts, shipped as a PWA.

**Out of scope (v1):** user accounts/login, merch print (t-shirt/mug/poster — fast-follow), photo-to-coloring-page conversion, watercolor/style-transfer AI filter (or any per-user live AI/inference call), licensed IP content.

## Success Criteria
- Primary signal: real usage of the export and named-book features by people who color a page — not signups, not page views. Specifically: colored-page → export action taken (watch-face and/or print), and pages actually added to a named book across multiple sessions.
- No numeric target set (hypothesis-stage) — the qualitative bar is "people who finish coloring a page actually use the export/book features," not just finish coloring.

## Open Questions
- Is "what you get out of it" a strong enough first-impression hook to overcome a visibly smaller content library? Untested — no landing-page positioning test run as part of this spec.
- Content production plan for the template library (hand-illustrated vs. AI-generated line art) not yet decided.
- Whether "nobody bundles watch-face + print + named-book" holds is unconfirmed — a direct competitor search pass (doc's Next Step 2) has not been run.
- Local-storage-only persistence means the book is lost on cleared browser data / device switch — accepted for MVP validation, but worth revisiting if usage signal is positive and a fast-follow toward lightweight accounts becomes worthwhile.
- Rich color & fill picker (journey 7): confirmed still region-based (fill style applied to the tapped region's SVG path, not free-hand strokes), which keeps it compatible in spirit with ADR-0003 — but gradient/pattern fills and textured-brush looks haven't gone through an architecture pass yet. Stories were drafted directly from this spec addition per explicit user direction (create stories first, run the architecture skill after to validate feasibility/approach) — this inverts the normal Stage 1→2→3→4 order and is flagged in each story's Architecture Notes as pending a Stage 3 pass before implementation.
