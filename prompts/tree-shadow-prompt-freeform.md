# 樹影 — tree-shadow backdrop, freeform version

*A prompt for Claude Code. This version hands your agent the research and the craft tricks, then gets out of the way — expect its own design, not a copy. For a faithful reproduction of the original prototype, use the sibling file `tree-shadow-prompt.md` in this gist.*

---

Build me an interactive HTML prototype of a **tree-shadow backdrop** (樹影 / komorebi) for my app's home screen — dappled leaf-shadow falling across the UI the way light through a tree lands on a wall. Publish it as an artifact if you can; otherwise write one self-contained HTML file I can open in a browser. Everything inline — no CDNs, no external images or fonts.

## Research (already done — build on it)

- **Komorebi (木漏れ日)** names not the light but the *relationship* — leaves, light, and the movement between them. A static shadow texture is not the thing: the layer must breathe, slowly, like wind. (https://wabisabi-jp.com/blogs/wabi-sabi-journal/komorebi , https://thursd.com/articles/komorebi-dance-of-sunlight-in-nature)
- **Every dapple is a picture of the sun.** Each gap in a canopy is a pinhole camera projecting the sun's disk — that's why dapples are round, and why they turn crescent during an eclipse. Softness grows with canopy height. So dappled light means soft ROUND spots punched out of a shadow field — never thresholded noise. (https://eclipse.aas.org/eye-safety/projection , https://petapixel.com/2012/05/21/crescent-shaped-projections-through-tree-leaves-during-the-solar-eclipse/ , https://www.edwardtufte.com/notebook/dappled-light/)
- **The stage-lighting word is "gobo",** and tree-shadow gobo overlays are a current design trend (stock Figma sets, overlay loops). The risk is reading as stock — so generate the shadow procedurally from a seed, never a baked image.
- **Wind is a sum of slow sines** (GPU Gems 3, ch. 6: https://developer.nvidia.com/gpugems/gpugems3/part-i-geometry/chapter-6-gpu-generated-procedural-wind-animations-trees): layer ~3 sine frequencies per branch, amplitude growing toward the tips, plus a faster small flutter on leaves. No physics engine needed.

## Craft tricks known to work (steal them, adapt them, or beat them)

- **No shader needed.** Draw sharp vector shapes on a plain Canvas 2D each frame; three CSS properties on the canvas element make the entire feel: `filter: blur()` is the penumbra, `mix-blend-mode: multiply` can only darken so it reads as shadow rather than gray paint, and `opacity` is the presence. Overdraw ~48px past the visible edge and clip it away, so the blur never vignettes.
- **Seed all randomness** (e.g. mulberry32) so the same seed always grows the same tree — and reseeding is a feature, not a refresh.
- **A recursive skeleton with slightly bowed limbs** (quadratic curves, children attached at random fractions *along* the parent) reads organic where straight segments read like a diagram. Wind: the summed sines above per limb, children inheriting the parent's swayed angle so motion accumulates trunk→tip. Keep every amplitude tiny — leaf flutter under ~0.1 rad. Breathing, never flapping.
- **Dapples are punched, not drawn:** fill the canvas with shadow, then cut drifting round holes with `destination-out` radial gradients.

## The freedom

The rest is yours. Design the page and the phone mock in your own voice — or in my repo's, if it has a design system (tokens, theme, palette: check first). Choose how many shadow variants to offer and what they are — a bare branch, a full canopy dapple, loose drifting leaves are the proven doors, but invent your own if you see one. Choose the controls that make exploring feel good, and keep them quiet: thin sliders, no dashboard chrome. Choose and defend your own tuned defaults.

## The bar (not negotiable)

**"Barely there."** A very low-contrast layer the UI wears — it must never read as an image pasted behind the app, and the motion must read as breathing, never flapping. If a first-time viewer can tell you HOW it's made, it's too loud; if they just feel late-afternoon light in the room, it's right. Before you finish, look at your own defaults and retune until they pass this bar. Respect `prefers-reduced-motion` (freeze to a still frame) and theme the page chrome for both light and dark.
