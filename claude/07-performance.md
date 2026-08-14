# 07 — Performance

**Target:** `https://web.telegram.org/a/` — Telegram Web A, version `12.0.38 A` (confirmed in-product via Settings → Active Sessions, and against `/a/version.txt`).
**Measured:** 2026-08-14. Deployment build timestamp `2026-08-11 15:24:14 UTC`.
**Raw data:** `/home/claude/audit/perfout/{cold,repeat,throttle,a11y,responsive,memidle}.json`; screenshots in the sibling `screenshots/` folder.

This file separates **measured numbers** from **judgement**. Every measured claim carries the caveats in §1. Every interpretive claim is tagged.

**Confidence scale used throughout:**

- **Confirmed** — directly observed in this environment, reproducible from the raw JSON, and not sensitive to the environment caveats.
- **Strong inference** — measured inputs plus a short, low-risk reasoning step.
- **Possible** — consistent with the evidence but with a plausible competing explanation.
- **Unknown** — not measurable here; stated so you don't infer it.

---

## 1. Methodology and caveats — read before quoting any number

### 1.1 Harness

- Python Playwright (`/home/claude/audit/perf.py`) driving headless Chromium, `launch_persistent_context` with a **fresh user-data-dir per cold run**, viewport 1600×1000.
- `PerformanceObserver`s for `largest-contentful-paint`, `layout-shift` and `longtask` installed via `context.add_init_script` — i.e. **before any page script runs**, all with `buffered: true`.
- Navigation `page.goto(..., wait_until="load")`, then wait for the login CTA, then a **6 s settle** before collection, so late LCP candidates and lazily-imported chunks are captured.
- **3 cold runs** as briefed, plus a second independent set of 3 after a harness fix, giving **6 usable timing samples**. Medians of n=6 are reported as the central estimate; both sets are retained so variance is visible.
- Throttled runs used CDP (`Network.emulateNetworkConditions`, `Emulation.setCPUThrottlingRate`), **n=1 each** — indicative only.
- Accessibility used axe-core **4.13.0** injected locally via `page.evaluate()` (CDN injection was correctly blocked by the site's own CSP — see §8 of `notes/performance.md`).

### 1.2 Caveats, ranked by how much they distort the result

| # | Caveat | Effect |
|---|---|---|
| **C1** | **HTTP/1.1 was forced** (`--disable-http2`). Confirmed: every resource reports `nextHopProtocol = "http/1.1"`. The real site serves **HTTP/2**. | With 48 subresources from a single origin, HTTP/1.1's 6-connection limit serialises requests that H/2 would multiplex. **All transfer and waterfall timings are pessimistic. Request counts and byte counts are unaffected.** |
| **C2** | The container's egress proxy **rejected Chromium's TLS ClientHello**, so a TLS-terminating local relay was inserted in the path (`127.0.0.1:8899`). | Adds per-request latency and caps throughput. Measured aggregate throughput on the *unthrottled* baseline was only **0.45 Mbps** — which is why the Fast-3G row is meaningless (§6.3). |
| **C3** | **The performance run's MTProto WebSockets failed.** `wss://zws2.web.telegram.org` closed with code 1006; `POST .../apiw1` failed `net::ERR_ABORTED`; the app logged `[WebSocket connection failed attempt: 1]` then `[Using fallback connection]`. | **Directly inflates time-to-CTA (§4).** *But see §4.3 — the separate authenticated session on the same day had fully working MTProto WebSockets, so this is a property of that run's relay configuration, not a categorical statement that the environment blocks WebSockets.* |
| **C4** | Headless Chromium, **datacenter IP, US egress**, single machine, n=3 (n=6 pooled). No real-user RTT, no thermal or CPU variance, no CDN-edge variance. | Absolute values are not field data. Use them for *relative* comparisons and for the deterministic byte/request facts. |
| **C5** | `performance.memory` is **quantized to 100 KB buckets** (`--enable-precise-memory-info` was deliberately not set). | Heap figures are ±100 KB and GC-timing sensitive. Order-of-magnitude only. |
| **C6** | Three cross-origin `_websync_` beacons (`t.me`, `telegram.me`, `telegram.dog`) report `transferSize = 0` — no `Timing-Allow-Origin`. | Their bytes are excluded from all totals. The undercount is small (redirect/beacon responses). |
| **C7** | **Run-to-run variance is 8–20 % on paint metrics.** | **Any difference under ~400 ms between runs in this report is noise.** Do not read significance into it. |

### 1.3 What is trustworthy regardless of environment

**Confirmed, byte-identical across all runs:** request counts, transfer/encoded/decoded byte totals, the loaded-vs-deployed JS file split, the storage and service-worker inventory, cache-hit behaviour on repeat visit, the accessibility findings, and the responsive breakpoints.

**Not measurable here — do not infer it from this report:** the production HTTP/2 waterfall shape, the true production time-to-CTA, real-user RTT and CDN-edge behaviour, precise heap values, and any field distribution (p75/p95) of anything.

---

## 2. Load timing — the headline table

Median of **n=6** cold runs, all values in ms from `navigationStart`. **Confirmed** (subject to C1/C2/C4/C7).

| Metric | Median (n=6) | Min | Max | Reading |
|---|---:|---:|---:|---|
| `responseStart` | **642** | 545 | 757 | TTFB for the 5.8 KB HTML shell (1,846 B gzipped) |
| `first-paint` | **1,734** | 1,576 | 2,156 | first pixels — background/chrome only |
| `first-contentful-paint` | **2,536** | 2,336 | 2,852 | the `<h1>` and instruction list |
| `largest-contentful-paint` | **2,536** | — | — | **identical to FCP in every single run** — see §2.1 |
| `domInteractive` | **2,266** | 2,157 | 2,700 | |
| `domContentLoadedEventEnd` | **2,438** | 2,260 | 2,767 | |
| `loadEventEnd` | **3,561** | 3,286 | 3,806 | |

Document itself: `transferSize` 2,146 B, `encodedBodySize` 1,846 B, `decodedBodySize` 5,806 B. There is **no SSR, no inlined state, no critical-CSS inlining** — the HTML is a 5.8 KB static shell.

### 2.1 Why LCP == FCP in every run — a measurement artifact, not a win

**Confirmed.** The LCP entry reports `element: "SPAN"`, area **9,799 px²**, no `url`. 9,799 px² ≈ 408 × 24 px — that is the `<h1>` line "Log in to Telegram by QR Code" inside the 408 px-wide `.auth-form`.

The visually dominant element on this screen is the **280 × 280 px QR block (78,400 px², 8× larger)** — but it is rendered as an **inline `<svg>`**, and **SVG elements are not LCP candidates in Chrome**. It therefore never registers. The 54 × 54 px Lottie plane is a `<canvas>`, also not a candidate.

Consequence, and this is the part worth internalising for taskrgram:

- Nothing larger than the heading ever becomes an LCP candidate, so **the first contentful paint is by definition also the largest** — LCP carries zero additional information on this screen.
- An LCP of 2,536 ms (pooled n=6; the n=3 subset used for the throttle comparisons medians 2,640 ms) therefore reads as "good-ish" while the screen is, in fact, **not usable for another six seconds** (§4).
- **Judgement:** if your primary screen's hero element is an SVG, a canvas, or a CSS background, your LCP number is measuring your heading, not your product. Treat LCP as invalid on such screens and instrument an explicit application-level "ready" mark instead. This is a general lesson, not a Telegram defect.

---

## 3. Byte budget

**Confirmed and deterministic** — byte-for-byte identical across all 3 runs (fully content-hashed immutable bundle). Unaffected by C1/C2.

**Totals: 49 requests · 686,041 B transferred · 1,818,438 B decoded.**

| Category | Requests | Transfer | Encoded | Decoded | Compression |
|---|---:|---:|---:|---:|---:|
| script (`.js`) | 31 | 601,876 | 592,576 | 1,635,274 | **2.76×** |
| css (`.css`) | 8 | 48,167 | 45,767 | 144,338 | 3.15× |
| font (`.woff2`) | 2 | 22,672 | 22,072 | 22,072 | 1.00× (pre-compressed) |
| media (`.mp3`) | 1 | 11,180 | 10,880 | 10,880 | 1.00× |
| img (`.png`) | 3 | 0 | 68 | 68 | served from Service Worker cache |
| other (cross-origin beacons) | 3 | 0\* | 0\* | 0\* | \*excluded, see C6 |
| **Subresource subtotal** | **48** | **683,895** | **671,363** | **1,812,632** | 2.70× |
| + document (HTML) | 1 | 2,146 | 1,846 | 5,806 | 3.15× |
| **GRAND TOTAL** | **49** | **686,041** | **673,209** | **1,818,438** | **2.70×** |

### 3.1 Code splitting: it works, and the login screen is still heavy

| Measure | Value |
|---|---|
| JS resource-timing entries on the login screen | 31 |
| JS **unique URLs** | 30 (`index.worker-DLzlUYNq.js` is fetched twice — one per media-worker instance) |
| JS files **in the deployment** | 461 |
| **Fraction of files loaded** | **30 / 461 = 6.5 %** |
| Decoded JS loaded | 1,635,274 B = 1.56 MiB |
| Decoded JS available in deployment | 5,532,659 B = 5.28 MiB |
| **Fraction of JS bytes loaded** | **29.6 %** |

**Confirmed.** Only **6.5 % of chunk files** are pulled, but they carry **29.6 % of all JS bytes** — the login path grabs a disproportionate share of the large chunks. The split itself is real and correct: `main-D0-d5I4l.js` (420 KB), `useConnectionStatus-DLbED6Q4.js` (480 KB), `Modal-BuOILwVN.js` (396 KB) and `editor-CL7uxqfp.js` (295 KB) are all correctly deferred past auth.

### 3.2 The four things on the login path that shouldn't be

| Decoded | Transfer | Asset | Finding |
|---:|---:|---|---|
| 742,096 | 241,222 | `worker-J7_WDuX0.js` | The MTProto/GramJS worker. **45 % of all decoded JS in one file**, and **it does not start downloading until ~2.8 s** — it is requested only when the app boots the worker. On the critical path to a usable login screen (§4). |
| 218,910 | 84,745 | `assets/InputText-CnVXgAD5.js` | A text-input component chunk on a screen with **zero `<input>` elements** (measured: 0). **Strong inference:** barrel-file / shared-chunk bleed from over-broad chunk grouping. |
| 180,125 | 56,211 | `assets/fallback-fgJSCTeC.js` | Second-wave chunk at ~2.4 s. |
| — | 11,180 | `notification.mp3` | **Fetched on the login screen, before any notification can exist.** Pure waste on the first-visit path. |

**Confirmed positive:** **no `.wasm` is requested on the login path** (0 requests) despite `wasm-unsafe-eval` in the CSP and 1.75 MiB of WASM existing in the deployment (`fasttext-wasm` 1.12 MB, `tlottie` 400 KB, opus decoder/encoder). WASM is correctly deferred past auth. That is exactly the right call and worth copying.

### 3.3 Waterfall shape (measured, but see C1)

- Document lands at **~630 ms**.
- **22 JS files and all 8 CSS files are requested in one burst at 789–792 ms** — 21 of them are the entry `<script type="module">` plus 20 `<link rel=modulepreload>`; the 22nd is a static import pulled in the same burst. The 8 `<link rel=stylesheet>` are declared in the HTML.
- A **second wave at ~2.4 s** brings `fallback`, `TLottie`, `qr-code-styling`, the 2 fonts, and `notification.mp3`.
- The **742 KB MTProto worker starts at ~2.8 s**.

**Confirmed absence:** the HTML contains **0 `preconnect`, 0 `dns-prefetch`, 0 `prefetch`, 0 `preload`** hints. There is no early warming of the MTProto WebSocket origins (`zws*.web.telegram.org`), which are on a different hostname from the asset origin. **Strong inference:** a `preconnect` to the DC hostnames would shave the TLS+TCP setup off the critical path to first MTProto response — cheap and low-risk.

**Confirmed (from header forensics, not this run):** assets are served **gzip only — no Brotli, no zstd**, verified by explicit `Accept-Encoding: br` probes. Brotli on the same corpus would realistically remove **~20 %** of transfer bytes for a one-line nginx change.

---

## 4. The readiness gap — the most important finding in this file

### 4.1 Measured

Time-to-CTA was measured with a `requestAnimationFrame` loop installed *pre-navigation*, recording `performance.now()` at the first frame where a `<button>` containing "LOG IN BY PHONE NUMBER" has a non-zero box, `visibility` not hidden, `display` not none, and `opacity > 0.01`.

| Set | Run 1 | Run 2 | Run 3 | Median |
|---|---:|---:|---:|---:|
| Cold set A | 7,965 | 10,042 | 8,341 | 8,341 |
| Cold set B | 12,664 | 9,354 | 8,353 | 9,354 |
| **Pooled (n=6)** | | | | **8,854 ms** (range 7,965–12,664) |

**That is 3.5× the LCP (2,536 ms) and 2.5× `loadEventEnd` (3,561 ms).** On a **fully warm cache** the CTA still takes **5,702 ms** (§7).

### 4.2 Filmstrip evidence

Screenshots in `screenshots/`. Filenames encode the *nominal* capture target; the recorded `performance.now()` at capture drifted slightly later.

| Recorded t | Frame | State |
|---:|---|---|
| 2,153 ms | `screenshots/filmstrip-2000ms.png` | **Blank.** No body text, no `<h1>`. |
| 2,989 ms | `screenshots/filmstrip-2500ms.png` | **`<h1>` + 3-step instruction list painted.** This is the FCP/LCP moment. **No QR code. No buttons.** |
| 3,192 ms | `screenshots/filmstrip-3000ms.png` | Unchanged. |
| 4,015 ms | `screenshots/filmstrip-4000ms.png` | Unchanged. |
| 5,015 ms | `screenshots/filmstrip-5000ms.png` | Unchanged. |
| 7,015 ms | `screenshots/filmstrip-7000ms.png` | Unchanged — still no QR, still no buttons. |
| 9,016 ms | `screenshots/filmstrip-9000ms.png` | **QR `<svg>` and both login buttons appear simultaneously.** |
| — | `screenshots/filmstrip-12000ms.png` | Steady state. |
| — | `screenshots/filmstrip-1500ms.png` | Pre-paint. |

**Measured conclusion (Confirmed for this environment):** the login screen paints a **static shell within ~2.6–3.0 s and then sits visually frozen for roughly six seconds** before the QR code and both CTAs appear together. FCP and LCP fire on the frozen shell and therefore **materially overstate readiness**.

### 4.3 Reconciling the two runs — how much of the six seconds is real?

This is the claim most at risk of being overstated, so it gets its own treatment.

**Evidence A — the performance run (`perfout/cold.json`, `notes/performance.md` §7).** MTProto WebSockets failed. Verbatim console: `Socket zws2.web.telegram.org closed. Code: 1006, reason: , was clean: false` (×4), `Error: Not connected` inside `worker-J7_WDuX0.js`, `[WebSocket connection failed attempt: 1]`, then `[Using fallback connection]` ~1 s later, and `POST https://zws2.web.telegram.org/apiw1 :: net::ERR_ABORTED`. The QR token requires a successful MTProto round trip; here the primary transport died and the client fell back to the MTProto HTTP long-poll path.

**Evidence B — the authenticated session (`notes/live-session-evidence.md` §4), same day, same container, different relay configuration.** MTProto WebSockets **worked**: 17 opens across six `zws*` hosts, **310 frames sent (85,464 B)** and **660 frames received (10,026,931 B ≈ 10 MB)**, with a max single received frame of 218,940 B. That session also saw occasional 1006 closures and `Error: TIMEOUT`, and **recovered from them** — the session continued.

**Therefore:**

- The blanket statement "this environment blocks WebSockets" is **false** — WebSockets demonstrably worked in the authenticated run. What is true is that **the performance run's relay path failed the MTProto WebSocket handshake and the app fell back**, which is a per-run configuration artifact.
- **It follows that the 8,854 ms cold / 5,702 ms warm CTA figures are inflated by an unknown amount and must not be quoted as production numbers.** How much? **Unknown.** We did not measure time-to-CTA in a run with healthy WebSockets, so we cannot bound it.
- Anyone repeating the harsher reading — "the CTA delay is purely architectural" — is **overstating the evidence**. We are not entitled to that claim.

**What survives the caveat (Strong inference, and it is the useful part):**

1. **The ordering is a property of the app, not the network.** The CTA is gated on a server round trip; that round trip cannot begin until the **742 KB MTProto worker is downloaded, parsed and booted**, and that worker is **not even requested until ~2.8 s** — after two prior waves of chunk loading. Faster transport shortens the round trip; it does not move the worker earlier.
2. **The residual on a warm cache is not about bytes.** On reload, 99.6 % of bytes come from cache and `loadEventEnd` halves to 1,547 ms — yet the CTA still lands at 5,702 ms. Bytes stopped being the constraint and the gap did not close proportionally. (Caveat: that warm run *also* ran through the failing-WebSocket path, so the 5,702 ms magnitude is equally suspect. The **shape** — "byte cost collapsed, CTA cost did not" — is what survives, not the number.)
3. **There is no skeleton or progressive affordance in the gap.** For 6 s the user sees a heading and a numbered list with an empty space where the QR belongs, and no indication that anything is loading. That is a UI decision, entirely independent of transport.

**For taskrgram — Judgement:** the transferable lesson is not "Telegram is slow to show its CTA". It is: **(a)** put the transport worker on the *first* wave, not the third, if the first meaningful interaction depends on it; **(b)** never let a network-gated control render into a screen that is already claiming to be painted — render a skeleton or a disabled control with a spinner so the paint metric and the perceived state agree; **(c)** instrument an explicit `performance.mark('ready')` and alert on *that*, because FCP/LCP will lie to you exactly as they lie here.

---

## 5. Stability

| Metric | Median | Detail |
|---|---:|---|
| **CLS** | **0.00035** | 1–2 layout-shift entries per run; largest single shift 0.00100. |

**Confirmed, and this one is genuinely excellent.** 3.5 × 10⁻⁴ is effectively zero. The reason is structural and copyable: the auth form is a **fixed-size 408 px centred block**, so there is nothing to reflow when late content arrives. Note the trade-off with §4 — the layout is stable *because* the QR slot is pre-sized and simply sits empty for six seconds. Zero CLS and a six-second dead zone are the same design decision viewed from two angles.

Responsive behaviour is equally stable: the login screen is **width-invariant from 1920 px down to 601 px** (byte-identical box metrics; only the centring offset changes), with exactly one breakpoint at 600/599 px, and **no horizontal scrollbar at any width down to 320 px**. See `screenshots/responsive-1920px-login-screen.png` through `screenshots/responsive-480px-login-screen.png`. The authenticated three-column layout has three breakpoints, located exactly: **1276/1275, 926/925, 601/600** — see `screenshots/responsive-1280px-channel-view-desktop-layout.png`, `screenshots/responsive-925px-channel-view-desktop-layout.png`, `screenshots/responsive-600px-channel-view-desktop-layout.png`.

---

## 6. Responsiveness — the app is CPU-bound, not byte-bound

### 6.1 Baseline main-thread cost

| Metric | Median (n=3) | Pooled (n=6) |
|---|---:|---:|
| Long tasks (count) | 2 | — |
| Long-task total duration | 160 ms | 164 ms (range 102–193) |
| **TBT proxy** `Σ max(0, dur − 50)` | **72 ms** | 79.5 ms (range 52–118) |

72 ms TBT is comfortably inside Lighthouse's "good" band. **Notable:** the dominant long task in the first set was **168 ms starting at 7,760 ms** — that is the *login-form render triggered by the MTProto response*, **not** bundle parse/eval. Bundle evaluation did not produce a >50 ms long task in most runs. **Strong inference:** the parse/eval cost of 1.64 MB of decoded JS is being absorbed well — plausibly helped by native ESM chunk loading (Rolldown's runtime is 716 bytes; there is no JSONP loader and no webpack runtime).

