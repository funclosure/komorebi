# New Defaults + zh-TW Localization Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship the author's tuned values (presence 20 / blur 6 / reach 92 / wind 138) as the site defaults, and add a runtime EN ↔ 繁體中文 toggle that localizes all copy including the copied Claude Code prompt.

**Architecture:** Task 1 edits the engine `state` initializer (a deliberate, user-approved relaxation of the engine-untouched rule, for default VALUES only) plus the sliders' HTML values/outputs. Task 2 adds `data-i18n` attributes to static copy, an `I18N` dictionary + `setLang()` inside the playground script, dictionary-driven re-rendering of the engine-owned dynamic strings (captions, statelines, wind words) via after-engine listeners, `lang=zh-tw` hash persistence, a header toggle button, and a language-keyed `buildPrompt`. Task 3 deploys.

**Tech Stack:** Plain HTML/CSS/JS in the self-contained `index.html`. Verification via local `python3 -m http.server` + Playwright MCP. Deploy via `git push`.

## Global Constraints

- Engine script (`<script id="engine">`): Task 1 may change ONLY the six literal default values named in Task 1 Step 1. Nothing else in the engine, ever. Task 2 does not touch the engine at all.
- The engine keeps writing its English captions/statelines/wind-words; the i18n layer re-renders them AFTER via its own listeners (accepted duplication of those few strings, commented as such).
- Ids `regrow`, `copyPrompt`, `copyLink`, `takeNote`, `copyFallback`, plus new `langToggle`, must exist.
- Hash param `lang=zh-tw` appears ONLY in Chinese mode; English URLs stay clean. All eight existing hash params unchanged.
- All zh strings are Traditional Chinese (Taiwan usage), given verbatim in this plan — do not re-translate or "improve" them.
- Everything inline in `index.html`; no external requests.
- Spec: `docs/superpowers/specs/2026-07-13-defaults-zhtw-design.md`.
- Commit messages end with `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>`.

---

### Task 1: New tuned defaults

**Files:**
- Modify: `index.html` — engine `state` initializer, four slider `value` attributes, four `<output>` initial texts.

**Interfaces:**
- Consumes: existing engine/markup.
- Produces: pristine-load state `{presence:20, blur:6, reach:92, wind:138}` that Task 2's verification relies on.

- [ ] **Step 1: Engine state initializer (the ONLY engine edit)**

In the `<script id="engine">` block, the state object currently reads:

```js
  const state = {
    variant:'branch',
    presence:0.14,
    blur:16,
    scale:1.0,
    wind:0.7,
    tint:'empty',
    seed:20260712
  };
```

Change exactly four values:

```js
  const state = {
    variant:'branch',
    presence:0.20,
    blur:6,
    scale:0.92,
    wind:1.38,
    tint:'empty',
    seed:20260712
  };
```

- [ ] **Step 2: Slider values and outputs**

In the panel HTML make these exact swaps:

- `<output id="presenceOut">14%</output>` → `<output id="presenceOut">20%</output>`
- `<input type="range" id="presence" min="4" max="32" value="14"` → `value="20"`
- `<output id="blurOut">16 px</output>` → `<output id="blurOut">6 px</output>`
- `<input type="range" id="blur" min="2" max="36" value="16"` → `value="6"`
- `<output id="scaleOut">1.0×</output>` → `<output id="scaleOut">0.9×</output>`
- `<input type="range" id="scale" min="60" max="170" value="100"` → `value="92"`
- `<output id="windOut">breath</output>` → `<output id="windOut">wind</output>`
- `<input type="range" id="wind" min="0" max="200" value="70"` → `value="138"`

(Readout texts must match what the input handlers would render for those values: 20%, 6 px, 0.9×, and 1.38 ≥ 1.2 → "wind".)

- [ ] **Step 3: Verify pristine defaults**

Local server runs at http://localhost:8017 (controller-provided). Playwright (load via about:blank first):

```js
() => ({
  state: window.__playground.currentState(),
  outs: ['presenceOut','blurOut','scaleOut','windOut'].map(id => document.getElementById(id).textContent),
  hash: location.hash,
  opacity: getComputedStyle(document.getElementById('shadow')).opacity,
  blur: getComputedStyle(document.getElementById('shadow')).filter
})
```

Expected: `state` = `{door:"branch",presence:20,blur:6,reach:92,wind:138,day:"empty",card:1,seed:20260712}`; `outs` = `["20%","6 px","0.9×","wind"]`; `hash` = `""`; `opacity` = `"0.2"`; `blur` = `"blur(6px)"`.

Then load `about:blank` → `http://localhost:8017/#presence=26&blur=9` and confirm `currentState().presence === 26 && currentState().blur === 9` (hash still wins).

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Retune defaults to the author's light: 20% presence, 6px softness, 0.92 reach, 1.38 wind

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 2: zh-TW localization

