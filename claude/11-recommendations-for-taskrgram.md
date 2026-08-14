# 11 — Recommendations for taskrgram

**Audience:** the taskrgram engineering team — internal, desktop-first team chat, built fresh, not a clone
**Basis:** this audit of Telegram Web A v12.0.38 (`web.telegram.org/a/`), the public repo `Ajaxy/telegram-tt`
@ `d915b1b9`, live authenticated session measurements, bundle forensics, and load-performance runs, all
2026-08-14
**Posture:** opinionated. Every recommendation below traces to a specific finding in this audit. Where a
recommendation contradicts what Telegram did, that is deliberate and the reason is stated.

---

## 1. Licensing — read this before anyone opens the repo

**I am not a lawyer. This is an engineering reading of license files, not legal advice. Get it confirmed by
whoever owns legal at your company before writing a line of code that was informed by this repo.**

### 1.1 The facts

| Component | License | Holder | Evidence |
|---|---|---|---|
| **Telegram Web A** (`Ajaxy/telegram-tt`) — the application | **GPL-3.0-or-later** | **Alexander Zinchuk** (`alexander@zinchuk.com`) — *not* Telegram FZ-LLC | `LICENSE` is the full unmodified GNU GPL v3, 674 lines; `package.json:5-6` declares `"author": "Alexander Zinchuk"`, `"license": "GPL-3.0-or-later"` |
| **Teact** (`Ajaxy/teact`) — the rendering framework, extracted | **MIT** | Alexander Zinchuk, © 2022 | `LICENSE` reads "MIT License / Copyright (c) 2022 Alexander Zinchuk"; `package.json` declares `"license": "MIT"`, zero dependencies |
| **GramJS** upstream (`gram-js/gramjs`) | **MIT** | GramJS authors | listed as MIT in telegram-tt's own README dependency section |
| Other vendored deps | MIT / Apache-2.0 | various | fflate MIT, `@cryptography/aes` Apache-2.0, tlottie MIT, Tiptap MIT, lowlight MIT, idb-keyval Apache-2.0, qr-code-styling MIT, music-metadata MIT |

Two details that matter and are easy to miss:

- **The repo has no per-file copyright headers.** A `grep -ri copyright src/` returns only Telegram's
  "report reason: copyright" enum values. The license notice lives *only* in `LICENSE` and `package.json`.
  So a copied file carries no marker — which makes accidental contamination easy and hard to detect later.
- **The Web A README does not restate the license.** GPL-3.0-or-later is discoverable only from `LICENSE`
  and `package.json`. Likewise, Teact being MIT rather than GPL is discoverable only from the *separate*
  `Ajaxy/teact` repo's own files. Do not assume anyone on your team knows either fact.

### 1.2 What this means, plainly

**GPLv3 obligations attach on *conveying*, not on running.** GPLv3 has no network-use clause — that is the
AGPL, and this is not the AGPL. I checked the license text: it is plain GPLv3 with only the standard
Affero-compatibility clause (§13), which permits *combining* with AGPL code and does not itself impose a
network condition.

Practical consequences:

- **Internal-only web use does not trigger source-disclosure obligations.** An employees-only chat client,
  served from your own servers, never distributed outside the legal entity, does not convey anything to
  anyone. Running a GPL'd program is not conveying it.
- **"Internal" is more fragile than it sounds.** Each of these is arguably conveying, and each one pulls in
  full corresponding-source obligations for the *whole combined work* under GPLv3, plus license and
  copyright notices, plus the anti-tivoization installation-information terms for User Products:
  - shipping a **desktop binary** — Electron, Tauri, anything. Note that telegram-tt already contains a
    Tauri target (`tauri/`, `docs/TAURI.md`, five `@tauri-apps/*` runtime deps compiled into the *web*
    bundle), so "we only run it on our servers" would not survive the first desktop build. **taskrgram is
    desktop-first. Assume you will ship a binary. Assume conveying.**
  - serving it to **contractors, a partner company, a customer, an acquired subsidiary on a different legal
    entity** — practitioners widely treat serving GPL'd JavaScript to a third party's browser as
    distribution of that JavaScript to them;
  - publishing it to an app store or an internal store that reaches a different entity.
- **The permissive escape hatch is real but narrow.** Teact is MIT and could legitimately be used directly.
  GramJS is described as MIT *upstream* (the version in telegram-tt is a modified fork inside a GPL repo — take
  it from upstream, not from here). Our only source for that is telegram-tt's own README, not the upstream
  project's LICENSE file: **verify `gram-js/gramjs`'s own LICENSE before relying on this.** The application around them — components, global state, selectors, the service
  worker, the media pipeline — is GPL.

### 1.3 The recommendation

1. **Treat `Ajaxy/telegram-tt` as reference architecture to read, not a codebase to fork or paste from.**
   Ideas are not copyrightable; expression is. Read it, understand it, write your own. Everything in this
   audit's §5 ("steal this") is an *idea*.
2. **No copy-paste. Not a component, not a hook, not a utility file, not a 40-line scheduler.** The absence
   of per-file headers means a pasted file leaves no trace, and "we'll clean it up before we ship" has never
   once worked.
3. **Do not use Teact**, even though it is MIT — for the engineering reasons in §4, not licensing ones.
4. **Keep a written record** of what you read and what you decided independently. If you later ship a
   binary and someone asks, "we read the GPL project for architectural ideas and implemented our own" is a
   defensible position only if you can show the decision trail.
5. **Get sign-off from legal now, before the first sprint**, on exactly one question: *"we are building a
   desktop-distributed internal chat app, informed by reading a GPL-3.0-or-later project; what is our
   contamination boundary?"* Getting that answer in week one costs an email. Getting it in month nine costs
   a rewrite.

---

## 2. P0 — do these before the first vertical slice ships

### P0-1 · Pick a real framework and write a 200-line frame scheduler on top of it

**Do:** React 19 (or Preact/Solid — see §6). Then port the *idea* of `fasterdom`: a module exposing
`requestMeasure(fn)` / `requestMutation(fn)` / `requestForcedReflow(fn)`, flushing once per `requestAnimationFrame`,
with reads batched before writes. Add the `stricterdom` half: in dev builds only, patch the layout-causing
property getters and **throw** if anything reads layout during the mutate phase or writes during the measure
phase.

**Why:** Finding — Telegram's frame model (`AGENTS.md`'s 7-step frame table, `src/lib/fasterdom/`, 3 files)
is the single highest-leverage idea in the audited codebase, and the author's own Fasterdom writeup
explicitly says this is *not* a React deficiency: *"Web frameworks such as React or Teact already respect
this during virtual DOM operations."* The differentiator is applying phase separation to **application
code**. That is framework-agnostic. Meanwhile Teact's costs — no error boundaries, no Suspense, no devtools,
cursor-based hooks that need a bespoke ESLint plugin to stay safe, manual GC on unmount — are all avoidable.

