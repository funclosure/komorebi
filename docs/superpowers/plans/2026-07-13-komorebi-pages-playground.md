# komorebi Pages Playground Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Host the 樹影 komorebi tree-shadow prototype on GitHub Pages with one-click copy of an "apply this effect to my project" prompt and shareable URL state.

**Architecture:** Single self-contained `index.html` (the gist prototype verbatim, engine untouched except a one-line export hook) plus a second `<script>` block adding URL-hash state sync, a "take it home" panel group, and clipboard copy. The copied prompt embeds the engine source read at runtime from the engine `<script>` tag's `textContent`, so it can never drift from what runs on screen. No build step, no external requests.

**Tech Stack:** Plain HTML/CSS/JS, Canvas 2D. Verification via a local `python3 -m http.server` plus the Playwright MCP browser tools (`mcp__playwright__browser_navigate`, `mcp__playwright__browser_evaluate`, `mcp__playwright__browser_click`). Deployment via `git push` + `gh api` (GitHub Pages from `main` branch root).

## Global Constraints

- The shadow engine's constants, layer order, control ranges, and tuned defaults must remain byte-identical to the gist prototype. The ONLY permitted engine edits: `id="engine"` on its `<script>` tag, and the `window.__komorebi` export hook appended at the end of the engine IIFE.
- All playground additions live in a separate second `<script id="playground">` block and new HTML/CSS — never inside the engine IIFE (except the hook above).
- Everything inline; no CDNs, no external images/fonts, no fetches. The file must work opened via `file://` and on Pages.
- Hash param names (exact): `door`, `presence`, `blur`, `reach`, `wind`, `day`, `card`, `seed`. `reach` maps to the `scale` slider, `card` to the `washCard` checkbox (`1`/`0`).
- Repo: `funclosure/komorebi` (public, currently empty). Live URL: `https://funclosure.github.io/komorebi/`.
- Source files from the gist are already downloaded at `/private/tmp/claude-501/-Users-victor-Documents-Workspace-Projects-komorebi/70f761a6-9b09-4b9a-bea6-21ca69e74c31/scratchpad/` (`tree-shadow-prototype.html`, `tree-shadow-prompt.md`, `tree-shadow-prompt-freeform.md`). If missing, re-download from gist `funclosure/ed79bf5abfcf0d3ba1f6561615469c6b` via the GitHub API.
- Commit messages end with `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>`.

---

### Task 1: Seed the repo — prototype as index.html, provenance prompts

**Files:**
- Create: `index.html` (byte-identical copy of the gist's `tree-shadow-prototype.html`)
- Create: `prompts/tree-shadow-prompt.md`, `prompts/tree-shadow-prompt-freeform.md` (gist originals)

**Interfaces:**
- Produces: `index.html` with the engine IIFE in its single `<script>` tag; element IDs `screen`, `shadow`, `wash`, `presence`, `blur`, `scale`, `wind`, `washCard`, `regrow`, `caption`, `stateline`; buttons `.seg[data-variant]` (branch/dapple/drift) and `.sw[data-tint]` (empty/gold/sage/dusk). Later tasks rely on these IDs/classes exactly.

- [ ] **Step 1: Copy files into the repo**

```bash
cd /Users/victor/Documents/Workspace/Projects/komorebi
SCRATCH="/private/tmp/claude-501/-Users-victor-Documents-Workspace-Projects-komorebi/70f761a6-9b09-4b9a-bea6-21ca69e74c31/scratchpad"
cp "$SCRATCH/tree-shadow-prototype.html" index.html
mkdir -p prompts
cp "$SCRATCH/tree-shadow-prompt.md" "$SCRATCH/tree-shadow-prompt-freeform.md" prompts/
```

- [ ] **Step 2: Start a local server (background, keep running for all tasks)**

Run (background): `python3 -m http.server 8017 --directory /Users/victor/Documents/Workspace/Projects/komorebi`

- [ ] **Step 3: Verify the page renders**

Playwright: `browser_navigate` to `http://localhost:8017/`, then `browser_evaluate`:

```js
() => ({
  title: document.title,
  canvas: !!document.getElementById('shadow'),
  doors: [...document.querySelectorAll('.seg')].map(b => b.dataset.variant),
  canvasPainted: document.getElementById('shadow').width > 0
})
```

Expected: `title` = "樹影 — Tree-Shadow Prototype", `canvas` = true, `doors` = `["branch","dapple","drift"]`, `canvasPainted` = true.

- [ ] **Step 4: Commit**

```bash
git add index.html prompts
git commit -m "Seed site with gist prototype and provenance prompts

Prototype from gist funclosure/ed79bf5abfcf0d3ba1f6561615469c6b, unmodified.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 2: Engine hook + URL hash state sync

**Files:**
- Modify: `index.html` — engine `<script>` tag and end of engine IIFE; append a new `<script id="playground">` block before `</body>`/end of file.

**Interfaces:**
- Consumes: element IDs and classes from Task 1; engine closure vars `state`, `rebuild`, `dirty`.
- Produces: `window.__komorebi` = `{ seed (getter), setSeed(n) }`; `window.__playground` = `{ currentState(), stateToHash(s), applyHash() }` where a state object is `{door, presence, blur, reach, wind, day, card, seed}` (all numbers except `door`/`day` strings, `card` 0|1). Task 3 extends `window.__playground`.

- [ ] **Step 1: Tag the engine script and add the export hook**

In `index.html`, change `<script>` (the only script tag, line ~402) to:

```html
<script id="engine">
```

At the end of the engine IIFE, immediately after the `resize();` line and before `})();`, insert:

```js
  /* playground hook — not part of the effect; omit when porting */
  window.__komorebi = {
    get seed(){ return state.seed; },
    setSeed(n){ state.seed = n >>> 0; rebuild(); dirty = true; }
  };
