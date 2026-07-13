# Viewport fit: full device + all controls above the fold — design

**Date:** 2026-07-13
**Status:** approved

## Problem

On typical laptop displays (~800–1000 CSS px of height) the playground's first
screen cuts the phone mock in half and hides the control panel's bottom —
including the "take it home" copy buttons, which defeats the site's point.
Measured at 1440×900: header 307px tall, panel ends at 1326px, phone at
1128px. The panel (986px) is the binding constraint.

## Goal

On the two-column desktop layout, the full phone mock, all controls, and both
copy buttons fit above the fold at ~900px viewport height. Tall monitors keep
the current airy layout. Mobile (single-column, ≤920px wide) is untouched —
it scrolls naturally.

## Approach (decided)

Height-adaptive compaction: two media-query tiers in the page-chrome
`<style>` block, both guarded with `(min-width: 921px)` so they never apply
to the single-column layout. The shadow engine is not touched; the canvas
already follows phone size via `ResizeObserver`.

### Tier 1 — `@media (min-width: 921px) and (max-height: 1360px)`

Spacing-only tightening, no content removed (~250px saved; screens ≥ ~1080px
tall then fit everything):

- `header.page` padding 64/40 → 28/20; `h1` margins → 8/8; `.dek` margin 0
- `.stage` padding-top 32 → 20
- `.panel` padding 26/24/20 → 16/20/12; `h2` margin-bottom → 2
- `.panel .group` padding 16/14 → 10/8
- `.label-row` margin-bottom 9 → 6; `.label-row.stack` margin-top 14 → 8
- `.seg` padding 10/8 → 7/4

### Tier 2 — `@media (min-width: 921px) and (max-height: 1080px)`

The laptop class; additionally:

- Hide the explanatory copy: `.dek`, `.sw-note`, `.take-note` (`display:none`)
- Swatches `.sw` 34px → 30px
- `.stage` padding-top → 14; `.panel` padding → 12/18/8; `.panel .group`
  padding → 7/5 (deep enough that ~820px-tall viewports also fit)
- Cap the phone by viewport height:
  `.phone{ width: min(372px, 90vw, calc((100dvh - 150px) * 0.4724)); }`
  (0.4724 = 393/832, the phone's aspect ratio; 150px ≈ compact header +
  stage padding + breathing room)

### HTML change (page chrome, not engine)

The two slider `label-row`s using inline `style="margin-top:14px"` become
`class="label-row stack"`, with base rule `.label-row.stack{margin-top:14px}`
— inline styles can't be tightened by media queries. No other markup changes.

## Verification

Browser screenshots + assertions at:
- 1440×900 and 1440×820: phone fully visible, `#copyPrompt` and `#copyLink`
  fully inside the viewport (getBoundingClientRect bottom ≤ innerHeight)
- 1440×1200: tier 1 only — dek and notes still visible
- 1440×1400: pristine current layout (no tier applies)
- 390×844: mobile single-column identical to today (tiers gated off)
- Existing behavior re-checks: hash restore round-trip and copy-prompt
  payload unchanged.

## Out of scope

Panel reordering, full-viewport app layout, any engine change, mobile layout
changes. The landing-page redesign remains deferred.