### 6.2 The 4× CPU throttle result — the trustworthy throttle row

CPU throttling is unaffected by the network caveats (C1/C2), so this row is quotable in a way the network rows are not. **n=1, indicative.**

| Condition | FCP | LCP | `loadEventEnd` | **TBT** | Long tasks | CLS |
|---|---:|---:|---:|---:|---:|---:|
| Baseline (median n=3) | 2,640 | 2,640 | 3,462 | **72** | 2 | 0.00035 |
| **4× CPU throttle** | 2,892 | 2,892 | 3,813 | **753** | 5 | 0.00100 |
| Fast 3G + 4× CPU | 3,528 | 3,528 | 4,286 | 674 | 6 | 0.00096 |

**FCP degraded 10 % (+252 ms). TBT degraded 10.5× (72 → 753 ms), with a single 437 ms task.**

**Measured conclusion (Confirmed):** the JS work is invisible on a fast desktop and **scales badly with CPU**. 753 ms TBT sits in Lighthouse's "poor" range (>600 ms).

**Judgement — and this is the single most actionable line in this file for taskrgram:** a team looking at "686 KB transferred" will instinctively go on a byte diet. On this evidence that would be optimising the wrong axis. Halving the bytes moves FCP by a fraction of a second; halving the main-thread work moves the number that actually determines whether typing feels laggy on a five-year-old corporate laptop. **Budget TBT/INP, not kilobytes.** Byte budgets are still worth having — they are cheap and they catch regressions — but they should be the secondary gate.