```

- [ ] **Step 2: Add the playground script with hash sync**

Append after the engine `</script>`:

```html
<script id="playground">
(function(){
  'use strict';
  const hook = window.__komorebi;

  const RANGES = { presence:[4,32], blur:[2,36], reach:[60,170], wind:[0,200] };
  const DOORS = ['branch','dapple','drift'];
  const DAY_TINTS = ['empty','gold','sage','dusk'];
  const SLIDER_IDS = { presence:'presence', blur:'blur', reach:'scale', wind:'wind' };

  function currentState(){
    return {
      door: document.querySelector('.seg[aria-pressed="true"]').dataset.variant,
      presence: +document.getElementById('presence').value,
      blur: +document.getElementById('blur').value,
      reach: +document.getElementById('scale').value,
      wind: +document.getElementById('wind').value,
      day: document.querySelector('.sw[aria-pressed="true"]').dataset.tint,
      card: document.getElementById('washCard').checked ? 1 : 0,
      seed: hook.seed
    };
  }

  function stateToHash(s){
    const p = new URLSearchParams();
    for (const k of ['door','presence','blur','reach','wind','day','card','seed']) p.set(k, s[k]);
    return '#' + p.toString();
  }

  function writeHash(){
    history.replaceState(null, '', location.href.split('#')[0] + stateToHash(currentState()));
  }

  function applyHash(){
    if (location.hash.length < 2) return;
    const p = new URLSearchParams(location.hash.slice(1));
    for (const key in SLIDER_IDS){
      if (!p.has(key)) continue;
      const n = parseFloat(p.get(key));
      if (!isFinite(n)) continue;
      const el = document.getElementById(SLIDER_IDS[key]);
      el.value = Math.min(RANGES[key][1], Math.max(RANGES[key][0], Math.round(n)));
      el.dispatchEvent(new Event('input'));
    }
    const door = p.get('door');
    if (DOORS.includes(door)) document.querySelector('.seg[data-variant="' + door + '"]').click();
    const day = p.get('day');
    if (DAY_TINTS.includes(day)) document.querySelector('.sw[data-tint="' + day + '"]').click();
    if (p.has('card')){
      const el = document.getElementById('washCard');
      const on = p.get('card') !== '0';
      if (el.checked !== on){ el.checked = on; el.dispatchEvent(new Event('change')); }
    }
    const seed = parseInt(p.get('seed'), 10);
    if (isFinite(seed) && seed > 0) hook.setSeed(seed);
  }

  /* restore first, THEN start writing — restore must not rewrite the hash */
  applyHash();
  Object.values(SLIDER_IDS).forEach(function(id){
    document.getElementById(id).addEventListener('input', writeHash);
  });
  document.getElementById('washCard').addEventListener('change', writeHash);
  document.querySelectorAll('.seg, .sw').forEach(function(b){ b.addEventListener('click', writeHash); });
  document.getElementById('regrow').addEventListener('click', writeHash);

  window.__playground = { currentState: currentState, stateToHash: stateToHash, applyHash: applyHash };
})();
</script>
```

(Engine listeners were attached first, so `writeHash` always runs after the engine has applied the change — including the regrow seed update.)

- [ ] **Step 3: Verify hash restore (round-trip in)**

Playwright: `browser_navigate` to
`http://localhost:8017/#door=dapple&presence=22&blur=8&reach=140&wind=120&day=gold&card=0&seed=123`,
then `browser_evaluate`:

```js
() => window.__playground.currentState()
```

Expected: `{door:"dapple", presence:22, blur:8, reach:140, wind:120, day:"gold", card:0, seed:123}`.

- [ ] **Step 4: Verify hash write (round-trip out) and clamping**

Playwright: `browser_navigate` to `http://localhost:8017/#presence=9999&door=nonsense`, then `browser_evaluate`:

```js
() => {
  const before = window.__playground.currentState();
  document.getElementById('wind').value = 150;
  document.getElementById('wind').dispatchEvent(new Event('input'));
  return { clampedPresence: before.presence, door: before.door, hash: location.hash };
}
```

Expected: `clampedPresence` = 32 (clamped to max), `door` = "branch" (invalid ignored), and `hash` contains `wind=150` plus all eight keys.

- [ ] **Step 5: Verify pristine load writes nothing**

Playwright: `browser_navigate` to `http://localhost:8017/`, `browser_evaluate`: `() => location.hash`
Expected: `""` (empty — no hash until the visitor changes something).

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Add shareable URL hash state sync

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 3: "Take it home" — copy prompt + copy link + footer credit

**Files:**
- Modify: `index.html` — new CSS rules in the `<style>` block (after the `.regrow` rules), new panel group after the regrow `.group`, one paragraph in `footer.page`, and additions inside the `#playground` script.

**Interfaces:**
- Consumes: `window.__komorebi`, `currentState()`, `stateToHash()` from Task 2; engine source via `document.getElementById('engine').textContent`.
- Produces: `window.__playground.buildPrompt(state)` → string; `window.__playground.shareURL(state)` → string; `window.__lastCopied` set on every copy click (for tests); buttons `#copyPrompt`, `#copyLink`; fallback `#copyFallback`.

- [ ] **Step 1: Add CSS (after the `.regrow .hz` rule in the style block)**

```css
  .take{
    background:none; border:none; cursor:pointer; padding:4px 0; display:block;
    font-family:var(--serif); font-size:14px; color:var(--quiet); text-align:left;
  }
  .take:hover{color:var(--accent)}
  .take .hz{margin-left:.4em; font-size:13px}
  .take.done{color:var(--accent)}
  .take-note{font-size:12.5px; color:var(--quiet); margin:8px 0 0; line-height:1.5}
  .copy-fallback{
    width:100%; margin-top:10px; height:120px; resize:vertical;
    font-family:var(--sans); font-size:11px; color:var(--quiet);
    background:none; border:1px solid var(--hairline-soft); border-radius:8px; padding:8px;
  }
```

- [ ] **Step 2: Add the panel group (after the regrow `.group`, still inside the `<aside class="panel">`)**

```html
      <div class="group">
        <div class="label-row"><span class="name">take it home<span class="hz">土産</span></span></div>
        <button class="take" id="copyPrompt">copy the prompt<span class="hz">持ち帰り</span></button>
        <button class="take" id="copyLink">copy this light<span class="hz">光を共有</span></button>
        <p class="take-note">The prompt carries your tuned values and the engine itself —
        paste it into Claude Code inside your own repo and the light follows you home.</p>
        <textarea id="copyFallback" class="copy-fallback" hidden readonly
          aria-label="copy manually — automatic copy failed"></textarea>
      </div>
```

- [ ] **Step 3: Add the footer credit (second `<p>` inside `footer.page`, after the existing one)**

```html
  <p style="margin-top:12px">Open source at
  <a href="https://github.com/funclosure/komorebi">github.com/funclosure/komorebi</a> ·
  born from <a href="https://gist.github.com/funclosure/ed79bf5abfcf0d3ba1f6561615469c6b">the original gist</a>.</p>
```

- [ ] **Step 4: Add prompt assembly + clipboard to the `#playground` script**

Insert before the `window.__playground = …` line, and extend that line:

```js
  /* ---------- take it home ---------- */
  const DOOR_NOTES = {
    branch:'枝 branch — a bare branch entering from the light’s corner',
    dapple:'隙 dapple — round light-gaps punched from shadow; every dapple is a pinhole image of the sun',
    drift:'葉 drift — sparse leaf clusters adrift on the surface'
  };
  const DAY_INKS = { empty:'#4d473f', gold:'#6e5f43', sage:'#4f5b4d', dusk:'#5f4d51' };

  function shareURL(s){
    const base = location.origin.indexOf('http') === 0
      ? location.href.split('#')[0]
      : 'https://funclosure.github.io/komorebi/';
    return base + stateToHash(s);
  }

  function buildPrompt(s){
    const engine = document.getElementById('engine').textContent.trim();
    return [
      'Add a komorebi (木漏れ日) tree-shadow backdrop to this project — dappled leaf-shadow falling across the UI the way light through a tree lands on a wall.',
      '',
      'I tuned the effect in the komorebi playground; these exact values are the required defaults:',
      '',
      '- door: ' + DOOR_NOTES[s.door],
      '- presence: ' + s.presence + '% (layer opacity)',
      '- softness: ' + s.blur + 'px (blur — the penumbra)',
      '- reach: ' + (s.reach/100).toFixed(1) + '× (scale)',
      '- wind: ' + (s.wind/100).toFixed(2) + ' (0 = still)',
      '- day tint: ' + s.day + ' (shadow ink ' + DAY_INKS[s.day] + ')',
      '- shadow falls on the content too: ' + (s.card ? 'yes' : 'no — behind content only'),
      '- seed: ' + s.seed,
      '- this exact state, live: ' + shareURL(s),
      '',
      'How to apply it (rules from the reference — not negotiable):',
      '1. Detect my platform and design system from this repo (plain web, React, SwiftUI, Compose, …) and implement natively — sharp vector shapes on a 2D canvas; no WebGL, no image assets.',
      '2. Port the generator as a pure seeded module (unit-testable). Keep my seed as its default; consider seeding from the date (e.g. month×31+day) so the same day always wears the same tree.',
      '3. Compositing belongs in the design-system layer, never inline in feature views: blur is the penumbra, a multiply blend so the layer can only darken (shadow, not gray paint), opacity is the presence.',
      '4. Overdraw ~48px past the visible edge and clip it away, so the blur never vignettes.',
      '5. Respect prefers-reduced-motion: freeze to a still frame.',
      '',
      'The acceptance bar — "barely there": the layer must never read as an image pasted behind the app, and the motion must read as breathing, never flapping. If a first-time viewer can tell HOW it is made, it is too loud; if they just feel late-afternoon light in the room, it is right.',
      '',
      'Reference engine — exactly the code I was watching in the playground. Implement ONLY the "' + s.door + '" door; the other two are context, drop them. The window.__komorebi hook at the bottom is playground plumbing — omit it.',
      '',
      '```js',
      engine,
      '```',
      ''
    ].join('\n');
  }

  function legacyCopy(text, done){
    const ta = document.createElement('textarea');
    ta.value = text; ta.style.position = 'fixed'; ta.style.opacity = '0';
    document.body.appendChild(ta); ta.select();
    let ok = false;
    try { ok = document.execCommand('copy'); } catch(e){}
    ta.remove(); done(ok);
  }
  function copyText(text, btn){
    const done = function(ok){ flash(btn, ok, text); };
    if (navigator.clipboard && navigator.clipboard.writeText){
      navigator.clipboard.writeText(text).then(function(){ done(true); }, function(){ legacyCopy(text, done); });
    } else legacyCopy(text, done);
  }
  function flash(btn, ok, text){
    if (!ok){
      const fb = document.getElementById('copyFallback');
      fb.hidden = false; fb.value = text; fb.focus(); fb.select();
      return;
    }
    if (btn.dataset.orig === undefined) btn.dataset.orig = btn.innerHTML;
    btn.innerHTML = 'copied ✓'; btn.classList.add('done');
    clearTimeout(btn._t);
    btn._t = setTimeout(function(){
      btn.innerHTML = btn.dataset.orig; btn.classList.remove('done');
    }, 1500);
  }

  document.getElementById('copyPrompt').addEventListener('click', function(){
    window.__lastCopied = buildPrompt(currentState());
    copyText(window.__lastCopied, this);
  });
  document.getElementById('copyLink').addEventListener('click', function(){
    window.__lastCopied = shareURL(currentState());
    copyText(window.__lastCopied, this);
  });
