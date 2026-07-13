# 樹影 komorebi

Light through a tree, landing on paper — a procedural tree-shadow backdrop
(木漏れ日) you can tune in the browser and take home to your own project.

**Play with it: https://funclosure.github.io/komorebi/**

繁體中文：頁面右上角有 EN／中文 切換。

Tune the three doors — 枝 branch, 隙 dapple, 葉 drift — then:

- **copy the prompt** — one click copies a Claude Code prompt carrying your
  tuned values and the reference engine; paste it into your own repo and it
  applies the effect natively to your stack.
- **copy this light** — a URL that reproduces your exact state, seed and all.

Everything is procedural Canvas 2D — seeded randomness (mulberry32), wind as
three summed sines (GPU Gems 3 ch. 6), dapples punched from shadow because
every canopy gap is a pinhole image of the sun. No WebGL, no image assets,
no build step: the site is one self-contained `index.html`.

Born from [the original gist](https://gist.github.com/funclosure/ed79bf5abfcf0d3ba1f6561615469c6b);
the original prompts live in [`prompts/`](prompts/).