**Strong inference:** a mid-tier laptop or mobile device is roughly a 4–6× CPU handicap relative to this container's CPU, so the 4× row is a reasonable proxy for the low end of a corporate fleet. This is an extrapolation, not a measurement — we did not test on real hardware.

### 6.3 The Fast-3G row is not quotable — say so out loud

**Do not cite the Fast-3G numbers as a 3G estimate.** The throttle was largely non-binding because the baseline was already slower than the cap:

- Baseline: 683,895 B over a 12,069 ms load window = **0.45 Mbps effective**.
- Fast-3G run: 683,895 B over a 14,096 ms window = **0.39 Mbps effective** — against a **1.6 Mbps cap**.

The relay (C2) plus forced HTTP/1.1 (C1) already constrained the connection to well below the Fast-3G cap. Throttling *did* bind on individual renderer-initiated fetches (`InputText-CnVXgAD5.js` 1,254 → 2,154 ms, +72 %; `fallback-fgJSCTeC.js` 276 → 718 ms, +160 %), but worker-initiated `importScripts` appears to bypass renderer throttling entirely (`worker-J7_WDuX0.js` measured 153 ms for 241 KB ≈ 12 Mbps, above the cap). **The row is internally inconsistent and should be dropped from any summary.**

---

## 7. Caching — the best-behaved part of the app, and one clear defect