```

And extend the export line to:

```js
  window.__playground = { currentState: currentState, stateToHash: stateToHash, applyHash: applyHash, buildPrompt: buildPrompt, shareURL: shareURL };
```

- [ ] **Step 5: Verify prompt content**

Playwright: `browser_navigate` to
`http://localhost:8017/#door=dapple&presence=22&blur=8&reach=140&wind=120&day=gold&card=0&seed=123`,
`browser_click` on `#copyPrompt`, then `browser_evaluate`:

```js
() => {
  const p = window.__lastCopied;
  return {
    hasDoor: p.includes('隙 dapple'),
    hasPresence: p.includes('- presence: 22%'),
    hasSeed: p.includes('- seed: 123'),
    hasEngine: p.includes('function mulberry32') && p.includes('drawDappleDoor'),
    hasBar: p.includes('barely there'),
    hasLink: p.includes('#door=dapple'),
    noHtml: !p.includes('<script')
  };
}
```

Expected: every field `true`. Also confirm the button visually reads "copied ✓" via `browser_evaluate`: `() => document.getElementById('copyPrompt').textContent` → `"copied ✓"`.

- [ ] **Step 6: Verify copy link**

`browser_click` on `#copyLink`, then `browser_evaluate`:

```js
() => window.__lastCopied
```

Expected: `http://localhost:8017/#door=dapple&presence=22&blur=8&reach=140&wind=120&day=gold&card=0&seed=123`.

- [ ] **Step 7: Full-page sanity in both themes + reduced motion**

Playwright `browser_evaluate`:

```js
() => {
  document.documentElement.dataset.theme = 'dark';
  const ok = getComputedStyle(document.querySelector('.take')).color !== '';
  document.documentElement.dataset.theme = 'light';
  return ok;
}
```

Expected: `true` (buttons styled in both themes). Take a `browser_take_screenshot` and eyeball: panel group sits below "another tree", matches the page's quiet style, no layout breakage.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "Add take-it-home copy prompt, copy link, and footer credit

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 4: README, push, enable Pages, live verification

**Files:**
- Create: `README.md`

**Interfaces:**
- Consumes: the finished `index.html` from Tasks 1–3.
- Produces: the live site at `https://funclosure.github.io/komorebi/`.

- [ ] **Step 1: Write README.md**

```markdown
# 樹影 komorebi

Light through a tree, landing on paper — a procedural tree-shadow backdrop
(木漏れ日) you can tune in the browser and take home to your own project.

**Play with it: https://funclosure.github.io/komorebi/**

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
```

- [ ] **Step 2: Commit and push**

```bash
git add README.md
git commit -m "Add README

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
git push -u origin main
```

- [ ] **Step 3: Enable GitHub Pages (main branch, root)**

```bash
gh api repos/funclosure/komorebi/pages -X POST -f "source[branch]=main" -f "source[path]=/"
```

Expected: JSON response with `"status"` (e.g. `null` or `"building"`). If it returns 409 (already exists), verify with `gh api repos/funclosure/komorebi/pages` instead.

- [ ] **Step 4: Wait for the build, then smoke-check over HTTP**

```bash
for i in $(seq 1 30); do
  status=$(gh api repos/funclosure/komorebi/pages --jq .status)
  echo "pages status: $status"
  [ "$status" = "built" ] && break
  sleep 10
done
curl -s -o /dev/null -w "%{http_code}" https://funclosure.github.io/komorebi/
```

Expected: final status `built`, curl prints `200`.

- [ ] **Step 5: Live verification**

Playwright: `browser_navigate` to
`https://funclosure.github.io/komorebi/#door=dapple&presence=22&seed=123`,
`browser_click` on `#copyPrompt`, `browser_evaluate`:

```js
() => ({
  state: window.__playground.currentState(),
  promptOk: window.__lastCopied.includes('- seed: 123') && window.__lastCopied.includes('function mulberry32'),
  shareOk: window.__playground.shareURL(window.__playground.currentState()).startsWith('https://funclosure.github.io/komorebi/#')
})
```

Expected: `state.door` = "dapple", `state.presence` = 22, `state.seed` = 123, `promptOk` = true, `shareOk` = true.

- [ ] **Step 6: Final commit if anything changed, and report the live URL**

```bash
git status --short   # expect clean
```

Report `https://funclosure.github.io/komorebi/` to the user.