**Files:**
- Modify: `index.html` — `data-i18n` attributes on static copy, toggle button + CSS in the header, and an i18n block inside `<script id="playground">` (dictionary, `setLang`, dynamic re-render hooks, hash `lang` param, language-keyed `buildPrompt`).

**Interfaces:**
- Consumes: Task 1's defaults; existing playground functions `currentState`, `stateToHash`, `writeHash`, `applyHash`, `buildPrompt`, `shareURL`; engine listener order (engine first, playground later, i18n hooks last).
- Produces: `window.__playground.setLang(lang)` and `window.__playground.lang()` for tests; `#langToggle`.

- [ ] **Step 1: Tag static copy with `data-i18n`**

Add a `data-i18n="KEY"` attribute to each element below (attribute only — inner content unchanged; `setLang` swaps innerHTML from the dictionary):

| Element (current content identifies it) | key |
|---|---|
| `<div class="micro">樹影 · a komorebi prototype</div>` (header) | `micro` |
| `<h1>` in header | `h1` |
| `<p class="dek">` | `dek` |
| `<div class="micro">2026 · July</div>` (mock) | `mockMicro` |
| `<h2 class="mock-hero">12 July</h2>` | `mockHero` |
| `<div class="mock-sub">Saturday</div>` | `mockSub` |
| `<h2>The layer</h2>` (panel) | `panelTitle` |
| door group `<span class="name">door<span class="hz">門</span></span>` | `lblDoor` |
| `.seg[data-variant=branch] .en` (BRANCH) | `segBranch` |
| `.seg[data-variant=dapple] .en` (DAPPLE) | `segDapple` |
| `.seg[data-variant=drift] .en` (DRIFT) | `segDrift` |
| `<span class="name">presence<span class="hz">濃</span></span>` | `lblPresence` |
| `<span class="name">softness<span class="hz">柔</span></span>` | `lblSoftness` |
| `<span class="name">reach<span class="hz">幅</span></span>` | `lblReach` |
| `<span class="name">wind<span class="hz">風</span></span>` | `lblWind` |
| `<span class="name">the day<span class="hz">日色</span></span>` | `lblDay` |
| `<p class="sw-note">` | `swNote` |
| `<span class="name" style="font-size:14.5px">shadow falls on the content too</span>` | `lblWash` |
| `<div class="rail-cap" aria-hidden="true">` | `railCap` |
| `<p class="sr-note" id="takeNote">` | `takeNote` |
| `<div class="micro">Field notes — what the research says</div>` | `notesMicro` |
| the six `<article class="note">` elements, in DOM order | `note1` … `note6` |
| first `<p>` in `footer.page` | `footer1` |
| second `<p>` in `footer.page` | `footer2` |

- [ ] **Step 2: Header toggle button + CSS**

Inside `<header class="page wrap">`, immediately after the `<div class="micro" data-i18n="micro">…</div>` line, add:

```html
  <button class="lang-toggle" id="langToggle" aria-label="switch language">中文</button>
```

In the `<style>` block, after the `.micro{…}` rule, add:

```css
  header.page{position:relative}
  .lang-toggle{
    position:absolute; top:30px; right:24px; background:none; border:none; cursor:pointer;
    font-family:var(--sans); font-size:11px; font-weight:600; letter-spacing:.18em;
    text-transform:uppercase; color:var(--quiet); padding:4px 0;
  }
  .lang-toggle:hover{color:var(--accent)}
  @media (min-width:921px) and (max-height:1360px){ .lang-toggle{top:14px} }
```

(Note: `header.page` is inside `.wrap`, whose horizontal padding is 24px — `right:24px` aligns the toggle with the content edge... actually the header IS the `.wrap` element (`<header class="page wrap">`), so its own padding creates the 24px inset; `right:24px` aligns with the text column. Keep exactly `right:24px`.)

- [ ] **Step 3: Insert the i18n block into the playground script**

Inside `<script id="playground">`, insert the following block immediately AFTER the `function applyHash(){ … }` definition and BEFORE the `/* restore first, THEN start writing … */` comment with its `applyHash();` call. Placement is load-bearing: `applyHash()` (as modified in Step 4b) calls `setLang`, which reads the `const I18N` — inserting the block after the `applyHash();` call would throw a temporal-dead-zone ReferenceError on any `lang=zh-tw` URL.

IMPORTANT on the `en` strings: every `en` value in `I18N.ui` MUST be copied byte-verbatim from the element's current innerHTML in index.html (same line breaks, indentation, apostrophe characters). The `en` strings printed below are references; if any differs from the file, THE FILE WINS. The `zh` strings below are final — use them verbatim, do not re-translate.

Complete block:

```js
  /* ---------- i18n (en ↔ zh-TW) ----------
     The engine keeps writing its English captions/statelines/wind words;
     refreshDynamic() re-renders them from this dictionary AFTER the engine
     (listener order), so the engine stays untouched. Those few strings are
     deliberately duplicated here. */
  let currentLang = 'en';
  function L(){ return currentLang === 'zh-tw' ? 'zh' : 'en'; }

  const I18N = {
    title: { en:'樹影 — Tree-Shadow Prototype', zh:'樹影 — 木漏れ日原型' },
    ui: {
      micro: { en:'樹影 · a komorebi prototype', zh:'樹影 · 木漏れ日原型' },
      h1: { en:'<span class="hanzi">樹影</span>light through a tree, landing on paper',
            zh:'<span class="hanzi">樹影</span>光穿過樹，落在紙上' },
      dek: { en:'Three procedural doors for the Home screen’s tree-shadow backdrop, tuned to the\n  moodboard’s quietest reference — a branch so diffuse it barely resolves. The layer rides\n  <em>multiply</em> on the paper, it breathes with the wind, and it never reads as an image.',
             zh:'為主畫面的樹影背景準備的三扇程序生成之門，調校自情緒板上最安靜的一張參考——一枝淡到幾乎化開的樹枝。這層影以 <em>multiply</em> 落在紙上，隨風呼吸，永遠不會讓人覺得是一張貼上去的圖。' },
      mockMicro: { en:'2026 · July', zh:'2026 · 七月' },
      mockHero: { en:'12 July', zh:'7月12日' },
      mockSub: { en:'Saturday', zh:'星期六' },
      panelTitle: { en:'The layer', zh:'這層影' },
      lblDoor: { en:'door<span class="hz">門</span>', zh:'門<span class="hz">door</span>' },
      segBranch: { en:'branch', zh:'樹枝' },
      segDapple: { en:'dapple', zh:'光斑' },
      segDrift: { en:'drift', zh:'飄葉' },
      lblPresence: { en:'presence<span class="hz">濃</span>', zh:'濃度<span class="hz">presence</span>' },
      lblSoftness: { en:'softness<span class="hz">柔</span>', zh:'柔和<span class="hz">softness</span>' },
      lblReach: { en:'reach<span class="hz">幅</span>', zh:'幅度<span class="hz">reach</span>' },
      lblWind: { en:'wind<span class="hz">風</span>', zh:'風<span class="hz">wind</span>' },
      lblDay: { en:'the day<span class="hz">日色</span>', zh:'日色<span class="hz">the day</span>' },
      swNote: { en:'空 is the empty day — the shadow is all the surface has. A fed day\n        tints the shadow toward its own color, so the layer rides <em>inside</em> the atmosphere\n        rather than competing with it.',
                zh:'「空」是還沒有顏色的日子——紙上只有這層影。有顏色的日子會把影子染向自己的色調，讓它<em>融進</em>當天的空氣裡，而不是與之相爭。' },
      lblWash: { en:'shadow falls on the content too', zh:'樹影也落在內容上' },
      railCap: { en:'take it <br>home<span class="hz">土産</span>', zh:'帶光 <br>回家<span class="hz">土産</span>' },
      takeNote: { en:'The prompt carries your tuned values and the engine itself —\n      paste it into Claude Code inside your own repo and the light follows you home.',
                  zh:'這段 prompt 帶著你調好的參數和引擎本身——貼進你自己 repo 裡的 Claude Code，這道光就跟你回家。' },
      notesMicro: { en:'Field notes — what the research says', zh:'田野筆記——研究怎麼說' },
      note1: { en:'<h3>The phenomenon has a name<span class="hz">木漏れ日</span></h3>\n        <p><strong>Komorebi</strong> names not the light itself but the relationship — leaves,\n        light, and the movement between them. The implication: a static shadow texture is\n        not the thing. The layer must breathe, or it’s just a stain. A rule worth keeping: waiting things breathe, never freeze.</p>\n        <p class="src"><a href="https://wabisabi-jp.com/blogs/wabi-sabi-journal/komorebi">wabisabi-jp</a> ·\n        <a href="https://thursd.com/articles/komorebi-dance-of-sunlight-in-nature">thursd</a></p>',
               zh:'<h3>這個現象有名字<span class="hz">木漏れ日</span></h3>\n        <p><strong>Komorebi</strong> 指的不是光本身，而是那層關係——葉、光、以及兩者之間的搖動。言下之意：一張靜態的影子貼圖不是這回事。這層影必須呼吸，否則只是一塊漬。一條值得留下的規則：等待中的東西要呼吸，永不凍結。</p>\n        <p class="src"><a href="https://wabisabi-jp.com/blogs/wabi-sabi-journal/komorebi">wabisabi-jp</a> ·\n        <a href="https://thursd.com/articles/komorebi-dance-of-sunlight-in-nature">thursd</a></p>' },
      note2: { en:'<h3>Every dapple is a picture of the sun</h3>\n        <p>Each gap in a canopy is a <strong>pinhole camera</strong> projecting the sun’s disk —\n        that’s why dapples are round, and why they turn crescent during an eclipse. Softness grows\n        with canopy height. So the 隙 door punches <strong>round light spots out of shadow</strong>\n        rather than thresholding noise; it’s physically honest and it reads instantly.</p>\n        <p class="src"><a href="https://eclipse.aas.org/eye-safety/projection">AAS eclipse projection</a> ·\n        <a href="https://petapixel.com/2012/05/21/crescent-shaped-projections-through-tree-leaves-during-the-solar-eclipse/">PetaPixel</a> ·\n        <a href="https://www.edwardtufte.com/notebook/dappled-light/">Tufte, “Dappled light”</a></p>',
               zh:'<h3>每個光斑都是太陽的照片</h3>\n        <p>樹冠上的每個縫隙都是一台<strong>針孔相機</strong>，把太陽的圓盤投影下來——所以光斑是圓的，日食時它們會變成月牙。樹冠越高，邊緣越柔。因此「隙」這扇門是<strong>從影子裡打出圓形亮點</strong>，而不是對雜訊取閾值；物理上誠實，一眼就能讀懂。</p>\n        <p class="src"><a href="https://eclipse.aas.org/eye-safety/projection">AAS eclipse projection</a> ·\n        <a href="https://petapixel.com/2012/05/21/crescent-shaped-projections-through-tree-leaves-during-the-solar-eclipse/">PetaPixel</a> ·\n        <a href="https://www.edwardtufte.com/notebook/dappled-light/">Tufte, “Dappled light”</a></p>' },
      note3: { en:'<h3>The stage word is gobo</h3>\n        <p>Lighting designers shape light with a <strong>gobo</strong> — and tree-shadow gobos are\n        currently everywhere as background textures (Figma sets, overlay loops). That’s the risk:\n        a baked overlay reads as stock. the shadow must be <strong>procedural and of the day</strong> —\n        seeded, tinted by the day’s own palette, never a PNG.</p>\n        <p class="src"><a href="https://www.figma.com/community/file/1360553650737919328/20-background-shadow-textures-light-mode-dark-mode-gobo-shadows">Figma gobo set</a> ·\n        <a href="https://www.schoolofmotion.com/blog/designing-shadows">School of Motion</a></p>',
               zh:'<h3>舞台上的說法叫 gobo</h3>\n        <p>燈光設計師用 <strong>gobo</strong> 為光塑形——而樹影 gobo 正是眼下隨處可見的背景素材（Figma 套件、循環影片）。風險就在這裡：烘焙好的貼圖看起來像庫存素材。這裡的影子必須<strong>程序生成、屬於今天</strong>——有種子、被當日色調染色，永遠不是一張 PNG。</p>\n        <p class="src"><a href="https://www.figma.com/community/file/1360553650737919328/20-background-shadow-textures-light-mode-dark-mode-gobo-shadows">Figma gobo set</a> ·\n        <a href="https://www.schoolofmotion.com/blog/designing-shadows">School of Motion</a></p>' },
      note4: { en:'<h3>Wind is a sum of slow sines</h3>\n        <p>The standard trick (GPU Gems): layer <strong>sines of different frequencies</strong>,\n        let amplitude grow toward branch tips, add a whisper of high-frequency flutter for the\n        leaves. This prototype does exactly that per branch segment — cheap enough for a\n        60fps backdrop, and directly portable to SwiftUI’s <code>Canvas</code> or a Metal shader.</p>\n        <p class="src"><a href="https://developer.nvidia.com/gpugems/gpugems3/part-i-geometry/chapter-6-gpu-generated-procedural-wind-animations-trees">GPU Gems 3, ch. 6</a> ·\n        <a href="https://codepen.io/DienoX/pen/DEegxZ">procedural tree pen</a></p>',
               zh:'<h3>風是幾條慢正弦的和</h3>\n        <p>GPU Gems 的標準做法：疊加<strong>不同頻率的正弦</strong>，讓振幅往枝梢增長，再給葉子添一絲高頻的顫。這個原型對每段樹枝正是這麼做——便宜到能當 60fps 的背景，也能直接移植到 SwiftUI 的 <code>Canvas</code> 或 Metal shader。</p>\n        <p class="src"><a href="https://developer.nvidia.com/gpugems/gpugems3/part-i-geometry/chapter-6-gpu-generated-procedural-wind-animations-trees">GPU Gems 3, ch. 6</a> ·\n        <a href="https://codepen.io/DienoX/pen/DEegxZ">procedural tree pen</a></p>' },
      note5: { en:'<h3>Where it lives in the code</h3>\n        <p>Shadow <strong>generation math → a pure, seeded, unit-testable module</strong>\n        (like color extraction), <strong>tint &amp; compositing → the design-system layer</strong> —\n        never inline in feature views. Seed it from the date (e.g. month×31+day) so the same\n        day always wears the same tree.</p>',
               zh:'<h3>它住在程式碼的哪裡</h3>\n        <p>影子的<strong>生成數學 → 一個純粹、有種子、可單元測試的模組</strong>（就像取色），<strong>染色與合成 → 設計系統層</strong>——永遠不要散落在功能視圖裡。用日期當種子（例如 月×31+日），同一天永遠長同一棵樹。</p>' },
      note6: { en:'<h3>What the controls probe</h3>\n        <p>The open questions, made draggable: <strong>日色</strong> probes whether the shadow\n        competes with the app’s own day-color atmosphere or rides inside it; <strong>空</strong> is the\n        empty-day answer (procedural, always available); the <strong>content toggle</strong> asks\n        whether light falls on the whole wall or only behind the text.</p>',
               zh:'<h3>這些控制在探什麼</h3>\n        <p>把懸而未決的問題做成可拖動的：<strong>日色</strong>在問影子是與 App 當日的色彩空氣相爭、還是融於其中；<strong>空</strong>是空日子的答案（程序生成，永遠可用）；<strong>內容開關</strong>在問光是落在整面牆上、還是只落在文字後面。</p>' },
      footer1: { en:'Prototype only — three doors, one paper. Next step: pick a door, then port the winner as a\n  seeded generator behind a native canvas compositor (SwiftUI <code>Canvas</code>, Jetpack Compose,\n  or plain layers). The extraction door — lifting leaf silhouettes from the user’s own photos —\n  stays open as a second source feeding the same compositor.',
                 zh:'僅為原型——三扇門，一張紙。下一步：選定一扇門，把贏家移植成有種子的產生器，放在原生畫布合成器之後（SwiftUI <code>Canvas</code>、Jetpack Compose 或純圖層）。抽取之門——從使用者自己的照片裡取出葉影——仍然敞開，作為餵給同一個合成器的第二個光源。' },
      footer2: { en:'Open source at\n  <a href="https://github.com/funclosure/komorebi">github.com/funclosure/komorebi</a> ·\n  born from <a href="https://gist.github.com/funclosure/ed79bf5abfcf0d3ba1f6561615469c6b">the original gist</a>.',
                 zh:'開源於 <a href="https://github.com/funclosure/komorebi">github.com/funclosure/komorebi</a>·源自<a href="https://gist.github.com/funclosure/ed79bf5abfcf0d3ba1f6561615469c6b">最初的 gist</a>。' }
    },
    actions: {
      regrow: { title:{ en:'another tree 再生', zh:'再長一棵 再生' },
                aria:{ en:'another tree — regrow a new random tree', zh:'再長一棵樹——以新的隨機種子重生' } },
      copyPrompt: { title:{ en:'copy the prompt 持ち帰り', zh:'複製 prompt 持ち帰り' },
                    aria:{ en:'copy the prompt — carries your tuned values and the engine to your own project', zh:'複製 prompt——帶著你調好的參數與整個引擎，去你自己的專案' } },
      copyLink: { title:{ en:'copy this light 光を共有', zh:'複製這道光 光を共有' },
                  aria:{ en:'copy this light — a URL that reproduces this exact state', zh:'複製這道光——一條能重現此刻狀態的網址' } }
    },
    wind: { still:{en:'still',zh:'靜'}, breath:{en:'breath',zh:'息'}, breeze:{en:'breeze',zh:'微風'}, wind:{en:'wind',zh:'風'} },
    stateline: {
      empty: { en:'Still quiet — waiting for color.', zh:'還很安靜——等著顏色到來。' },
      fed:   { en:'Gathering — color is taking shape.', zh:'漸漸聚攏——顏色正在成形。' }
    },
    captions: {
      branch: { en:'枝影 — a bare branch entering from the light’s corner, the way the reference tile falls across a white wall.',
                zh:'枝影——一枝裸枝從光的角落斜入，就像參考圖裡那道落在白牆上的影。' },
      dapple: { en:'葉隙 — komorebi: every bright spot is a pinhole image of the sun, drifting as the canopy shifts.',
                zh:'葉隙——木漏れ日：每一個亮點都是太陽的針孔成像，隨樹冠搖動而漂移。' },
      drift:  { en:'疏影 — sparse leaf clusters adrift on the paper, the loosest and quietest of the three.',
                zh:'疏影——幾簇疏落的葉影漂在紙上，三扇門裡最鬆、最安靜的一扇。' }
    }
  };

  function refreshDynamic(){
    const lang = L();
    const s = currentState();
    document.getElementById('caption').textContent = I18N.captions[s.door][lang];
    document.getElementById('stateline').textContent = I18N.stateline[s.day === 'empty' ? 'empty' : 'fed'][lang];
    const w = s.wind / 100;
    const word = w === 0 ? 'still' : w < 0.5 ? 'breath' : w < 1.2 ? 'breeze' : 'wind';
    document.getElementById('windOut').textContent = I18N.wind[word][lang];
  }

  function setLang(lang){
    currentLang = lang;
    const k = L();
    document.documentElement.lang = lang === 'zh-tw' ? 'zh-Hant' : 'en';
    document.title = I18N.title[k];
    document.querySelectorAll('[data-i18n]').forEach(function(el){
      const entry = I18N.ui[el.dataset.i18n];
      if (entry) el.innerHTML = entry[k];
    });
    Object.keys(I18N.actions).forEach(function(id){
      const el = document.getElementById(id), a = I18N.actions[id];
      el.title = a.title[k];
      el.setAttribute('aria-label', a.aria[k]);
    });
    document.getElementById('langToggle').textContent = lang === 'zh-tw' ? 'EN' : '中文';
    refreshDynamic();
  }
```