Same profile, `page.reload()`. Both halves measured in one session. **Confirmed.**

| Metric | Cold | Warm reload | Change |
|---|---:|---:|---|
| Subresource requests | 48 | 44 | −4 |
| **Subresource transfer** | 683,895 B | **2,975 B** | **−99.6 %** |
| Subresource decoded | 1,812,632 B | 1,043,834 B | −42 % (fewer chunks needed) |
| Requests served from cache | 1 | **39** | — |
| `responseStart` | 477 ms | 145 ms | −70 % |
| `first-paint` | 1,676 ms | 872 ms | −48 % |
| FCP / LCP | 2,236 ms | 1,396 ms | −38 % |
| `domInteractive` | 2,117 ms | 870 ms | −59 % |
| **`loadEventEnd`** | 3,195 ms | **1,547 ms** | **−52 %** |
| Login CTA visible | 7,837 ms | 5,702 ms | −27 % |
| Long tasks / TBT | 1 / 84 ms | **0 / 0 ms** | eliminated |

Only **5 resources still touch the network**, totalling **2,975 B**: `compatTest.js` (1,450 B) and `redirect.js` (625 B), both unhashed and therefore not immutably cacheable, plus three 300 B worker `importScripts` revalidations.

A service worker (`service.worker-BSeu-kQn.js`, hand-written — **0 hits for `workbox`**) is registered, `activated`, and **already controlling the page on the very first unauthenticated load** (`navigator.serviceWorker.controller` non-null), implying `skipWaiting()` + `clients.claim()`. The `tt-assets` cache grows from **1 entry on cold load to 42 after reload** — the SW precaches the shell in the background after first paint.

