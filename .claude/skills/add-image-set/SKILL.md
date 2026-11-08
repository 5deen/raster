---
name: add-image-set
description: Add an image set (a Medigeist block library) to Raster's Image set picker from an attached or referenced AssemblyScript .ts file — an `export const libN = new Map<string,string>()` of base64 block SVGs. Use when the user attaches such a file, or asks to add, import or ship a new image set or library. Covers deriving the id, building the WASM, vendoring it, registering it in LIBRARIES, measuring whether its blocks are visibly distinct, and updating the docs.
---

# Add an image set

Turns one attached `lib*.ts` file into an image set the demo can select. The
picker calls it an **image set**; the code and docs call it a **library**; they
are the same thing. Every step below is load-bearing; skipping the build and
hand-editing `LIBRARIES` produces a picker entry that 404s at runtime.

Work on a branch, never on `main`: a push to `main` deploys the site, and
whether a set ships can still be open after it is built (step 6).

An attached file is large (≈0.5 MB+). Do not read it whole; inspect it with
`grep`/`head` and decode individual blocks as needed.

## 1. Read the file and derive the name

The file looks like:

```ts
export const lib0 = new Map<string, string>();
lib0.set("cascading_orientation_pattern_v3.12_001", "data:image/svg+xml;base64,…");
```

The **library id** is the prefix every key shares, minus the trailing `_NNN`:
`cascading_orientation_pattern_v3.12` above. Derive it, and assert that *every*
key starts with it — a file whose keys disagree is not one library and must be
rejected rather than guessed at:

```bash
grep -o 'lib[0-9]*\.set("[^"]*"' FILE | sed 's/.*set("//; s/"$//' | sed 's/_[0-9]*$//' | sort -u
```

One line out means one library. More than one usually means stop and ask, but
first check whether the keys are simply not all `_NNN`. `network_pattern_v3.20`
runs `_001`–`_003`, then descriptive suffixes such as `_001_edge3_grey_07` and
`_black`, which this check prints as a dozen lines. Take the shortest line as
the candidate id and confirm that every key starts with `<id>_`:

```bash
grep -o 'lib[0-9]*\.set("[^"]*"' FILE | sed 's/.*set("//; s/"$//' | grep -vc "^${ID}_"   # must print 0
```

Descriptive keys are harmless. `setImageSets()` takes the map's values in
insertion order and never reads a key, so only the shared prefix matters.

The **binding** is the id with `.` replaced by `_`
(`cascading_orientation_pattern_v3_12`). AssemblyScript identifiers and module
filenames cannot carry a dot, so the id and the binding differ whenever the name
carries a version number. The id — with the dot — is what the directory, the
`LIBRARIES` entry and the saved filename use.

`base64/config.json` in asc-set-generator holds the `name` the export takes when
these files are generated (`lib8` today). Do not rely on it being `lib0`: match
whatever the file declares.

## 2. Rename the map binding

Rewrite `libN` to the binding, **anchored to the statement forms only**:

```python
out = re.sub(r'^export const libN\b', f'export const {binding}', text, flags=re.M)
out = re.sub(r'^libN\.set\(', f'{binding}.set(', out, flags=re.M)
```

Never a bare string replace: `lib0` occurs inside the base64 payloads, and
replacing those corrupts block images silently — the build still succeeds and
the pictures come out wrong.

## 3. Build

Clone `https://github.com/5deen/asc-set-generator` (shallow is fine) and
`npm install`. Then, with the renamed source copied to
`assembly/lib/<binding>.ts`:

```bash
cat > assembly/imagesets.ts <<INNER
// image selection
import { $BINDING } from './lib/$BINDING';
import { SetParams, setImageSets } from './utils';

export const SETS = new Map<string, SetParams>();

SETS.set("set0", new SetParams(setImageSets([$BINDING])));
INNER
npx asc assembly/index.ts --target release \
  --outFile "build/$ID/release.wasm" \
  --textFile "build/$ID/release.wat" --sourceMap false
git checkout assembly/imagesets.ts
```

Import the lib file **directly**, never through the `assembly/lib` barrel: the
barrel re-exports all ten libraries, defeats dead-code elimination, and leaves
the build at 1 MB whatever was asked for.

`SETS.set("set0", …)` is not a free choice. `setMapsArray()` names sets
positionally, so the single set in a single-library build is always `set0`,
which is the name `src/medigeist.js` passes to `createSVGDocument`.

`--sourceMap false` writes a stray file literally named `false` next to the
output. Delete it, along with `release.wat`; neither is vendored.

## 4. Vendor into the app

```
public/medigeist/<id>/release.js
public/medigeist/<id>/release.wasm
medigeist-src/<binding>.ts        # the renamed source, for rebuilding later
```