- [ ] **Step 4: Wire the hash, listeners, and exports**

Four small edits inside the playground script:

a. In `stateToHash`, after the `for (const k of [...]) p.set(k, s[k]);` loop, add:

```js
    if (currentLang === 'zh-tw') p.set('lang', 'zh-tw');
```

b. In `applyHash`, at the END of the function body (after the seed handling), add:

```js
    if (p.get('lang') === 'zh-tw') setLang('zh-tw');
```

c. In the listener-attachment section (after the `document.getElementById('regrow').addEventListener('click', writeHash);` line), add:

```js
  /* i18n re-renders engine-written strings; these run after the engine's own listeners */
  document.querySelectorAll('.seg, .sw').forEach(function(b){ b.addEventListener('click', refreshDynamic); });
  document.getElementById('wind').addEventListener('input', refreshDynamic);
  document.getElementById('langToggle').addEventListener('click', function(){
    setLang(currentLang === 'zh-tw' ? 'en' : 'zh-tw');
    writeHash();
  });
```

d. Extend the export line to:

```js
  window.__playground = { currentState: currentState, stateToHash: stateToHash, applyHash: applyHash, buildPrompt: buildPrompt, shareURL: shareURL, setLang: setLang, lang: function(){ return currentLang; } };
```

- [ ] **Step 5: Language-keyed buildPrompt**