### 7.1 The defect: `cache-control: max-age=3600` on immutable assets

**Confirmed** from response headers: **every** asset — including content-hashed, immutable ones like `assets/index-IZ97MA_m.js` — is served with `cache-control: max-age=3600`, no `immutable`, weak ETags (`W/"6a7b3e9e-7117"`, shared build-mtime prefix).

The URL scheme is already correct for long-lived caching: `assets/<name>-<hash8>.js` with a base64url 8-char content hash. **The one-hour TTL throws that away** — every returning user more than an hour later revalidates up to ~490 assets. The service worker masks most of the damage for repeat visitors, which is precisely why the warm numbers above look so good; the header is still wrong, and any user whose SW is not yet installed or has been evicted pays for it.

**Fix, for anyone copying this:** `max-age=31536000, immutable` on `assets/*`, short TTL with revalidation on `index.html` only. Free win, one config line. Same for enabling Brotli (§3.3).

**Judgement:** the contrast is instructive. Telegram got the hard part right (content hashing, a hand-written SW that reaches 99.6 % byte elimination) and the trivial part wrong (a header). **Verify your cache headers empirically — don't assume the build pipeline's naming scheme implies the CDN's policy.**

---

## 8. Runtime performance under real use — authenticated session

All from `notes/live-session-evidence.md`. Same environment caveats apply to timings; the structural facts are **Confirmed**.

