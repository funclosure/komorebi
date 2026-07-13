# New tuned defaults + zh-TW localization — design

**Date:** 2026-07-13
**Status:** approved

## Ask

1. Make the author's tuned state the site default:
   `presence=20, blur=6, reach=92, wind=138` (door=branch, day=empty,
   card=1, seed=20260712 already are the defaults).
2. A Traditional Chinese (zh-TW) version of the page.

## Part 1 — New defaults

This deliberately relaxes the historical "engine untouched" rule for the
engine's *tuned default values only* — the author is retuning the reference.
The engine's algorithms, constants unrelated to defaults, layer order, and
control ranges remain untouched. Changes:

- Engine `state` initializer: `presence:0.14→0.20`, `blur:16→6`,
  `scale:1.0→0.92`, `wind:0.7→1.38`.
- Slider HTML `value` attributes: presence `14→20`, blur `16→6`,
  scale `100→92`, wind `70→138`.
- Initial `<output>` texts: `14%→20%`, `16 px→6 px`, `1.0×→0.9×`,
  `breath→wind` (must match what the input handler would render).
- The copied prompt then carries the new defaults automatically (it reads
  live state and embeds live engine source). Old share links are unaffected —
  hash values always win over defaults.

## Part 2 — zh-TW localization (single page, runtime toggle)

### Mechanism

- A `LANG` dictionary (keys → `{en, zh}` strings) and `setLang(lang)`
  function inside the `<script id="playground">` block. The engine script
  stays untouched.
- Static translatable elements get `data-i18n="key"` attributes; `setLang`
  swaps their innerHTML. It also sets `<html lang>` (`en` ↔ `zh-Hant`),
  `document.title`, and the seals' `title`/`aria-label`/sr-note.
- Dynamic strings render from the dictionary at use time: wind words
  (still/breath/breeze/wind → 靜／息／微風／風), day statelines, door
  captions, copied-✓ flash needs no text.
- Language persists in the URL hash as `lang=zh-tw` (absent = English), so
  shared links keep their language; `applyHash` reads it, `writeHash`/
  `shareURL` include it only when zh-tw (English URLs stay clean).
- Toggle: a small button top-right of the header in the `.micro` small-caps
  style — reads `中文` in English mode, `EN` in Chinese mode. Clicking calls
  `setLang` and updates the hash.

### Translation scope (all zh-TW copy in Traditional Chinese, Taiwan usage)

- `<title>`, header micro line, h1 tagline, dek.
- Phone mock: `2026 · JULY → 2026 · 七月`, `12 July → 7月12日`,
  `Saturday → 星期六`, statelines (`Still quiet — waiting for color.` →
  `還很安靜——等著顏色到來。`, gathering line likewise).
- Panel: heading THE LAYER, all label pairs flip primary/accent
  (`door 門 → 門 door`, `presence 濃 → 濃度 presence`, etc.), door segment
  captions (BRANCH/DAPPLE/DRIFT → small-caps pinyin-free Chinese: 枝影/葉隙/疏影
  as the en-caption slot), day swatch aria, toggle line
  (`shadow falls on the content too → 樹影也落在內容上`), rail caption
  (`take it / home 土産 → 帶一片光回家`), seal tooltips/aria-labels.
- Phone captions under the mock (three door captions — already half hanzi).
- The six field-note articles and the "Field notes" micro heading — full
  translations, sources/links unchanged.
- Footer paragraphs (the open-source line keeps the same links).
- The copied prompt: fully zh-TW in Chinese mode — intro, tuned values
  block, porting rules, acceptance bar (「若有似無」), the implement-only-
  this-door instruction — with the embedded engine source byte-identical to
  the English prompt's. `buildPrompt` becomes language-keyed.
- The share URL copied by 光 is the same in both modes apart from the
  `lang` param.

### Non-goals

README stays English (may mention 中文 toggle in one line). The gist
provenance prompts under `prompts/` are historical — untouched. No
auto-detection of browser language.

## Verification

- Pristine load: sliders/outputs/engine state equal the new defaults; the
  copied prompt lists them.
- Toggle: EN→中文 swaps title, `<html lang>`, dek, panel labels, captions,
  statelines, tooltips; 中文→EN restores the exact original strings.
- URL: toggling to 中文 puts `lang=zh-tw` in the hash; loading a zh URL
  renders Chinese immediately; English URLs carry no `lang` param.
- Prompt: in zh mode the copied prompt is Chinese and still contains the
  unchanged engine code (`function mulberry32`) and the tuned values; in EN
  mode identical to today's except new defaults.
- Regressions: hash restore of non-default values, regrow, copy-link, fit
  checks at 1440×900, docked rail at 1100×900, mobile 390×844, both themes.
