# Viewport Fit Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** On desktop, fit the full phone mock, all controls, and both copy buttons above the fold at ~900px viewport height via height-adaptive CSS compaction.

**Architecture:** Two height-gated media-query tiers appended to the page-chrome `<style>` block of `index.html`, both guarded with `(min-width: 921px)` so the single-column/mobile layout is untouched. Tier 1 (≤1360px tall) tightens spacing only; tier 2 (≤1080px tall) additionally hides three explanatory notes and caps the phone width by viewport height. Two inline `style="margin-top:14px"` attributes in the panel become a `stack` class so the tiers can tighten them. The shadow engine is not touched — the canvas follows phone size via its existing `ResizeObserver`.

**Tech Stack:** Plain CSS/HTML in the existing self-contained `index.html`. Verification via local `python3 -m http.server` + Playwright MCP tools (`browser_resize`, `browser_navigate`, `browser_evaluate`). Deploy via `git push` (GitHub Pages rebuilds from main).

## Global Constraints

- The `<script id="engine">` block must not be modified in any way.
- The `<script id="playground">` block must not be modified in this plan.
- Everything stays inline in `index.html`; no external requests.
- Both media queries MUST carry the `(min-width: 921px)` guard — mobile/single-column stays pixel-identical.
- Spec: `docs/superpowers/specs/2026-07-13-viewport-fit-design.md`.
- Commit messages end with `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>`.

---

### Task 1: Height-adaptive compaction in index.html

**Files:**
- Modify: `index.html` — (a) one CSS rule added after the `.label-row output` rule, (b) two media-query blocks appended at the end of the `<style>` block just before `</style>`, (c) two HTML attribute swaps in the control panel.

**Interfaces:**
- Consumes: existing classes `header.page`, `.dek`, `.stage`, `.panel`, `.group`, `.label-row`, `.seg`, `.sw`, `.sw-note`, `.take-note`, `.phone` — all already in index.html.
- Produces: class `label-row stack` on the two mid-group slider label rows; media tiers keyed at `max-height: 1360px` and `max-height: 1080px`. Task 2 deploys this unchanged.

- [ ] **Step 1: Add the base `.stack` rule**

In the `<style>` block, directly after this existing rule:

```css
  .label-row output{font-family:var(--sans); font-size:12px; color:var(--quiet); font-variant-numeric:tabular-nums}
```

add:

```css
  .label-row.stack{margin-top:14px}
```

- [ ] **Step 2: Swap the two inline styles for the class**

There are exactly two occurrences of this line in the panel HTML (the softness and reach label rows):

```html
        <div class="label-row" style="margin-top:14px">
```

Replace each with:

```html
        <div class="label-row stack">
```

Verify with `grep -c 'label-row stack' index.html` → `2` and `grep -c 'label-row" style' index.html` → `0`.

- [ ] **Step 3: Append the two media-query tiers**

At the end of the `<style>` block, immediately before `</style>`, add:

```css
  /* ============ viewport fit — two-column layout only ============ */
  @media (min-width:921px) and (max-height:1360px){
    header.page{padding:28px 0 20px}
    header.page h1{margin:8px 0}
    header.page .dek{margin:0}
    .stage{padding-top:20px}
    .panel{padding:16px 20px 12px}
    .panel h2{margin-bottom:2px}
    .panel .group{padding:10px 0 8px}
    .label-row{margin-bottom:6px}
    .label-row.stack{margin-top:8px}
    .seg{padding:7px 4px 4px}
  }
  @media (min-width:921px) and (max-height:1080px){
    header.page .dek{display:none}
    .sw-note{display:none}
    .take-note{display:none}
    .sw{width:30px; height:30px}
    .stage{padding-top:14px}
    .panel{padding:12px 18px 8px}
    .panel .group{padding:7px 0 5px}
    .phone{width:min(372px, 90vw, calc((100dvh - 152px) * 0.4723))}
  }
```