### 8.1 The message list is a bounded sliding window, not a virtualizer

Public channel @TelegramTips (11.19 M subscribers, years of history), viewport 1600×1000, scrolling upward:

| Action | `.Message` nodes in DOM | `scrollHeight` | JS heap |
|---|---:|---:|---:|
| Chat opened | 29 | 20,317 px | — |
| scrollTop −4,000 px | 29 | 20,317 px | — |
| scrollTop −12,000 px | 29 | 20,317 px | — |
| scrollTop −25,000 px | 89 | 61,266 px | 54.6 MB |
| scrollTop −40,000 px | **89** | **61,266 px** | **52.3 MB** |

**Confirmed:** nodes are **not** recycled per-pixel (react-window style). `scrollHeight` grows in discrete jumps as slices load, node count rises 29 → 89 and then **stops**. The **heap went down** between the last two samples (54.6 → 52.3 MB) while scrolling *further*, i.e. an older slice was released. (Heap values ±100 KB per C5, and GC timing is not controlled — but a 2.3 MB *decrease* under continued scrolling is well outside the quantization noise and is the right direction.)

The source confirms the design: `useInfiniteScroll.ts` keeps a viewport slice (`DEFAULT_LIST_SLICE = 30`) recomputed around an anchor; `InfiniteScroll.tsx` stores `currentAnchor`/`currentAnchorTop` and restores `scrollTop` in a layout effect via `requestForcedReflow`; `ChatList.tsx` renders rows **absolutely positioned** at computed offsets so container height is constant and reordering is a transform rather than a reflow.

**Judgement for taskrgram:** this is the right trade for a chat log and it is markedly simpler than true virtualization. You keep **native scrollbars, working find-in-page, and variable-height messages for free**; you pay with a hard cap on how much history can be mounted at once. For an internal team-chat app with threads of tens-to-hundreds of messages, take this design. See `screenshots/23-desktop-message-list-scrolled-up-virtualization-test.png`.

### 8.2 Storage growth

`navigator.storage.estimate()` after a few minutes browsing **one** channel:

```
usage: 16,652,983 B   (quota 162,331,904,409 B)
  caches:                      16,321,280 B   (98.0 %)
  indexedDB:                      322,094 B
  serviceWorkerRegistrations:       9,609 B
```

**Confirmed:** **16.65 MB after minutes on a single channel**, 98 % of it media in Cache Storage. Five cache buckets are provisioned (`tt-media`, `tt-media-avatars`, `tt-media-progressive`, `tt-lang-packs-v52`, `tt-assets`). **Strong inference:** on a real workload across dozens of chats this grows into the hundreds of MB — which is why the product ships an explicit **Data and Storage → Clear Media Cache** control plus per-chat-type auto-download matrices and a max-file-size cap (`screenshots/28-desktop-settings-data-and-storage-cache-controls.png`). If you cache media aggressively, you owe the user a cache manager. Budget for it in v1, not v3.

### 8.3 Media comes down the WebSocket, not from a CDN

**Confirmed.** Over the session: **310 frames sent = 85,464 B; 660 frames received = 10,026,931 B**, a **~117:1 downstream asymmetry**, max single received frame 218,940 B. Photos, video and stickers arrive as MTProto `upload.getFile` chunks over the WebSocket and are handed to the page as `blob:` URLs. The service worker synthesises **HTTP 206 Partial Content** responses at `/a/progressive/document<id>` so a native `<video>` element can seek — confirmed at runtime.

**Judgement:** this is the single biggest architectural divergence from a conventional web chat app, and it is *forced* by MTProto (files are addressable by `(dc_id, file_reference)`, not by URL). **taskrgram should not copy this.** If your files can live behind signed HTTPS URLs, put them there: you get CDN edge caching, browser HTTP cache, Range requests, and `<img>`/`<video>` native loading for free, none of which Telegram gets. The 10 MB over WebSocket is a cost of their protocol, not a technique to emulate.

