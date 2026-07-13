# Seal rail: floating action buttons beside the panel — design

**Date:** 2026-07-13
**Status:** approved

## Problem / ask

The three panel actions — another tree 再生, copy the prompt 持ち帰り, copy
this light 光を共有 — live as quiet text links at the bottom of the control
panel. The user sketched them pulled out of the panel into a vertical rail of
rounded-rect buttons floating just right of the panel.

## Design

### The rail

A vertical column of three rounded-rect "seal" buttons, anchored to the
sticky panel: a `.rail` container inside `<aside class="panel">`, positioned
`position:absolute; left:calc(100% + 14px); top:0` so it rides with the
panel's sticky behavior. Buttons stack with a 10px gap.

Each seal (~52×52px, `border-radius:14px`, 1px `var(--hairline-soft)`
border, `var(--panel)` background):

- a thin inline-SVG line icon (16px, `stroke:currentColor`,
  `stroke-width:1.5`, no fill) on top,
- one serif hanzi (~19px) beneath:

| id (unchanged) | hanzi | icon | title / aria-label |
|---|---|---|---|
| `regrow` | 再 | circular arrow | `another tree 再生` / `another tree — regrow a new random tree` |
| `copyPrompt` | 帰 | document/scroll lines | `copy the prompt 持ち帰り` / `copy the prompt — carries your tuned values and the engine to your own project` |
| `copyLink` | 光 | link | `copy this light 光を共有` / `copy this light — a URL that reproduces this exact state` |

States: hover/focus-visible warms text+border toward `var(--accent)`
(existing focus outline token applies); on successful copy the face content
is replaced by a single ✓ for ~1.5s (see script change below). Both themes
come free via the existing tokens.

### Docking (narrow screens)

The floating rail needs free space right of the panel, which exists only at
viewport widths ≥ ~1220px. Below that:

- `@media (max-width:1219px)`: the `.rail` becomes a static horizontal row
  (three seals, same markup) rendered as the panel's last group —
  `position:static; flex-direction:row` with a top hairline border, matching
  the panel's group rhythm. This covers both the narrow two-column band
  (921–1219px) and the single-column/mobile layout.

### What leaves the panel / what stays

Removed: the "another tree" group and the whole "take it home" group (label
row, both text buttons, visible take-note). Kept for correctness:

- `#copyFallback` textarea stays in the panel as its own hidden last group
  (shown only on clipboard failure, exactly as today).
- The take-note text moves to a permanently screen-reader-only paragraph
  (`id="takeNote"`, sr-only pattern already used by the viewport-fit tiers)
  so `#copyPrompt`'s `aria-describedby="takeNote"` keeps resolving.
- The `.take`/`.take-note` CSS rules are replaced by the new `.rail`/`.seal`
  rules; the `.regrow` CSS rule is removed with its group.

### Script impact (one line)

Element ids `regrow`, `copyPrompt`, `copyLink` move onto the seals unchanged,
so every existing binding (engine regrow handler, playground hash write,
copy handlers, flash) works without modification — except `flash()` sets
`btn.innerHTML = 'copied ✓'`, which cannot fit a seal face. Change that one
line to `btn.innerHTML = '✓'`. `btn.dataset.orig` already restores the
icon+hanzi markup afterward. No other script edits; the engine script is
untouched.

### Interaction with the viewport-fit tiers

The panel loses ~230px of height, comfortably preserving the ≤1080px-tier
fit. The tier-2 phone cap and existing fit thresholds are not retuned in
this change. The `.take-note` selector disappears from the tier-2
visually-hidden rule (the note is now sr-only unconditionally).

## Verification

- 1440×900: rail floats right of the panel, three seals render icon+hanzi,
  panel no longer shows the old text buttons; fit checks (phone + panel above
  fold) still pass.
- 1100×900 and 390×844: rail docked as a horizontal row inside the panel;
  no horizontal overflow.
- Click `#copyPrompt`: `window.__lastCopied` carries the tuned values and
  engine source (regression); face shows ✓ then restores icon+hanzi.
- `#regrow` still regrows and updates the hash seed.
- Both themes styled (`data-theme` light/dark).
- `aria-describedby` on `#copyPrompt` resolves to the sr-only note.

## Out of scope

Fourth button (theme toggle), reordering panel groups, changing tuned
defaults, mobile-specific redesign.