**Effort:** ~3 days for the scheduler + strict mode. (Compare: writing a framework, which is what the
alternative actually is.)

**If skipped:** you either (a) write your own framework and spend a permanent senior headcount on it, or
(b) skip the frame discipline entirely and discover layout thrashing in a profiler in month nine, when the
offending code is spread across 200 components and nobody remembers why.

### P0-2 · HttpOnly, SameSite session cookie. Never a token in `localStorage`

**Do:** authenticate against your own backend, set an `HttpOnly; Secure; SameSite=Lax` session cookie, and
put a CSRF token on state-changing requests. Your WebSocket authenticates from the same cookie or a
short-lived ticket fetched over the authenticated channel.

**Why:** Finding — we enumerated storage on a live authenticated session and found **MTProto auth keys
sitting in plaintext `localStorage`**: `dc1_auth_key`, `dc2_auth_key`, `dc4_auth_key`, 514 chars each
(≈256-byte key, hex), plus `user_auth` and `account1`. `document.cookie` was **empty**. Any XSS, any
malicious browser extension with storage permission, or anyone with filesystem access to the browser profile
takes over the account permanently. Telegram has a structural excuse — MTProto requires the client to hold
the key material to do its own crypto, so there is nowhere else to put it. **You do not have that excuse.**
Your transport is HTTPS to your own server; the browser can hold the credential where JavaScript cannot
reach it.

**Effort:** ~1 day if done at the start. Weeks if retrofitted.

**If skipped:** one XSS is a full, silent, persistent account takeover for every affected user, on an
internal app where the "accounts" are employees and the contents are company data. This is the single worst
pattern in the audited product to copy by accident.

### P0-3 · Bound the message list; do not reach for a virtualizer

**Do:** a bounded sliding window. Load by page (~50 messages), keep at most ~2 pages mounted, trim the far
end, drive load-more from an `IntersectionObserver` sentinel, and preserve scroll by anchoring on a stable
element whose top offset you re-apply in a layout effect.

**Why:** Finding — measured live on an 11.19M-subscriber channel with years of history: DOM held **29 nodes,
then 89 nodes, and stopped growing**; `scrollHeight` went 20,317 → 61,266 px in discrete jumps; **heap went
*down*** (54.6 → 52.3 MB) as the older slice was released
(`screenshots/23-desktop-message-list-scrolled-up-virtualization-test.png`). Source confirms the intent:
`MESSAGE_LIST_SLICE = isBigScreen ? 60 : 40`, `MESSAGE_LIST_VIEWPORT_LIMIT = SLICE * 2`. This is *not*
`react-window`; nodes are never recycled. The payoff is that you keep native scrollbars, native find-in-page,
correct variable heights (an image loads, a reaction row appears, a thread preview expands), and correct
keyboard scrolling — all for free.

**Effort:** ~1 week including scroll anchoring, which you would have to write for a virtualizer anyway.

**If skipped:** you adopt a virtualizer, then fight it for a quarter over variable heights and scroll jumps.
Scroll jumping during reading is the most viscerally bad failure mode a chat app has.

### P0-4 · Decide the multi-window model on day one

**Do:** split your state shape *now* into **per-window UI state** (open panel, scroll position, draft focus,
modal stack, active channel) and **per-user shared state** (settings, read cursors, theme, notification
prefs). Even if you keep one WebSocket per tab, keep the shapes separate and route read-state and
notification suppression through the shared half.

**Why:** Finding — Telegram bolted multi-tab on in **1.59.0 (2023-01-30), two years after 1.0.0**, and the
retrofit cost is visible: `tabId` threaded through hundreds of actions and selectors, a **custom ESLint
plugin (`eslint-plugin-tt-multitab`) written solely to enforce that threading**, a `SharedWorker` for shared
state, `BroadcastChannel` proxying for non-master tabs, master election in `establishMultitabRole.ts`, and
`AGENTS.md` having to say *"**`tabId` is required**"* in bold. taskrgram is **desktop-first**: your users
will have three windows open and two of them will be the same channel.

**Effort:** ~0 extra if decided up front; multiple weeks and a lint plugin if not.

**If skipped:** double notifications, unread counts that fight each other, drafts overwriting each other,
and a painful refactor exactly when the app is too big to refactor.

### P0-5 · Self-hosted error tracking from commit one — invert Telegram's telemetry stance

**Do:** GlitchTip or self-hosted Sentry inside your perimeter. Capture unhandled exceptions, unhandled
promise rejections, failed sends, reconnect storms, and service-worker install failures. **Scrub message
content and file names at the SDK layer, not the server.** Upload source maps at deploy; serve none (P1-3).