One directory per library, and the two build files must sit side by side:
`release.js` finds its binary with `new URL("release.wasm", import.meta.url)`.
They live under `public/` because Vite copies it verbatim; bundling `release.js`
would rewrite that URL and break the fetch.

`medigeist-src/` is outside `public/` on purpose — it is the rebuild source, not
something to serve to visitors.

## 5. Register it

Add one entry to `LIBRARIES` in `src/medigeist.js`:

```js
{ id: 'cascading_orientation_pattern_v3.12', label: 'Cascading orientation v3.12' },
```

That is the only code change. The picker is generated from this list and reveals
itself once there is more than one entry; the first entry is the default.

## 6. Verify before reporting done

Do not skip this — a wrong build fails silently and looks fine in the markup.

1. `npm run build`, then `npx vite preview`.
2. In a browser: the picker offers the new label, selecting it fetches
   `/<id>/release.wasm` **and nothing else**, a pattern draws, and a download is
   named after the library. The phrase must appear nowhere in the saved SVG.
   Block payloads carry their own `<desc>` elements, which `toFileDocument()`
   drops along with the rest. Note the saved size too, because it varies far
   more than build size does (~150 KB for most sets, ~550 KB for `v3.20`).
3. The generator agrees about itself: `setNames()` returns `['set0']`, and
   `createSVGDocument(800, 'set0', 'x')` called twice returns identical strings.
4. **Check the pictures actually differ.** Render one phrase and the same phrase
   with one character changed, rasterize both to a canvas and compare pixels. A
   library whose blocks share a single canvas fill can be fully deterministic and
   still look like a uniform field — around 0.6% of pixels changing rather than
   ~9%. That breaks the "one wrong character is visible" claim the whole page
   rests on, so report the number rather than shipping it silently.

   `rect.canvas` is the only fill that varies per block, and the fix when it does
   not is `recolor.py`, next to this file:

   ```bash
   python3 .claude/skills/add-image-set/recolor.py medigeist-src/<binding>.ts \
     --reference medigeist-src/<a-library-that-reads-well>.ts \
     --hue 196 224 --light 38 77 --sat 24 52
   ```

   It gives each block the reference library's *position* within its hue,
   lightness and saturation ranges, remapped into the bands you pass — so the
   two libraries share a cadence and differ only in colour family. Pick bands
   around the library's existing fill so it keeps its character; the light range
   is what does most of the work, since contrast between neighbouring blocks is
   what a reader actually sees. Rebuild and re-measure afterwards: it rewrites
   the source, not the wasm.

   **The number is evidence, not a verdict, and the tool does not fit every
   library.** Measure two unrelated phrases as well: that is the library's
   ceiling. The five cascading libraries clear it comfortably, so for them a low
   one-character figure means the change is small, not that the library cannot
   discriminate. Two shipped sets do fail it, and they are the ones to compare
   against: `v3.16` (ceiling 2.4%) and `v3.20` (ceiling 12.7%, against about
   50% for the weakest cascading set on the same harness).

   Use several one-character pairs, not one; a single pair can land on two
   near-identical blocks and read 0.00%. Measure every library you compare on
   the **same** harness, re-measuring the existing ones alongside, because
   figures from different phrases, rasterizers or tolerances do not compare.
   And use a per-channel tolerance a person can see (±16, then ±48 to confirm).
   A recolor of `v3.20` looked like a rescue at ±4 (82% ceiling), but at ±48 it
   fell back to 0.01% on some pairs, and side by side it looked like the
   original. When a pixel figure and the screenshot disagree, believe the
   screenshot. `v3.13` measures 2.18% and ships unrecoloured because it
   is line art whose blocks differ by drawing; `v3.14` measures 3.03% and has no
   `rect.canvas` rule at all — its background is four `<rect>` quadrants with
   inline fills — so `recolor.py` stops rather than guessing, which is right.
   Recolour when the *rendering* is uniform, as `v2.11`'s was, not when a number
   is low. Report both figures either way.

5. **If it fails, stop and ask before the docs.** Whether to ship a set that a
   reader cannot read a changed character out of is a product decision, not
   yours. Lay out the figures and three options: ship as supplied, as `v3.16`
   and `v3.20` were; recolor or rework it, then re-measure and show the result;
   or drop it. Leave the docs until the answer is in, because what they say
   depends on it.

## 7. Update the docs in the same change

- `public/medigeist/README.md` — the list of shipped sets and their sizes.
- `CLAUDE.md` — the library table, the set count and total vendored size, the
  exact-fit list if the set has 102 blocks, and the build-size range if it moved.
  A set kept despite failing step 6 gets its own subsection under **Judging a
  new library**, with the figures and the decision, as `v3.16` and `v3.20` have.
  Never make it the default or cite it in copy about visible differences.
- `privacy.html` only if something about requests changed. Adding a vendored
  library does not change it: everything is still served from this origin.
