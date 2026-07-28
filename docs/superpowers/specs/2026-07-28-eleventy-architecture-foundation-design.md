# Architecture Foundation: Migrate to Eleventy Static Site

**Status:** Approved, pending implementation plan
**Date:** 2026-07-28

## Context

The project is a single-page, self-contained Git/GitHub reference guide (`index.html`, ~2,700 lines, no build step, no dependencies). The user wants to grow the project substantially, in three directions:

- **B. Content depth** — more advanced Git topics, deeper GitHub coverage, possibly other VCS tools.
- **C. Interactivity** — an in-browser command sandbox, quizzes, progress tracking.
- **D. Platform features** — accounts, search, community contributions, versioning by Git version.

All three get significantly harder to build well on top of a single unmanageable HTML file. This spec covers only the prerequisite foundation: **A. Architecture** — restructuring the current site into a maintainable, buildable, multi-page site. B, C, and D are separate, later sub-projects, each to be brainstormed and planned independently once this foundation exists.

## Goal

Replace the single `index.html` with an Eleventy-built static site: content authored as plain HTML/JS templates (same authoring style as today, no new format), compiled into multiple real pages with shared layout, styles, and scripts factored out.

**This is a pure restructuring pass** — no visual, content, or behavioral changes. Success is: the site looks and behaves identically to today, but is now organized as maintainable, independently-loadable pages instead of one file.

## Decisions

- **Generator:** Eleventy (11ty). Chosen over Astro (more setup than the current problem — one unmanageable file — requires) and a hand-rolled build script (reinvents dev server / incremental builds / data cascade that Eleventy already provides). Eleventy consumes plain HTML/Nunjucks templates directly, preserving today's authoring style. Revisit Astro later only if the interactivity sub-project (C) needs component islands.
- **Content authoring format:** Stays HTML/JS, not Markdown. The user explicitly wants to keep authoring guide content the way it's written today; only the command-reference data moves into a structured data file (it already is structured data, just currently embedded inline in a `<script>` tag).
- **Navigation model:** Real multi-page navigation — each section gets its own URL and a real page load (not client-side tab-switching / SPA routing). This is what actually solves the "one giant page" scaling problem as content grows in sub-project B.
- **Hosting/deployment:** Out of scope for this pass. Build locally only; hosting pipeline (GitHub Pages, Netlify, etc.) is a follow-up decision made separately.
- **Testing:** No automated test framework introduced. The project has none today and this is a pure restructuring — adding a test suite here would be scope creep. Verification is a manual browser pass (see below).

## Directory Structure

```
Git-Github/
├── package.json                  # eleventy as sole devDependency
├── .eleventy.js                  # Eleventy config (dirs, passthrough copy)
├── src/
│   ├── _includes/
│   │   └── base.html             # shared <head>, header, nav, theme-toggle, footer
│   ├── _data/
│   │   └── commands.js           # command-reference array, moved out of inline JS
│   ├── assets/
│   │   ├── css/styles.css        # extracted from today's single <style> block
│   │   └── js/
│   │       ├── theme.js          # dark/light toggle (shared, all pages)
│   │       ├── commands.js       # search/filter logic (commands page only)
│   │       └── visual-guide.js   # SVG branch-diagram interactions (that page only)
│   ├── index.html                # hero/landing
│   ├── installation.html
│   ├── commands.html
│   ├── visual-guide.html
│   └── guide/
│       ├── index.html            # Beginner's Guide overview
│       ├── first-project.html
│       ├── pull-requests.html
│       └── github-actions.html
├── _site/                        # build output, gitignored
└── README.md
```

Each page pulls in only the JS/CSS it actually needs (e.g. `visual-guide.js` is not shipped on the Installation page). This is the key scaling property as content grows in sub-project B — pages stay independently sized instead of one page's payload growing forever.

## Data Flow

- Every `src/**/*.html` page declares front matter (`layout: base.html`, `title`, `navActive`) and Eleventy wraps its body in `_includes/base.html`.
- The command-reference data (`_data/commands.js`) is available to any page via Eleventy's data cascade. `commands.html` renders the initial list server-side at build time (so the page has real content with JS disabled), and `assets/js/commands.js` handles client-side search/filter on top of it — same behavior as today, just backed by a real data file instead of an inline array in a `<script>` tag.
- The Beginner's Guide, currently three walkthroughs switched via tabs within one section, becomes four real pages (an overview index page plus one page per walkthrough) linked from shared nav.
- The Visual Guide (SVG branch diagram) becomes its own page with its own dedicated interaction script, rather than one section among many sharing a global script scope.

## Build Workflow

- `npx eleventy` → builds `_site/`
- `npx eleventy --serve` → local dev server with live reload

## Verification

No automated tests. Manual browser pass on every generated page, checking:

1. Build completes without errors.
2. Nav links resolve correctly between all pages.
3. Theme toggle (light/dark/system) persists across page loads via `localStorage` and renders correctly on every page.
4. Command reference search/filter behaves identically to today.
5. SVG visual guide interactions (branch diagram) still work.
6. Responsive layout is intact at mobile/tablet/desktop widths.
7. Visual diff check: each page's rendered content matches the corresponding section of the current `index.html` (no accidental content loss during the split).

## Non-Goals (explicitly deferred to later sub-projects)

- No Markdown authoring format.
- No hosting/CI/deployment pipeline.
- No new guide content or topics (that's sub-project B).
- No command sandbox, quizzes, or progress tracking (that's sub-project C).
- No accounts, community contributions, or cross-device sync (that's sub-project D).

## Open Questions

None — all decisions above were confirmed with the user during brainstorming.