(0.4723 rounds down from 393/832 (rounding up overflows the viewport by a sub-pixel), the phone's aspect ratio. Later rules of equal specificity win, so the tier-2 `.phone` width overrides the base `width:min(372px,90vw)` without `!important`.)

- [ ] **Step 4: Verify at laptop heights (tier 2)**

A local server is running at http://localhost:8017 (controller-provided). Using Playwright MCP: `browser_resize` to 1440×900, `browser_navigate` to `about:blank` then `http://localhost:8017/`, then `browser_evaluate`:

```js
() => {
  const vh = innerHeight;
  const b = s => document.querySelector(s).getBoundingClientRect();
  return {
    vh,
    phoneTop: Math.round(b('.phone').top),
    phoneFits: b('.phone').bottom <= vh && b('.phone').top >= 0,
    copyPromptVisible: b('#copyPrompt').bottom <= vh,
    copyLinkVisible: b('#copyLink').bottom <= vh,
    dekHidden: getComputedStyle(document.querySelector('.dek')).display === 'none',
    notesHidden: getComputedStyle(document.querySelector('.sw-note')).display === 'none'
  };
}
```

Expected: `phoneFits`, `copyPromptVisible`, `copyLinkVisible`, `dekHidden`, `notesHidden` all `true`.

Repeat at `browser_resize` 1440×820 (fresh navigate via about:blank): same five fields `true`.

- [ ] **Step 5: Verify tier 1 and untouched layouts**

- `browser_resize` 1440×1200, fresh navigate: evaluate the same function → `phoneFits` `true`, `copyPromptVisible` `true`, `copyLinkVisible` `true`, `dekHidden` `false` (tier 1 keeps the dek).
- `browser_resize` 1440×1400, fresh navigate: evaluate `() => getComputedStyle(document.querySelector('header.page')).paddingTop` → `"64px"` (no tier applies; pristine layout).
- `browser_resize` 390×844, fresh navigate: evaluate `() => ({dek: getComputedStyle(document.querySelector('.dek')).display !== 'none', oneCol: getComputedStyle(document.querySelector('.stage')).gridTemplateColumns.split(' ').length === 1})` → both `true` (mobile untouched).

- [ ] **Step 6: Regression check (hash + copy prompt)**

Fresh navigate (about:blank first) to `http://localhost:8017/#door=dapple&presence=22&blur=8&reach=125&wind=120&day=gold&card=0&seed=123`, click `#copyPrompt`, evaluate:

```js
() => ({
  state: window.__playground.currentState(),
  promptOk: window.__lastCopied.includes('- seed: 123') && window.__lastCopied.includes('function mulberry32')
})
```

Expected: `state` = `{door:"dapple",presence:22,blur:8,reach:125,wind:120,day:"gold",card:0,seed:123}`, `promptOk` = `true`.

- [ ] **Step 7: Take a screenshot for the record**

`browser_resize` 1440×900, fresh navigate to `http://localhost:8017/`, `browser_take_screenshot` (type png, scale css, filename `fit-1440x900.png`). Eyeball: full phone visible, panel ends above the fold with both copy buttons, layout still reads calm (not cramped). Note the screenshot path in your report; do not commit the image.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "Fit full device and all controls above the fold on laptop heights

Two height-gated media tiers (spacing-only at ≤1360px; note-hiding and
viewport-capped phone at ≤1080px), desktop layout only — mobile untouched.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 2: Deploy and live verification

**Files:**
- None created; pushes the Task 1 commit.

**Interfaces:**
- Consumes: the Task 1 commit on `main`.
- Produces: the live site at `https://funclosure.github.io/komorebi/` serving the fit.

- [ ] **Step 1: Push**

```bash
git push origin main
```

- [ ] **Step 2: Wait for Pages to serve the change**

```bash
for i in $(seq 1 30); do
  curl -s https://funclosure.github.io/komorebi/ | grep -q 'max-height:1080px' && { echo updated; break; }
  sleep 10
done
```

Expected: prints `updated` within a few minutes.

- [ ] **Step 3: Live check**

Playwright: `browser_resize` 1440×900, navigate `about:blank` then `https://funclosure.github.io/komorebi/`, run the Step-4 evaluate function from Task 1.
Expected: `phoneFits`, `copyPromptVisible`, `copyLinkVisible`, `dekHidden`, `notesHidden` all `true`.
