# Unicode Code Point Viewer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a minimal, dependency-free single-page Unicode code point viewer that breaks text into code points and turns a code point back into its character.

**Architecture:** One self-contained `codepoints/index.html` with inline `<style>` and inline `<script>`, no external resources except the Google Fonts stylesheet already used by the repo's other apps. Two independent panels share no state: a text→codepoints breakdown (using `for...of`, which is code-point-accurate), and a codepoint→character lookup (parsing hex via `parseInt(..., 16)` and rendering via `String.fromCodePoint`). The entire logic is three pure functions: `toCodePoints(str)`, `formatCodePoint(cp)`, `parseCodePoint(input)`. A third card is added to the root `index.html` app listing.

**Tech Stack:** Vanilla HTML/CSS/JS (no framework, no build step, no package manager). This repo is a static GitHub Pages site; each app is a folder with one `index.html`. There is no test runner — the existing apps are verified by opening them in a browser. To match this, the three pure logic functions are verified with `node` by loading the actual function source, and the DOM integration is verified by the spec's manual browser cross-checks.

**Reference for design conventions:** `myanmar-text-cards/index.html` — copy its CSS variable tokens, card styling, header structure, and `.control-row` / `.btn` patterns so the viewer reads as a sibling app.

---

## File Structure

- **Create:** `codepoints/index.html` — the entire app (HTML structure, inline `<style>` reusing repo tokens, inline `<script>` with the three pure functions plus the two panels' DOM wiring).
- **Modify:** `index.html` (root app listing) — add a third `.app-card` linking to `codepoints/`, after the Myanmar Text Cards card, using the same card markup/styling as the existing two.

No other files. No `fonts/` folder needed (glyphs render with the Google-Fonts-loaded Noto Sans Myanmar stack plus the OS fallbacks). No `package.json`, no build config.

---

## Task 1: Scaffold the app with design-system styles and header

This task produces a page that renders correctly with the repo's look-and-feel but has no interactive logic yet. The logic functions and panel wiring come in Tasks 2–4.

**Files:**
- Create: `codepoints/index.html`

- [ ] **Step 1: Create `codepoints/index.html` with the full HTML skeleton, styles, and empty panel containers**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Unicode Code Points</title>
    <link href="https://fonts.googleapis.com/css2?family=Noto+Sans+Myanmar:wght@400;500;600;700&display=swap" rel="stylesheet">
    <style>
        :root {
            --card-bg: #ffffff;
            --card-shadow: 0 8px 40px rgba(180, 120, 60, 0.12), 0 2px 8px rgba(180, 120, 60, 0.08);
            --text-dark: #3d2b1f;
            --text-medium: #6b4d3a;
            --text-light: #9b7b65;
            --accent: #f0784a;
            --accent-glow: rgba(240, 120, 74, 0.35);
            --font-burmese: 'Noto Sans Myanmar', 'Padauk', 'Myanmar Text', sans-serif;
        }

        * { margin: 0; padding: 0; box-sizing: border-box; }

        body {
            font-family: system-ui, -apple-system, 'Segoe UI', sans-serif;
            background: linear-gradient(170deg, #fff8f0 0%, #fef3e8 25%, #fef7f2 50%, #fff5ed 75%, #fef9f5 100%);
            background-attachment: fixed;
            min-height: 100vh;
            padding: 24px 20px 60px;
            color: var(--text-dark);
        }

        .app-container {
            position: relative;
            max-width: 760px;
            margin: 0 auto;
            display: flex;
            flex-direction: column;
            gap: 20px;
        }

        .header { text-align: center; margin-bottom: 4px; }
        .header h1 {
            font-family: var(--font-burmese);
            font-size: clamp(1.4rem, 3.5vw, 1.8rem);
            font-weight: 700;
            color: var(--text-dark);
        }
        .header .subtitle {
            font-size: 0.85rem;
            color: var(--text-light);
            margin-top: 4px;
        }

        .card {
            background: var(--card-bg);
            border-radius: 24px;
            box-shadow: var(--card-shadow);
            padding: clamp(18px, 3vw, 26px);
            border: 1.5px solid #fef0e4;
            display: flex;
            flex-direction: column;
            gap: 16px;
        }

        .card label {
            font-size: 0.8rem;
            font-weight: 600;
            color: var(--text-medium);
            text-transform: uppercase;
            letter-spacing: 0.03em;
        }

        textarea, .cp-input {
            width: 100%;
            padding: 14px 16px;
            font-family: var(--font-burmese);
            font-size: 1.15rem;
            line-height: 1.6;
            color: var(--text-dark);
            background: #fffdf9;
            border: 1.5px solid #f0d8c4;
            border-radius: 14px;
            outline: none;
            transition: border-color 0.25s ease, box-shadow 0.25s ease;
        }
        textarea { min-height: 120px; resize: vertical; }
        textarea:focus, .cp-input:focus {
            border-color: var(--accent);
            box-shadow: 0 0 0 3px var(--accent-glow);
        }
        textarea::placeholder, .cp-input::placeholder { color: #c4ab95; }

        .error-msg {
            font-size: 0.82rem;
            color: #c0392b;
            min-height: 1em;
        }

        /* Text → code points result list */
        .cp-list { display: flex; flex-direction: column; gap: 6px; }
        .cp-row {
            display: grid;
            grid-template-columns: 56px 1fr auto;
            align-items: center;
            gap: 14px;
            padding: 8px 12px;
            background: #fffdf9;
            border: 1px solid #f3e3d2;
            border-radius: 12px;
        }
        .cp-row .glyph {
            font-family: var(--font-burmese);
            font-size: 1.8rem;
            text-align: center;
            color: var(--text-dark);
            line-height: 1.2;
            overflow: hidden;
        }
        .cp-row .hex { font-size: 0.95rem; font-weight: 600; color: var(--accent); }
        .cp-row .dec { font-size: 0.8rem; color: var(--text-light); font-variant-numeric: tabular-nums; }

        .hint {
            font-size: 0.85rem;
            color: #c4ab95;
            font-style: italic;
        }

        /* Code point → character output */
        .cp-out {
            font-family: var(--font-burmese);
            font-size: 3.2rem;
            text-align: center;
            padding: 16px;
            background: #fffdf9;
            border: 1px solid #f3e3d2;
            border-radius: 14px;
            min-height: 80px;
            display: flex;
            align-items: center;
            justify-content: center;
        }

        .back-link {
            text-align: center;
            font-size: 0.85rem;
            color: var(--text-light);
        }
        .back-link a { color: var(--accent); text-decoration: none; }
        .back-link a:hover { text-decoration: underline; }
    </style>
</head>
<body>

    <div class="app-container">
        <div class="header">
            <h1>Unicode Code Points</h1>
            <div class="subtitle">Break text into code points, or look one up</div>
        </div>

        <div class="card">
            <label for="textInput">Text → code points</label>
            <textarea id="textInput" placeholder="Type or paste any text… ဥပမာ — မင်္ဂလာပါ"></textarea>
            <div class="cp-list" id="textList"></div>
        </div>

        <div class="card">
            <label for="cpInput">Code point → character</label>
            <input class="cp-input" id="cpInput" type="text" placeholder="e.g. U+1015, 0x1015, or 1015" autocomplete="off">
            <div class="error-msg" id="cpError"></div>
            <div class="cp-out" id="cpOut"></div>
        </div>

        <div class="back-link"><a href="../">← All apps</a></div>
    </div>

    <script>
        // Logic functions are added in Task 2.
        // Panel wiring is added in Tasks 3 and 4.
    </script>

</body>
</html>
```

- [ ] **Step 2: Verify the page renders**

Open `codepoints/index.html` in a browser (or run `open codepoints/index.html` on macOS). Expected: cream gradient background, centered title "Unicode Code Points", two white rounded cards with labels and empty inputs, a "← All apps" link. No console errors.

- [ ] **Step 3: Commit**

```bash
git add codepoints/index.html
git commit -m "Scaffold Unicode code point viewer with design-system styles"
```

---

## Task 2: Add the three pure logic functions and verify with node

The entire app logic lives in these three pure functions. We write them, then verify each one by running the actual source through `node` before wiring them to the DOM.

**Files:**
- Modify: `codepoints/index.html` (the `<script>` block)

- [ ] **Step 1: Replace the placeholder `<script>` comment with the three functions**

Replace this block:

```html
    <script>
        // Logic functions are added in Task 2.
        // Panel wiring is added in Tasks 3 and 4.
    </script>
```

with:

```html
    <script>
        'use strict';

        // Break a string into its Unicode code points.
        // Uses for...of which iterates by code point, NOT UTF-16 code unit,
        // so combining marks and supplementary-plane characters are never mis-split.
        function toCodePoints(str) {
            const out = [];
            for (const ch of str) {
                out.push(ch.codePointAt(0));
            }
            return out;
        }

        // Format a code point as U+hex, zero-padded to at least 4 digits.
        function formatCodePoint(cp) {
            return 'U+' + cp.toString(16).toUpperCase().padStart(4, '0');
        }

        // Parse a code point from user input.
        // Accepts: U+1015, u+1015, 0x1015, or bare 1015 (always hex).
        // Returns the integer code point, or null if invalid / out of range.
        // Lone surrogates (0xD800–0xDFFF) return null — String.fromCodePoint
        // cannot render them, so we reject them up front.
        function parseCodePoint(input) {
            if (typeof input !== 'string') return null;
            const trimmed = input.trim();
            if (!trimmed) return null;
            let cleaned = trimmed.replace(/^u\+/i, '').replace(/^0x/i, '');
            if (!/^[0-9a-fA-F]+$/.test(cleaned)) return null;
            const cp = parseInt(cleaned, 16);
            if (Number.isNaN(cp)) return null;
            if (cp < 0 || cp > 0x10FFFF) return null;
            if (cp >= 0xD800 && cp <= 0xDFFF) return null;
            return cp;
        }

        // Panel wiring is added in Tasks 3 and 4.
    </script>
```

- [ ] **Step 2: Write the node verification script**

Create a temporary file `/tmp/cp_verify.js` that extracts the three functions from the source file and asserts their behavior against every case from the spec's Testing section:

```js
// /tmp/cp_verify.js
const fs = require('fs');
const html = fs.readFileSync('codepoints/index.html', 'utf8');

// Pull the three function definitions out of the <script> block and eval them.
const scriptMatch = html.match(/<script>([\s\S]*?)<\/script>/);
if (!scriptMatch) { console.error('no script block found'); process.exit(1); }
// The script begins with 'use strict';. A direct eval of strict-mode source does
// not leak function declarations to this scope, so strip that directive first.
eval(scriptMatch[1].replace(/^\s*'use strict';\s*/, ''));

let pass = 0, fail = 0;
function eq(actual, expected, label) {
    const a = JSON.stringify(actual);
    const e = JSON.stringify(expected);
    if (a === e) { pass++; }
    else { fail++; console.error(`FAIL: ${label}\n  expected ${e}\n  got      ${a}`); }
}

// --- toCodePoints ---
eq(toCodePoints(''), [], 'empty string -> []');
eq(toCodePoints('a'), [0x61], 'a -> [0x61]');
eq(toCodePoints('ပါ'), [0x1015, 0x102B], 'ပါ -> two code points (combining mark not merged)');
eq(toCodePoints('😀'), [0x1F600], 'emoji -> one code point (supplementary plane, not split into surrogates)');
eq(toCodePoints('𐍈'), [0x10348], 'U+10348 -> one code point');

// --- formatCodePoint ---
eq(formatCodePoint(0x61), 'U+0061', '0x61 -> U+0061 (4-digit padding)');
eq(formatCodePoint(0x1015), 'U+1015', '0x1015 -> U+1015');
eq(formatCodePoint(0x102B), 'U+102B', '0x102B -> U+102B');
eq(formatCodePoint(0x1F600), 'U+1F600', '0x1F600 -> U+1F600 (>4 digits)');
eq(formatCodePoint(0x10348), 'U+10348', '0x10348 -> U+10348');

// --- parseCodePoint ---
eq(parseCodePoint('U+1015'), 0x1015, 'U+1015 -> 0x1015');
eq(parseCodePoint('u+1015'), 0x1015, 'u+1015 (lowercase prefix) -> 0x1015');
eq(parseCodePoint('0x1015'), 0x1015, '0x1015 -> 0x1015');
eq(parseCodePoint('1015'), 0x1015, 'bare 1015 parsed as HEX -> 0x1015');
eq(parseCodePoint('1F600'), 0x1F600, '1F600 -> 0x1F600');
eq(parseCodePoint('1f600'), 0x1F600, '1f600 lowercase hex -> 0x1F600');
eq(parseCodePoint('  U+1015  '), 0x1015, 'whitespace trimmed');
eq(parseCodePoint(''), null, 'empty -> null');
eq(parseCodePoint('GGGG'), null, 'non-hex -> null');
eq(parseCodePoint('110000'), null, 'above U+10FFFF -> null');
eq(parseCodePoint('D800'), null, 'lone surrogate D800 -> null');
eq(parseCodePoint('DFFF'), null, 'lone surrogate DFFF -> null');

console.log(`\n${pass} passed, ${fail} failed`);
process.exit(fail === 0 ? 0 : 1);
```

- [ ] **Step 3: Run the verification and confirm all assertions pass**

Run from the repo root:

```bash
node /tmp/cp_verify.js
```

Expected: `21 passed, 0 failed` and exit code 0. (21 = the count of `eq(...)` calls above.)

- [ ] **Step 4: Commit**

```bash
git add codepoints/index.html
git commit -m "Add pure code-point logic functions (toCodePoints, formatCodePoint, parseCodePoint)"
```

---

## Task 3: Wire the text → code points panel

**Files:**
- Modify: `codepoints/index.html` (the `<script>` block)

- [ ] **Step 1: Add the text-panel rendering and event wiring**

In the `<script>` block, replace the comment line:

```js
        // Panel wiring is added in Tasks 3 and 4.
```

with:

```js
        // --- Text → code points panel ---
        const textInput = document.getElementById('textInput');
        const textList = document.getElementById('textList');

        function renderTextPanel() {
            const text = textInput.value;
            textList.innerHTML = '';
            if (text.length === 0) {
                const hint = document.createElement('div');
                hint.className = 'hint';
                hint.textContent = 'Code points will appear here as you type…';
                textList.appendChild(hint);
                return;
            }
            for (const ch of text) {
                const cp = ch.codePointAt(0);
                const row = document.createElement('div');
                row.className = 'cp-row';

                const glyph = document.createElement('div');
                glyph.className = 'glyph';
                glyph.textContent = ch;

                const hex = document.createElement('div');
                hex.className = 'hex';
                hex.textContent = formatCodePoint(cp);

                const dec = document.createElement('div');
                dec.className = 'dec';
                dec.textContent = cp;

                row.appendChild(glyph);
                row.appendChild(hex);
                row.appendChild(dec);
                textList.appendChild(row);
            }
        }

        textInput.addEventListener('input', renderTextPanel);
        renderTextPanel();
```

- [ ] **Step 2: Verify the text panel in the browser**

Open `codepoints/index.html`. Type `မင်္ဂလာပါ`. Expected: one row per code point, each showing the glyph, `U+xxxx` in orange, and the decimal number. Confirm `ပါ` near the end shows as **two** rows (`U+1015`, `U+102B`). Type `a` → one row `U+0061` / `97`. Type an emoji like `😀` → one row `U+1F600` / `128512` (not two surrogate rows). Clear the box → the faint hint text returns.

- [ ] **Step 3: Commit**

```bash
git add codepoints/index.html
git commit -m "Wire text-to-codepoints panel with live breakdown"
```

---

## Task 4: Wire the code point → character panel

**Files:**
- Modify: `codepoints/index.html` (the `<script>` block)

- [ ] **Step 1: Add the codepoint-panel rendering and event wiring**

Append to the end of the `<script>` block (after the text-panel code from Task 3):

```js
        // --- Code point → character panel ---
        const cpInput = document.getElementById('cpInput');
        const cpError = document.getElementById('cpError');
        const cpOut = document.getElementById('cpOut');

        function renderCpPanel() {
            cpError.textContent = '';
            cpOut.textContent = '';
            const cp = parseCodePoint(cpInput.value);
            if (cp === null) {
                const trimmed = cpInput.value.trim();
                if (trimmed) {
                    cpError.textContent = 'Enter a hex code point up to U+10FFFF (e.g. U+1015).';
                }
                return;
            }
            cpOut.textContent = String.fromCodePoint(cp);
        }

        cpInput.addEventListener('input', renderCpPanel);
        renderCpPanel();
```

- [ ] **Step 2: Verify the codepoint panel in the browser**

Open `codepoints/index.html`, focus the lower input. Try each: `U+1015` → `ပ`; `1F600` → `😀`; `0x0061` → `a`. Confirm these match the glyphs the text panel produces for the same code points (the two panels agree). Then invalid inputs: `GGGG` → red error, no glyph; `110000` → red error, no glyph; `D800` → red error, no glyph; empty → no error, no glyph.

- [ ] **Step 3: Commit**

```bash
git add codepoints/index.html
git commit -m "Wire codepoint-to-character panel with validation"
```

---

## Task 5: Add the viewer to the root app listing

**Files:**
- Modify: `index.html` (root app listing)

- [ ] **Step 1: Add a third app card after the Myanmar Text Cards card**

In `index.html`, find the Myanmar Text Cards card block:

```html
            <a class="app-card" href="myanmar-text-cards/">
                <span class="icon">🖼️</span>
                <div class="info">
                    <h3>Myanmar Text Cards</h3>
                    <p>Turn Burmese text into color-coded PNG images</p>
                </div>
                <span class="arrow">→</span>
            </a>
```

Add this new card immediately after it (inside the same `<div class="apps">`):

```html
            <a class="app-card" href="codepoints/">
                <span class="icon">#</span>
                <div class="info">
                    <h3>Unicode Code Points</h3>
                    <p>Break text into code points, or look one up</p>
                </div>
                <span class="arrow">→</span>
            </a>
```

- [ ] **Step 2: Verify the listing renders**

Open the root `index.html`. Expected: three app cards — KG Practice, Myanmar Text Cards, and the new Unicode Code Points card. Click the new card → navigates to `codepoints/` and loads the viewer. No layout breakage; the new card matches the styling of the other two.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Add Unicode Code Points to app listing"
```

---

## Final verification

After all tasks complete, run the spec's full cross-check matrix in the browser:

1. Open `codepoints/index.html` via the root listing.
2. **Text panel:** `မင်္ဂလာပါ` → code points including `ပါ` as two rows (`U+1015`, `U+102B`); `a` → `U+0061`/`97`; `😀` → `U+1F600`/`128512` (one row, not two).
3. **Codepoint panel:** `1F600` → `😀`; `U+1015` → `ပ` (agrees with the text panel's glyph for U+1015); `GGGG` / `110000` / `D800` → red error, no glyph.
4. No console errors on either page. No network requests beyond the Google Fonts stylesheet (confirm in DevTools → Network that there are no third-party script/data fetches — the viewer must be dependency-free per spec).

The plan is complete when all four checks pass.