### 8.4 Idle memory on the login screen

Three independent load+60 s-idle pairs: **heap delta exactly 0 in all three** (20.5 → 20.5, 13.4 → 13.4, 20.5 → 20.5 MB). Load-time spread across 5 samples was 13.4–26.0 MB — GC-timing noise, since the same bytes load every run. `jsHeapSizeLimit` 2,330 MB.

**No idle leak detectable on the login screen at 60 s (Confirmed).** **Unknown:** leak behaviour in a real long-lived session — 60 s unauthenticated exercises almost none of the app. Do not generalise this.

---

## 9. Performance techniques the source shows they use deliberately

Each row pairs a technique read from the public source with something observable in the measurements. Techniques are **Confirmed** (quoted from source); the linkage is **Strong inference** unless noted.

| Technique | Source evidence | What it plausibly buys, observably |
|---|---|---|
| **RAF-ordered update pass** | `teact.ts`: a documented single frame pipeline — *effects → measures (DOM reads) → render → layout effects → mutations → forced-reflow measure → forced-reflow mutate*. `fasterdom.ts` maintains `pendingMeasureTasks`/`pendingMutationTasks`/`pendingForceReflowTasks`, flushed once per RAF, phases sequenced through promise microtasks. | Read/write phases never interleave, so layout thrashing is structurally prevented. Consistent with **CLS 0.00035** and with bundle eval producing no >50 ms long task in most runs. |
| **A runtime linter for layout thrashing** | `stricterdom.ts` — `enableStrict()` in DEBUG **throws** when code reads layout in the mutate phase or writes in the measure phase; `layoutCauses.ts` enumerates the offending properties. | Not directly observable at runtime (DEBUG-only), but it is why the discipline above actually holds across 183 k LOC of components. **Highest-leverage single idea in the repo.** |
| **Deferring updates during heavy animation** | `runUpdatePassOnRaf` checks `getIsBlockingAnimating()` and re-queues itself; TeactN's `runCallbacks` does the same with `getIsHeavyAnimating()`; `beginHeavyAnimation(duration, isBlocking)` is called by every screen transition. | State updates and container re-mapping simply do not run during a transition — animation frames are protected by construction rather than by hoping the work is cheap. |
| **Signals that bypass render** | `src/util/signals.ts` (~90 LOC); `useEffect` dependencies may be signals, in which case the effect *subscribes* rather than comparing, and the component never re-renders. Context is implemented **as signals**, so a context change does not cascade re-renders. | The intended targets are the hot paths (typing, caret position, animation state). **Strong inference:** part of why baseline TBT is only 72 ms despite a large component tree. |
| **Idle-gated cache writes** | `cache.ts`: `UPDATE_THROTTLE = 5000`, `throttle(() => onFullyIdle(() => updateCache()), …)`, and `updateCache` bails outright if `getIsHeavyAnimating()`. Persistence is IndexedDB via `idb-keyval`, with explicit whitelisting (`reduceGlobal`) and caps (500 users / 200 chats / 150 custom emoji). | Explains **IndexedDB at only 322 KB** while caches hold 16.3 MB — the persisted state is a deliberately slimmed projection, not the live store. |
| **Manual code splitting** | No router-driven splitting. 6 explicit barrel bundles (`Auth`, `Main`, `Extra`, `Calls`, `Stars`, `Editor`) behind `moduleLoader.ts`, plus **151 `*.async.tsx`** shims that receive `isOpen` and fetch on first need. | Directly visible as the **6.5 % of files / 29.6 % of bytes** split, and in the correct deferral of `main`, `Modal`, `editor`, `useConnectionStatus`. Also visible as **373 of 461 chunks being single-language highlight.js grammars** — an unusually aggressive per-feature split. |
| **CI bundle-size diffing** | `rollup-plugin-bundle-stats` with a baseline file for PR comparison, plus a custom plugin that merges **worker** bundles into the report so worker size is not invisible. `build:stats` / `build:stats:visualizer` scripts. | The one guard that keeps the split honest over time. **Note the detail worth stealing: they explicitly instrument workers**, because the 742 KB MTProto worker would otherwise be invisible to a naive bundle report — and it is the single largest asset in the deployment. |
| **Compositable-only animation, enforced** | `stylelint-high-performance-animation` blocks animating non-compositable properties at lint time; `will-change` applied through a side-channel rather than statically. | Architecture rules encoded as lint rules — the pattern, more than any individual rule, is the transferable idea. |
| **A user-facing performance panel** | `perfomanceSettings.ts` maps a 3-stop slider (`Power Saving / Nice and Fast / Lots of Stuff`) plus **10 individually-toggleable interface animations** to body classes; the two `backdrop-filter` effects (Context Menu Blur, Message Blur) are broken out separately from the animations they accompany. | The `with-message-blur` class was observed live on `<body>`. See `screenshots/26-desktop-settings-animations-and-performance-toggles.png` and `screenshots/27-desktop-settings-animations-performance-expanded-granular-toggles.png`. Blur is the expensive one and they knew it. |

