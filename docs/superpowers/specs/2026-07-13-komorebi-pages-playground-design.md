# komorebi playground on GitHub Pages — design

**Date:** 2026-07-13
**Status:** approved

## Goal

Host the 樹影 komorebi tree-shadow prototype (from gist
`funclosure/ed79bf5abfcf0d3ba1f6561615469c6b`) as a public GitHub Pages site at
`https://funclosure.github.io/komorebi/`. Visitors play with the effect, then
one click copies a prompt that applies their tuned effect **to their own
project** via Claude Code. A second click copies a shareable URL of the exact
state.

The full-page redesign is explicitly out of scope — this ships the existing
prototype page plus the copy features.

## Approach (decided)

Single self-contained `index.html`, no build step, no external requests. The
copied prompt embeds the engine source read at runtime from the page's own
`<script>` tag, so the code in the prompt can never drift from the code running
on screen. Works on Pages and opened via `file://`.

## Repo layout

```
index.html                          — the site (prototype + additions)
README.md                           — what it is, live link, gist link
prompts/tree-shadow-prompt.md       — original gist prompt (provenance)
prompts/tree-shadow-prompt-freeform.md
docs/superpowers/specs/…            — this spec
```

Pages is served from the `main` branch root.

## index.html changes

Base: the gist's `tree-shadow-prototype.html` verbatim. The shadow engine —
its constants, layer order, control ranges, and tuned defaults — is not
modified. Additions:

1. **Engine script gets `id="engine"`** so its `textContent` can be read when
   assembling the prompt. The new UI/clipboard/hash code lives in a separate
   second `<script>` block, keeping the engine source clean in the copied
   prompt.
2. **New panel group "take it home"** at the bottom of the control panel, in
   the page's existing visual language (serif labels, hanzi accents, hairline
   separators):
   - **copy the prompt 持ち帰り** — assembles the apply-prompt (below) and
     copies it.
   - **copy this light 光を共有** — copies the canonical share URL for the
     current state.
   - Both flash "copied ✓" for ~1.5s then revert. Clipboard via
     `navigator.clipboard.writeText`, falling back to a hidden
     textarea + `document.execCommand('copy')`. If both fail, the payload is
     shown in a selectable `<textarea>` inline so the visitor can copy
     manually — never a silent failure.
3. **Footer GitHub link** to `https://github.com/funclosure/komorebi`.

## The copied prompt

Assembled client-side at click time from the live state. Structure:

1. **Ask:** "Add a komorebi (木漏れ日) tree-shadow backdrop to this project" —
   dappled leaf-shadow across the UI. Explicitly aimed at the visitor's repo:
   the agent should detect the platform (plain web, React, SwiftUI, Compose,
   etc.) and the design system from the repo itself.
2. **Tuned values (required defaults):** door, presence %, softness px,
   reach ×, wind, day tint, seed — the visitor's current state, plus the share
   URL they came from.
3. **Porting rules** (distilled from the gist): generator = a pure seeded
   module (unit-testable; consider seeding from the date); compositing
   (blur = penumbra, multiply = shadow-not-paint, opacity = presence) in the
   design-system layer, never inline in feature views; overdraw past the
   visible edge so blur never vignettes; respect `prefers-reduced-motion`
   (freeze to a still frame).
4. **Reference engine** in a fenced `js` block — the full engine source read
   from `#engine`. All three doors are included with the instruction to
   implement only the selected door; per-door slicing is deliberately avoided
   as brittle.
5. **Acceptance bar** (from the gist, non-negotiable): "barely there" — never
   reads as an image pasted behind the app; motion reads as breathing, never
   flapping.

## Shareable URL state

State ↔ hash, e.g.
`#door=dapple&presence=14&blur=16&reach=100&wind=70&day=gold&card=1&seed=20260712`.

- **On load:** parse hash; clamp numeric values to each control's min/max;
  ignore unknown keys and invalid values (fall back to that control's
  default); apply to state, controls, and engine (rebuild if seed differs).
- **On change** (any control, door, tint, toggle, regrow): update the hash via
  `history.replaceState` — no history spam.
- Seed is included so regrown trees are shareable.
- An empty/absent hash means pristine defaults and writes nothing to the URL
  until the visitor changes something.

## Deployment

1. Commit to `main`, push to the existing empty `funclosure/komorebi` remote
   (public).
2. Enable Pages via `gh api` (source: `main` branch, `/` root).
3. Live URL: `https://funclosure.github.io/komorebi/`.

## Verification

Local first (Playwright against the file): each of the three doors renders;
hash round-trip (set controls → reload with produced hash → same state); copy
prompt contains the tuned values and the engine source; copy link produces the
canonical hash URL; reduced-motion and both themes unaffected. Then the same
smoke checks against the live Pages URL after deploy.

## Out of scope

Full landing-page redesign, analytics, prompt-style variants (freeform vs
faithful), build tooling. The original gist stays up; the README links to it.