**Why:** Finding — Telegram ships **zero** error reporting: 0 hits for `sentry`, `google-analytics`, `gtag`,
`amplitude`, `mixpanel`, `segment`, `posthog` across all 461 chunks; 737 HTTP responses in a live session
touched **zero third-party hosts**. Combined with Teact having **no error boundaries** (a throwing component
silently re-renders its last good output via `safeExec`'s `rescue`), they are structurally blind. We found a
live user-visible defect in one hour of walkthrough — a chat-header button whose accessible name is the raw
i18n key `aria-label="AccDescrPollVoteDown"` — plus a `console.error(undefined)` call, an error path that
swallows its own diagnostic. Telegram pays that price for a brand promise. **You have no such promise, and
you owe your colleagues a working app.** Take "no data leaves the perimeter"; reject "no data".

**Effort:** ~2 days including the scrubbing config.

**If skipped:** you find out your app is broken for the Berlin office when someone walks over.

### P0-6 · Error boundaries around the four risky surfaces

**Do:** React error boundaries around the message list, the composer, the right-hand panel, and each modal —
each with a small "this panel crashed, reload" affordance and a report to P0-5.

**Why:** Finding — Teact has no error-boundary concept at all; `renderComponent`'s rescue path logs to
console and reuses the previous render. This is one of the concrete capabilities you get *free* by not
writing your own framework (P0-1), and it converts "the whole app is a white screen" into "one panel is
broken and we already know about it".

**Effort:** ~half a day.

**If skipped:** a single bad message payload white-screens the app for everyone in that channel.

### P0-7 · Accessibility baseline in CI, not in a backlog ticket

**Do:** `axe-core` in your component tests and on 5–10 key screens in CI, failing the build on serious/critical
violations. Plus a hard rule set: real `<button>`s, a visible focus ring at ≥3:1 contrast, one `<h1>` per
screen, landmarks (`<main>`, `<nav>`), no `user-scalable=no`, and `prefers-reduced-motion` honoured.

**Why:** Finding — the audited product's a11y posture is exactly what a rubric with no accessibility line
produces. GitHub issue #128, *"Improve screen reader accessibility"*, opened **2022-04-05** by an actual
screen-reader user, is **still open with no assignee, no label, no milestone**. Our own axe run on the login
screen found: **both** primary CTAs failing WCAG AA contrast (`#3390ec` on `#ffffff` = **3.31:1**, needs
4.5:1); **zero landmarks of any kind**; `user-scalable=no` blocking pinch-zoom; and a focus indicator that is
an 8%-alpha background tint measuring **~1.08:1** against white, where WCAG 2.2 SC 1.4.11 requires 3:1 —
axe does not catch that one, we measured it by pixel diff
(`screenshots/a11y-button-unfocused.png`, `screenshots/a11y-button-focused.png`). Deeper: the architecture
that makes transitions cheap — **keeping inactive screens mounted inside `Transition` containers** — is the
same architecture that makes the accessibility tree ambiguous. Every `querySelector` in our audit matched 3+
elements, only one visible; we had to qualify everything with `:visible`. A screen reader has the identical
problem. If you copy the "keep screens mounted" trick, you must `aria-hidden` and `inert` the inactive ones.

**Effort:** ~3 days to set up; ongoing cost near zero if it starts on day one.

**If skipped:** an internal tool with a legal accessibility obligation, retrofitted at 100k lines. Ask
anyone who has done it.

---

## 3. P1 — do these in the first two months

### P1-1 · Animation gating: `beginHeavyAnimation()` / `onFullyIdle()`

**Do:** a module exposing `beginHeavyAnimation(durationMs)` returning an `end()` function, plus
`onFullyIdle(cb)` = `requestIdleCallback` **and** no animation running. Gate on it: non-critical store
subscriber flushes, persisted-cache writes, IntersectionObserver processing, search index rebuilds, unread
recomputation.

**Why:** Finding — `src/lib/teact/heavyAnimation.ts` is **73 lines**, and its consumers are the whole
smoothness story: Teact's `runUpdatePassOnRaf` defers component updates, TeactN's `runCallbacks` defers
container mapping, `cache.ts` defers IndexedDB writes, `useIntersectionObserver` freezes observers,
`folderManager` defers recompute. `AGENTS.md`'s performance checklist rule #1 is literally *"Animations
first"*. This is why a transition in that app does not stutter when 40 messages arrive mid-slide.

**Effort:** ~2 days including wiring the consumers.

**If skipped:** every animation in your app is hostage to whatever else happened to fire that frame, and the
jank is non-reproducible.

### P1-2 · Signals for high-frequency values

**Do:** keep typing text, caret position, scroll offset, upload progress, playback time, presence ticks, and
"is dragging" **out of the render path**. Solid has signals natively; React has `useSyncExternalStore`;
Preact has `@preact/signals`.

**Why:** Finding — `AGENTS.md`: *"Signals deliver updates **without causing component renders**. Use for
frequently-updated values"* — and it names exactly these cases. The genuinely novel bit in Teact is that a
signal can appear in a `useEffect` dependency array and converts that effect into a push-based subscriber.
You can reproduce the *effect* with `useSyncExternalStore` + a subscription without reproducing the
framework. Related: their store's `withGlobal` costs O(mounted containers × selector cost) per tick, which
is why `AGENTS.md` bans loops in selectors and bans allocating objects in them. Whatever store you pick,
adopt those two rules.

**Effort:** ~2 days, then a convention.

**If skipped:** a composer that re-renders the message list on every keystroke. This is the most common way
chat apps become slow.

### P1-3 · Fix the delivery layer Telegram did not

**Do:**

- `Cache-Control: max-age=31536000, immutable` on content-hashed assets; short TTL + revalidation on
  `index.html` only;
- Brotli (and zstd if your edge supports it);
- HTTP/2 minimum, HTTP/3 if available;
- HSTS with a sensible max-age;
- `sourcemap: 'hidden'` — emit maps, strip the `sourceMappingURL` comment, upload to P0-5, **serve none**.

**Why:** Findings, all measured on the live deployment:

- `cache-control: max-age=3600` on **everything**, including content-hashed immutable assets like
  `index-IZ97MA_m.js`. A full revalidation storm every hour across ~490 assets. This is the single clearest
  performance defect in an otherwise obsessively performance-tuned product.
- **gzip only.** Verified: `Accept-Encoding: br` → served identity; `br, gzip, deflate, zstd` → gzip. Brotli
  alone would cut roughly 20% off a 2.32 MiB transfer.
- **No `alt-svc`** → no HTTP/3. **No `strict-transport-security`** on any response.
- **453 of 461 chunks ship publicly-resolving source maps**, exposing 2,056 original file paths including
  internal codenames (`src/lib/vibecalls`, `src/components/gili`). Defensible for a GPL project; not for a
  proprietary internal one.

**Effort:** ~1 day of config. This is the highest ratio of benefit to effort in the entire audit.

**If skipped:** you inherit a documented, avoidable defect and hand your module graph to anyone who looks.

### P1-4 · A bounded, whitelisted, migration-versioned persisted cache

**Do:** persist to **IndexedDB** (via `idb-keyval` or equivalent), not `localStorage`. Whitelist what gets
persisted rather than blacklisting. Bound every list. Strip optimistic/local-only state before writing.
Throttle writes and gate them on idle. **Every state-shape change requires a migration entry**, enforced in
review.

**Why:** Finding — `src/global/cache.ts` (1,013 LOC) is a model implementation:
`GLOBAL_STATE_CACHE_USER_LIST_LIMIT = 500`, `CHAT_LIST_LIMIT = 200`, `ARCHIVED_CHAT_LIST_LIMIT = 10`,
`CUSTOM_EMOJI_LIMIT = 150`; `reduceGlobal()` explicitly `pick()`s the persisted keys and runs per-slice
reducers; optimistic media is stripped (`omitLocalMedia`, `omitLocalPhoto`, `clearCachedDraftLocalFlags`);
writes go through `throttle(() => onFullyIdle(updateCache), 5000)` and bail if
`getIsHeavyAnimating()`; and `unsafeMigrateCache` is ~250 lines of versioned fixups with a hard rule in
`AGENTS.md`: *"When adding a new required section to `GlobalState`, always add a corresponding entry in
`migrateCache`."* This is what makes their warm start fast (`screenshots/39-desktop-warm-reload-restored-chat-list-from-cache.png`)
without the cache becoming a liability.

**Effort:** ~1 week.

**If skipped:** an unbounded cache that grows until it is slower than the network, and a deploy that
white-screens every returning user because the persisted shape no longer matches the selectors.

### P1-5 · Stale-write detection in dev — 10 lines, whole bug class gone

**Do:** stamp each store snapshot with a random id on read; throw on write if the id is stale.

**Why:** Finding — verbatim from `teactn.tsx`:

```ts
if (!options?.forceOutdated && newGlobal.DEBUG_randomId && newGlobal.DEBUG_randomId !== DEBUG_currentRandomId) {
  throw new Error('[TeactN.setGlobal] Attempt to set an outdated global');
}
```

This kills the classic `read global → await → write stale global` bug — which in a chat app manifests as
messages reappearing, unread counts resurrecting, and drafts resurrecting after send. It is ten lines and it
is dev-only.

**Effort:** ~1 hour. This audit's source notes flag it as "worth stealing outright" and that is correct.

**If skipped:** you will debug a resurrected-message bug for two days in month six.

### P1-6 · Encode your architecture as lint rules

**Do:** write custom ESLint rules for whatever invariants you decide are non-negotiable. Candidates for
taskrgram: no direct DOM writes outside `requestMutation`; no `localStorage` for anything credential-shaped;
no raw `fetch` outside the API layer; window-scope threading if you adopt one; no allocation inside store
selectors. Plus `stylelint-high-performance-animation` (blocks animating non-compositable properties) and a
whole-pixel rule.

**Why:** Finding — Telegram runs three custom/pinned ESLint plugins (`tt-multitab`, `no-null`,
`react-hooks-static-deps`) and four Stylelint plugins including `high-performance-animation` and
`@mytonwallet/stylelint-whole-pixel`. A rule in a wiki page is gone in two quarters; a rule that fails CI
outlives everyone who agreed to it. This is the cheapest durability mechanism in the audited codebase.

**Effort:** ~1 day per rule.

**If skipped:** your architecture decays to whatever the last three PRs did.

### P1-7 · An explicit lazy boundary per heavy feature, plus a CI size budget

**Do:** one small `Feature.async.tsx`-style shim per heavy feature that takes `isOpen` and loads the chunk
when it first becomes true; a memory cache in front so re-opens are synchronous. Diff bundle size in CI
against a stored baseline, **including worker bundles**, and fail on a threshold.

**Why:** Finding — 151 `*.async.tsx` shims + 6 named barrels in `src/bundles/`, with
`rollup-plugin-bundle-stats` against a baseline and a **custom `createWorkerBundleCollectorPlugin` so worker
bundles cannot hide from the budget**. Measured payoff: on the login screen only **6.5% of chunk files** load,
and `main` (420 KB), `useConnectionStatus` (480 KB), `Modal` (396 KB) and `editor` (295 KB) are all correctly
deferred until after auth. `[Size]` is a first-class commit tag.

**Effort:** ~2 days. Set the budget generously (see §4 — you are on a LAN).

**If skipped:** you will not notice the day someone imports a charting library into the message row.

### P1-8 · A motion/effects setting, defaulting to `prefers-reduced-motion`

**Do:** one "Reduce motion and effects" toggle in settings, defaulting on when the OS asks. Under it, break
out the genuinely expensive things individually: **backdrop blur**, autoplay of animated content, and any
continuously-running animation. Implement as body classes, exactly as they do.

**Why:** Finding — the audited product's **"Animations and Performance"** screen contains a three-stop slider
(`Power Saving | Nice and Fast | Lots of Stuff`) and a section literally headed **"Resource-Intensive
Processes"** with **ten** individually-toggleable interface animations — and, tellingly, **"Context Menu
Blur" and "Message Blur" are separate switches from the animations they decorate**, because
`backdrop-filter` is the expensive one
(`screenshots/26-desktop-settings-animations-and-performance-toggles.png`,
`screenshots/27-desktop-settings-animations-performance-expanded-granular-toggles.png`). We observed
`with-message-blur` on the live `<body>`; the mapping is in `src/util/perfomanceSettings.ts`. This is the
strongest single piece of evidence in the audit about what this team values.

And the gap to improve on: **there is no `prefers-reduced-motion` usage anywhere in their stylesheets.** They
built something richer and skipped the platform signal. Do both — one control, defaulted from the OS.

Why it matters more for you than for them: corporate fleets are old and heterogeneous, and our CPU-throttling
run measured **TBT 72 ms → 753 ms at 4× CPU** on a page with almost nothing on it. Remote workers on
VDI/Citrix/RDP get zero benefit from compositor effects and pay full price.

**Effort:** ~3 days. Two toggles, not ten.

**If skipped:** the app is unusable for a slice of your own colleagues and you will hear about it as "it's
slow", with no diagnosis attached.

---

## 4. P2 — worth doing, but not before the app works

### P2-1 · Service-worker Range streaming for auth-gated attachments

**Do:** *only if* legal or security requires that attachment bytes never be fetchable without a live session.
Intercept `/media/<id>` in a service worker, answer `206 Partial Content` from an authenticated fetch, and let
native `<video>`/`<audio>` do seeking.

**Why:** Finding — Telegram's `/progressive/<id>` route, confirmed live returning **HTTP 206**, with
`DEFAULT_PART_SIZE = 0.5 MB`, *"We only cache the first 2 MB of each file"*, and MSE used **only** as a
Safari fallback (`if (!IS_SAFARI …) return undefined`). It gets native seek, buffering and PiP for free, with
no HLS and no custom player. But: they only did it because MTProto files have **no URL** — per
`core.telegram.org/api/datacenter`, files are addressable only by `(dc_id, file_reference)` and *"encryption
keys are not copied between DCs"*.

**Effort:** ~1–2 weeks. **If you can use short-TTL signed URLs, do that instead** — an order of magnitude
less code. Sharp edge: the app breaks where service workers are unavailable.

### P2-2 · Consider a worker for the transport layer — later, and only if measured

**Do:** structure your code so this is *possible* (transport module → DTO builders → store → UI, with only
plain serializable objects crossing the boundary, and a dev-time assertion that no class instances cross).
Actually move it to a worker only if you measure main-thread time in parsing or crypto.

**Why:** Finding — the boundary discipline is the most valuable structural decision in the repo
(`AGENTS.md`: *"UI code uses plain objects (`Api*` types)"*, 23 `apiBuilders` files, a DEBUG assertion that
responses contain no `VirtualClass` instances). But the *worker* exists because MTProto forces mandatory
client-side AES/RSA/DH on every request. Their worker is **742 KB — the largest asset in the deployment and
45% of decoded JS on the login screen** — and it sits on the critical path to a usable login CTA: the
filmstrip shows the shell painting at ~3.0 s and the CTA appearing at ~9.0 s
(`screenshots/filmstrip-3000ms.png`, `screenshots/filmstrip-9000ms.png`). If you speak JSON over HTTPS, a
worker buys complexity and nothing else.

**Effort:** boundary discipline ~free; worker ~1 week if ever needed.

### P2-3 · Fixed-height rows at absolute offsets for the channel/DM list

**Do:** render list rows at computed `top` offsets inside a constant-height container, so reordering is a
transform rather than a reflow.

**Why:** Finding — `ChatList.tsx`:
`offsetTop = panesHeight + archiveHeight + (viewportOffset + i) * CHAT_HEIGHT_PX`, with
`<InfiniteScroll withAbsolutePositioning maxHeight={totalHeight} />` and `teactOrderKey` telling the
reconciler when a node actually needs to move. It makes an animated, frequently-reordering list cheap.

**Effort:** ~3 days. Only worth it once your sidebar animates.

### P2-4 · Faceted search as a distinct surface

**Do:** if search matters (it does, for an internal team app — search is the killer feature of a work chat),
build it as a faceted surface, not a filtered list.

**Why:** Finding — the live product's global search has tabs `Chats · Channels · Apps · Posts · Media ·
Links · Files · Music · Voice`, sectioned into "Chats and Contacts" / "Global Search" / "Messages"
(`screenshots/18-desktop-global-search-results-chats-channels-messages.png`), and the right column carries a
parallel per-channel facet set. For taskrgram the analogous facets are people, channels, files, links, and
tasks. Note the supporting architecture: a denormalized, incrementally-maintained index
(`src/util/folderManager.ts`, `UPDATE_THROTTLE = 500`) so list selectors never iterate everything.

**Effort:** ~2 weeks for a good version.

### P2-5 · Three-column responsive behaviour, measured from theirs

**Do:** copy the *breakpoint strategy*, which we measured exactly rather than guessed:

| Range | Behaviour |
|---|---|
| ≥ 1276 px | all three columns scale proportionally (left ≈25%, right ≈24%) |
| 926–1275 px | left switches to ~33%, **right column becomes a fixed 408 px** — stop scaling the reading column, start protecting it |
| 601–925 px | left column goes **off-canvas** as a fixed 424 px drawer (`x = -424`); middle takes the full viewport; right overlays at 408 px |
| ≤ 600 px | every column is exactly viewport-width — one screen at a time |

The left column is never edge-to-edge: `x = 16` at every desktop width. That 16 px gutter is the whole
"floating card" aesthetic — left column, chat header and composer are rounded, inset surfaces over the
wallpaper.

**Why:** Finding — measured with `getBoundingClientRect()` at 13 viewport widths; breakpoints located
precisely at 1276/1275, 926/925, 601/600. Compare
`screenshots/responsive-1920px-channel-view-desktop-layout.png`,
`screenshots/responsive-1100px-channel-view-desktop-layout.png`,
`screenshots/responsive-925px-channel-view-desktop-layout.png`,
`screenshots/responsive-600px-channel-view-desktop-layout.png`. The pattern worth stealing is
**"fix the secondary panel, protect the primary reading column"** — most three-column apps scale everything
and end up with an unreadably wide message column at 1920 and an unusably narrow one at 1100.

**Effort:** ~3 days.

### P2-6 · Use the platform where it is better than your abstraction

**Do:** native `<dialog>` for the media/file viewer. Native `Intl` (`PluralRules`, `NumberFormat`,
`ListFormat`, `DisplayNames`) for formatting. CSS cascade layers declared up front to fix async-CSS ordering.

**Why:** Findings — the media viewer is `<dialog open id="MediaViewer" aria-modal="true">`, and Escape closed
it in one press, verified (`screenshots/44-desktop-media-viewer-native-dialog-element-open.png`); top-layer
stacking and backdrop come free. Their i18n layer uses native `Intl` throughout with a documented fallback.
And the only inline `<style>` in the document is the layer declaration:

```css
@layer reset, variables, ui, components;
@layer ui { @layer tablist, spinner, button, input, layout; }
```

so async CSS chunks land in the right layer regardless of load order — specificity decided by layer, not by
network timing. That is genuinely elegant and costs 4 lines.

Note the inconsistency worth *not* copying: everything else in that app goes through a `#portals` div, so
there are two overlay systems. Pick one.

**Effort:** near zero; these are decisions, not projects.

### P2-7 · Generate i18n types from the strings file

**Do:** codegen a `LangKey` union from your source strings; re-run it on save. A missing or renamed key
becomes a **compile error**.

**Why:** Finding — they do exactly this (`npm run lang:ts` → `src/types/language.d.ts`, re-run by a Vite
watch plugin) — and yet we still caught `aria-label="AccDescrPollVoteDown"` leaking a raw key into an
accessible name in the live product. So: do the codegen *and* add a runtime dev-mode assertion that a lookup
never returns its own key. The type check covers the code path; only the assertion covers the data path.

**Effort:** ~2 days.

---

## 5. Steal this

Ideas, not code. Every one of these is portable to any framework and none requires touching GPL'd source.

1. **The two-phase measure → mutate discipline, enforced at runtime in dev.** `requestMeasure` /
   `requestMutation`, flushed once per RAF, with a strict mode that *throws* on a phase violation. Three
   files. Framework-agnostic. The highest-leverage idea in the audited codebase, and the author's own
   writeup says it is complementary to React, not a replacement for it.
2. **Gate expensive work on animation and idle state.** `beginHeavyAnimation(duration)` and
   `onFullyIdle(cb)` — 73 lines — with store flushes, cache writes, observers and index rebuilds all
   consuming them.
3. **Signals for high-frequency values.** Typing, caret, scroll, upload progress, playback time never enter
   the render path.
4. **A bounded, whitelisted, migration-versioned persisted cache.** Whitelist keys, bound every list, strip
   optimistic state, throttle + idle-gate the write, and require a migration entry for every shape change.
5. **Stale-write detection in dev.** The `DEBUG_randomId` stamp that throws on `read → await → write stale`.
   Ten lines, one whole bug class.
6. **Protocol/transport client behind a hard DTO boundary**, with a dev-time assertion that no class
   instances cross it — worker optional, boundary mandatory.
7. **Service-worker Range streaming for auth-gated media**, so native `<video>` gets seeking without MSE,
   HLS or a custom player.
8. **Performance as a user-facing setting**, with the genuinely expensive effects (backdrop blur, autoplay)
   broken out individually — plus the `prefers-reduced-motion` default they forgot.
9. **Architecture encoded as lint rules.** Custom ESLint plugins for your non-negotiable invariants;
   Stylelint blocking non-compositable animations.
10. **An explicit, reviewable lazy boundary per heavy feature**, plus a CI size budget that includes worker
    bundles.
11. **The token/theming structure — minus the runtime injection.** Global semantic tokens on `:root`,
    component-scoped locals with a `--_` private-token prefix convention, and cascade layers declared up
    front in the HTML.
12. **A bounded sliding window instead of virtualization** for anything with variable-height rows.
13. **Fixed-height rows at absolute offsets** for lists that reorder and animate.
14. **Optimistic UI as a default, with placeholders that are cheap.** Vector-path thumbnails decoded to
    inline SVG and worker-blurred mini-thumbnails, rather than a spinner.
15. **A denormalized, incrementally-maintained index** for anything a list selector would otherwise iterate
    (their `folderManager.ts`).
16. **Install-script gating.** Their `package.json` carries an `allowScripts` allowlist (`core-js: false`,
    esbuild/fsevents `true`). Cheap, real supply-chain hardening.
17. **A compatibility gate before the app boots.** Their `compatTest.js` feature-checks 14 APIs and, on
    failure, replaces the body with a static "unsupported browser" page. For a corporate fleet with a long
    tail of old browsers, this converts a white screen into an explanation.

---

## 6. Do NOT copy this

Each of these is a real pattern in the audited product with a specific reason not to follow it.

1. **Writing your own framework.** Teact exists because of a 2019 contest rule banning third-party UI
   frameworks, not because React was benchmarked and found wanting — the README says so, and the
   Teact-vs-React benchmark postdates the framework by two years and publishes no numbers. The costs are
   concrete: **no error boundaries** (a throwing component silently reuses its last render), no Suspense, no
   concurrent rendering, no devtools, hooks in four parallel cursor arrays with a bespoke ESLint plugin
   required to keep dep arrays safe, double-buffered `useState` where same-tick reads see the old value,
   `memo` as a *positional* compare over `Object.values(props)`, and manual GC on unmount because the
   virtual tree holds live DOM references. Note that the "no dependencies" doctrine is **no longer enforced
   by its own authors** — 33 runtime + 68 dev dependencies now, including 17 `@tiptap/*` packages; and Web K,
   the other contest winner, now ships SolidJS.
2. **Plaintext credentials in `localStorage`.** Measured on a live session: `dc1_auth_key`, `dc2_auth_key`,
   `dc4_auth_key` (514 chars each), `user_auth`, `account1`, and an **empty `document.cookie`**. Forced on
   them by MTProto. Not forced on you. Use an `HttpOnly` cookie.
3. **A single-SHA-256 KDF with a hardcoded salt.** `src/util/passcode.ts` derives the passcode encryption
   key with **one SHA-256 pass** and the literal salt string `'harder better faster stronger'`. If you ever
   derive a key from a user secret, use Argon2id or PBKDF2 with a high iteration count and a per-user random
   salt. This one is simply a defect, not a trade-off.
4. **Publishing source maps in production.** 453 of 461 chunks serve resolving `.map` files, exposing 2,056
   original source paths and internal codenames. Coherent for a GPL project whose source is already public;
   an own-goal for a proprietary internal app. Use `sourcemap: 'hidden'` + upload to your error tracker.
5. **`max-age=3600` on content-hashed assets.** Measured on every asset including
   `index-IZ97MA_m.js`. Hashed filenames exist precisely so you can cache them forever. Use
   `max-age=31536000, immutable`.
6. **Committing build output to git.** `dist/` is **230 MB** in the repo and the release script is
   `build && git add -A && git commit -m '[Build]' --no-verify && git push`. Every clone pays for it, history
   is build noise, and `--no-verify` skips whatever the hooks were protecting. Artifacts belong in an
   artifact store.
7. **373 lazy `highlight.js` grammar chunks.** 81% of all JS chunks, 928,928 B raw across 373 files, one per
   language — on a `max-age=3600` cache, so an exotic language costs an extra round trip. Ship a bundle with
   the ~15 languages your team actually pastes and lazy-load the rest as **one** chunk.
8. **The contest-era bundle-size obsession, applied to an internal app on a LAN.** Their warm reload
   transfers **2,975 bytes — 0.4% of the cold load** — with `loadEventEnd` cut in half. For an internal app
   with a returning, cached, LAN-connected user base, cold-load bytes are close to irrelevant. Keep a budget
   so you notice a 2 MB regression; do not architect around 50 KB.
9. **Zero telemetry.** See P0-5. Their blindness is a brand promise; yours would be negligence.
10. **`user-scalable=no` and sub-3:1 focus indicators.** Measured. Both are accessibility failures, both are
    one line to avoid.
11. **Keeping inactive screens mounted without hiding them from assistive tech.** The `Transition` container
    keeps previous and next screens in the DOM simultaneously — great for cheap, cancellable transitions,
    but every control matched 3+ times in our audit and only one was visible. If you use this trick,
    `aria-hidden` + `inert` the inactive ones.
12. **Two overlay systems.** A native `<dialog>` for the media viewer and a `#portals` div for everything
    else. Pick one.
13. **Hardcoded infrastructure endpoints with a `TODO`.** `src/lib/gramjs/Utils.ts:171` hardcodes the DC list
    with `// TODO Move to external config`. Put your endpoints in runtime config from day one.
14. **Git-pinned dependencies.** Five of them, including three ESLint plugins and `opus-recorder`
    (`Ajaxy/opus-recorder#116830a`). Reproducibility and supply-chain risk. Publish to your internal
    registry instead.
15. **The feature surface.** This is a full consumer client: payments, star gifts and gift *auctions*
    (14 sub-folders for gifts alone), stories, mini-apps, premium tiers, group video calls, and a
    32,000-character document editor inside the composer
    (`screenshots/17-desktop-composer-attachment-menu-open.png` shows "Checklist", "Date" and "Article" as
    *attachment types*). Roughly 10–15% of these 184,000 lines of component code maps to anything an internal
    team chat needs. Scope discipline is the cheapest performance optimisation available to you.

---

## 7. A suggested stack for taskrgram

Opinionated, with the trade-offs stated and the contrast against Telegram's choices made explicit.

| Layer | Recommendation | Why, and how it differs from Telegram Web A |
|---|---|---|
| **UI framework** | **React 19** (Preact if bundle size ever actually bites; **Solid** if the team already knows it) | Telegram wrote Teact because a contest rule banned frameworks. You want error boundaries, Suspense, devtools, a hiring pool, and testing libraries. Add the frame scheduler (P0-1) yourself. **Trade-off:** React's scheduler is less controllable than Teact's — mitigated by keeping expensive work out of render via signals (P1-2) and gating on animation state (P1-1). |
| **Language / build** | **TypeScript strict**, **Vite** | Telegram runs Vite 8 on **Rolldown** with LightningCSS in production — a 716-byte bundler runtime and native ESM chunk loading, versus 8–15 KB for a webpack runtime at this scale. Their choice is validated at real scale; take stock Vite and adopt Rolldown when it is boring. |
| **Styling** | **CSS Modules + SCSS**, semantic tokens on `:root`, cascade layers declared in `index.html`, **palette in CSS** | Copy their three-tier token structure (566 custom properties; `--_` prefix for private component tokens) and their layer declaration. **Reject the runtime injection**: their 78 `[light, dark]` tuples live in a JS chunk written into a `<style>` element, which is exactly why their CSP needs `style-src 'unsafe-inline'`. If you need a few dynamic tokens (per-team accent), use `element.style.setProperty()` — not subject to `style-src`. |
| **State** | **Zustand** (or Redux Toolkit if the team prefers ceremony) with **selector-level subscriptions**, plus **signals** for high-frequency values | Their store is ~400 lines: a module-level global, a flat registry of containers, shallow-equal mapped props, an action **trampoline** rather than recursion, and `execAfterActions`. Adopt: normalized-by-id slices, pure allocation-free selectors, no loops in selectors, and the stale-write guard (P1-5). Adopt their **two-dimension split** (per-window vs per-user) from day one (P0-4). |
| **Persistence** | **IndexedDB** via `idb-keyval`, whitelisted + bounded + migrated (P1-4) | Same as theirs, minus storing credentials in `localStorage`. |
| **Transport** | **Boring.** HTTPS + JSON (or protobuf if you have a reason) for RPC; **one WebSocket** for realtime, with a documented reconnect/backoff policy and a resume cursor | Telegram needs obfuscated MTProto over WSS with AES-CTR framing, a forbidden-prefix table, per-DC auth keys, `auth.exportAuthorization` across DCs, an HTTP long-poll fallback, and a **client-side implementation of the full update-gap algorithm** (`pts`/`seq` continuity, `SortedQueue` per box, difference recovery). **You need none of it.** What you *should* copy is the *shape* of the update pipeline: a monotonic sequence number per stream, gap detection, and a "fetch difference since cursor" endpoint. Resyncing after a laptop sleeps is the single most common realtime bug. |
| **Auth** | **HttpOnly + Secure + SameSite session cookie**, SSO/OIDC against your IdP, CSRF token on mutations | The inverse of theirs (P0-2). You get session revocation, device listing, and MDM integration free from the IdP — and their "Active Sessions" screen (`screenshots/29-desktop-settings-active-sessions-device-list.png`) is worth copying as a *UI*, backed by your IdP rather than by per-DC auth keys. |
| **Media / files** | **Short-TTL signed URLs** from object storage, behind your auth | Their SW Range pipeline exists because MTProto files have no URL. Only adopt P2-1 if security requires bytes be unfetchable without a live session. **Trade-off:** signed URLs can leak via history/logs; SW streaming cannot, but breaks where service workers are unavailable. |
| **Rich text** | **Tiptap/ProseMirror** — but scoped to *chat*, not documents | Telegram converged on this too, and published an unusually candid reason for rewriting their composer: *"All content is treated as HTML string, so every edit triggers re-parsing"*, *"We can't directly render components"*, *"A RegExp-based Markdown parser has too many problems"*, *"Composer component is bloated"* (`t.me/webachannel`, 2025-02-04). Learn from that: **do not build a Markdown-regex composer**. But also do not build their editor — the live product ships tables, headings, pullquotes, details blocks and LaTeX in a chat box. You want bold/italic/code/link/list/mention/code-block and nothing else. |
| **Search** | **Server-side**, Postgres FTS or OpenSearch | They do client-side search over a bounded local cache because the server is Telegram's. Yours is yours. Server-side search is simpler, complete, and permission-aware. |
| **i18n** | Generated key types + native `Intl`, even if you ship English only | Cheap now, structural later (P2-7). |
| **Desktop shell** | **Tauri** | Telegram uses it (5 `@tauri-apps/*` packages compiled into the web bundle, one codebase for web and desktop). Smaller than Electron, and the one-codebase pattern works. **This is also the moment the GPL question becomes real** — see §1. |
| **Observability** | **Self-hosted GlitchTip/Sentry** + a first-party metrics endpoint; no third-party hosts | The deliberate inversion of their zero-telemetry stance (P0-5). |
| **Testing** | Vitest + Testing Library + Playwright | They run exactly this (`vitest`, `@playwright/test`, `jsdom`, `fake-indexeddb`) — and a real framework means Testing Library actually works, which it only partially does on Teact. |

**Where an internal team app should deliberately diverge from Telegram, summarised:**

- **a real framework** instead of a bespoke one — you are optimising for the fifth engineer, not the first;
- **real accessibility** as a CI gate rather than a four-year-old open issue;
- **HttpOnly sessions** instead of JS-readable key material;
- **boring transport** instead of a protocol implementation;
- **self-hosted telemetry** instead of blindness;
- **far fewer features** — 10–15% of their component surface;
- **warm-load and input latency** as the headline metrics instead of cold-load bytes.

---

## 8. Phased build plan

The principle: **a thin vertical slice through every layer before any layer is finished.** Every architectural
risk in this audit — the frame model, the persisted cache, multi-window, realtime resync — is cheap to
validate in week 3 and brutal to discover in month 6.

### Phase 0 — Decisions and skeleton (1 week)

**Build:** repo, CI, TypeScript strict, lint baseline including two custom rules, the frame scheduler +
strict mode (P0-1), error tracking (P0-5), the state-shape split (P0-4), and the delivery-layer config
(P1-3).

**Prove:** that `stricterdom`-equivalent actually throws on a deliberate violation in dev and is a no-op in
prod; that a thrown error reaches your tracker with a readable stack from hidden source maps; that legal has
answered the §1 question.

**Exit criteria:** a blank app that already has the invariants. Nothing here is retrofittable cheaply.

### Phase 1 — Thin vertical slice (3 weeks)

**Build:** login via SSO with an HttpOnly cookie → a channel list → one channel → a bounded message list →
send a plaintext message → see it arrive in a second browser via WebSocket. Persisted cache with **one**
migration already written and exercised. Error boundaries around the four surfaces.

**Prove:**

- **Warm start.** Second load restores the channel list from IndexedDB before the network answers. Measure
  it. (Their benchmark: warm reload transfers 0.4% of cold bytes and halves `loadEventEnd` —
  `screenshots/39-desktop-warm-reload-restored-chat-list-from-cache.png`.)
- **Resync.** Close the laptop lid for five minutes; on wake, the client detects the gap, fetches the
  difference since its cursor, and shows no duplicates and no holes. This is the highest-risk correctness
  behaviour in any chat app and the reason Telegram implements a full `pts`/`seq` gap algorithm client-side.
- **Bounded DOM.** Scroll a channel with 10,000 messages; node count plateaus and heap does not grow
  monotonically. (Their numbers to beat: 29 → 89 nodes, heap 54.6 → 52.3 MB.)
- **The migration path.** Deploy a state-shape change to a client with an old cache. Nobody white-screens.

**Exit criteria:** two people can talk to each other reliably across a sleep/wake cycle. If any of the four
proofs fails, fix it *now*.

### Phase 2 — The real composer and real media (3 weeks)

**Build:** Tiptap composer scoped to chat formatting, mentions, drafts persisted per channel, file upload
with progress, image/file rendering with cheap placeholders, optimistic send with a failure state.

**Prove:**

- typing in the composer does **not** re-render the message list (signals, P1-2 — verify with a render
  counter, the way their DEBUG build dumps per-component render counts on double-click);
- optimistic send: message appears instantly, reconciles with the server id, and shows an honest failed
  state when the socket is down;
- upload progress does not enter the render path;
- a message that fails to render takes down one row, not the app.

### Phase 3 — Multi-window, notifications, presence (2 weeks)

**Build:** per-window vs per-user state actually exercised across three windows; web notifications with
suppression when a window is focused on that channel; read cursors; presence.

**Prove:** three windows open, two on the same channel — exactly one notification, unread counts agree
everywhere, drafts do not overwrite each other, and closing the "primary" window does not break realtime.
This is the phase that retroactively justifies P0-4.

### Phase 4 — Search, files, admin (3 weeks)

**Build:** server-side faceted search (people / channels / files / links / messages), a per-channel files
panel, channel management, member management.

**Prove:** search returns permission-filtered results in <300 ms p95 on your real corpus; the files panel
does not load the world.

### Phase 5 — Polish, performance settings, a11y pass, desktop shell (3 weeks)

**Build:** motion/effects setting (P1-8), three-column responsive behaviour (P2-5), keyboard shortcuts, focus
management, the Tauri build.

**Prove:**

- axe-core clean on every key screen; full keyboard traversal with a visible ≥3:1 focus ring; screen-reader
  walkthrough of send/receive/search by an actual user of one;
- the app is usable at 4× CPU throttle (their measured degradation: TBT 72 ms → 753 ms — know your own
  number);
- performance budget green in CI, worker bundles included;
- and **before the desktop binary ships, the §1 licensing answer is in writing.**

---

## 9. Open questions — need source access or a decision from the team

*This section doubles as the audit's "questions requiring source-code access" deliverable. Items marked
**[Telegram]** could only be answered with access we do not have; items marked **[taskrgram]** are decisions
your team must make and no amount of auditing will make for you.*

### Requires access we do not have

1. **[Telegram] Were the `dist/` build artifacts ever the *only* record of a deploy?** The public repo is the
   artifact store and every release body is `[Build]`. Whether the served bundle is byte-reproducible from
   the committed sources is unverifiable from outside — relevant if you ever want to argue that the public
   source corresponds to the public binary.
2. **[Telegram] Is `sourcesContent` populated in the 453 published maps?** We confirmed the `sources` arrays
   (2,056 paths) but did not check whether full original source text is embedded. If it is, the maps are a
   complete second copy of the source, not just a path listing.
3. **[Telegram] What is the actual measured Teact-vs-React delta?** The benchmark at
   `ajaxy.github.io/teact/benchmark/` is **runnable** — it loads React 18 and Preact 10 from unpkg alongside
   Teact and computes results client-side. Nobody has published numbers. **Actionable:** if the framework
   question is ever seriously reopened on your team, run it yourselves rather than citing either side's
   claims. Note it postdates the framework by two years, so it is validation, not justification.
4. **[Telegram] Does `beginHeavyAnimation` deferral ever starve a critical update?** The gate defers
   component updates, container mapping, cache writes and observers while animating, with
   `AUTO_END_TIMEOUT = 1000`. What happens to an incoming message during a 900 ms transition is a design
   question we could only answer by instrumenting the source.
5. **[Telegram] What actually drives the `stricterdom` violation rate in their CI?** Whether the strict mode
   is genuinely clean or routinely bypassed determines how much the pattern is worth. Not observable from
   the outside.
6. **[Telegram] Web A's current official feature-gap list.** The only first-party gap list is Web K's, dated
   **2021-04-13**, and much of it is now stale. Our `Business`/`Affiliate` findings come from grepping a
   3,140-line fallback strings file and are weak evidence.
7. **[Telegram] The provenance of the vendored `tlottie.wasm`.** Documented as built from
   `dkaraush/tlottie` commit `c461cd5` with a size-optimized Cargo profile, but not reproducibly verified in
   CI. If you ever vendor a prebuilt WASM binary, this is the standard you should exceed.
8. **[Telegram] What is `src/components/gili`?** 21 files, aliased as `@gili/*`, a self-contained newer
   primitive set (`Island`, `Surface`, `Control`, `Interactive`, `SwitchField`) sitting alongside the legacy
   `src/components/ui/` (114 files). It looks like an in-progress design-system migration. Whether it is the
   future or an abandoned branch would tell you a lot about how they handle design-system replacement — a
   problem you will have too.

### Decisions your team must make

9. **[taskrgram] Does a desktop binary ship, and to whom?** This is the licensing hinge (§1). If the answer
   is "yes, and to contractors", the contamination boundary must be airtight and legal must have said so in
   writing before Phase 0 ends.
10. **[taskrgram] One WebSocket per tab, or an elected master?** Decide in Phase 0. If your server can carry
    N sockets per user (it almost certainly can), take the simple path and keep only the *state-shape*
    discipline. If not, you are building master election, and you should know that on day one.
11. **[taskrgram] Must attachment bytes be unfetchable without a live session?** Signed URLs vs the
    service-worker Range pipeline is a 1-day vs 2-week decision, and it is a security/legal question, not an
    engineering one.
12. **[taskrgram] What is your accessibility obligation, precisely?** WCAG 2.2 AA? A public-sector
    procurement requirement? An internal policy? The answer determines whether P0-7 is a CI gate or a
    guideline, and retrofitting is the expensive path.
13. **[taskrgram] How much history must be searchable and from where?** Server-side search assumes the
    server has everything. If there is a retention policy, an e-discovery requirement, or a
    client-side-encryption ambition, that changes the entire persistence design — and note that
    **end-to-end encryption is precisely the feature both Telegram web clients still do not have**, because
    durable per-device key storage is not something a browser provides safely.
14. **[taskrgram] What is your real browser support floor?** Telegram's `compatTest.js` gates on 14 APIs
    including `navigator.locks`, `BroadcastChannel` and `DecompressionStream`, and their browserslist target
    is `"baseline widely available"`. Corporate fleets have long tails. Decide the floor before you pick
    APIs, and ship the compat gate (§5.17) so an unsupported browser gets an explanation rather than a white
    screen.
15. **[taskrgram] Who owns the frame-model discipline?** Every mechanism in this audit that survived — the
    phase rules, the `tabId` threading, the selector purity rules — survived because it was encoded in lint
    and in an agent-readable instruction file (`AGENTS.md`, symlinked as `CLAUDE.md`) rather than in a wiki.
    Decide now who writes and maintains yours. That file is the single most valuable artifact in the entire
    audited repository, and it is the cheapest thing on this list to copy.