**One negative finding worth carrying forward:** across all runs there were **zero application-level JS exceptions and zero `pageerror` events**. Every console message was transport (WebSocket/MTProto) or GPU (software-WebGL fallback in headless). The one genuine app-side smell is a bare `[error] undefined` — a `console.error` logging an undefined value, i.e. an error path that swallows its own diagnostic. Also note there is **no error-reporting SaaS in the bundle at all** (0 hits for `sentry`, and 0 for every analytics vendor searched). That is a deliberate privacy stance; for an internal tool it is a gap you must fill yourself.

---

## 10. Prioritised — what we would fix

Ordered by (impact × confidence) ÷ effort. "Impact" is judgement; the evidence column is measured.

| # | Fix | Evidence | Effort | Confidence in the impact |
|---|---|---|---|---|
| 1 | **`cache-control: max-age=31536000, immutable` on `assets/*`** (short TTL on `index.html` only) | Every hashed asset served `max-age=3600` | One config line | **Confirmed** defect; impact **Strong inference** |
| 2 | **Move the transport worker to the first request wave**, or start the auth round trip from a small bootstrap module and hydrate the full worker after | 742 KB worker not requested until ~2.8 s; CTA gated behind it | Medium — needs a worker-boot split | **Strong inference** |
| 3 | **Render a skeleton/disabled state in the CTA slot** instead of empty space | 6 s of visually frozen shell across `screenshots/filmstrip-3000ms.png` → `filmstrip-7000ms.png` | Low | **Confirmed** UX gap; independent of transport |
| 4 | **Enable Brotli** | gzip-only confirmed by explicit `Accept-Encoding` probes | One config line | ~20 % transfer reduction, **Strong inference** |
| 5 | **Budget TBT/INP, not bytes** — CI gate on main-thread blocking under 4× CPU throttle | TBT 72 → 753 ms (10.5×) while FCP moved 10 % | Medium (harness) | **Confirmed** measurement; framing is judgement |
| 6 | **Split the `InputText` chunk** off the auth path | 219 KB decoded on a screen with 0 `<input>` elements | Low–medium | **Strong inference** (barrel bleed) |
| 7 | **Don't fetch `notification.mp3` before login** | 11,180 B on the pre-auth path | Trivial | **Confirmed** waste, small |
| 8 | **Add `preconnect` to the realtime origin** | 0 resource hints of any kind in the HTML; WS origin differs from asset origin | Trivial | **Possible** — unquantified here |
| 9 | **Instrument an explicit app-level `ready` mark and alert on it** | LCP == FCP in 6/6 runs; LCP is 3.5× optimistic vs CTA | Low | **Confirmed** that LCP is uninformative here |
| 10 | **Ship a media-cache manager from v1** | 16.65 MB after minutes on one channel, 98 % in Cache Storage | Medium | **Strong inference** |

---

## 11. What we would need to measure properly

Everything above is a lab measurement through a distorting path. To make performance decisions for taskrgram on real evidence, these are the gaps, roughly in order of how much they would change the picture:

- **Real HTTP/2 (and ideally H/3).** C1 is the largest single distortion. The site serves H/2; we forced H/1.1, which serialises 48 same-origin subresources behind 6 connections. The waterfall shape in §3.3 is the metric most likely to be wrong.
- **A direct network path with no TLS-terminating relay.** C2 capped aggregate throughput at 0.45 Mbps, which invalidated the Fast-3G comparison entirely and inflates every transfer timing.
- **A run with verified-healthy MTProto WebSockets, for time-to-CTA specifically.** This is the one number we most want and least have. The authenticated session proved healthy WebSockets are achievable in this container; the performance run did not have them. Repeating §4 under a known-good transport would let us separate the architectural component of the readiness gap from the transport component — which is exactly the split we currently cannot make.
- **Real hardware, plural.** Headless Chromium in a datacenter container is not a corporate laptop. The 4× CPU row is a proxy; measure on the actual fleet, including the oldest supported machine.
- **Residential networks and realistic geography.** US datacenter egress to a DC2 (Amsterdam) origin is not representative of anyone's office.
- **RUM (field data), with p75 and p95, not medians.** n=6 lab runs with 8–20 % variance cannot detect anything smaller than ~400 ms. Field data is the only way to see the tail, which is where user complaints live. Note Telegram itself has *no* telemetry of any kind in the bundle, so they are flying on lab data plus user reports too — that is a choice you do not have to copy for an internal tool, where instrumenting your own employees' clients is comparatively uncontroversial.
- **Authenticated-session profiling over hours, not minutes.** Everything in §8 is minutes-scale on one channel. Memory-leak behaviour, cache growth, and update-storm handling in a long-lived multi-tab session are **Unknown**.
- **INP under real interaction** (typing in the composer, opening the media viewer, switching chats). We measured load, not interaction. For a chat app used all day, INP matters more than LCP.
