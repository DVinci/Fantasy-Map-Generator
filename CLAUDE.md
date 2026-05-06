# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm install          # Install dependencies (Node.js >= 24.0.0 required)
npm run dev          # Start dev server at http://localhost:5173
npm run build        # TypeScript check (tsc) + Vite production build → dist/
npm run preview      # Preview production build locally
npm run lint         # Biome lint & auto-fix
npm run format       # Biome format & auto-fix
npm run test         # Vitest unit tests (headless)
npm run test:browser # Vitest unit tests (Chromium browser)
npm run test:e2e     # Playwright E2E tests (requires built app on port 4173 in CI)
```

Run a single test file:
```bash
npx vitest run src/utils/commonUtils.test.ts
npx playwright test tests/e2e/burgs.spec.ts
```

## Architecture

The codebase is migrating from vanilla JavaScript to TypeScript. The four-layer architecture (in order of abstraction):

```
State (PackedGraph)         — world data only, no logic or rendering
    ↑
Generators (src/modules/)   — procedural simulation, populates State
    ↑
Editors (src/controllers/)  — user-driven mutations, interactive generators
    ↑
Renderer (src/renderers/)   — pure SVG visualization, never modifies State
```

**Critical invariants:**
- Renderers must be pure: they read State and emit SVG, never mutate world data.
- The State layer (PackedGraph) contains data types and typed arrays, no logic.
- Generators are the only non-interactive path that writes to State.

### Global State

World state lives on `window` (legacy JS compatibility):
- `window.pack` — the `PackedGraph` instance (cells, rivers, burgs, states, cultures, etc.)
- `window.options`, `window.settings` — generation and display settings
- `window.grid` — the unpacked Voronoi grid before packing

Types are declared in [src/types/global.ts](src/types/global.ts) and [src/types/PackedGraph.ts](src/types/PackedGraph.ts).

### Entry Points

- HTML: [src/index.html](src/index.html) — main entry, includes pre-bundled SVG data
- TS bootstrap: [src/utils/index.ts](src/utils/index.ts) — exposes utility globals to `window`
- Module aggregators: `src/modules/index.ts`, `src/controllers/index.ts`, `src/renderers/index.ts`

### Key Dependencies

- `d3` — Voronoi diagrams, scales, geo utilities
- `delaunator` — Delaunay triangulation (foundation for terrain grid)
- `alea` — seeded RNG (reproducible map generation)
- `polylabel` — optimal label placement inside polygons

## Tooling

**Linting & Formatting:** [Biome](https://biomejs.dev/) (`biome.json`)
- Enforces double quotes, 2-space indent, `useTemplate`, `useParseIntRadix`, `noGlobalIsNan`
- `noExplicitAny` and `noNonNullAssertion` are **off** — expected during TS migration
- Run `npm run lint` before committing; CI enforces `biome ci .` on PRs

**Build:** Vite 7 with root `./src`, output `../dist`. Base path is `/Fantasy-Map-Generator/` except on Netlify (`NETLIFY=true` sets `/`).

**Testing:**
- Unit tests: Vitest, files match `src/**/*.{test,spec}.ts`
- E2E tests: Playwright, Chromium only, fixed 1280×720 viewport, files in `tests/e2e/`
- E2E dev server: `http://localhost:5173`; CI preview: `http://localhost:4173`

## TypeScript Migration Notes

New code should be TypeScript. When touching legacy JS files, prefer converting them rather than patching. Use typed arrays (`Float32Array`, `Uint16Array`, etc.) for large datasets that mirror the existing PackedGraph patterns — they are critical for performance at map scale.
