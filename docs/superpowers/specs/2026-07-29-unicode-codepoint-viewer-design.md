# Unicode Code Point Viewer — Design

**Date:** 2026-07-29
**Status:** Approved
**Location:** `codepoints/` (new folder) + a card on the root `index.html` app listing.

## Purpose

A minimal, dependency-free web tool that does two things:

1. **Break text into Unicode code points** — inspect existing text (Burmese, emoji, anything).
2. **Turn a code point into its character** — look up what a hex value is.

No character names, no Unicode block info, no external data. Everything the app
needs is native browser JavaScript.

## Scope

In scope:
- Two panels: text → code points, and code point → character.
- Per character: the glyph, its code point (`U+1015`), and the decimal value.
- One row per Unicode code point (code-point-accurate, not UTF-16-unit-accurate).
- New `codepoints/` folder with a single `index.html`.
- A third card on the root app listing (`index.html`).

Out of scope (explicitly excluded to keep it simple):
- Character names and Unicode block names. (Considered; rejected — naming every
  character requires a ~1.2 MB dataset, which violates the "keep simple" goal.)
- General category, UTF-8 bytes, and other rich metadata.
- Grapheme-cluster grouping. The viewer is code-point-accurate; combining marks
  such as `ပါ` render as two rows.
- Persistence (no `localStorage`), network fetching (no `fetch`), or CDN scripts.

## Architecture

One self-contained `index.html`: inline `<style>` and inline `<script>`. No
external resources beyond the Google Fonts stylesheet already used by the repo's
other apps. The two panels are independent and share no state.

### Panel 1 — Text → code points

A `<textarea>` for input. As the text changes (on every `input` event), a result
list renders below it, one row per code point.

Iteration uses `for...of` on the string, which yields code points rather than
UTF-16 code units. This means:

- `ပါ` (U+1015 U+102B) renders as **two** rows.
- Characters outside the BMP, such as `𐍈` (U+10348) or `😀` (U+1F600), render as
  **one** row each — never mis-split into lone surrogates, which a `split('')` or
  `charCodeAt` loop would do.

Each row shows:
- The glyph, rendered large with the Myanmar font stack so it is legible.
- `U+` + zero-padded uppercase hex (e.g. `U+1015`). Padding is to at least 4
  digits; code points above U+FFFF show their full width (e.g. `U+1F600`).
- The decimal value (e.g. `4117`).

Empty input shows a faint hint row instead of an empty list.

### Panel 2 — Code point → character

A single text `<input>` that accepts any of these forms, case-insensitive:

- `U+1015`
- `u+1015`
- `0x1015`
- `1015`

On each `input` event:

1. Strip the `U+` / `u+` / `0x` prefix if present.
2. Parse with `Number.parseInt(cleaned, 16)`.
3. Validate: must be a valid hex number and in range `0 ≤ cp ≤ 0x10FFFF`.
4. If valid, render the glyph with `String.fromCodePoint(cp)`.
5. If invalid, show a small red helper message under the input and render nothing.

Lone surrogates are not special-cased. `String.fromCodePoint` throws for them;
that case is caught and treated as invalid input (same red helper message).

### Layout

The two panels sit as stacked white cards inside a centered container
(`max-width: 760px`), matching the layout of `myanmar-text-cards/index.html`:
a header with the app title and subtitle, then the cards, then a small footer.

## Design system

Reuse the established tokens from the repo's other apps so the viewer reads as a
sibling:

- Cream gradient background.
- White rounded cards (`border-radius` ~20–24px) with the warm shadow
  (`0 8px 40px rgba(180, 120, 60, 0.12), 0 2px 8px rgba(180, 120, 60, 0.08)`).
- `--accent: #f0784a` orange for input focus rings and any accent.
- Noto Sans Myanmar font stack (`'Noto Sans Myanmar', 'Padauk', 'Myanmar Text', sans-serif`)
  for glyph rendering; `system-ui` for UI chrome.
- Responsive sizing via `clamp()` for headings and padding.
- Card `label`s styled uppercase, small, medium-color — matching the existing
  `label` style in `myanmar-text-cards`.

## Error handling

- **Text panel:** always succeeds. Empty input → faint hint row. There is no
  invalid input for this panel.
- **Code-point panel:** invalid (non-hex, empty, out of range, or lone
  surrogate) → red helper text under the input, no glyph rendered. The error
  message is specific enough to guide: "Enter a hex code point up to U+10FFFF."

## Testing

Manual verification, cross-checking the two panels against each other:

- `ပါ` → two rows: `U+1015` (4117) and `U+102B` (4139).
- `a` → `U+0061` (97).
- `😀` → `U+1F600` (128512). Confirms BMP+ handling in the text panel.
- Reverse: `1F600` in the code-point panel → `😀`. Confirms Panel 2 handles
  supplementary-plane code points.
- Reverse: `U+1015` in the code-point panel → `ပ`. Confirms the prefixed form
  parses and that Panel 1 and Panel 2 agree.
- Invalid: `GGGG`, `99999999`, empty → red helper text, no glyph.

These are run in a browser after implementation.

## Open questions

None. All design decisions resolved during brainstorming.
