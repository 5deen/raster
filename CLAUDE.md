# CLAUDE.md

Guidance for Claude Code when working in this repository.

## Project

**Raster** is a product of 5deen, published at
<https://5deen.github.io/raster/>. It turns a text phrase into a deterministic
grid of colored geometric shapes.

The name is written **Raster**, capitalized, in site copy; the lowercase form
appears only in the repository name, the package name, the `base` path and the
saved-file prefix. It was questioned once and kept. The objection was real — the
product's central claim is that the SVG *is* the exact document and the PNG only
a picture of it, so naming the product after the lossy half invites the wrong
reading. The copy answers that rather than the name: SVG is the primary download
button and `### Downloads` below says why hash-comparing a PNG does not work.
Reopen it only with a concrete replacement, and note that the name is now in the
repository, the deployed URL, the saved-file prefix and the published copy.

The core flow:

1. A user enters a text phrase.
2. Each character is mapped to a block via the selected library.
3. The site renders a **public** grid of those blocks — the picture is safe to
   display, since it reveals nothing without the mapping rules.
4. Re-entering the phrase re-derives the pattern. If the character sequence
   matches, the regenerated pattern is identical to the original. That identity
   check is the verification.

Say **phrase**, never "password". The word promises storage, reset and recovery,
none of which exist here — there is no account, no server and nothing held
anywhere. Keep it out of the copy and out of these notes.

**Key invariant: generation is deterministic.** The same phrase and the same
library always produce a byte-identical document. Any randomness would have to
be seeded from the phrase, never from `Math.random()`, the clock, or iteration
order of an unordered collection. A timestamp *inside* a document would make one
phrase produce two pictures and destroy the comparison the whole product rests
on.

Site copy is **US English**, and every page carries `lang="en-US"`.

## The landing page copy

Written for raster and **nothing is lifted from geistgrid**. The sections are
`#what-this-is`, `#proof`, `#how-it-works`, `#demo` and `#use-cases`; the hero,
the proof-by-comparison argument, the three steps and the use cases are all
raster's own. `#proof` makes the strongest claim on the page — that matching
pictures prove ownership — and closes by pointing at `#what-this-is`, which
bounds it: a public generator means a guessable phrase can be found from its own
pattern. Keep that qualifier attached to the claim.

## Commands

```bash
npm install      # install dependencies
npm run dev      # Vite dev server with HMR
npm run build    # production build to dist/
npm run preview  # serve the production build locally
```

There is no test runner. If you add one, add the script here.

## Stack

- **Vite 8** — dev server and bundler, ESM (`"type": "module"`).
- **Tailwind CSS 4** — via `@tailwindcss/vite`. Tailwind v4 is configured **in
  CSS**, not in a `tailwind.config.js`: `src/styles/main.css` does
  `@import 'tailwindcss'`, and theme customization belongs in an `@theme { … }`
  block in that file.
- Vanilla JavaScript — no framework. Keep it that way unless asked.

## Deployment and the base path

Published to GitHub Pages at <https://5deen.github.io/raster/> by
`.github/workflows/deploy.yml`, which builds on every push to `main` and uploads
`dist/` as a Pages artifact. The Pages source in repository settings must be
**GitHub Actions**, not "Deploy from a branch".

Because a project repo is served from a subpath rather than a domain root,
`vite.config.js` sets `base: '/raster/'`. Vite rewrites the absolute URLs it
owns — script and stylesheet tags, `<link rel="preload">`, `url()` in CSS — but
**not** plain `href` attributes on anchors. So every in-site link must be
written **relative**: `href="./"`, `href="./privacy.html"`, `href="#demo"`, and
`href="./#demo"` from a sub-page. A leading `/` silently leaves the project site
and lands on `5deen.github.io`.

`base` tracks the **repository name**, not the product name. They agree today.
Rename the repo and this is the one line to change, or every asset 404s.

Hosting is **settled on GitHub Pages**, with no custom domain: there is no
`public/CNAME`, and `5deen.com` is the contact domain only. This is what keeps
section 4 of `privacy.html` true, which names GitHub, Inc. and its address as
the host and processor and declares the transfer to the United States. That
section is a factual statement about a processor, not boilerplate. Moving the
site to Vercel, or putting a custom domain in front, means changing the
workflow, the `base` path and that section of the privacy policy in one change.

If raster later moves to its own domain at a root, `base` becomes `/`, a
`public/CNAME` is needed, and the relative-link discipline above stops being
load-bearing — but nothing breaks by keeping it.

**Each page is a separate Vite entry.** Vite discovers only `index.html` on its
own, so any new page must be added to `build.rollupOptions.input` in
`vite.config.js` or it is silently missing from `dist/`.

## The pattern generator