Replace the entire existing `function buildPrompt(s){ … }` (including its `DOOR_NOTES` dependency — DELETE the old `const DOOR_NOTES = {...};` block too) with:

```js
  const PROMPT = {
    doorNotes: {
      branch: { en:'枝 branch — a bare branch entering from the light’s corner',
                zh:'枝 branch——一枝裸枝從光的角落斜入' },
      dapple: { en:'隙 dapple — round light-gaps punched from shadow; every dapple is a pinhole image of the sun',
                zh:'隙 dapple——從影子裡打出的圓形光斑；每一個都是太陽的針孔成像' },
      drift:  { en:'葉 drift — sparse leaf clusters adrift on the surface',
                zh:'葉 drift——幾簇疏葉漂在表面上' }
    },
    lines: {
      en: {
        intro: 'Add a komorebi (木漏れ日) tree-shadow backdrop to this project — dappled leaf-shadow falling across the UI the way light through a tree lands on a wall.',
        tuned: 'I tuned the effect in the komorebi playground; these exact values are the required defaults:',
        door: '- door: ', presence: '- presence: ', presenceSuf: '% (layer opacity)',
        blur: '- softness: ', blurSuf: 'px (blur — the penumbra)',
        reach: '- reach: ', reachSuf: '× (scale)',
        wind: '- wind: ', windSuf: ' (0 = still)',
        day: '- day tint: ', daySuf1: ' (shadow ink ', daySuf2: ')',
        card: '- shadow falls on the content too: ', cardYes: 'yes', cardNo: 'no — behind content only',
        seed: '- seed: ', link: '- this exact state, live: ',
        rulesHead: 'How to apply it (rules from the reference — not negotiable):',
        rule1: '1. Detect my platform and design system from this repo (plain web, React, SwiftUI, Compose, …) and implement natively — sharp vector shapes on a 2D canvas; no WebGL, no image assets.',
        rule2: '2. Port the generator as a pure seeded module (unit-testable). Keep my seed as its default; consider seeding from the date (e.g. month×31+day) so the same day always wears the same tree.',
        rule3: '3. Compositing belongs in the design-system layer, never inline in feature views: blur is the penumbra, a multiply blend so the layer can only darken (shadow, not gray paint), opacity is the presence.',
        rule4: '4. Overdraw ~48px past the visible edge and clip it away, so the blur never vignettes.',
        rule5: '5. Respect prefers-reduced-motion: freeze to a still frame.',
        bar: 'The acceptance bar — "barely there": the layer must never read as an image pasted behind the app, and the motion must read as breathing, never flapping. If a first-time viewer can tell HOW it is made, it is too loud; if they just feel late-afternoon light in the room, it is right.',
        engineHead1: 'Reference engine — exactly the code I was watching in the playground. Implement ONLY the "',
        engineHead2: '" door; the other two are context, drop them. The window.__komorebi hook at the bottom is playground plumbing — omit it.'
      },
      zh: {
        intro: '為這個專案加上一層木漏れ日（komorebi）樹影背景——斑駁的葉影落在 UI 上，就像光穿過樹、落在牆上。',
        tuned: '我在 komorebi 遊樂場調好了這個效果；以下數值就是必須採用的預設值：',
        door: '- 門：', presence: '- 濃度：', presenceSuf: '%（圖層不透明度）',
        blur: '- 柔和：', blurSuf: 'px（模糊——半影）',
        reach: '- 幅度：', reachSuf: '×（縮放）',
        wind: '- 風：', windSuf: '（0 = 靜止）',
        day: '- 日色：', daySuf1: '（影子墨色 ', daySuf2: '）',
        card: '- 樹影也落在內容上：', cardYes: '是', cardNo: '否——只落在內容後面',
        seed: '- 種子：', link: '- 此刻狀態的連結：',
        rulesHead: '如何套用（參考實作的規則——不可妥協）：',
        rule1: '1. 從這個 repo 判斷我的平台與設計系統（純網頁、React、SwiftUI、Compose……），用原生方式實作——2D 畫布上的銳利向量圖形；不需要 WebGL，不需要圖片素材。',
        rule2: '2. 把產生器移植成純粹、有種子的模組（可單元測試）。保留我的種子作為預設；考慮用日期當種子（例如 月×31+日），同一天永遠長同一棵樹。',
        rule3: '3. 合成屬於設計系統層，永遠不要散落在功能視圖裡：模糊是半影，multiply 混合讓這層只能變暗（是影子，不是灰漆），不透明度是存在感。',
        rule4: '4. 向可見邊緣外多畫約 48px 再裁掉，模糊才不會在邊緣露餡。',
        rule5: '5. 尊重 prefers-reduced-motion：凍結成靜止的一幀。',
        bar: '驗收標準——「若有似無」：這層影永遠不能讓人覺得是一張貼在 App 後面的圖，而且動起來要像呼吸，永遠不能像拍打。如果第一次看到的人說得出它是怎麼做的，就太吵了；如果他們只是覺得房間裡有午後的光，就對了。',
        engineHead1: '參考引擎——正是我在遊樂場裡看著跑的程式碼。只實作「',
        engineHead2: '」這扇門；另外兩扇是脈絡，可刪。底部的 window.__komorebi 是遊樂場的水電——略過。'
      }
    }
  };

  function buildPrompt(s){
    const k = L(), t = PROMPT.lines[k];
    const engine = document.getElementById('engine').textContent.trim();
    return [
      t.intro, '', t.tuned, '',
      t.door + PROMPT.doorNotes[s.door][k],
      t.presence + s.presence + t.presenceSuf,
      t.blur + s.blur + t.blurSuf,
      t.reach + (s.reach/100).toFixed(2) + t.reachSuf,
      t.wind + (s.wind/100).toFixed(2) + t.windSuf,
      t.day + s.day + t.daySuf1 + DAY_INKS[s.day] + t.daySuf2,
      t.card + (s.card ? t.cardYes : t.cardNo),
      t.seed + s.seed,
      t.link + shareURL(s),
      '', t.rulesHead, t.rule1, t.rule2, t.rule3, t.rule4, t.rule5,
      '', t.bar, '',
      t.engineHead1 + s.door + t.engineHead2,
      '', '```js', engine, '```', ''
    ].join('\n');
  }
```

