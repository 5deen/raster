---
name: remove-image-set
description: Remove an image set (a Medigeist block library) from Raster's Image set picker by naming an existing one — by id, picker label or a close variant. Deletes its vendored WASM build, its AssemblyScript source and its LIBRARIES entry, and updates the docs. Use when the user asks to remove, drop or delete an image set or library.
---

# Remove an image set

Takes the name of an image set and removes every trace of it. The picker calls
it an **image set**; the code and docs call it a **library**; they are the same
thing. Four places hold one, and missing any of them leaves either a picker
entry that 404s or 1–3 MB of dead weight in the repository.

Work on a branch, never on `main`: a push to `main` deploys the site.

## 1. Resolve the name

The user may give the id (`cascading_orientation_pattern_v3.12`), the picker
label (`Cascading orientation v3.12`), or something in between. Resolve it
against `LIBRARIES` in `src/medigeist.js` and confirm which id you matched
before deleting anything. If the name matches nothing, or matches more than one,
stop and list what is actually there.

The **binding** — the id with `.` replaced by `_` — is what
`medigeist-src/<binding>.ts` is named.

`CLAUDE.md` records `v3.16` and `v3.20` as **kept by explicit decision** despite
failing the library bar. Removing either is fine when the user names it. Do not
remove one on your own initiative as a tidy-up, and do not offer to remove a
different set than the one named.

## 2. Refuse to empty the list

If this is the only entry in `LIBRARIES`, do not proceed. Removing it leaves the
demo with no library to draw with: `DEFAULT_LIBRARY` reads `LIBRARIES[0].id` and
throws on an empty array, so the Try it section breaks on load. Say so and ask
whether a replacement is coming first.

## 3. Delete

```bash
git rm -r "public/medigeist/<id>" "medigeist-src/<binding>.ts"
```

Then remove its one entry from `LIBRARIES` in `src/medigeist.js`.

If the removed library was first in the list it was also the default, so the
next entry silently becomes the default. That is usually right, but say which
library the demo now opens with rather than letting the user find out.

## 4. What not to touch

- **Saved files elsewhere.** Patterns are only comparable within one library, so
  any SVG or PNG a user saved from the removed library can no longer be
  reproduced by this site. That is a real consequence worth stating in the
  report; it is not a reason to keep the build.
- **`src/medigeist.js` beyond `LIBRARIES`.** `SET`, `RATIO` and the loader are
  library-independent.
- **`privacy.html`.** Removing a vendored file changes none of its claims.

## 5. Verify

1. `npm run build` succeeds and `dist/medigeist/` no longer contains the id.
2. `npx vite preview`, then in a browser: the picker no longer offers the label,
   the remaining default draws a pattern, and no request is made for the removed
   directory.
3. If exactly one library is left, the picker field hides itself again — a
   select with one option is a control that cannot do anything. Confirm that
   rather than assuming it.

## 6. Update the docs in the same change

`public/medigeist/README.md` and `CLAUDE.md` both name the shipped libraries.
Update them in this change, not a follow-up. Search both for the id and its
short form (`v3.12`). The set can appear in several places: the library table,
the set count and total vendored size, the exact-fit and reachable-block lists,
the build-size range, both measurement tables, and, for `v3.16` and `v3.20`, a
subsection of its own under **Judging a new library**. Remove that subsection
whole, and reword any sentence that compared another set against the removed
one.