The demo runs [Medigeist](https://github.com/5deen/asc-set-generator), an
AssemblyScript program compiled to WebAssembly. `src/medigeist.js` wraps it;
`createSVGDocument(ratio, set, text)` returns a complete, self-contained SVG.

**One build per block library.** `LIBRARIES` in `medigeist.js` is the single
list everything is built from: adding a build means adding an entry there and
nothing else. The **Image set** picker is generated from it and hides itself
while there is only one library, since a select with one option is a control
that cannot do anything. The first entry is the default.

Adding and removing a library are written up as the `add-library` and
`remove-library` skills in `.claude/skills/`. They exist because the work spans
four places — the build, `public/medigeist/<id>/`, `medigeist-src/` and
`LIBRARIES` — and missing one leaves either a picker entry that 404s or a
megabyte of dead weight. Use them rather than doing it by hand.

### The libraries

The first five are inherited from geistgrid. Each is named after the prefix its
block keys share; keys run `<id>_001` upwards.

| Library | Blocks | Build | What it looks like |
| --- | --- | --- | --- |
| `cascading_orientation_pattern_v3.12` | 193 | ~1.7 MB | orange, per-block canvas fill |
| `cascading_orientation_pattern_v3.13` | 102 | ~1.3 MB | black line art on white |
| `cascading_orientation_pattern_v3.14` | 128 | ~1.6 MB | four-quadrant split, sage/brown |
| `cascading_orientation_pattern_v3.15` | 102 | ~1.3 MB | lavender/navy half-split, per-block rotation |
| `cascading_orientation_pattern_v2.11` | 193 | ~1.7 MB | blue; recolored, see below |
| `cascading_square_pattern_v3.16` | 102 | ~3.3 MB | one black cell in a 10×10 grid; **does not discriminate**, see below |

**The shipping set is settled at these six, 10.7 MB vendored**, and that
includes `v3.16`. Only the selected library is ever downloaded, so the cost is
repository size rather than page weight, and growing the set is one
`add-library` pass.

`v3.16` is the first library outside the `cascading_orientation_pattern` family
and the only one that does not meet the bar in **Judging a new library** below.
It was added on request, measured, and **kept on an explicit decision** with the
figures recorded rather than buried. Do not drop it as a tidy-up: removing it is
a product decision, the same as adding it was. What the measurement means for
the page is the part to keep in view — the picker offers a set that a reader
cannot read a changed character out of, so do not cite `v3.16` in copy about
patterns being visibly different, and do not make it the default.

A library id can carry a **dot**, as all six do. AssemblyScript identifiers and
module filenames cannot, so the map binding and the file under `medigeist-src/`
use `_` where the id uses `.` (`cascading_orientation_pattern_v3_12`). The id
with the dot is what the vendored directory, the `LIBRARIES` entry and the saved
filename use; the filename slug turns it into `v3-12`.

Sources live in `medigeist-src/`, outside `public/` so they are not served.
Keep them: without them a library cannot be rebuilt, and the generator is a
separate repository this one cannot push to.

### Constraints the generator puts on the page

- **It reads only the first 16 characters.** Longer phrases are truncated
  silently, so the demo shows a notice past that length.
- **Its glyph table is 102 characters**
  (`a–z A–Z 0–9`, punctuation, `ÄÖÜäöü ß`), and the mapping is positional: the
  Nth character draws block N. So **only blocks 001–102 are ever reachable**.
  `v3.12` and `v2.11` carry 193 blocks and can never draw 91 of them; `v3.14`
  can never draw 26. `v3.13`, `v3.15` and `v3.16` are exact fits. Trimming the
  oversized libraries to 102 blocks would cut roughly 0.4 MB each with provably
  identical output — worth doing only if the glyph table is not going to grow.
- **One document shows at most 16 distinct blocks**, one per distinct character
  read. This is a property of the glyph limit, not a bug.
- **Every build exposes exactly one image set, always `set0`**, because
  `setMapsArray()` names sets positionally. That name is hard-coded in
  `medigeist.js`; it is not a free choice.
- **A build is 1.3–3.3 MB.** Builds are fetched lazily — when the demo scrolls
  near or the field is focused, never on page load — cached per library, and
  keystrokes are debounced. Do not move the first render back to load time.
  Only the selected library is downloaded; do not preload the others.
  Within the `cascading_orientation_pattern` family the size tracks the block
  count, but what it really tracks is the byte length of the base64 payloads —
  AssemblyScript holds those strings as UTF-16, so a build is about twice its
  `medigeist-src/` file. That is why `v3.16` reaches 3.3 MB on 102 blocks: each
  of its blocks is a hundred `<rect>` elements. Its renders are ~290 KB rather
  than ~150 KB.
- **It runs entirely in the browser**, every file served from this origin, which
  is what keeps the privacy claims true.
- **Its output carries the phrase in plain text.** Every document opens with a
  `<desc id="sequences">` listing the phrase and all its rotations. Inert on the
  page; not inert in a file, which is the thing people send each other. Anything
  that hands a document to a user must take that block out first.

### Downloads

Both buttons go through `toFileDocument()` in `medigeist.js`, which does two
things and touches nothing that paints:

- **Drops every `<desc>`**, for the reason above. Verify after changing a
  vendored build: the test is that the phrase appears nowhere in the saved
  bytes, not that one element is gone.
- **Restates `width`/`height` as `DOWNLOAD_SIZE`** (1080), because the generator
  writes `100%` for both — right for a responsive panel, unusable in a file,
  which has no containing box to resolve a percentage against. The `viewBox` is
  left alone, so only the intrinsic size changes.

It is string surgery rather than `DOMParser` so the module keeps working outside
a browser. It saves the document the generator returned, not the copy in the
panel, which has picked up a `role`, an `aria-label` and layout classes.

**PNG** rasterizes that same document with `toPNG()` in `main.js`: blob URL →
`Image` → `drawImage` onto a `DOWNLOAD_SIZE`² canvas → `toBlob()`. It lives in
`main.js` because it needs the DOM. Three things hold it up:

- **The stated `width`/`height` are what make it work.** Always hand `toPNG()`
  the output of `toFileDocument()`, never the raw generator string.
- **The canvas must stay origin-clean.** Blob URLs are same-origin and the
  document is self-contained, so `toBlob()` returns pixels. An SVG that
  referenced anything external — a font URL, a remote image, a `foreignObject` —
  would taint the canvas and break the PNG *while leaving the SVG download
  working*, which is a quiet way to break it.
- **PNG is a picture of the document, not the document.** Rasterizers differ
  across browsers in antialiasing and color management, so one phrase gives
  byte-different PNGs on different machines while giving byte-identical SVGs
  everywhere. Comparing by eye works; comparing by hash does not. Keep SVG the
  primary button.

Saved files are `raster-<library>-YYYYMMDD-HHMMSS.<ext>` in local time, and are
**not** named after the phrase: a filename is the most visible part of a file,
and naming it after the phrase would undo the stripping above. The library
segment comes from the library **id**, not the picker label, because patterns
are only comparable within one library. Reading the clock for the *name* does
not break determinism — it never reaches the document.

### Judging a new library

**A library is only useful if its blocks look different from each other**, and
that is a property of the blocks, not the generator. Measure it: render a phrase
and the same phrase one character off, rasterize both, compare pixels. Measure
two *unrelated* phrases too — that is the ceiling.

| Library | One character off | Unrelated phrase |
| --- | --- | --- |
| `v3.12` | 9.47% | 97.9% |
| `v3.13` | 2.18% | 94.9% |
| `v3.14` | 3.39% | 93.6% |
| `v3.15` | 2.26% | 95.0% |
| `v2.11` | 9.47% | — |

Those five clear 93% on unrelated phrases, so they all discriminate. What
varies is how much a *single* character moves, and a low figure means something
different in each case — `v3.13` is line art whose blocks differ by drawing,
`v3.15` by the angle of a high-contrast edge. **The number is evidence, not a
verdict: read it alongside the rendering.**

The one real failure in the inherited set looked like this: as supplied, every
block in `v2.11` shared one canvas fill, so it rendered as a uniform dot field —
deterministic, and unreadable, at 0.6%. `.claude/skills/add-library/recolor.py`
fixed it to 9.47% by giving each block the reference library's *position* within
its hue, lightness and saturation ranges, remapped into `v2.11`'s own bands.
Recolor when the **rendering** is uniform, not when a number is low. The script
rewrites a `rect.canvas` fill and stops on a library that colors itself another
way — `v3.14` paints inline fills on quadrant rects — which is the correct
outcome.

#### `v3.16` is the case the bar exists for

Every block is the same white square carrying a 10×10 grid of 100 cells with
exactly **one** black cell. Block `_001` is blank, `_102` is solid, and the
hundred between them are a single black dot in a different one of the hundred
positions. Combined with the positional mapping above, the Nth character simply
moves a dot to the Nth cell. The blocks are distinct and the generator is
perfectly deterministic, but any two blocks agree on 98 of their 100 cells, so a
changed character moves one dot and the picture does not change in a way a
reader can see.

Measured across all six libraries on one harness at 1080², a 16-character phrase
against the same phrase one character off, and against an unrelated phrase:

| Library | One character off | Unrelated phrase |
| --- | --- | --- |
| `v3.12` | 5.60% | 83.7% |
| `v3.13` | 1.45% | 33.8% |
| `v3.14` | 3.67% | 43.3% |
| `v3.15` | 1.37% | 32.9% |
| `v2.11` | 5.20% | 82.3% |
| `v3.16` | **0.13%** | **2.87%** |

These read lower than the table above — different phrases, rasterizer and
per-channel tolerance — which is why all six are re-measured together here
rather than compared across harnesses. What survives the change of harness is
the shape: `v3.16`'s **ceiling** is 2.87%, an order of magnitude below the
lowest of the other five and *below their one-character figures*. Two unrelated
phrases render near-identically. That is the one property a library cannot
lack, because the whole claim is that a wrong character is visible.

`recolor.py` cannot rescue it and should not be tried: the grid cells carry
inline `fill` attributes with no `rect.canvas` rule, so the script stops — and
the uniformity is geometric anyway. Ninety-eight of a hundred cells are shared
between any two blocks, and no palette changes that. A library of this shape
needs more black cells per block, varying in *position and count*, before it
discriminates at all.

**It ships anyway, by decision.** This was weighed with the figures in hand and
`v3.16` was kept, so treat it as settled rather than as an oversight to correct.
What follows from keeping it:

- It is **not** the default. `LIBRARIES[0]` is `v3.12`, and the entry order is
  what sets the default — do not reorder the list without re-reading this.
- No page copy may cite it as an example of a changed character being visible,
  because for this library that is not true.
- A better version of the same idea would vary the black cells in *count* as
  well as position. If one arrives, replace `v3.16` rather than adding a
  seventh set beside it.

Dropping it later is a product decision and takes one `remove-library` pass.

## Legal pages

`privacy.html` and `contact.html` are plain static pages. Two things are
load-bearing:

- **The privacy policy makes factual claims about the site**: no cookies, no
  analytics, no external requests, nothing in `localStorage`/`sessionStorage`,
  and the phrase never leaving the browser. Adding an embed, a CDN asset, a font
  service, an analytics snippet or a form would falsify one of them. If you add
  anything of that kind, update `privacy.html` in the same change.
- **Section 4 names the host by company and address.** It must match reality.

There is no shared layout — header and footer markup is duplicated across every
page. Editing navigation means editing all of them.

## The beta list

There is no login and no backend. `signup.html` **links** to a form;
`signin.html` says accounts are not open. Three constraints hold it together:

- **Never call the form provider's API from this site.** It needs a token, and
  on a static site the token ships inside the JS bundle, published to a public
  repo and a public site. A shared form URL in `src/config.js` is not a
  credential: it accepts submissions and exposes nothing else.
- **Link the form, never embed it.** An `<iframe>` makes every visitor's browser
  call a third party on page load, disclosing their IP whether or not they sign
  up, and falsifying the "no external requests" claim.
- **Do not gate the demo.** Generation runs client-side, so a login in front of
  it hides a button rather than protecting anything. Making the gate real would
  mean sending the phrase to a server, which the privacy policy says never
  happens.

With the form URL empty, the button is disabled and a visible notice says what
is missing — an unconfigured deploy should say so rather than link to nowhere.

## Design tokens

Inherited from geistgrid and **kept deliberately** — they were reviewed and not
rebranded, so this is the palette rather than a placeholder. Changing it is an
ordinary design task, not a pending decision. The contrast floors below it are
not negotiable either way:

| Token | Value | Used for |
| --- | --- | --- |
| `primary` | `#818cf8` indigo-400 | accent text, borders, tinted backgrounds |
| `secondary` | `#ffffff` | contrast color on top of `primary` |
| `link` | `#ec4899` pink-500 | the far end of the gradient |
| `brand` | `#f54905` | the orange in the logo |

- `primary` is indigo-**400**, not indigo-500: on a `slate-900` panel
  indigo-500 measures 3.99:1, under the 4.5:1 WCAG AA floor.
- Button *fills* use `bg-indigo-600`, not `bg-primary`, so white labels pass.
  White on indigo-500 is 4.47:1 — also just short.
- Do not drop below `slate-400` for body or caption text on a panel;
  `slate-500` measures 3.69:1 at the sizes used here.

Fonts are self-hosted from `public/fonts`, so the page makes no external request
and works offline. Only weights 400–700 exist as files: do **not** use
`font-extrabold` or heavier, or the browser synthesizes the weight.

## Conventions

- Two-space indent, single quotes, semicolons in JS.
- Prefer Tailwind utilities in markup over hand-written CSS. Reach for
  `src/styles/main.css` only for `@theme` tokens and genuine one-offs.
- Keep pattern-generation logic free of DOM access so it stays usable outside a
  browser. `toPNG()` is in `main.js` for exactly this reason.

## Notes

- This is a visual-encoding scheme, **not** a vetted cryptographic primitive. Do
  not describe it as encryption or substitute it for a cipher.
- The generator is public and runs client-side, so anyone holding a pattern can
  test candidate phrases offline until one matches. Phrases must be chosen like
  passwords, and the page should say so wherever it invites someone to publish a
  pattern.