(Note: the old `DOOR_NOTES` const must be deleted; `DAY_INKS` stays.)

- [ ] **Step 6: Verify**

All loads via about:blank; local server http://localhost:8017.

a. **Pristine EN:** load `/`; evaluate:
```js
() => ({ lang: window.__playground.lang(), htmlLang: document.documentElement.lang || 'en',
  toggle: document.getElementById('langToggle').textContent, hash: location.hash,
  dekEn: document.querySelector('.dek').textContent.includes('procedural doors') })
```
Expected: `lang` `"en"`, `toggle` `"中文"`, `hash` `""`, `dekEn` `true`.

b. **Toggle round-trip:** evaluate:
```js
async () => {
  document.getElementById('langToggle').click();
  await new Promise(r => setTimeout(r, 300));
  const zh = {
    lang: window.__playground.lang(), htmlLang: document.documentElement.lang,
    title: document.title, dekZh: document.querySelector('.dek').textContent.includes('程序生成'),
    note1: document.querySelector('[data-i18n="note1"] h3').textContent.includes('這個現象有名字'),
    windOut: document.getElementById('windOut').textContent,
    caption: document.getElementById('caption').textContent.includes('枝影——'),
    stateline: document.getElementById('stateline').textContent.includes('還很安靜'),
    tooltip: document.getElementById('copyPrompt').title,
    hashLang: location.hash.includes('lang=zh-tw')
  };
  document.getElementById('langToggle').click();
  await new Promise(r => setTimeout(r, 300));
  const restored = document.querySelector('.dek').textContent.includes('procedural doors')
    && document.getElementById('caption').textContent.includes('the way the reference tile')
    && document.getElementById('windOut').textContent === 'wind'
    && !location.hash.includes('lang');
  return { zh, restored, backToEn: document.documentElement.lang === 'en' };
}
```
Expected: `zh.lang` `"zh-tw"`, `zh.htmlLang` `"zh-Hant"`, `zh.title` `"樹影 — 木漏れ日原型"`, `zh.dekZh`/`zh.note1`/`zh.caption`/`zh.stateline` all `true`, `zh.windOut` `"風"`, `zh.tooltip` `"複製 prompt 持ち帰り"`, `zh.hashLang` `true`, `restored` `true`, `backToEn` `true`. (Semantic restore checks, not byte-equality — the en dictionary strings are file-verbatim per Step 3, but the assertion shouldn't depend on that.)

c. **zh URL loads Chinese:** load `/#door=dapple&presence=26&lang=zh-tw`; evaluate:
```js
() => ({ lang: window.__playground.lang(), presence: window.__playground.currentState().presence,
  caption: document.getElementById('caption').textContent.includes('葉隙——') })
```
Expected: `lang` `"zh-tw"`, `presence` `26`, `caption` `true`.

d. **zh prompt:** on that page click `#copyPrompt`; evaluate:
```js
() => ({ zhIntro: window.__lastCopied.includes('為這個專案加上一層木漏れ日'),
  zhBar: window.__lastCopied.includes('若有似無'),
  engine: window.__lastCopied.includes('function mulberry32'),
  values: window.__lastCopied.includes('- 濃度：26%'),
  link: window.__lastCopied.includes('lang=zh-tw') })
```
Expected: all `true`.

e. **EN prompt regression:** load `/` fresh, click `#copyPrompt`; evaluate `() => window.__lastCopied.includes('- presence: 20%') && window.__lastCopied.includes('barely there') && window.__lastCopied.includes('function mulberry32')` → `true`.

f. **Layout sanity:** at 1440×900 the toggle is visible top-right (`document.getElementById('langToggle').getBoundingClientRect().top < 60`), fit checks still pass (phone + `#copyLink` bottoms ≤ vh); at 390×844 no horizontal overflow in zh mode (toggle to zh first). Take a zh-mode screenshot at 1440×900 and eyeball for overflow/typography issues; do not commit it.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "Add zh-TW: runtime language toggle, localized copy and prompt

繁體中文 via an I18N dictionary in the playground script — data-i18n swaps
for static copy, after-engine listeners re-render dynamic strings, lang=zh-tw
rides the hash, and the copied Claude Code prompt is fully localized (engine
code identical in both languages).

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 3: Deploy and live verification

**Files:**
- Modify: `README.md` — one line.

**Interfaces:**
- Consumes: Tasks 1–2 commits on `main`.
- Produces: live site with defaults + zh-TW.

- [ ] **Step 1: README line**

In `README.md`, after the "**Play with it: …**" line, add:

```markdown
繁體中文：頁面右上角有 EN／中文 切換。
```

Commit:
```bash
git add README.md
git commit -m "README: note the zh-TW toggle

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

- [ ] **Step 2: Push and wait**

```bash
git push origin main
for i in $(seq 1 30); do
  curl -s https://funclosure.github.io/komorebi/ | grep -q 'langToggle' && { echo updated; break; }
  sleep 10
done
```
Expected: `updated`.

- [ ] **Step 3: Live check**

Playwright at 1440×900 (about:blank first): load `https://funclosure.github.io/komorebi/`; run Task 2 Step 6a (expect EN defaults incl. `currentState().presence === 20`); click `#langToggle`, confirm `document.documentElement.lang === 'zh-Hant'` and `location.hash.includes('lang=zh-tw')`; click `#copyPrompt`, confirm `window.__lastCopied.includes('為這個專案加上一層木漏れ日')`. Then load `https://funclosure.github.io/komorebi/#lang=zh-tw` fresh and confirm it renders Chinese immediately (`window.__playground.lang() === 'zh-tw'`).
