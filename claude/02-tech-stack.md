# 02 — Technology Stack

**Target:** Telegram Web A — `https://web.telegram.org/a/`, app version `12.0.38`, deployment built `2026-08-11 15:24:14 UTC`.
**Audit date:** 2026-08-14.
**Evidence base:** live authenticated session (headless Chromium, real account), passive fetch of all 461 JS / 25 CSS / 4 WASM assets plus 453 source maps, and direct reading of the public GPLv3 source repo `Ajaxy/telegram-tt` @ `d915b1b9ae856eb9eae737d1816d9c05df66b1d3`.

Every substantive claim below carries one of four tags:

- **Confirmed** — directly visible in source, headers, requests, or runtime behaviour observed in this audit.
- **Strong inference** — supported by multiple independent signals.
- **Possible** — plausible, not sufficiently verified.
- **Unknown** — cannot be determined from the access we had.

Full row-by-row evidence with verbatim quotes: `10-evidence-log.md`.

---

## 1. Headline stack table

| Layer | Technology | Version | Confidence | Evidence (verbatim) |
|---|---|---|---|---|
| Language | TypeScript | `^6.0.3` (declared) | Confirmed | `package.json` devDeps `"typescript": "^6.0.3"`; 917 `.tsx` + 836 `.ts` under `src/`; 2,056 source paths recovered from maps |
| Type config | `strict: true`, `target: esnext`, `moduleResolution: bundler` | — | Confirmed | `tsconfig.base.json` |
| JSX runtime | **Teact** (not React) | in-repo, ~2,800 LOC | Confirmed | `tsconfig.base.json` → `"jsxImportSource": "@teact"`; `assets/teact-CUUQAb7N.js` (12,259 B) |
| UI framework | **Teact** — in-house VDOM | — | Confirmed | bundle strings `teactFastList`, `teactOrderKey`, `TeactNContainer`, `TeactMemoWrapper`, `DEBUG_contentComponentName` |
| State store | **TeactN** — in-house flat global store | 397 LOC | Confirmed | `src/lib/teact/teactn.tsx`; `assets/reducers-BlsyOM6t.js` (55,351 B), `assets/initial-CskBLhZ6.js` (52,944 B) |
| Bundler | **Vite on Rolldown** (Rust) | `vite: ^8.2.1` | Confirmed | `assets/rolldown-runtime-hePW80VL.js` (716 B); `__vite__mapDeps`; `vite:preloadError`; `vite.config.ts` imports types from `'rolldown'` |
| Minifier | **OXC minifier** (Rolldown's) | Unknown | Strong inference | every string literal rewritten to a backtick template (`` `#3390EC` ``, `` `gramjs` ``) + single-letter identifiers — Terser preserves quote style |
| CSS compile | SCSS → CSS Modules → **LightningCSS** | `sass: ^1.101.0` | Confirmed | `html.theme-dark{--lightningcss-light: ;--lightningcss-dark:initial;color-scheme:dark}` in `index-CTuaTAxZ.css`; 358 `*.module.scss` in repo |
| CSS scoping | CSS Modules, `[hash:base64:8]` | — | Confirmed | `vite.config.ts` `generateScopedName: '[hash:base64:8]'`; live `<body class="Y7owXZmb …">`, composer `class="… NKP0M5xy ProseMirror">` |
| Cascade control | CSS Cascade Layers | — | Confirmed | inline `<style>` in `index.html`: `@layer reset, variables, ui, components;` |
| Protocol client | **GramJS**, custom fork | in-repo, 46,119 LOC | Confirmed | `baseLogger:\`gramjs\``, `langPack:\`weba\``, `$t=\`GramJs:apiCache:2\`` |
| Wire protocol | MTProto, **TL layer 227** | 227 | Confirmed | `new z.InvokeWithLayer({layer:227,query:new z.InitConnection({apiId:…` |
| Transport | Obfuscated MTProto over **WSS** | — | Confirmed | live `wss://zws1.web.telegram.org/apiws` (512 events); `getWebSocketLink` builds `wss://${ip}:${port}/apiws` |
| Fallback transport | MTProto HTTP long-poll (`http_wait`) | — | Confirmed | `HttpConnection` → `shouldLongPoll = true`; `Api.HttpWait({maxDelay, waitAfter, maxWait})` |
| Crypto | `@cryptography/aes` (AES-IGE / AES-CTR) + WebCrypto | `^0.1.1` | Confirmed | `src/lib/gramjs/crypto/{IGE,CTR,RSA,AuthKey,Factorizator}`; `crypto.subtle` in `compatTest.js` gate list |
| Rich-text editor | **TipTap 3 / ProseMirror** | `@tiptap/core ^3.29.0` | Confirmed | `this.className=\`tiptap\``; `animation: ProseMirror-cursor-blink 1.1s steps(2, start) infinite;`; 27 `@tiptap/*` + 13 `prosemirror-*` package paths in maps |
| Syntax highlighting | `highlight.js` + in-house `src/lib/hljs-tl` fork; `lowlight ^3.3.0` bridge | — | Confirmed | 375 `node_modules/highlight.js/…` paths in maps; 373 single-grammar chunks |
| Math rendering | `temml` | `^0.13.3` | Confirmed | `assets/Temml-Local-ChnUH04l.css` (5,474 B) |
| Sticker/Lottie renderer | **tlottie** (rlottie fork) via WASM | — | Confirmed | `assets/tlottie-HZNSEMV6.wasm` (400,534 B); `m.tlottie_render_alpha8_color(…)` |
| Language ID | **fastText** via WASM | — | Confirmed | `assets/fasttext-wasm-zrRkeJ3U.wasm` (1,122,181 B); `fasttext.worker--naZEx7i.js` |
| Voice codec | `opus-recorder` (git pin `Ajaxy/opus-recorder#116830a`) | — | Confirmed | `_opus_encoder_ctl = asm["i"]`; `decoderWorker.min.wasm` (137,424 B) |
| Calls | Hand-rolled WebRTC, `src/lib/vibecalls` | — | Confirmed | `new RTCPeerConnection({iceServers:…})`; raw SDP: `` d(`m=application ${+!t} UDP/DTLS/SCTP webrtc-datachannel`) `` |
| Charts | `lovely-chart` (Telegram's own) | `^2.0.2` | Confirmed | `assets/LovelyChart-_d0rNxo5.js` (50,025 B) |
| Popover positioning | `@floating-ui/dom` | — | Confirmed | `node_modules/@floating-ui/{dom,core,utils}` in maps |
| Audio metadata | `music-metadata` | `^11.13.0` | Confirmed | 91 `music-metadata` files in maps + tokenizer stack (`strtok3`, `token-types`) |
| QR login | `qr-code-styling` | `^1.9.2` | Confirmed | `assets/qr-code-styling-DyCNGGn8.js` (46,552 B) |
| Emoji dataset | `emoji-data-ios` (git pin `#30529a2`) | — | Confirmed | `assets/emoji-data-Cp82hEFa.js` (55,591 B) |
| Colour maths | `colorjs.io` | `^0.6.1` | Confirmed | `src/util/switchTheme.ts` interpolates every theme var over 200 ms |
| Compression (JS) | `fflate` | `^0.8.3` | Confirmed | listed in `package.json` deps; `.tgs` files are gzipped Lottie |
| Persistence | IndexedDB via `idb-keyval` + Cache Storage + `localStorage` | `idb-keyval ^6.2.5` | Confirmed | live: IDB `tt-data` v1; caches `tt-media`, `tt-media-avatars`, `tt-media-progressive`, `tt-lang-packs-v52`, `tt-assets` |
| Desktop shell | **Tauri v2** — bridge compiled into the web bundle | `@tauri-apps/api ^2.11.1` | Confirmed | `node_modules/@tauri-apps/api` + plugins `shell`, `notification`, `updater`, `process` in maps |
| Service worker | Hand-written, ES module, 6 source files | — | Confirmed | `service.worker-BSeu-kQn.js` (9,609 B); registered with `{type:\`module\`}`; **zero `workbox` strings** |
| Web server | **nginx** | `1.30.1` | Confirmed | `server: nginx/1.30.1` on every response |
| Wire protocol (HTTP) | HTTP/2, gzip only | — | Confirmed | negotiated h2; no `alt-svc`; `Accept-Encoding: br` → identity |
| Hosting | Self-hosted, **AS62041 Telegram Messenger Inc**, Amsterdam | — | Confirmed | A `149.154.167.99`, AAAA `2001:67c:4e8:f004::9`; `x-served-by: meta4240719` rotating |
| License | **GPL-3.0-or-later** | — | Confirmed | `LICENSE` = unmodified GPLv3 (674 lines); `package.json` `"license": "GPL-3.0-or-later"`, author Alexander Zinchuk |

---

## 2. Language and type system

**Confirmed.** TypeScript end to end, `strict: true`, no JS source files in `src/` other than three vendored helpers (`fastBlur.js`, `punycode.js`, `twemojiRegex.js`). Measured size: 362,996 LOC of `.ts` + `.tsx` under `src/`, of which `src/components/` is 183,963 and `src/lib/gramjs/` is 46,119.

Compiler configuration that actually matters:

```json
"target": "esnext", "lib": ["dom","webworker","esnext"], "strict": true,
"module": "preserve", "moduleResolution": "bundler", "noEmit": true,
"jsx": "react-jsx",
"jsxImportSource": "@teact",
"paths": { "@teact": ["./src/lib/teact/teact.ts"], "@gili/*": ["./src/components/gili/*"] }
```

`"noEmit": true` means TypeScript is a checker only — Vite/Rolldown does the transform. `"jsxImportSource": "@teact"` is the single line that redirects every JSX element in the codebase away from React. **Confirmed.**

Engine floor is aggressive: `"node": "^24.11 || ^26"`, `"npm": "^11 || ^12"`, browserslist `"baseline widely available"`. **Confirmed.**

Three custom/pinned ESLint plugins encode architectural invariants as lint errors — `eslint-plugin-no-null` (the codebase forbids `null`), `eslint-plugin-react-hooks-static-deps` (dep arrays must be statically analysable, required because Teact hooks are cursor-based), `eslint-plugin-tt-multitab` (enforces `tabId` threading). Stylelint adds `stylelint-high-performance-animation` (blocks animating non-compositable properties) and `@mytonwallet/stylelint-whole-pixel`. **Confirmed.**

---

## 3. Teact — the UI framework

### 3.1 What it is

**Confirmed.** Teact is Telegram's own React reimplementation, ~2,800 LOC across seven files:

| File | LOC |
|---|---:|
| `teact.ts` | 1,109 |
| `teact-dom.ts` | 1,009 |
| `teactn.tsx` (the store) | 397 |
| `dom-events.ts` | 188 |
| `heavyAnimation.ts` | 73 |
| `jsx-runtime.ts` | 18 |
| `jsx-dev-runtime.ts` | 1 |

React is a **devDependency only** (`react ^19.2.7`, `@types/react ^19.2.17`); `@types/react` exists purely to supply ambient JSX and event types. The shipped bundle contains **zero** occurrences of `react`, `react-dom`, `preact`, or `__REACT_DEVTOOLS_GLOBAL_HOOK__`. **Confirmed.**

The repo's own agent-instruction file states the rule flatly:

```
* All components use JSX and render with Teact.
* Do not import "react". React types are available globally in React namespace (e.g. React.MouseEvent).
* Built-in hooks live in Teact library. Import them from there.
```

### 3.2 Why it exists — be precise about the sequence

**Confirmed (stated origin).** The README says:

> "According to the original contest rules, it has nearly zero dependencies and is fully based on its own Teact framework (which re-implements React paradigm)."

The rule it refers to is Telegram's JavaScript Contest brief: *"create a simplified web version of Telegram without using third-party UI frameworks."* Telegram later clarified (2025-02-04) that the rule means *"you can use any existing dependencies. You just can't add new ones."*

**Do not invert this sequence.** There is **no** interview, conference talk, HN thread or blog post in which the author explains "why not React" on technical grounds — we searched HN via the Algolia API for `Teact`, `Telegram Web Z`, `Telegram Web A`, `telegram-tt`, and the author's two listed conference talks both predate the contest. **Confirmed absence within the sources we checked.** Any narrative claiming "they benchmarked React, found it too slow, therefore wrote Teact" is unsupported.

A Teact-vs-React-18-vs-Preact-10 benchmark does exist and is runnable in-browser at `ajaxy.github.io/teact/benchmark/`, but results are computed client-side at page load and **no numbers are published anywhere**. It also postdates the framework (Teact LICENSE dated 2022; contest 2019–2020), so it is validation, not justification. **Confirmed.** Actionable: a team evaluating this can run it and produce first-party numbers rather than citing anyone's claim.

**Strong inference** on *retention*: the origin rationale and the retention rationale are different. Teact has since been specialised in ways that are genuinely hard to retrofit onto React, and the "no dependencies" policy has visibly lapsed anyway (35 runtime deps today, including a whole third-party editor framework). Teact is now maintained because it exists and is deeply coupled, not because dependencies are banned.

### 3.3 How it differs from React, concretely

All **Confirmed** from source reading.

| Area | React | Teact |
|---|---|---|
| Virtual node kinds | fiber tags | 5-member enum: `Empty, Text, Tag, Component, Fragment` |
| Host node reference | on the fiber | **on the virtual element itself** (`VirtualElementTag.target` is the live DOM node) |
| Hook storage | one linked list | **four parallel cursor arrays** — `state`, `effects`, `memos`, `refs`, each with its own cursor |
| `memo` | `Object.is` per prop | `useMemo(() => createElement(C, props), Object.values(props))` — **positional** shallow compare over `Object.values` |
| Error handling | error boundaries | **none.** `renderComponent` runs in `safeExec` whose `rescue` reuses the previous rendered value: `console.error('[Teact] Error while rendering component …'); newRenderedValue = componentInstance.renderedValue;` |
| `useState` | batched queue | **double-buffered** — setter writes `nextValue`; `prepareComponentForFrame()` copies `nextValue → value` at the start of the update pass. Same-tick reads see the old value; functional updaters see `nextValue` |
| Keyed lists | keyed diff by default | **index-matched by default**; keyed reconciliation is opt-in via `teactFastList` + `teactOrderKey` |
| Context | re-renders the subtree | **implemented as signals** — `createContext` stashes a signal; consumers opt in via `useDerivedState`. Context change does **not** cascade renders |
| Signals | none | **first-class in the dep array** — a signal in `useEffect` deps makes the effect a push-based subscriber that never re-renders the component |
| Scheduling | React's own scheduler | RAF pass with an **explicitly documented order**, that defers itself while a heavy animation runs |
| Events | synthetic event system | delegation to `document` for all bubbleable non-capture listeners, with React quirks deliberately reproduced (`change` → `input` for non-`SELECT`; `focus` → `focusin`) |
| Unmount | GC handles it | **manual GC** — `helpGc()` nulls every hook slot, `$element`, `renderedValue`, `onUpdate`, because the virtual tree holds direct DOM pointers |

The scheduling order is documented in the source verbatim:

```
/*
  Order:
  - component effect cleanups
  - component effects
  - measure tasks
  - mutation tasks
  - component updates
  - component layout effect cleanups
  - component layout effects
  - forced layout measure tasks
  - forced layout mutation tasks
 */

const runUpdatePassOnRaf = throttleWith(requestMeasure, () => {
  if (getIsBlockingAnimating()) {
    getIsBlockingAnimating.once(runUpdatePassOnRaf);
    return;
  }
```

That `getIsBlockingAnimating()` check is the load-bearing difference. **The framework's update pass, the store's container-mapping pass, IndexedDB cache writes, and IntersectionObserver callbacks all voluntarily stall while an animation is in flight.** You cannot express that with React's scheduler. **Confirmed.**

Effects are keyed by `componentInstance.id * MAX_EFFECT_CURSORS_PER_INSTANCE + cursor` with `MAX_EFFECT_CURSORS_PER_INSTANCE = 10000`, sorted so higher (deeper/newer) instance ids run first — that is how "child to parent" effect ordering is guaranteed. **Confirmed.**

### 3.4 The companion libraries that make it work

- **`fasterdom`** (`assets/fasterdom-vRnhxblf.js`, 19,666 B, on the critical path). Read/write phase batching: `requestMeasure`, `requestMutation`, `requestNextMutation`, `requestForcedReflow`, flushed once per RAF, sequenced through promise microtasks with the comment `// We use promises to provide correct order for Mutation Observer callback microtasks`. **Confirmed.**
- **`stricterdom`** — the enforcement half. `enableStrict()` runs in DEBUG builds and **throws** when code reads layout in the mutate phase or writes in the measure phase. This is a runtime linter for layout thrashing. **Confirmed.**
- **`heavyAnimation.ts`** — `beginHeavyAnimation(duration, isBlocking)`, `onFullyIdle(cb)`, `throttleWithFullyIdle(fn)`, `AUTO_END_TIMEOUT = 1000`. **Confirmed.**
- **Signals** (`src/util/signals.ts`, ~90 LOC) — `createSignal<T>()` returning `[getter & {subscribe, once}, setter]`, identity-guarded, with a global `unsubscribesByEffect` registry. **Confirmed.**

**One licensing fact that matters more than any other in this section:** Teact is separately extracted at `Ajaxy/teact` under **MIT** (`LICENSE`: "MIT License / Copyright (c) 2022 Alexander Zinchuk", `package.json` `"license": "MIT"`, zero dependencies). The Telegram application around it is GPLv3. **The rendering framework can be lifted; the app cannot.** **Confirmed.**

---

## 4. Build pipeline

### 4.1 It is Vite on Rolldown, not webpack, not stock Rollup

This is the headline structural finding of the bundle analysis.

| Marker | Files containing it | Meaning |
|---|---:|---|
| `__webpack_require__` | **0** | webpack fully absent |
| `webpackChunk` / `webpackChunkName` | **0** | no webpack magic comments |
| `esbuild` | 0 | — |
| `rolldown` | 28 | dedicated runtime chunk |
| `__vite__mapDeps` | 10 | Vite's chunk-dependency map preserved |
| `vite:preloadError` | 1 | Vite's `preload-helper` |
| `--lightningcss-light` / `--lightningcss-dark` | 1 | LightningCSS `light-dark()` lowering |

**The entire bundler runtime is 716 bytes** (`assets/rolldown-runtime-hePW80VL.js`), exporting three helpers:

```js
var e=Object.create,t=Object.defineProperty,n=Object.getOwnPropertyDescriptor,r=Object.getOwnPropertyNames,
i=Object.getPrototypeOf,a=Object.prototype.hasOwnProperty,o=(e,t)=>()=>(t||(e((t={exports:{}}).exports,t),
e=null),t.exports),s=(e,n)=>{ ... };export{s as n,l as r,o as t};
```

A webpack runtime for an app this size is typically 8–15 KB. Chunk loading is **native ESM `import()`** — no JSONP loader, no `__webpack_require__.e`, no script injection. **Confirmed.**

Vite's fingerprints survive intact. `__vite__mapDeps` sits at the very top of the entry chunk:

```js
const __vite__mapDeps=(i,m=__vite__mapDeps,d=(m.f||(m.f=["./core-DFiqCqIj.js","./rolldown-runtime-hePW80VL.js",
"./dist-js-Pjhpf2xF.js","./dist-js-Cw4ugO5N.js","./window-R9mtfgf9.js","./event-Cg2-4Juw.js",
"./dist-js-Cq0Z38wh.js","./qr-code-styling-DyCNGGn8.js"])))=>i.map(i=>d[i]);
```

…and `preload-helper-HclGiUj8.js` is `__vitePreload` verbatim, down to the event name:

```js
function o(e){let t=new Event(`vite:preloadError`,{cancelable:!0});if(t.payload=e,window.dispatchEvent(t),
!t.defaultPrevented)throw e}
```

It also reads `document.querySelector('meta[property=csp-nonce]')` for nonce propagation, though no nonce meta is present in the served HTML. **Confirmed.**

### 4.2 CSS pipeline

LightningCSS is in the chain — its `light-dark()` polyfill toggle appears once in `index-CTuaTAxZ.css`:

```css
html.theme-dark{--lightningcss-light: ;--lightningcss-dark:initial;color-scheme:dark}
```

**Confirmed.** The repo also declares `sass ^1.101.0`, `postcss`, `postcss-load-config` and `autoprefixer`, and 358 `*.module.scss` files — so the authored language is SCSS, compiled and then processed by LightningCSS. **Confirmed.**

### 4.3 Minification

**Strong inference (not Confirmed).** Identifiers are single-letter and *all* string literals are rewritten to backtick template literals — `` `#3390EC` ``, `` `gramjs` ``, `` `tlottie-canvas` ``. Terser and esbuild both preserve quote style; this rewriting is characteristic of the Rolldown/OXC minifier. We did not find a version string for the minifier itself. **Unknown:** exact OXC version.

### 4.4 Build-time hardening and codegen

From `vite.config.ts`, all **Confirmed**:

- CSP is **generated in the build config** and injected into both dev-server headers and the HTML `<meta>`, so dev and prod cannot drift.
- `sourcemap: true` in production — source maps are deliberately published.
- Credentials are mandatory outside tests: `if (appEnv !== 'test' && (!telegramApiId || !telegramApiHash)) { throw new Error('Missing required Telegram API credentials'); }`.
- A `watchAndRun` plugin regenerates codegen artifacts on save: localisation types (`npm run lang:ts` on `fallback.strings`), the TL schema (`npm run gramjs:tl`), and the icon font (`npm run icons:build`). A missing i18n key is therefore a **compile error**, not a runtime `???`.
- `rollup-plugin-bundle-stats` runs in CI against a baseline, and a custom `createWorkerBundleCollectorPlugin` merges *worker* bundles into the report so worker size is not invisible. `[Size]` is a first-class commit tag.

### 4.5 Asset URL and hashing scheme

```
/a/index.html                          (unhashed, 1h TTL)
/a/assets/<chunkName>-<hash8>.js       e.g. teact-CUUQAb7N.js
/a/assets/<chunkName>-<hash8>.css      e.g. index-CTuaTAxZ.css
/a/assets/<name>-<hash8>.wasm          e.g. tlottie-HZNSEMV6.wasm
/a/<workerName>-<hash8>.js             workers live at ROOT, not under assets/
/a/version.txt, /a/redirect.js, /a/compatTest.js   (unhashed)
```

Hash is 8 chars from the base64url alphabet (`IZ97MA_m`, `-2DJCARP`, `_d0rNxo5`). **Chunk names are derived from the first/primary module in the chunk and are not reliable content indicators** — `usePrevious-BOJ4DmJ9.js` is 40,966 B of unrelated code, `useConnectionStatus-DLbED6Q4.js` is 480,535 B. Name collisions are resolved by hash alone: two `core-*.js`, two `main-*.js`, two `calls-*.js`, two `Modal-*.css`. Some chunks carry a doubled extension (`arduino.js-DB177i_z.js`, `cpp.js-DBV_MYDV.js`) because the source module name `arduino.js` is used verbatim as the chunk name. **Confirmed.**

---

## 5. Module graph and chunking strategy — the real numbers

### 5.1 Totals

| Class | Files | Raw | Transfer (gzip) |
|---|---:|---:|---:|
| JavaScript | **461** | **5,532,659 B (5.28 MiB)** | **2,194,436 B (2.09 MiB)** |
| CSS | 25 | 850,207 B (830.3 KiB) | 236,707 B (231.2 KiB) |
| WASM | 4 | 1,832,565 B (1.75 MiB) | *served uncompressed* |
| **JS + CSS** | **486** | **6,382,866 B (6.09 MiB)** | **2,431,143 B (2.32 MiB)** |

**Confirmed** — every asset fetched and measured; table at `/home/claude/audit/raw/size_table.tsv`.

### 5.2 Critical path

Everything referenced directly by `index.html`: 1 entry + 20 `modulepreload` + 8 stylesheets.

| Class | Count | Raw | gzip |
|---|---:|---:|---:|
| JS | 21 | 636,791 B | **266,948 B** |
| CSS | 8 | 144,338 B | **45,767 B** |
| **Total** | **29** | **781,129 B (763 KiB)** | **312,715 B (305.4 KiB)** |

So **~305 KiB gzip / 763 KiB parsed before first paint — about 14 % of the total JS graph.** The remaining ~86 % is genuinely deferred. **Confirmed.**

Note a deliberate discrepancy between two numbers in this audit: the *static* enumeration of `index.html` gives 29 files / 305 KiB, while *runtime* resource timing on the login screen records **48 subresources / 683,895 B transferred / 1,818,438 B decoded**, because a second wave at ~2.4 s pulls `fallback`, `TLottie`, `qr-code-styling`, two fonts and `notification.mp3`, and the 742 KB MTProto worker starts at ~2.8 s. Both are correct measurements of different things. **Confirmed.**

Measured split on the login screen: **30 unique JS URLs of 461 files (6.5 %), carrying 1,635,274 B of 5,532,659 B decoded (29.6 %)** — code splitting works, but the login path grabs a disproportionate share of the large chunks. **Confirmed.**

### 5.3 The highlight.js long tail

**373 of the 461 JS chunks (81 %) are single-language `highlight.js` grammar definitions**, one chunk per language, each lazily imported on demand.

| | Raw | gzip |
|---|---:|---:|
| 373 highlight.js grammar chunks | 928,928 B | 433,732 B |
| All other JS (88 chunks) | 4,603,731 B | 1,760,704 B |

Confirmed by source maps: those 373 maps have `sources` arrays containing *only* `node_modules/highlight.js/...` paths. Largest: `mathematica-FGVCSpsQ.js` (109,901 B), `isbl-D8SR3GGH.js` (71,725 B), `gml-5I1j5qRd.js` (55,777 B), `1c-CM1CBEvN.js` (55,468 B). **Confirmed.**

This is excellent for the median user and bad for the tail: a code block in an exotic language costs an extra round trip against a `max-age=3600` cache. For taskrgram, a 10–15 language allowlist bundled into one chunk is almost certainly the right call instead.

### 5.4 The 742 KB MTProto worker

`worker-J7_WDuX0.js` — **742,096 B raw / 240,922 B gzip — the single largest asset in the deployment, and 45 % of all decoded JS on the login screen.** It contains the entire GramJS client: TL serialisation, AES-IGE/CTR, RSA, the DH handshake, connection pool, update pipeline. It is served from `/a/` root, not `/a/assets/`, and is not requested until the app boots the worker (~2.8 s in our environment). **Confirmed.**

Architectural consequence, and it is not an artifact of our proxy: **the login CTA is render-blocked on an MTProto round trip that cannot begin until a 742 KB worker has downloaded, parsed and booted.** On a fully warm cache the login CTA still took 5,702 ms. **Confirmed** (the *absolute* number is environment-inflated; the *ordering* is a property of the app).

### 5.5 Splitting is manual, not route-based

There is **no router-driven splitting**. Six named barrel modules plus a tiny loader:

```ts
export enum Bundles { Auth, Main, Extra, Calls, Stars, Editor }
...
      case Bundles.Extra:
        LOAD_PROMISES[Bundles.Extra] = import('../bundles/extra');
```

with **151 `*.async.tsx`** shim files, each taking `isOpen` so the chunk is fetched exactly when the feature is first needed:

```tsx
const MediaViewerAsync: FC<OwnProps> = ({ isOpen }) => {
  const MediaViewer = useModuleLoader(Bundles.Extra, 'MediaViewer', !isOpen);
  return MediaViewer ? <MediaViewer /> : undefined;
};
```

**Confirmed.** The lazy boundary is a reviewable convention rather than a scattering of `React.lazy`, and CI bundle-size diffing keeps it honest.

### 5.6 Top 12 assets by raw size

| # | Asset | Raw | gzip |
|---|---|---:|---:|
| 1 | `worker-J7_WDuX0.js` | 742,096 | 240,922 |
| 2 | `assets/useConnectionStatus-DLbED6Q4.js` | 480,535 | 194,621 |
| 3 | `assets/main-D0-d5I4l.js` | 420,528 | 148,175 |
| 4 | `assets/Modal-BuOILwVN.js` | 396,332 | 147,962 |
| 5 | `assets/encoderWorker.min-BL5medTV.js` | 339,281 | 179,234 |
| 6 | `assets/editor-CL7uxqfp.js` | 294,925 | 110,610 |
| 7 | `assets/useConnectionStatus-CnGFpDOS.css` | 243,854 | 59,067 |
| 8 | `assets/InputText-CnVXgAD5.js` | 218,910 | 84,445 |
| 9 | `assets/extra-cut7UxU8.css` | 209,067 | 57,950 |
| 10 | `assets/fallback-fgJSCTeC.js` | 180,125 | 55,911 |
| 11 | `assets/stars-3qKrSpXD.js` | 164,403 | 63,892 |
| 12 | `assets/ActionMessage-CZsmVjMQ.js` | 129,017 | 52,944 |

**Inference worth flagging:** `InputText-CnVXgAD5.js` (218,910 B) is on the login critical path, on a screen with **zero `<input>` elements** (measured). That points to barrel-file / shared-chunk bleed. **Strong inference**, not Confirmed — we did not trace the import graph that produces that chunk.

---

## 6. CSS approach

### 6.1 Structure — split in parallel with JS

25 CSS files, 830.3 KiB raw / 231.2 KiB gzip, only 8 on the critical path. Each lazy JS chunk pulls its own stylesheet via the preload helper. Largest: `useConnectionStatus-CnGFpDOS.css` (243,854 B), `extra-cut7UxU8.css` (209,067 B), `index-CTuaTAxZ.css` (92,523 B). **Confirmed.**

### 6.2 Cascade layers solve the async-CSS ordering problem

The **only** inline `<style>` in the document declares layer order up front, so async CSS chunks land in the right layer regardless of load order:

```css
@layer reset, variables, ui, components;
@layer ui {
  @layer tablist, spinner, button, input, layout;
}
```

Chunked CSS then declares `@layer ui { ... }` (11 occurrences), `@layer variables`, `@layer reset`, `@layer component`. **Specificity is decided by layer, not by load order.** **Confirmed.** This is the cleanest single idea in the CSS architecture and is directly copyable.

### 6.3 Hybrid naming — hashed and semantic coexist

Production `generateScopedName: '[hash:base64:8]'` produces classes like `Y7owXZmb` (on `<body>`, observed live), `NKP0M5xy` (the composer's ProseMirror node, observed live), `x2doFvA9`, `qVl-H51g`, `Up-yv5MS`, `-HTWONyy`, `G7yJB56g`, `kJPBA-4R`. These coexist with human-readable BEM-ish names still present in the shipped CSS — `.Message.own`, `.MessageList`, `.CommentButton_right`, `.Checkbox-main`, `.ActionMessage`, `.CodeBlock .code-block`. Repo counts: **358 `*.module.scss` vs 148 plain `.scss`.** **Confirmed:** the migration to CSS Modules is real, deliberate, and incomplete.

Authoring rules, from the repo's own guide: camelCase class names, `buildClassName.ts` to merge, always extract styles to files, prefer `rem` (`N px = N / 16 rem`), no complex or broad selectors, no tag-based selectors. And a Teact-specific constraint — inline styles must be **template strings**, because Teact does not accept style objects:

```
    style={`transform: translateX(${value}%)`}   // ✅ CORRECT
    style={{ transform: ... }}                   // ❌ WRONG
```

**Confirmed.**

### 6.4 Token system — three tiers, and the palette is not in the CSS

Measured across all served CSS:

- **566** distinct custom properties defined anywhere
- **252** defined on `:root` (250 of them in one block in `index-CTuaTAxZ.css`)
- **577** component-scoped selectors define local vars, with a `--_`-prefix convention marking private tokens (`--_accent-color-rgb`, `--_selected-color`, `--_size`)
- only **2** `--var` declarations sit under `html.theme-dark` in the static CSS — and they are the LightningCSS toggles, not colours

**The primary light/dark palette is injected at runtime from JS.** `assets/initial-CskBLhZ6.js` holds a `[light, dark]` tuple map of **78 tokens**:

```js
var Rr={"--color-primary":[`#3390EC`,`#8774E1`],"--color-primary-opacity":[`#50A2E91E`,`#8378DB1E`],
  "--color-background":[`#FFFFFF`,`#212121`],"--color-background-secondary":[`#F4F4F5`,`#0F0F0F`],
  "--color-background-own":[`#EEFFDE`,`#766AC8`], ... }
```

…and `assets/Checkbox-Cxf2-dWf.js` owns a `<style>` element it rewrites on every theme change:

```js
var Ht=document.createElement(`style`);document.head.appendChild(Ht);
function Kt(){Ht.textContent=`
    html { ${qt(zt)} }
    html.theme-light { ${qt(Bt)} }
    html.theme-dark { ${qt(Vt)} }
  `}
function qt(e){return Array.from(e.entries()).map(([e,t])=>`--${e}: ${t};`).join(` `)}
```

**Confirmed.** Three consequences follow directly:

1. This is exactly why the CSP needs `style-src 'unsafe-inline'`.
2. A static scrape of the CSS finds almost no dark-mode variables — anyone auditing the theme from CSS alone will conclude, wrongly, that dark mode barely exists.
3. It enables **per-peer accent colours** with server-driven palettes: `Ut(\`color-peer-${e}\`, …)`, `Ut(\`color-peer-bg-${e}\`, …)`, `Ut(\`color-peer-gradient-${e}\`, …)`, with a 7-colour fallback `#D45246 #F68136 #6C61DF #46BA43 #5CAFFA #408ACF #D95574`.

Theme switching is animated: `colorjs.io` interpolates each variable over `DURATION_MS = 200`, `no-animations` is toggled on `<html>` for 500 ms, and `<meta name="theme-color">` is swapped (`c.setAttribute(\`content\`,r?\`#212121\`:\`#fff\`)`). See `screenshots/14-desktop-settings-general-dark-theme-selected.png` and `screenshots/15-desktop-main-layout-dark-theme-chat-list-and-message-list.png`. **Confirmed.**

Beyond colour, the token system carries structural values: **z-index is fully tokenised** (`--z-overlay-effects: 12000` down to `--z-below: -1`, ~45 tokens), as are radii (`--border-radius-modal: 2rem`, `--border-radius-messages: .9375rem`), layout widths (`--right-column-width: 26.5rem`, `--messages-container-width: 47.5rem`, `--folders-sidebar-width: 5rem`), transitions (`--layer-transition: .3s cubic-bezier(.33, 1, .68, 1)`), and safe areas (`--safe-area-top: env(safe-area-inset-top)`). **Confirmed.**

Font stacks are token-driven and platform-branched — `html,body` gets `--font-family: "Roboto", -apple-system, …`, while `body.is-ios, body.is-macos` overrides to `system-ui, -apple-system, BlinkMacSystemFont, "Roboto", …` and bumps `--font-weight-semibold` from 500 to 600. A Persian override exists (`html[lang=fa]` → `"Vazirmatn", …`). **Confirmed.**

There is also a **14-token Mini-App theme contract** exposed to bot Web Apps (`bg_color`, `text_color`, `hint_color`, `link_color`, `button_color`, `button_text_color`, `secondary_bg_color`, `header_bg_color`, `accent_text_color`, `section_bg_color`, `section_header_text_color`, `subtitle_text_color`, `destructive_text_color`, `section_separator_color`) — a deliberately small, stable public surface over a 566-property private one. **Confirmed.**

---

## 7. Networking stack

### 7.1 Layering

```
UI (Teact components)
  └─ global actions  (src/global/actions/api/*)
       └─ callApi('methodName', args)                     src/api/gramjs/worker/connector.ts   [main thread]
            └── postMessage ──▶ dedicated Worker           src/api/gramjs/worker/worker.ts      [worker thread]
                 └─ methods/*  (fetchUsers, sendMessage…)  src/api/gramjs/methods/*
                      └─ invokeRequest(new GramJs.X(...))  src/api/gramjs/methods/client.ts
                           └─ TelegramClient.invoke        src/lib/gramjs/client/TelegramClient.ts
                                └─ MTProtoSender           src/lib/gramjs/network/MTProtoSender.ts
                                     └─ ConnectionTCPObfuscated / HttpConnection
                                          └─ PromisedWebSockets (wss) | HttpStream (https fallback)
```

**Confirmed.** The boundary rule is stated by the authors: *"We use GramJS inside a web worker; UI code uses plain objects (`Api*` types) in `src/api/types`."* Conversion runs through `apiBuilders/` (23 files, `buildApi*`) and `gramjsBuilders/` (3 files, `buildInput*`). There is a DEBUG-time assertion in `connector.ts` that responses contain no TL `VirtualClass` instances.

This is the single most valuable structural decision in the codebase: **the TL/GramJS object graph never crosses into UI code**, so the UI is decoupled from the wire protocol and every response is cheaply serialisable across `postMessage`.

Worker-bridge mechanics, all **Confirmed**:

- Messages are **coalesced in both directions**, flushed once per microtask tail (`throttleWithTickEnd`).
- Binary payloads use **transferables** (`postMessage(data, transferables)` keyed off `response.arrayBuffer`).
- Requests correlate by `messageId` in `requestStates: Map<string, RequestState>`, with progress callbacks and `cancelProgress`.
- **Only the elected master tab owns the worker.** Non-master tabs proxy via `BroadcastChannel`: `callApiOnMasterTab(payload)` / `makeRequestToMaster({ name, args })`.
- A Safari/Tauri watchdog exists: `if (IS_SAFARI || (initialArgs.platform === 'macOS' && IS_TAURI)) setupHealthCheck();` with `HEALTH_CHECK_TIMEOUT = 150` and the comment `// Workaround for iOS sometimes stops interacting with worker`.

### 7.2 Transport, measured live

Every WebSocket frame in an authenticated session was logged:

| Endpoint | Events |
|---|---:|
| `wss://zws1.web.telegram.org/apiws` | 512 |
| `wss://zws4.web.telegram.org/apiws` | 204 |
| `wss://zws4-1.web.telegram.org/apiws` | 116 |
| `wss://zws2.web.telegram.org/apiws` | 93 |
| `wss://zws2-1.web.telegram.org/apiws` | 43 |
| `wss://zws1-1.web.telegram.org/apiws` | 35 |

```
opens: 17   closes: 16
frames sent:  310  =    85,464 bytes  (max single frame  4,764 B)
frames recv:  660  = 10,026,931 bytes (max single frame 218,940 B)
```

**Confirmed.** The upstream/downstream asymmetry is ~117:1, and it is the tell: **media is not fetched over HTTP.** Photos, video and stickers come down *through the MTProto WebSocket* as `upload.getFile` chunks and are handed to the page as `blob:` URLs — 102 of 737 HTTP responses in the session had an empty host, i.e. they were `blob:` URLs. This is forced by the protocol: files live behind MTProto, addressable by `(dc_id, file_reference)`, not by URL. **Confirmed.**

Multiple `zwsN` and `zwsN-1` hosts were live simultaneously — the client holds parallel connections to several datacenters (one for updates, others for file downloads), and holds **per-DC auth keys concurrently** (`dc1_auth_key`, `dc2_auth_key`, `dc4_auth_key`, all present in `localStorage` at once). **Confirmed.**

### 7.3 Obfuscation, and why it exists

Telegram's spec requires it: *"Transport obfuscation is required to use the WebSocket transports."* `src/lib/gramjs/network/connection/TCPObfuscated.ts` implements the 64-byte random header with AES-CTR streams and a forbidden-prefix rejection list:

```ts
const FORBIDDEN_OBFUSCATED_PREFIXES = [ bufferFromHex('48454144'), /* HEAD */ bufferFromHex('504f5354'), /* POST */
  bufferFromHex('47455420'), /* GET  */ bufferFromHex('4f505449'), /* OPTI */ bufferFromHex('16030102'), /* TLS  */
  bufferFromHex('dddddddd'), bufferFromHex('eeeeeeee') ];
```

Framing is the **Abridged** codec. **Confirmed.**

### 7.4 Endpoint configuration

**Meaningful negative result:** no `149.154.x.x` / `91.108.x.x` DC IPs and no `venus`/`pluto`/`aurora`/`vesta`/`flora` hostnames appear anywhere in the shipped JS. The WebSocket URL is templated:

```js
getWebSocketLink(e,t,n,r){
  return t===443
    ? `wss://${e}:${t}/apiws${n?`_test`:``}${r?`_premium`:``}`
    : `ws://${e}:${t}/apiws${n?`_test`:``}...`
}
```

The only literal transport strings are `/apiws`, `/apiws_test`, `/apiws_premium`. **`api_id` / `api_hash` are never hardcoded** — only referenced as `this.apiId` / `apiHash`, injected at build/runtime. **Confirmed.**

The DC *hostnames* are, however, hardcoded in the repo's `Utils.ts` (`zws1.web.telegram.org` … `zws5`, port 443, with a `-1` variant for download DCs) behind a `// TODO Move to external config` comment. **Confirmed.**

### 7.5 Resilience machinery worth naming

All **Confirmed** from source:

- **DC migration** discards the auth key, with the reason in a comment: `// authKey's are associated with a server, which has now changed / so it's not valid anymore. Set to None to force recreating it.`
- **Full MTProto error taxonomy** handled in `invoke()`'s retry loop: `ServerError`, `RPC_CALL_FAIL`, `RPC_MCGET_FAIL`, `/INTERDC_\d_CALL(_RICH)?_ERROR/` (sleep 2 s), `FloodWaitError` (sleep if `<= floodSleepLimit`), `PhoneMigrateError | NetworkMigrateError | UserMigrateError` → `_switchDC(e.newDc)`, `MsgWaitError`, `CONNECTION_NOT_INITED`, `TimedOutError`. Error→class mapping is regex-driven in `RPCErrorList.ts`.
- **Bandwidth governor** (`src/util/dcBandwithManager.ts`): `MAX_CONCURRENT_CONNECTIONS = 3` (premium 6), `MAX_ACTIVE_REQUEST_SIZE = 9 MB` (premium 20 MB), with a priority queue.
- **Parallel file transfer**: `DOWNLOAD_WORKERS = 16`, `UPLOAD_WORKERS = 16`, `MAX_UPLOAD_FILEPART_SIZE = 524288`.
- **Full client-side update-gap algorithm** (`updateManager.ts`, 673 LOC + `mtpUpdateHandler.ts`, 1,223 LOC): ordered `SortedQueue`s per box, pts/seq continuity checks, `getDifference` recovery, `SHORTPOLL_CHANNEL_DIFFERENCE_LIMIT = 100`, `CATCH_UP_CHANNEL_DIFFERENCE_LIMIT = 1000`.

Reconnect was observed working live: after `Socket zws4.web.telegram.org closed. Code: 1006`, the worker logged `Error: TIMEOUT`, retried, and the session continued. **Confirmed.**

### 7.6 Credential storage — the security finding

Measured directly after login:

```json
"localStorageKeys": [
  { "key": "tgme_sync", "len": 36 }, { "key": "account1", "len": 1698 },
  { "key": "dc2_auth_key", "len": 514 }, { "key": "user_auth", "len": 28 },
  { "key": "dc", "len": 1 }, { "key": "tt-multitab_1", "len": 1 },
  { "key": "dc1_auth_key", "len": 514 }, { "key": "dc4_auth_key", "len": 514 }
],
"cookies": ""
```

**Confirmed.** Two things follow:

1. **`document.cookie` is empty.** There is no session cookie and no CSRF token. CSRF as a concept does not apply — every authenticated call is an MTProto message signed with a key held in JS-readable storage, not a browser-attached ambient credential.
2. **MTProto auth keys sit in plaintext `localStorage`**, one per datacenter, 514 chars each (≈256-byte key, hex-encoded). Any XSS, any extension with storage access, or filesystem access to the browser profile takes the account. Optional encryption exists behind a user-set passcode — and its KDF is **a single SHA-256 with a hardcoded salt string** (`SALT = 'harder better faster stronger'`, `IV_LENGTH = 12`, AES-GCM), which is weak. **Confirmed.**

---

## 8. Workers and WASM

### 8.1 Worker inventory (served from `/a/` root)

| Worker | Raw | gzip | Purpose |
|---|---:|---:|---|
| `worker-J7_WDuX0.js` | 742,096 | 240,922 | GramJS/MTProto — crypto + TL serialisation off the main thread |
| `fasttext.worker--naZEx7i.js` | 64,400 | 24,532 | fastText language-ID |
| `index.worker-DLzlUYNq.js` | 7,911 | — | media/offscreen worker entry (pool) |
| `service.worker-BSeu-kQn.js` | 9,609 | — | PWA service worker |
| `sharedState.worker-zNhJnNN3.js` | 1,736 | — | cross-tab shared state (**SharedWorker**) |

**Confirmed.** Media worker pool sizing is `MAX_WORKERS = Math.min(navigator.hardwareConcurrency || 4, 4)`. Workers share a generic RPC layer (`PostMessageConnector.ts` / `createPostMessageInterface.ts`) with a `channel` string so several logical APIs multiplex one worker — tlottie and offscreen-canvas coexist under channel `'media'`. **Confirmed.**

### 8.2 WASM inventory

| Module | Bytes | What it is for |
|---|---:|---|
| `assets/fasttext-wasm-zrRkeJ3U.wasm` | 1,122,181 | **Facebook fastText** — client-side language identification, so the app can offer "Translate" without sending your text anywhere to find out what language it is |
| `assets/tlottie-HZNSEMV6.wasm` | 400,534 | **tlottie** — Telegram's rlottie fork, rasterises `.tgs` (gzipped Lottie) animated stickers and animated emoji |
| `decoderWorker.min.wasm` | 137,424 | `opus-recorder` **Opus decoder** — playing voice notes |
| `encoderWorker.min.wasm` | 1,587* | `opus-recorder` **Opus encoder** — recording voice notes (*small response; likely a stub/redirect) |

**Confirmed.** Note the export naming is `tlottie_*`, not `rlottie_*` — Telegram renamed the exports. Rendering is dispatched round-robin across the media worker pool, with device-aware quality (`HIGH_PRIORITY_QUALITY = (IS_ANDROID || IS_IOS) ? 0.75 : 1`) and a shared-canvas mode so many small emoji share one canvas.

**All four WASM modules are served as `application/wasm` with no compression** — 1.75 MiB uncompressed on the wire the first time voice or animation features activate. **Confirmed.** Zero `.wasm` requests occur on the login screen, so the deferral is correct. **Confirmed.**

`'wasm-unsafe-eval'` in the CSP exists for exactly these four modules. **Confirmed.**

### 8.3 Service worker — the clever part

Hand-written, 6 source files, registered as an ES module with `skipWaiting()` + `clients.claim()` (inferred from it controlling the very first cold load — **Strong inference**). Routing:

```ts
const CACHE_FIRST_ASSET_EXTENSIONS = 'js|css|woff2?|svg|png|jpe?g|tgs|json|wasm';
const RE_NETWORK_FIRST_ASSETS = /\.(wasm|html)$/;
const RE_CACHE_FIRST_ASSETS = new RegExp(`(?:/assets/[^/]+|/(?:[^/]+\\.)?worker|/index)-[\\w-]{8}\\.(${CACHE_FIRST_ASSET_EXTENSIONS})$`);
```

Routes: `/progressive/` → range serving, `/download/` → streamed save-as, `/share/` → OS share target, else cache-first / network-first. **Confirmed.**

**Progressive media over Range requests is the headline technique.** The SW intercepts `<video src="./progressive/…">` range requests, converts them to `postMessage` requests to the page (which forwards to the MTProto worker), and answers with `206 Partial Content`. Observed live:

```
206 GET https://web.telegram.org/a/progressive/document5109473995049145023
206 GET https://web.telegram.org/a/progressive/document5109473995049145021
206 GET https://web.telegram.org/a/notification.mp3
```

**Confirmed at runtime.** Constants: `DEFAULT_PART_SIZE = 0.5 MB`, `MAX_END_TO_CACHE = 2 MB - 1` (`// We only cache the first 2 MB of each file`), `PART_TIMEOUT = 60000`, plus an explicit `// Optimization for Safari` branch for the `start===0 && end===1` probe. Downloads use `DOWNLOAD_PART_SIZE = 1 MB`, `QUEUE_SIZE = 8`.

The net effect: `<video>` and `<audio>` stream and seek natively over content that only exists as encrypted MTProto parts. **No MSE, no HLS, no DASH, no custom player** — MSE appears only as a Safari-specific fallback in `src/hooks/useStreaming.ts`. **Confirmed.**

Cache Storage policy: `CACHE_TTL = 5 days`, `ACCESS_THROTTLE = 1 day`, `CLEANUP_INTERVAL = 1 hour`, LRU implemented by writing an `X-Last-Access` header and sweeping hourly while yielding to the main thread. `MEDIA_CACHE_MAX_BYTES = 512 * 1024` — anything larger goes down the progressive path instead. **Confirmed.**

Measured effect on a real session: 16.3 MB of 16.6 MB total storage was Cache Storage after a few minutes browsing one channel. **Confirmed.**

---

## 9. The rich-text editor

**Confirmed.** The composer is not a `contenteditable` div and not a `<textarea>`. The live DOM node carries:

```html
class="form-control allow-selection NKP0M5xy ProseMirror ProseMirror-focused"
```

The dependency footprint is the largest third-party UI surface in the app: **27 `@tiptap/*` packages and 13 `prosemirror-*` packages** recovered from source maps.

- `@tiptap/`: `core`, `starter-kit`, `suggestion`, `markdown`, `extensions`, `pm`, plus extensions `bold, italic, strike, underline, code, code-block, code-block-lowlight, blockquote, heading, paragraph, text, document, hard-break, horizontal-rule, link, list, placeholder, table, details, subscript, superscript`
- `prosemirror-`: `state, view, model, transform, commands, keymap, history, schema-list, tables, dropcursor, gapcursor`, plus `orderedmap`, `rope-sequence`, `w3c-keyname`, `linkifyjs`

Chunk: `assets/editor-CL7uxqfp.js` (294,925 B raw / 110,610 B gzip) + `assets/editor-CNGiJ7-T.css` (10,622 B), loaded as its own `Bundles.Editor`. Telegram's adapter layer is `src/util/tiptap` (17 files) plus `src/components/middle/composer/richInput`. **Confirmed.**

The observed toolbar surface is a **document editor, not a chat formatting bar**: `Open full editor`, `Insert block`, `Lists`, `Table`, `Add Link`, `Code block`, `Equation`, `Undo`, `Redo`; block menu items `Bulleted list`, `Numbered list`, `Checklist`, `List options`, `Heading`, `Footer`, `Quote`, `Pullquote`, `Details`, `Divider`. In-product copy read live in @TelegramTips confirms the intent: *"Rich Text Editor. You can create articles and long posts of over 32,000 characters in a visual editor that supports dozens of formatting options like tables, lists…"* **Confirmed.** See `screenshots/09-desktop-chat-opened-service-message-composer-visible.png`.

**This is also the clearest documented "why" in the whole project.** Telegram published the reasoning for the 2025 rewrite:

> "- All content is treated as HTML string, so every edit triggers re-parsing before you can do any work with it.
> - We can't directly render components, so custom emojis rely on workarounds, and it is problematic to support markdown syntax preview.
> - A RegExp-based Markdown parser has too many problems.
> - Composer component is bloated, making it hard to reuse in more lightweight places, like editing folder titles or gift messages."

**Confirmed** (first-party, `t.me/webachannel`, 2025-02-04). Note what it means: the "no new dependencies" constraint that produced Teact was **formally relaxed** for the editor problem. The team that wrote its own React would not write its own ProseMirror. That is a useful calibration for taskrgram: rich text editing is the one place where building your own is reliably a mistake.

Supporting libraries in the same area: `lowlight ^3.3.0` (hljs↔ProseMirror bridge), `temml ^0.13.3` (LaTeX → MathML), `marked` (Markdown parsing), `linkifyjs`. **Confirmed.**

---

## 10. WebRTC and calls

**Confirmed.** 1:1 and group calls are implemented in-house under `src/lib/vibecalls` (7 files — an internal codename leaked by the source maps) with UI in `src/components/calls/{phone,group}` (26 files), loaded as `Bundles.Calls` (`calls-BLCcyQCT.js` 89,858 B + `calls-BUedN57l.js` 52,407 B + `calls-D3LYymlO.css` 14,857 B).

Evidence from the bundle:

```js
let r=new RTCPeerConnection,i=n?void 0:Vt(r);return e.forEach(e=>e.getTr...
async function hn(e,t,n,r,i,a){let o=new RTCPeerConnection({iceServers:vn(e,i),iceTransportPolicy:i?`al...
d(`m=application ${+!t} UDP/DTLS/SCTP webrtc-datachannel`),d(`c=IN IP4 0.0.0.0`),d(`a=mid...`)
t.navigator.mediaDevices.getUserMedia({audio:this.config.mediaTrackConstraints})
```

**SDP is constructed and parsed by hand** — raw `m=` / `c=` / `a=mid` line building. There is **no `simple-peer`, no `mediasoup-client`, no `livekit`**. **Confirmed.**

That hand-rolled SDP layer has a documented cost: Mozilla bug 1903659 records that voice/video calls do not work on `web.telegram.org` in Firefox — Web A does not surface the call UI to Firefox at all (needs a UA spoof), and the sibling client's custom SDP parser chokes on Firefox's spec-compliant directional RTP header-extension attributes. Mozilla shipped a **user-agent override intervention in Firefox Nightly** spoofing Chrome for `/a/`. **Confirmed** (third-party bug tracker, primary source).

Feature surface confirmed in source: screen sharing (`IS_SCREENSHARE_SUPPORTED`, `startSharingScreen`), noise suppression, speaker toggle, video layout (`useGroupCallVideoLayout`), SCTP signalling (`sctpSignaling.ts`). **Confirmed.** We did **not** place a call during this audit — call quality, codec selection and TURN behaviour are **Unknown**.

---

## 11. Tauri desktop coupling

**Confirmed.** The web bundle carries the Rust-desktop bridge. Source maps list `node_modules/@tauri-apps/api` plus plugins `shell`, `notification`, `updater`, `process` (10 files); `package.json` declares `@tauri-apps/api ^2.11.1`, `plugin-notification ^2.3.3`, `plugin-process ^2.3.1`, `plugin-shell ^2.3.5`, `plugin-updater ^2.10.1`, with `tauri:build` / `tauri:dev` scripts and a `tauri/` directory plus `docs/TAURI.md`.

Two runtime tells of this coupling leaking into the browser UI:

- The `__vite__mapDeps` array in the entry chunk lists `./window-R9mtfgf9.js` and `./event-Cg2-4Juw.js` — Tauri API chunks — as eager dependencies.
- Privacy settings expose a **"Window title bar → Show chat name"** toggle in the *web* client, which is a desktop-app affordance. **Confirmed**, observed live; see `screenshots/25-desktop-settings-privacy-and-security-panel.png`.

The desktop product name is **contested and should be flagged**. Releases are tagged `air_v2.10.3`, `air_v2.9.3`; the Teact README says Teact "powers the official **Telegram Air** (Web A) client"; a SourceForge mirror carries `Telegram.Air-aarch64.dmg`. **However:** `telegram.org/apps` lists no product called "Telegram Air", and `web.telegram.org/a/get` is titled "**Telegram A Desktop**". Best supported statement: **"Telegram Air" is the internal/release name for packaged Tauri desktop builds of the Web A codebase; its status as a publicly branded product is unconfirmed.** — **Possible**, explicitly not Confirmed.

**Licensing consequence, and it is sharp:** GPLv3 is triggered by *conveying*, not by running a service. An internal-only, employees-only web app is the escape hatch. **Shipping a desktop binary is conveying.** This repo already ships a Tauri target, so "we only run it on our servers" would not survive a desktop build. **Inference, flagged for counsel, not legal advice.**

---

## 12. What is NOT there

This is one of the most interesting findings in the whole audit, and it deserves to be read as a positive architectural statement rather than a list of gaps. Each row below is a **Confirmed** zero-hit search across the complete 461-file JS graph and all 25 stylesheets, plus a runtime host census of 737 HTTP responses.

| Searched | Result | What it means |
|---|---|---|
| `react`, `react-dom`, `preact`, `__REACT_DEVTOOLS_GLOBAL_HOOK__` | **0 hits** | No React or Preact anywhere. Teact is the whole framework. |
| `webpack`, `__webpack_require__`, `webpackChunk` | **0 hits** | Not a webpack build in any form. |
| `sentry`, `Sentry` | **0 hits** | **No crash reporting SaaS.** Errors are handled locally or not at all. |
| `google-analytics`, `gtag`, `googletagmanager`, `amplitude`, `mixpanel`, `segment`, `posthog` | **0 hits** | **Zero third-party analytics or product telemetry.** |
| Third-party hosts at runtime | **0** | 737 responses: 629 `web.telegram.org`, 102 `blob:`, 2 `t.me`, 2 `telegram.me`, 2 `telegram.dog`. The three t.me aliases are Telegram's own domains, contacted only for deep-link/session sync. |
| CDN vendor headers (`cf-ray`, `x-amz-cf-id`, `x-cache`, `age`, `via`, `server-timing`) | **0** | **No commercial CDN.** Self-hosted nginx in Telegram's own AS. |
| Cookies | `document.cookie === ""` | **No cookies at all**, session or otherwise. |
| `ffmpeg` | **0 hits** | Despite Emscripten glue being present (for fastText/tlottie/opus), there is no ffmpeg.wasm. All media work is native `<video>` + Range requests. |
| `workbox` | **0 hits** | The service worker is hand-written, 6 files. |
| `tonweb`, `TonWeb`, `tonconnect`, `@tonconnect/ui`, `tonapi`, `walletconnect` | **0 hits** | **No TON SDK is bundled**, despite the CSP whitelisting 11 TON wallet URL schemes in `frame-src`. TON support is deep-link/iframe only. |
| `twemoji`, `joypixels` | **0 hits** | `src/lib/twemojiRegex.js` is the regex only, not the sprite library. |
| `lodash`, `moment`, `dayjs`, `axios`, `zod` | **0 hits** | No utility-belt, no date library, no HTTP client, no schema validator. Native `Intl` does date/number/plural formatting. |
| Font CDN / third-party font host | **0** | 2 `.woff2` served from origin (22,672 B), incl. a generated icon font built by `npm run icons:build`. |
| Ad tech | **0** | Sponsored messages exist as a product concept (`ads.telegram.org` string) but no ad-serving JS. |
| Hardcoded DC IPs, `api_id`, `api_hash` | **0** | All runtime-configured. |
| HLS / DASH | **0** | Range-request streaming via the service worker instead. |

The only external hosts referenced anywhere in the bundle graph that are actually contacted at runtime are: `*.web.telegram.org` (MTProto over WSS), `translations.telegram.org` (live language packs), `ss3.4sqi.net` (Foursquare venue-category icons, also whitelisted in CSP `img-src`), and — **only inside the payments flow** — `api.stripe.com/v1/tokens` and `tgb.smart-glocal.com/cds/v1/tokenize/card`. **Confirmed.**

**Why this matters for taskrgram.** A team-chat product will feel enormous pressure to add Sentry on day one and a product-analytics SDK by week three. Telegram Web A demonstrates that a top-50 website with payments, calls, video streaming and a document editor ships with **none of it** — and that the resulting property is that you can characterise the app's entire network behaviour from a single HAR file. That is worth something real in an internal tool handling employee conversations. Note the trade-off honestly: with no crash reporting, the only signal of a production defect is a user report routed through a suggestions platform, and the audit found evidence of the cost (an untranslated i18n key leaking into an `aria-label`, a `console.error(undefined)` swallowing its own diagnostic — see `10-evidence-log.md` rows 88–89).

---

## 13. Hosting and delivery

| Property | Value | Confidence |
|---|---|---|
| A record | `149.154.167.99` | Confirmed |
| AAAA record | `2001:67c:4e8:f004::9` | Confirmed |
| ASN / org | **AS62041 — Telegram Messenger Inc** (both v4 and v6) | Confirmed |
| Geolocation | Amsterdam, North Holland, NL (52.374, 4.890) — Telegram DC2 | Confirmed |
| Server | `nginx/1.30.1` | Confirmed |
| Backend node | `x-served-by: meta4240719`, `meta4240822`, `meta4240420`, `meta4240118` — rotates per request | Confirmed |
| Protocol | **HTTP/2**; no `alt-svc` → **HTTP/3 not advertised** | Confirmed |
| Compression | **gzip only.** `Accept-Encoding: br` → identity (28,951 B); `br, gzip, deflate, zstd` → gzip | Confirmed |
| `cache-control` | **`max-age=3600` on everything**, including content-hashed assets | Confirmed |
| `x-frame-options` | `deny` on all responses | Confirmed |
| HSTS | **absent** on every observed response | Confirmed |
| CSP | delivered via `<meta http-equiv>`, **not** a header | Confirmed |
| `etag` | `W/"6a7b3e9e-7117"` — weak; `6a7b3e9e` is the shared build mtime (hex unix ts = 2026-08-11 15:24:14 UTC), suffix is hex content-length | Confirmed |
| TLS cert | `CN = *.web.telegram.org`, issuer `Go Daddy Secure Certificate Authority - G2`, valid `Aug 29 2025` → `Sep 30 2026` | Confirmed |
| `robots.txt` | 302 → `/`. No robots.txt is served | Confirmed |
| Source maps | **453 of 461 chunks ship a working, publicly fetchable `.map`** | Confirmed |

### 13.1 Three defects worth naming

**1. `cache-control: max-age=3600` on content-hashed immutable assets.** This is the single clearest performance defect in the deployment. `assets/index-IZ97MA_m.js` is content-hashed — its URL changes when its content changes — and it gets the same 1-hour TTL as the HTML. There is no `immutable` and no `max-age=31536000`. That is a full revalidation storm across ~490 assets every hour for every active client. The fix is one line of nginx config. **Confirmed.**

Mitigating factor, also measured: the service worker's cache-first strategy for hashed assets largely masks this. On a warm reload only **5 resources touched the network, totalling 2,975 B** (`compatTest.js` 1,450 B and `redirect.js` 625 B — both unhashed, therefore uncacheable — plus three 300 B worker `importScripts` revalidations), a **99.6 % reduction** in transfer. So the defect costs revalidation round trips rather than bytes, for clients that reach the SW. **Confirmed.**

**2. gzip only, no Brotli, no zstd, no HTTP/3.** Verified by explicit `Accept-Encoding` probing, not assumed. Enabling Brotli alone would realistically cut ~20 % off a 2.32 MiB JS+CSS transfer. **Confirmed** (the 2.32 MiB is measured; the ~20 % improvement is an estimate — **Strong inference**).

**3. Complete source maps served publicly.** 453 maps resolve with HTTP 200 (`index-IZ97MA_m.js.map` → 115,171 B; `InputText-CnVXgAD5.js.map` → 1,003,890 B), exposing **2,056 original source paths**. CSS maps are *not* served (302). This is a deliberate choice (`build.sourcemap: true`) and entirely defensible for a GPLv3 client whose source is on GitHub anyway. For a proprietary internal app it would be a straightforward information leak — the maps handed us the complete module graph, the third-party dependency inventory, and two internal codenames (`src/lib/vibecalls`, `src/components/gili`) that are not mentioned anywhere public. **Confirmed.**

### 13.2 CSP, verbatim

```
default-src 'self';
connect-src 'self' wss://*.web.telegram.org blob: http: https: ;
script-src 'self' 'wasm-unsafe-eval' https://t.me/_websync_ https://telegram.me/_websync_ https://telegram.dog/_websync_;
worker-src 'self';
style-src 'self' 'unsafe-inline';
font-src 'self' data:;
img-src 'self' data: blob: https://ss3.4sqi.net/img/categories_v2/;
media-src 'self' blob: data:;
object-src 'none';
frame-src http: https: bitkeep: bnc: bybitapp: echooo: imtokenv2: mytonwallet-tc: nicegram-tc:
          safepal-tc: tonkeeper-pro-tc: tonkeeper-tc:;
base-uri 'none';
form-action 'none';
```

**Confirmed.** Reading it honestly:

- `script-src` is genuinely tight — no `unsafe-inline`, no `unsafe-eval`, strict `'self'` allowlist. It **actively blocked** our attempt to inject axe-core from a CDN during the accessibility pass, which is correct behaviour and a positive finding.
- `'wasm-unsafe-eval'` is required by the four WASM modules.
- `style-src 'unsafe-inline'` is a real weakening, forced by the runtime theme-token injector (§6.4).
- `connect-src` is effectively wide open (`http: https:`) despite the specific `wss://*.web.telegram.org` — that specificity is decorative.
- `frame-src http: https:` plus 11 TON wallet custom schemes is the widest directive here, driven by Mini Apps and the in-app browser.

### 13.3 Three generations live simultaneously

| Path | What | `last-modified` / version |
|---|---|---|
| `/` | **Legacy Webogram** — AngularJS (`ng-app`, `manifest=webogram.appcache`) | `Wed, 25 Oct 2023` |
| `/k/` | **Telegram Web K** | `Fri, 14 Aug 2026 10:32:09`, version `2.2 (669)` |
| `/a/` | **Telegram Web A** — this app | `Tue, 11 Aug 2026 15:24:14`, version `12.0.38` |

**Confirmed.** Both current clients now build with Vite/Rolldown — both index pages reference an **identically-hashed** `rolldown-runtime-hePW80VL.js`. `/a/manifest.json` 302-redirects to `/`, i.e. the legacy app. `/a/redirect.js` (325 B, render-blocking) rewrites `/z` → `/a` and bounces `weba.telegram.org` / `webz.telegram.org` → `web.telegram.org/a` when `localStorage['tt-global-state']` is absent — the residue of the 2023 Web Z → Web A rename. **Confirmed.**

---

## 14. How we would fingerprint this ourselves

Reproducible method, in the order we ran it. Nothing here requires authentication except step 7.

1. **Pull the shell and read it as a manifest.** `GET /a/` → 5,806 B of HTML. Enumerate `<script type="module">`, every `<link rel="modulepreload">` and `<link rel="stylesheet">`. That is the exact critical path, no guessing: 29 files here.
2. **Walk the graph by bare specifier.** Rolldown/Vite chunks reference each other by relative path (`"./teact-CUUQAb7N.js"`) and reference workers/WASM by `new URL('../service.worker-….js', import.meta.url)`. A regex over each fetched chunk for `\.\/[\w.\-]+-[\w\-]{8}\.(js|css|wasm)` plus a work queue reaches the full 461-file closure without a browser.
3. **Take the source maps.** `tail -c 60` each chunk for `//# sourceMappingURL=`, fetch the sibling `.map`, parse the `sources` array. This is the highest-yield single step in the whole audit: 2,056 original paths, which gives you the folder architecture, the internal library names, and — via `node_modules/<pkg>/…` paths — a **complete third-party dependency inventory with per-package file counts**, without ever reading package.json.
4. **Fingerprint the bundler by negative search.** `grep -c __webpack_require__` → 0 tells you more than any positive match. Then look for the runtime chunk (716 B here), `__vite__mapDeps`, `vite:preloadError`, `--lightningcss-*`. Minifier identification is a style question: backtick-rewritten string literals ⇒ OXC, preserved quote style ⇒ Terser.
5. **Probe the encoding and cache policy explicitly.** Request the same asset with `Accept-Encoding: br`, then `br, gzip, deflate, zstd`, then nothing, and compare `content-encoding` and byte counts. Never infer Brotli support from the browser's default request. Repeat requests and diff `x-served-by` to detect a node pool vs. a CDN edge.
6. **Read the real certificate through a raw tunnel.** In an environment with an inspecting proxy, `openssl s_client -proxy …` passes the origin's actual chain, which is how we got the GoDaddy issuer rather than the proxy's. DNS + ASN lookup on both A and AAAA closes out the hosting question.
7. **Drive the live app under CDP and enumerate everything.** `navigator.storage.estimate()`, `indexedDB.databases()`, `caches.keys()` + `caches.open(n).keys()`, `Object.keys(localStorage)` **recording key names and value lengths only**, `document.cookie`, `navigator.serviceWorker.getRegistrations()`. Hook `Network.webSocketFrameSent/Received` for frame counts and byte totals — the upstream/downstream asymmetry (117:1 here) is what exposes media-over-WebSocket.
8. **Bisect breakpoints, do not read them.** Binary-search viewport width and read `getBoundingClientRect()` on the layout containers at each step. That produced exact boundaries (1276/1275, 926/925, 601/600) rather than the approximate ones a stylesheet grep would suggest.
9. **Count DOM nodes across scroll, not once.** `document.querySelectorAll('.Message').length` plus `scrollHeight` plus `performance.memory` at several scroll offsets distinguishes windowed virtualization (node count constant, nodes recycled) from a bounded sliding window (node count steps up, then plateaus). Here: 29 → 89 → 89, with heap *falling* 54.6 → 52.3 MB.
10. **Separate environment artifacts from app defects before writing anything down.** Our egress proxy forced HTTP/1.1 and broke WebSockets; headless Chromium lacked a GPU and proprietary codecs. Every timing number was tagged accordingly, and the console output was triaged into transport (proxy), GPU (headless), and application (2 items) before any of it was reported.

---

## 15. Stack decisions: load-bearing vs. incidental

Ranked by how much of the observed behaviour actually depends on them.

**Load-bearing — copy the idea, and understand it before you do:**

1. **Protocol client in a worker, DTOs across the boundary.** The 742 KB of crypto and TL serialisation never touches the main thread; only plain serializable objects cross `postMessage`, batched once per microtask tail with `ArrayBuffer` transferables.
2. **Two-phase DOM discipline enforced at runtime.** `fasterdom` + `stricterdom` turn layout thrashing from a profiling exercise into a dev-time exception. Framework-agnostic — bolt it onto anything.
3. **Gate expensive work on animation state.** `beginHeavyAnimation()` / `onFullyIdle()` pausing renders, store mapping, IDB writes and IntersectionObservers is why transitions stay smooth.
4. **Service worker + Range requests for media.** Lets native `<video>` stream and seek content that has no URL. No MSE, no HLS, no custom player.
5. **Cascade layers declared inline before any async CSS loads.** Solves chunk-order specificity permanently, in eight lines.
6. **Multi-tab as an architecture decision.** One elected master owns the socket and the worker; others proxy over `BroadcastChannel`; per-tab UI state (`byTabId`) is separated from shared state (SharedWorker). Retrofitting this is painful.
7. **Architectural rules as lint rules.** Three custom ESLint plugins and `stylelint-high-performance-animation` mean the invariants survive staff turnover.

**Incidental — do not cargo-cult:**

1. **Teact itself.** It is genuinely good and it is MIT-licensed, but it was born of a contest rule, it has no error boundaries, no concurrent rendering, no ecosystem, and cursor semantics that diverge from React subtly rather than obviously. Every advantage it has can be reproduced *on top of* React/Preact/Solid at a fraction of the onboarding cost.
2. **373 per-language highlight.js chunks.** Correct for a client serving every language on earth; wrong for an internal tool.
3. **Publishing source maps.** Right for a GPLv3 client; an information leak for a proprietary one.
4. **`max-age=3600` on hashed assets.** Simply a bug.
5. **The 5.28 MiB total JS graph.** That number is payments, gifts, auctions, stories, mini-apps, premium and calls. A team building an internal chat app needs perhaps 10–15 % of that surface.
