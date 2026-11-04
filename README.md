# raster

Turn a text phrase into a deterministic grid of geometric blocks, entirely in
the browser. A product of [5deen](https://5deen.com).

Published to GitHub Pages at <https://5deen.github.io/raster/>.

```bash
npm install
npm run dev      # dev server with HMR
npm run build    # production build to dist/
npm run preview  # serve the production build
```

The landing copy is deliberately minimal: the page states only what the demo
provably does, pending raster's own positioning. See `CLAUDE.md` for the
constraints that are not up for revision — determinism, the glyph-table
ceiling, the metadata stripping on download — and for the open decisions.

The pattern generator is [Medigeist](https://github.com/5deen/asc-set-generator),
vendored per block library under `public/medigeist/`. Adding or removing a
library is scripted: see `.claude/skills/`.
