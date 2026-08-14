# 06 — Design Decisions and the Reasoning Behind Them

**Subject:** Telegram Web A — `https://web.telegram.org/a/`, v12.0.38, deployment built 2026-08-11 15:24:14 UTC
**Repo:** `Ajaxy/telegram-tt`, HEAD `d915b1b9`, author Alexander Zinchuk, GPL-3.0-or-later
**Audit date:** 2026-08-14
**Reader:** the taskrgram team — desktop-first internal team chat, built fresh, not a clone

This document answers "why is it like this", not "what is it". Each decision is presented in five
beats: **what they did → the evidence → the why (sourced where a source exists) → what it cost them →
whether the reasoning transfers to you.**

## How to read the confidence tags

- **Confirmed** — directly observed in the shipped bundle, the live session, the response headers, or
  stated verbatim in a first-party document (Telegram channel post, repo file).
- **Strong inference** — not stated anywhere, but only one reading fits multiple independent pieces of
  evidence.
- **Possible** — a plausible reading consistent with the evidence, but other readings also fit.
- **Unknown** — we looked and found nothing. Said plainly rather than papered over.

The single most important thing to know before reading: **there is no engineering blog, no design-principles
document, and no author interview for this project.** GitHub release bodies all read `[Build]`.
`CHANGELOG.md` stops at 10.0.0 (2023-08-14) while the shipped version is 12.0.38. The announcement channel
`t.me/webachannel` has not posted since 2025-02-04. The *only* substantial first-party rationale document is
`AGENTS.md` in the repo — an instruction file written for coding agents, symlinked as `CLAUDE.md`. Everything
else has to be read out of the contest briefs, the code, and the running product. That constraint shapes this
whole document and is why the confidence tags matter.

---

## 0. Three myths to kill before we start

An audit that repeats a good story it cannot source is worse than no audit. Three popular narratives about
this codebase are **not supported by the evidence**, and one of them is load-bearing for a lot of blog posts.

### Myth 1 — "They benchmarked React, found it too slow, and wrote Teact"

**Status: unsupported. The sequence is inverted.** — *Confirmed as unsupported*

The project's own README states the origin as a rule, not a measurement:

> "According to the original contest rules, it has nearly zero dependencies and is fully based on its own
> Teact framework (which re-implements React paradigm)."
> — `README.md`, `Ajaxy/telegram-tt`

The rule it refers to is the contest task: *"create a simplified web version of Telegram **without using
third-party UI frameworks**"* (`https://t.me/contest/146`, 2020-01-31). Telegram later clarified in
`t.me/webachannel` (2025-02-04) that the rule means *"you can use any existing dependencies. You just can't
add new ones."*

There **is** a Teact-vs-React-18-vs-Preact-10 benchmark — `https://ajaxy.github.io/teact/benchmark/` — but it
computes results client-side at page load, publishes no numbers anywhere, and the Teact repo's LICENSE is
dated 2022, i.e. **two years after the framework was written**. It is validation after the fact, not the
justification. We searched Hacker News (Algolia API) for `Teact`, `telegram-tt`, `Telegram Web Z/A` and found
only the 2021 launch thread with no author participation. The author's two recorded conference talks (Helsinki
and Zurich) both predate the contest.

The honest formulation: **Teact exists because of a contest rule; it survives because of frame-phase
scheduling.** Those are two different arguments and only the second one is transferable.

### Myth 2 — "Telegram has a published 60fps philosophy for its web clients"

**Status: no such statement exists.** — *Confirmed as unsupported*

We checked `telegram.org/faq`, `telegram.org/apps`, both client announcement channels, the contest pages, and
the 2025 design contest results. There is no Telegram design-principles document for first-party clients and
no use of "60fps" as a stated web target. The only first-party frame-rate number anywhere is from a **Web K**
contest brief:

> "This issue causes noticeable lagging on devices with **120 FPS displays**…"
> — `https://t.me/WebK_en/17`, 2024-07-07

That is Web K, not Web A, and it is a defect report rather than a philosophy. What *can* be sourced as the
performance doctrine is the judging rubric (Decision 2) and `AGENTS.md`. Cite those; do not cite "60fps".

### Myth 3 — "Jolly Cobra and Ace Monkey map to specific named authors"

**Status: inference, never published.** — *Strong inference, explicitly not fact*

Telegram awarded the contest to codenames and never linked them to identities. The Jolly Cobra → `Ajaxy`
mapping rests on the telegram-tt README claiming first prize at `contest.com/javascript-web-3`, which the
contest channel awarded to Jolly Cobra. Ace Monkey → `morethanwords` is weaker still. Both are high-confidence
inference. If you repeat them in an internal doc, label them.

Two further items worth flagging as **Unknown / unresolved** rather than asserted:

- **"Telegram Air"** — the repo publishes releases tagged `air_v2.10.3`, the Teact README says Teact "powers
  the official Telegram Air (Web A) client", and there are five `@tauri-apps/*` dependencies. But
  `telegram.org/apps` lists no product called "Telegram Air", and `web.telegram.org/a/get` is titled
  "Telegram A Desktop". Best supported statement: *"Telegram Air" is the internal release name for packaged
  Tauri desktop builds of the Web A codebase.* — *Possible*
- **The 2021 "only a 400 KB download" launch claim** cannot be reconciled with our measurement (~305 KiB gzip
  eager critical path in 2026) because the blog post never says what it measured. Do not present them as a
  before/after. — *Unknown*

---

## Decision 1 — Ship two independent web clients (`/k/` and `/a/`)

### What they did

Telegram runs two full, separately-authored, feature-divergent web clients on the same domain: `/k/`
(`morethanwords/tweb`, a TypeScript rewrite descended from the AngularJS Webogram codebase) and `/a/`
(`Ajaxy/telegram-tt`, a clean-sheet rewrite on its own framework). A third generation, the original AngularJS
Webogram, is *still being served* at `/`.

### The evidence

- **Confirmed.** `GET https://web.telegram.org/` returns the legacy Webogram app (`ng-app`,
  `manifest=webogram.appcache`), `last-modified: Wed, 25 Oct 2023`. `/k/` returns version `2.2 (669)`,
  `last-modified: 2026-08-14`. `/a/` returns `12.0.38`. Three generations live simultaneously.
- **Confirmed.** Webogram's own README carries the lineage: *"superseeded by 2 new official Telegram Web
  Apps… Telegram Web K, **based on source code of Webogram, rewritten in TypeScript**… Telegram Web Z, **based
  on its own Teact framework**."*
- **Confirmed.** Bonus-round results, `https://t.me/contest/196` (2020-10-06): *"1st PLACE – $5,000 🥇 Jolly
  Cobra 🥇 Ace Monkey"*. Two co-winners of the final round.

### The why

There is no engineering rationale, because there wasn't one — **it is the mechanical consequence of a tie in
a cash contest.** The only policy statement Telegram has ever made is operational, not architectural:

> "those are two separate independent apps, so the supported features and languages may vary"
> — Admin Dog, `https://bugs.telegram.org/c/4003/11`

The 2021 launch blog frames it jokily ("two new, fully-featured Telegram web apps") rather than technically.
— *Confirmed that no technical rationale was published; **Strong inference** that the tie is the cause.*

### What it cost them

Permanent duplicated effort on two full MTProto clients; divergent feature sets by explicit policy; users
redirected to one of the two without a choice; two announcement channels that have both gone silent (Web K
2024-07-07, Web A 2025-02-04); and a bus factor of roughly one named individual per client. Feature parity
work has to be done twice, and community discussion reflects the confusion.

### Does it transfer

**No, and it is worth naming why.** This is the clearest case in the audit of a structure that exists for
non-engineering reasons. Telegram could afford two clients because the contest paid for them and because
duplication is a hedge for a consumer product with hundreds of millions of users. An internal team app has
neither the budget nor the reason. The *transferable* observation is the negative one: when you find a
strange structure in someone else's system, check whether the cause is technical before you copy it.

---

## Decision 2 — Treat the judging rubric as the design brief

### What they did

Web A was not designed against a product spec. It was designed against a scoring rubric, by an individual
competing for money, under adversarial public review. That rubric is still visible in the shipped product
five years later.

### The evidence — the rubric verbatim

**Confirmed**, all first-party, from Telegram's Contests channel:

> "The design implementation should be identical to the updated mockups attached below. **As usually, our
> main criteria for evaluation will be speed, size of the apps and attention to detail.**"
> — `https://t.me/contest/146`, Stage 2, 2020-01-31

> "2. **Optimize app speed, both actual and perceived** (increase page loading speed, as well as …).
> 3. **Ensuring that smooth animations exist wherever applicable, increasing the smoothness of existing
> animations.**"
> — bonus round task list, `https://t.me/contest/177`, 2020-08-18

> "2nd PLACE – $3,500 🥈 Posh Ram 🥈 Merry Ant, **bundle size progress: + $500**"
> — `https://t.me/contest/196`, 2020-10-06

And the penalties, which price defects in cash:

> "2nd PLACE – $7,000 🥈 Posh Ram 🥈 Ace Monkey > **penalty: -$1,000 (broken 2FA)** … 🥉 Giant Parrot >
> **penalty: -$500 (broken in Chrome on iOS)** … 🏅 Tall Tiger > penalty: -$500 (half-broken 2FA) 🏅 Sacred
> Parrot > penalty: -$500 (certain chats not loaded at all) 🏅 Hip Hyena > penalty: -$1,000 (broken in Chrome
> on Android)."
> — `https://t.me/contest/177`

### The why, and how visibly it is encoded in the product

Four scored axes — **speed (actual), speed (perceived), size, defect-freedom across browsers** — map almost
one-to-one onto architectural decisions that are still in the codebase:

| Rubric line | What it produced, still visible in v12.0.38 |
|---|---|
| "size of the apps" + "$500 bundle size progress" | `[Size]` is a **first-class commit tag** in the project's commit convention alongside `[Perf]`, `[Security]`. `rollup-plugin-bundle-stats` runs in CI against a stored baseline, with a custom plugin that merges *worker* bundles into the report so worker weight is not invisible. 6 hand-maintained named bundles, 151 `*.async.tsx` lazy shims. — *Confirmed* |
| "speed, actual" | Protocol client in a worker; `fasterdom` measure/mutate batching; `stricterdom` runtime enforcement; 16 parallel download workers; folder index maintained incrementally so chat-list selectors never iterate all chats. — *Confirmed* |
| "speed, **perceived**" | Optimistic message sending; vector-path and blurred mini-thumbnails as placeholders before media arrives; skeletons; screens kept mounted inside `Transition` so navigation is instant rather than correct. — *Confirmed* |
| "smooth animations wherever applicable" | 16 named transition animations in `Transition.tsx`; `beginHeavyAnimation()` pausing state updates, container mapping, IDB writes and IntersectionObservers during animation; `stylelint-high-performance-animation` blocking non-compositable animated properties at lint time. — *Confirmed* |
| "attention to detail", enforced by cash penalties for cross-browser breakage | A long tail of named per-browser workarounds shipped in production code: Safari `IntersectionObserver` staleness, Firefox scroll bug 1753188, "An attempt to fix freezing UI on iOS" in the service worker, "Workaround for iOS sometimes stops interacting with worker" in the worker connector, `patchSafariProgressiveAudio.ts`, MSE fallback used **only** on Safari. — *Confirmed* |

The strongest single statement of the retained doctrine is `AGENTS.md`'s performance checklist, whose rule #1
is not "correctness" and not "simplicity":

> "1. **Animations first** – Evaluate if code negatively impacts animations."

— *Confirmed. The causal link from rubric to architecture is **Strong inference**, but every link in the chain
is independently evidenced.*

### What it cost them

The rubric contains no line for **accessibility**, none for **maintainability**, none for **onboarding cost**,
and none for **observability**. All four are correspondingly weak in the shipped product. GitHub issue #128,
"Improve screen reader accessibility", opened 2022-04-05 by an actual screen-reader user, is still open with
no assignee, no label and no milestone. Our own axe-core run on the login screen found both primary CTAs
failing WCAG AA contrast (`#3390ec` on `#ffffff` = 3.31:1), zero landmarks, pinch-zoom disabled, and a focus
indicator at ~1.08:1 contrast against white — see `screenshots/a11y-button-focused.png` and
`screenshots/a11y-button-unfocused.png`. A product scores what it is measured on.

### Does it transfer

**The method transfers; this rubric does not.** Write down the three or four axes you will actually score
your product on, and then let them drive architecture the way this rubric did. But your axes are different:
for an internal desktop team-chat app they are probably *time-to-first-useful-screen for an authenticated
returning user*, *input latency in the composer and message list*, *correctness of unread/notification state*,
and *accessibility*, with bundle size well down the list because your users are on a corporate LAN. Copying
Telegram's *axes* rather than their *method* is the single most likely way to waste six months.

---

## Decision 3 — Write and keep a custom framework (Teact)

### What they did

The entire UI runs on Teact — ~2,800 LOC across 7 files (`teact.ts` 1109, `teact-dom.ts` 1009, `teactn.tsx`
397, `dom-events.ts` 188, `heavyAnimation.ts` 73, `jsx-runtime.ts` 18). React is a **devDependency only**;
`@types/react` supplies ambient JSX and event types. `tsconfig.base.json` sets `"jsxImportSource": "@teact"`
— the single most consequential line in the repo.

### The evidence

- **Confirmed.** No `react`, `react-dom`, `preact`, or `__REACT_DEVTOOLS_GLOBAL_HOOK__` string exists
  anywhere in the 461-chunk production graph. Teact-specific identifiers are all over it: `teactFastList`,
  `teactOrderKey`, `TeactNContainer`, `TeactMemoWrapper`, `DEBUG_randomId`.
- **Confirmed.** `AGENTS.md`: *"Do not import 'react'. React types are available globally in React
  namespace."*
- **Confirmed.** The origin sentence in `README.md` (quoted under Myth 1).

### The why — honest version

**Origin: a contest rule.** *"without using third-party UI frameworks"* (`t.me/contest/146`).
**Retention: four capabilities that are genuinely hard to bolt onto React**, all visible in the source:

1. **An RAF-driven, explicitly ordered global update pass.** The authors documented the order in a comment
   block in `teact.ts:381-395`: component effect cleanups → effects → measure tasks → mutation tasks →
   component updates → layout-effect cleanups → layout effects → forced-reflow measures → forced-reflow
   mutations. The render pass is *fused with the DOM read/write scheduler*, not adjacent to it. — *Confirmed*
2. **The update pass defers itself while a heavy animation runs.**
   ```ts
   const runUpdatePassOnRaf = throttleWith(requestMeasure, () => {
     if (getIsBlockingAnimating()) { getIsBlockingAnimating.once(runUpdatePassOnRaf); return; }
   ```
   The same gate appears in the store (`teactn.tsx` `runCallbacks`), in the IndexedDB cache writer, and in
   the IntersectionObserver controller. — *Confirmed*
3. **Signals are valid `useEffect` dependencies.** A signal in a dep array converts that effect into a
   push-based subscriber that fires *without rendering the component*. `AGENTS.md`: *"Signals deliver updates
   **without causing component renders**. Use for frequently-updated values"* — typing text, caret position,
   animation state, playback time. This is a Solid idea grafted into a React-shaped hook API. — *Confirmed*
4. **Opt-in keyed reconciliation for exactly the two hot lists.** By default children are matched **by
   index** — there is no keyed diff. `teactFastList` turns on a keyed reconciler for a subtree, and
   `teactOrderKey` tells it when a preserved node actually needs to move:
   ```ts
   const shouldMoveNode = (currentChildInfo.index !== currentPreservedIndex
     && (!newOrderKey || currentChildInfo.orderKey !== newOrderKey));
   ```
   It is applied to the message container and the chat list, and nowhere else. Contrast React, which pays
   keyed-diff cost everywhere by default. — *Confirmed*

Also worth noting: **Context is implemented as signals**, so a context change does not cascade re-renders
through the subtree; consumers opt in with `useDerivedState`. And `memo` is `useMemo` over
`Object.values(props)` — a *positional* shallow compare, which is why `AGENTS.md` bans passing unmemoized
objects into `memo()` components and bans allocating new arrays/objects in selectors.

### What it cost them

- **No error boundaries.** `renderComponent` wraps the call in `safeExec` whose `rescue` reuses the previous
  rendered value and logs to console. A throwing component silently re-renders its last good output. There
  is no recovery UI, no error reporting, and — combined with Decision 10 — nowhere for the error to go.
  — *Confirmed*
- **No concurrent rendering, no Suspense, no transitions API.** — *Confirmed*
- **No ecosystem.** No devtools, no React Testing Library integration beyond partial, no third-party
  component libraries, no Stack Overflow. Every UI primitive is hand-built (`src/components/ui/`, 114 files),
  and a second design-system layer (`src/components/gili/`, 21 files) is mid-migration alongside it.
  — *Confirmed*
- **Subtle-not-obvious divergences that hurt onboarding.** Hooks live in **four parallel cursor arrays**
  (state/effects/memos/refs) rather than React's single linked list, so conditionally calling a `useMemo`
  shifts only the memo cursor. `useState` is double-buffered: the setter writes `nextValue`, and
  `prepareComponentForFrame()` copies `nextValue → value` at the start of the update pass, so same-tick reads
  see the *old* value while functional updaters see the new one. Effects run child-to-parent, and "sibling
  behavior is not defined". `useSyncEffect` runs *during render*. None of this is wrong, and all of it is a
  trap for someone whose priors are React. It is severe enough that the project ships a custom ESLint plugin
  (`react-hooks-static-deps`) to make dep arrays statically analyzable, because the cursor model requires it.
  — *Confirmed*
- **Manual GC.** Because the virtual tree holds direct DOM references (`VirtualElementTag.target` is the live
  node), unmount has to null out every hook slot, `$element`, `renderedValue` and `onUpdate` by hand
  (`helpGc()`, `teact.ts:596`). React avoids this class of leak structurally. — *Confirmed*

### Does it transfer

**No. Do not write your own framework.** This is the bluntest verdict in the audit, and the audit's own
source notes reach it independently.

Every advantage Teact has is reproducible on top of React, Preact or Solid at a fraction of the cost:

- the frame-phase scheduler is `fasterdom` — **a separate 3-file library that has nothing to do with the
  framework** and can be dropped into a React app tomorrow;
- animation gating is a signal plus a check in your store's subscriber — framework-agnostic;
- signals for high-frequency values: Solid has them natively; React has `useSyncExternalStore`, Preact has
  `@preact/signals`;
- non-cascading context: use a store with selector subscriptions (Zustand, Jotai) instead of raw Context;
- application-tuned list reconciliation: use keys properly and a virtualizer, or fixed-height rows.

What you lose by writing your own is error boundaries, Suspense, devtools, hiring, and roughly one senior
engineer permanently. Telegram accepted that trade because a contest rule forced their hand in 2019 and
because by 2026 the framework is too deeply coupled to remove. **You have no such rule.** Note also that the
"no dependencies" doctrine that justified Teact **is no longer enforced by its own authors**: `package.json`
now lists 33 runtime and 68 dev dependencies, including 17 `@tiptap/*` (ProseMirror) packages for the
composer rewrite; Web K, the other contest winner, now ships SolidJS. — *Confirmed*

---

## Decision 4 — A two-phase frame model, enforced at runtime and by lint

### What they did

The app has an explicit, documented frame model that separates DOM **reads** from DOM **writes**, and it is
enforced three ways: by API design (`requestMeasure` / `requestMutation`), by a **runtime throwing linter in
dev builds** (`stricterdom`), and by custom ESLint/Stylelint rules that survive staff turnover.

### The evidence

**Confirmed.** `AGENTS.md`, under a heading literally marked "⚠️ IMPORTANT: Fasterdom & Rendering Phases":

```
--- frame start ---
1. effects
2. requested measures (DOM reads)
3. render JSX → DOM
4. layout effects
5. requested mutations (DOM writes)
6. forced reflow measure (avoid!)
7. forced reflow mutate (avoid!)
--- frame end ---
```

with a phase-permission table:

| Hook / context | Can READ (measure) | Can WRITE (mutate) |
|---|---|---|
| `useLayoutEffect` | ❌ NO | ✅ YES |
| Event handlers (default) | ✅ YES | ❌ NO — use `requestMutation` |
| `requestMeasure` callback | ✅ YES | ❌ NO |
| `requestMutation` callback | ❌ NO | ✅ YES |

`src/lib/fasterdom/fasterdom.ts` keeps `pendingMeasureTasks`, `pendingMutationTasks`,
`pendingForceReflowTasks`, flushed once per RAF and sequenced through **promise microtasks** — with the
authors' own comment explaining why: `// We use promises to provide correct order for Mutation Observer
callback microtasks`. `stricterdom.ts` is the enforcement half: `enableStrict()` is switched on in DEBUG
(`STRICTERDOM_ENABLED = DEBUG`) and **throws** when code reads layout during the mutate phase or writes
during the measure phase, with `layoutCauses.ts` enumerating the offending properties.

The lint layer encodes four more architectural invariants mechanically:

- `eslint-plugin-tt-multitab` — enforces `tabId` threading through actions and selectors (Decision 12);
- `eslint-plugin-no-null` — `null` is banned outright in favour of `undefined`;
- `eslint-plugin-react-hooks-static-deps` — dep arrays must be statically analyzable, because Teact's hooks
  are cursor-based;
- `stylelint-high-performance-animation` — blocks animating non-compositable CSS properties;
- plus `@mytonwallet/stylelint-whole-pixel` and `stylelint-plugin-use-baseline`.

### The why

Sourced, and unusually well-sourced for this project. The author's one substantial public technical writeup,
*"Fasterdom: Optimized Browser Rendering"* (May 2024, hosted under the Teact repo), states it:

> "The browser stakes all your mutation calls until the rendering moment, which is either the end of an
> animation frame or a forced reflow caused by a measurement call." … "forced reflows are bad (because the
> browser needs to recalculate layout and styles instead of leveraging computation cache)."

And critically, the article **does not claim React is bad at this**: *"Web frameworks such as React or Teact
already respect this during virtual DOM operations."* The differentiator is extending read/write phase
separation **to application code**, not just to reconciliation. — *Confirmed*

### What it cost them

Verbosity and a learning curve. Ordinary code that would be three lines becomes a `requestMeasure` closure
followed by a `requestMutation` closure, and everything you measure is one frame stale. There is an escape
hatch (`requestForcedReflow`) that `AGENTS.md` rations explicitly: *"Use `requestForcedReflow` – Only as last
resort for sync measure+mutate."* The cost is real but bounded, and it is paid in developer time rather than
runtime.

### Does it transfer

**Yes — this is the single highest-leverage idea in the entire codebase, and it is framework-agnostic.**
`fasterdom` + `stricterdom` turn layout thrashing from "something you discover in a profiler in month nine"
into "an exception in dev on the commit that introduces it". It is three files. Bolt it onto React.

The meta-lesson transfers even more strongly: **encode your architecture as lint rules.** A rule that lives
in a wiki page is gone in two quarters. A rule that throws in CI outlives everyone who agreed to it. Three
custom ESLint plugins and a Stylelint plugin is a very cheap way to make an architecture survive turnover.

---

## Decision 5 — Put the protocol client in a worker; let only plain DTOs cross the boundary

### What they did

The entire MTProto stack — TL serialization, AES-IGE/CTR, RSA, the Diffie–Hellman handshake, the connection
pool, the update-gap algorithm — lives in a dedicated Web Worker. The UI never sees a wire object. A
generated builder layer converts TL objects into plain `Api*` DTOs at the boundary.

### The evidence

- **Confirmed.** `AGENTS.md`: *"We use GramJS inside a web worker; UI code uses plain objects (`Api*` types)
  in `src/api/types`. … Conversion from and to `Api*` objects is done by `apiBuilders` (function name starts
  with `buildApi*`) and `gramjsBuilders` (`buildInput*`)."* 23 `apiBuilders` files, 3 `gramjsBuilders`.
- **Confirmed.** `worker-J7_WDuX0.js` is **742,096 B raw / 240,922 B gzip — the largest single asset in the
  deployment, and 45% of all decoded JS loaded on the login screen.** All of it is off the main thread.
- **Confirmed.** Messages are coalesced in both directions once per microtask tail
  (`throttleWithTickEnd`), and binary payloads use `postMessage` transferables keyed off
  `response.arrayBuffer`.
- **Confirmed.** There is a DEBUG-time assertion in `connector.ts` that responses contain **no `VirtualClass`
  instances** — i.e. the boundary is machine-checked, not just documented.

### The why

Two forcing functions, both structural:

1. **MTProto makes client-side crypto mandatory.** Per `core.telegram.org/mtproto`, the client must perform a
   DH authorization key exchange before any API call, and every message is AES-encrypted and TL-serialized.
   That is CPU work on every single request. Run it on the main thread and animations die. `AGENTS.md`'s rule
   #1 is "Animations first"; this is that rule applied at the largest possible scale. — *Strong inference,
   from the protocol requirement plus the stated rule*
2. **A serialization boundary is a decoupling boundary.** Because only plain serializable objects cross
   `postMessage`, the UI is structurally prevented from depending on the wire format, and the same DTOs can
   be proxied to non-master tabs over `BroadcastChannel` (Decision 12) with no extra work.

### What it cost them

A 742 KB worker on the critical path to a usable login screen — the filmstrip shows the shell painting at
~3.0 s and then the QR code and both CTAs appearing together at ~9.0 s, because the CTA is gated on an MTProto
round trip that cannot begin until the worker downloads and boots (`screenshots/filmstrip-3000ms.png` vs
`screenshots/filmstrip-9000ms.png`; the absolute numbers are inflated by our proxy environment, the *ordering*
is not). Plus request/response correlation plumbing, progress-callback marshalling, a Safari/Tauri health-check
watchdog (`HEALTH_CHECK_TIMEOUT = 150`, *"Workaround for iOS sometimes stops interacting with worker"*), and
remote console piping so worker logs surface in the page.

### Does it transfer

**Yes, with a caveat.** Put whatever is CPU-heavy and protocol-shaped in a worker, and enforce that only
plain DTOs cross the boundary — that part is unconditionally good, and the DEBUG-time "no class instances in
the response" assertion is worth stealing verbatim.

The caveat: **you will not have MTProto.** If taskrgram speaks HTTP/JSON plus a WebSocket for realtime, your
transport does no meaningful CPU work and a worker buys you nothing but complexity. Adopt the *layering*
(transport module → DTO builders → store → UI, with a machine-checked boundary) without the worker, and add
the worker later only if you measure main-thread time in serialization or crypto. Do not put a worker in the
critical path of first paint the way this app does.

---

## Decision 6 — A bounded sliding window, not true virtualization

### What they did

The message list is not a windowed virtual scroller. It is a **bounded sliding window over loaded slices**:
real DOM nodes, native scrollbar, variable heights, with a hard cap on how many messages are mounted.

### The evidence

**Confirmed**, from both source and live measurement.

`src/config.ts`:

```ts
export const MESSAGE_LIST_SLICE = isBigScreen ? 60 : 40;
export const MESSAGE_LIST_VIEWPORT_LIMIT = MESSAGE_LIST_SLICE * 2;
export const CHAT_LIST_SLICE = isBigScreen ? 30 : 25;
export const CHAT_LIST_LOAD_SLICE = 100;
```

Measured live against `@TelegramTips` (11.19 M subscribers, years of history) at 1600×1000 — see
`screenshots/23-desktop-message-list-scrolled-up-virtualization-test.png`:

| Action | `.Message` nodes | `scrollHeight` | JS heap |
|---|---|---|---|
| Chat opened | 29 | 20,317 px | — |
| scrollTop −4,000 px | 29 | 20,317 px | — |
| scrollTop −12,000 px | 29 | 20,317 px | — |
| scrollTop −25,000 px | 89 | 61,266 px | 54.6 MB |
| scrollTop −40,000 px | **89** | **61,266 px** | **52.3 MB** |

Nodes are **not** recycled as you scroll; `scrollHeight` grows in discrete jumps as slices load, then stops.
Heap went *down* between the last two samples — the older slice was released. That is the signature of a
bounded window, not of `react-window`.

The chat list is handled differently and more cleverly: **fixed-height rows at computed absolute offsets**,
so the container height is constant and reordering is a transform rather than a reflow:

```tsx
const offsetTop = panesHeight + archiveHeight + (viewportOffset + i) * CHAT_HEIGHT_PX;
return <Chat teactOrderKey={isPinned ? i : getOrderKey(id, isSaved)} offsetTop={offsetTop} … />;
```

### The why

No published statement. But `MessageList.tsx` (1,464 lines) is full of comments describing the failure modes
this design is *avoiding* and the ones it creates:

> `// We avoid the very first item as it may be a partly-loaded album // and also because it may be removed when messages limit is reached`
> `// Suppresses spurious load-more triggers caused by Safari delivering stale IntersectionObserver entries between DOM mutation and scroll restore`
> `// Remove snap when scrolling up to avoid scroll bug // https://bugzilla.mozilla.org/show_bug.cgi?id=1753188`

The reading: **messages have unknown, variable, and *changing* heights** — an image finishes loading, a
sticker starts animating, a reaction row appears, a translation expands — and pixel-accurate virtualization
requires knowing heights before layout. Fighting that produces scroll jumps, which are the most viscerally
bad thing a chat log can do. The team chose to keep real nodes, cap the count, and hand-write scroll
anchoring (`currentAnchor` / `currentAnchorTop` restored in a `useLayoutEffect` via `requestForcedReflow`).
— *Strong inference*

### What it cost them

A hard cap on visible history — roughly 80–120 messages mounted (2× slice) — plus a long tail of
browser-specific scroll bugs, visible one by one in the comments above. Find-in-page only finds what is
mounted.

### Does it transfer

**Yes, and it is the right trade for a chat log.** You get correct native scrollbars, correct find-in-page,
correct variable heights, and correct keyboard scrolling *for free*, at the cost of not being able to show
40,000 rows at once — which no chat user wants. It is also **much less code than true virtualization**, once
you count the scroll-anchoring you have to write either way.

Concretely for taskrgram: bound the mounted set at ~2× a page, load by page on an `IntersectionObserver`
sentinel, anchor scroll on a stable element, and reserve true virtualization for lists that genuinely are
uniform and enormous (a member picker, an audit log). Steal the chat-list trick too: **fixed-height rows at
absolute offsets** turns reordering into a transform and is a big win for any list that animates.

---

## Decision 7 — Expose performance as a user-facing product surface

**This is the strongest single piece of evidence about this team's values in the entire audit.** Treat it as
such.

### What they did

Settings has a top-level entry called **"Animations and Performance"**. It contains a three-stop slider —
`Power Saving | Nice and Fast | Lots of Stuff` ("Choose the desired animations amount") — and then a section
**literally headed "Resource-Intensive Processes"**, with three expandable groups:

- **Interface Animations** — Page Transitions · Message Sending Animation · Media Viewer Animations ·
  Message Composer Animations · Context Menu Animation · **Context Menu Blur** · **Message Blur** ·
  Right Column Animation · Dust-effect deletion · Text Streaming
- **Stickers and Emoji** — Allow Animated Emoji · Loop Animated Stickers · Reaction Effects · Sticker Effects
- **Media Autoplay** — Autoplay GIFs · Autoplay Videos

Ten individually-toggleable interface animations. And **the two backdrop-filter effects are broken out
separately from the animations they accompany** — "Context Menu Blur" is its own switch, distinct from
"Context Menu Animation".

### The evidence

**Confirmed** in the live authenticated product — `screenshots/26-desktop-settings-animations-and-performance-toggles.png`
and `screenshots/27-desktop-settings-animations-performance-expanded-granular-toggles.png`.

Confirmed in source: `src/util/perfomanceSettings.ts` maps these settings to body classes —
`no-page-transitions`, `no-message-sending-animations`, `no-media-viewer-animations`,
`no-message-composer-animations`, `no-context-menu-animations`, `no-menu-blur`, `with-message-blur`,
`no-right-column-animations` — with levels `ANIMATION_LEVEL_MIN/MED/MAX/CUSTOM` in `config.ts` and
`ANIMATION_LEVEL_DEFAULT = ANIMATION_LEVEL_MED`. **We observed `with-message-blur` on the live `<body>`**
alongside `is-pointer-env is-linux`. The feature shipped in 1.61.0 (2023-04-26) as "Power Saving Mode":

> "You can extend battery life and improve performance by turning on Power Saving Mode or individually
> disabling autoplay, animations and effects in Settings > Animations and Performance."
> — `CHANGELOG.md`

### The why

No published statement, and none is needed — the design *is* the statement. Three claims follow directly:

1. **They know animation has a real cost and refuse to pretend otherwise.** Most products treat "we made it
   beautiful" as free and hide the cost. Shipping a settings screen called "Resource-Intensive Processes" is
   an admission, in the user's own language, that the pretty things cost something.
2. **They know *blur specifically* is the expensive one.** Breaking `backdrop-filter` out from the animation
   it decorates is not a UX nicety; it is a compositor-cost distinction surfaced as a user control. You only
   design that control if you have profiled it. — *Strong inference*
3. **They shipped a control rather than a heuristic.** They could have auto-detected device class. They
   chose to let the user decide, with a coarse three-stop default and a granular escape hatch. That is a
   values choice: user agency over cleverness.

The revealing gap: **there is no `prefers-reduced-motion` usage anywhere in `src/styles`.** They built a
bespoke, richer control and skipped the platform's standard signal. That is the same instinct that produced
Teact, and it is the wrong half of it. — *Confirmed (the absence), **Strong inference** (the interpretation)*

### What it cost them

Every one of those toggles is a code path that has to keep working — ten body classes × every animated
surface = a combinatorial test matrix nobody is testing exhaustively. It is also a settings screen that most
users will never open, which is a real cost in a consumer product's information architecture.

### Does it transfer

**Yes, and it is the recommendation that will most surprise your PM.** Ship a "reduce motion / reduce
effects" control in taskrgram, honour `prefers-reduced-motion` as its default, and make the expensive
individual effects (backdrop blur, autoplay, any continuous animation) independently switchable.

Three reasons this matters *more* for an internal corporate app than it did for Telegram:

- corporate fleets are heterogeneous and old — you will have users on machines that are a 4–6× CPU handicap
  versus your dev laptop, and our own throttling run showed TBT going **72 ms → 753 ms at 4× CPU** on a page
  with almost nothing on it;
- remote workers on VDI / Citrix / RDP get *nothing* from compositor effects and pay full price for them;
- an accessibility requirement you can point at ("we ship a motion-reduction control, on by default when the
  OS asks for it") is cheap here and expensive to retrofit.

The deeper transferable idea: **make the cost of your own polish visible and controllable, to users and to
yourselves.** A toggle is also a measurement instrument.

---

## Decision 8 — Inject the dark palette from JS at runtime, not from CSS

### What they did

The primary light/dark palette does not exist in the stylesheets. It is a map of **78 `[light, dark]` colour
tuples in a JS chunk**, written into a `<style>` element that the app owns and rewrites on every theme change.

### The evidence

**Confirmed** by bundle forensics. In `assets/initial-CskBLhZ6.js`:

```js
var Rr={"--color-primary":[`#3390EC`,`#8774E1`],"--color-background":[`#FFFFFF`,`#212121`],
  "--color-background-secondary":[`#F4F4F5`,`#0F0F0F`],"--color-background-own":[`#EEFFDE`,`#766AC8`], … }
```

and in `assets/Checkbox-Cxf2-dWf.js`:

```js
var Ht=document.createElement(`style`); document.head.appendChild(Ht);
function Kt(){ Ht.textContent = `html { ${qt(zt)} } html.theme-light { ${qt(Bt)} } html.theme-dark { ${qt(Vt)} }`; }
```

Corroborating: a static scrape of all 25 CSS files finds **566 distinct custom properties, 252 on `:root`, but
only 150 `--var` declarations under `.theme-dark` — and those are *local* overrides** (code-block syntax
colours, individual modals), not the main palette. The main palette is simply absent from CSS.

Theme switching is animated: `src/util/switchTheme.ts` uses `colorjs.io` to interpolate each variable over
`DURATION_MS = 200`, toggling a `no-animations` class on `<html>` for 500 ms and swapping
`<meta name="theme-color">`. See `screenshots/14-desktop-settings-general-dark-theme-selected.png` and
`screenshots/15-desktop-main-layout-dark-theme-chat-list-and-message-list.png`.

### The why

**What it bought: per-peer accent colours.** The same injector writes server-driven, per-entity tokens —
`color-peer-${n}`, `color-peer-bg-${n}`, `color-peer-gradient-${n}` — with a 7-colour fallback palette
(`#D45246 #F68136 #6C61DF #46BA43 #5CAFFA #408ACF #D95574`). Telegram's peer-colour feature lets the *server*
assign a user or channel a colour scheme. You cannot express a server-driven, unbounded, per-entity palette in
a static stylesheet. Once you need a runtime style channel for that, putting the base palette through the same
channel is nearly free — and it also makes the 200 ms interpolated theme transition trivial, because the
interpolator already owns the tokens. — *Strong inference; no first-party statement exists*

### What it cost them

**`style-src 'unsafe-inline'` in the CSP.** — *Confirmed.* The deployed policy is otherwise genuinely tight:

```
default-src 'self'; script-src 'self' 'wasm-unsafe-eval' https://t.me/_websync_ …;
worker-src 'self'; style-src 'self' 'unsafe-inline'; object-src 'none';
base-uri 'none'; form-action 'none';
```

We verified the script side works: when our accessibility harness tried to inject axe-core from a CDN, the
page's own CSP blocked it. But `style-src 'unsafe-inline'` is a real weakening — it re-enables a family of
CSS-injection and data-exfiltration-by-selector attacks that a nonce-based or hash-based policy would close.
The palette is also invisible to static analysis: any tooling that reads your CSS to check contrast, generate
docs, or diff a design system sees nothing.

There is a second, smaller cost: the theme tuples are in `initial-CskBLhZ6.js` (52,944 B) and the injector in
`Checkbox-Cxf2-dWf.js` (32,650 B), both on the eager critical path. Theming is not deferrable.

### Does it transfer

**Take the token structure; reject the injection mechanism.** The three-tier token system is genuinely good
and worth copying: global semantic tokens on `:root`, component-scoped locals with a `--_`-prefix convention
marking private tokens (577 selectors define these), plus cascade layers declared up front in the HTML to fix
the async-CSS ordering problem:

```css
@layer reset, variables, ui, components;
@layer ui { @layer tablist, spinner, button, input, layout; }
```

That inline `<style>` is the *only* one in the document and it is a genuinely elegant solution — chunked CSS
then declares `@layer ui { … }` and specificity is decided by layer, not by load order.

But put your palette in CSS. `html.theme-dark { --color-background: #212121; }` costs nothing, is statically
analyzable, needs no `unsafe-inline`, and works before your JS loads. If you later need a small number of
runtime-dynamic tokens (per-team accent colour, per-project tag colour), set **those specific properties** on
an element via `element.style.setProperty()` — which is not subject to `style-src` — rather than injecting a
stylesheet. You keep the capability and keep the CSP.

---

## Decision 9 — Serve media through the service worker with Range requests, not as URLs

### What they did

`<video>` and `<audio>` elements point at `./progressive/<id>` URLs that **do not exist on any server**. A
hand-written service worker intercepts the range request, forwards it via `postMessage` to the page, which
forwards it to the API worker, which fetches MTProto file parts, and the SW answers with `206 Partial
Content`. The browser's own media stack drives seeking. No MSE, no HLS, no DASH, no custom player.

### The evidence

**Confirmed at runtime.** Observed in the live session:

```
206 GET https://web.telegram.org/a/progressive/document5109473995049145023
206 GET https://web.telegram.org/a/progressive/document5109473995049145021
```

**Confirmed in source.** `src/serviceWorker/progressive.ts`: `DEFAULT_PART_SIZE = 0.5 * MB`,
`MAX_END_TO_CACHE = 2 * MB - 1` (*"We only cache the first 2 MB of each file"*), `PART_TIMEOUT = 60000`, plus
`// Optimization for Safari` for the `start === 0 && end === 1` probe request. `download.ts` streams full
files for "save as" with `DOWNLOAD_PART_SIZE = 1 MB`, `QUEUE_SIZE = 8` and a `Content-Disposition` header.
The service worker is hand-written — **zero `workbox` strings in the entire bundle graph**.

**Confirmed by traffic shape.** WebSocket aggregate for the session: 310 frames sent = 85,464 B; 660 frames
received = 10,026,931 B. **A ~117:1 downstream asymmetry**, because photos, video and stickers all come down
through the MTProto socket as `upload.getFile` chunks. HTTP host breakdown: 629 `web.telegram.org`, 102
`blob:`, and nothing else but three t.me aliases.

### The why

**Forced by the protocol, then exploited well.** Per `core.telegram.org/api/datacenter`, files are addressable
only by `(dc_id, file_reference)`, are downloadable only from the DC where the query executed, and
*"encryption keys are not copied between DCs"* — cross-DC access requires `auth.exportAuthorization` /
`auth.importAuthorization`. There is no URL to put in an `<img src>`. — *Confirmed, from Telegram's own docs*

The clever part is what they did with that constraint: instead of buffering whole files into `blob:` URLs
(which they still do for small media — `MEDIA_CACHE_MAX_BYTES = 512 * 1024`, above which the progressive path
takes over), they **re-created HTTP semantics inside the service worker** so the platform's native media
stack does the hard part. Seeking, buffering, playback-rate, picture-in-picture: all free.

MSE appears exactly once, in `src/hooks/useStreaming.ts`, and only for Safari:

```ts
if (!IS_SAFARI || !mimeType || !MediaSourceClass?.isTypeSupported(mimeType)) return undefined;
```

Underneath sits a Cache Storage LRU: buckets `tt-media`, `tt-media-avatars`, `tt-media-progressive`,
`tt-custom-bg`, `tt-lang-packs-v52`, `tt-assets`, with `CACHE_TTL = 5 days`, an `X-Last-Access` header, an
hourly sweep, and `yieldToMain()` between entries so eviction does not block a frame.

### What it cost them

A hand-written service worker with a three-hop request path (SW → page → worker → network), per-browser
workarounds (`// An attempt to fix freezing UI on iOS` around `clients.claim()` racing a 3 s timeout), and
media that is invisible to every HTTP-layer tool you own — no CDN, no cache headers, no `Content-Length`
you did not synthesise.

### Does it transfer

**Yes, for exactly one situation: auth-gated media that must not be reachable by URL.** For an internal team
app that is a *very* plausible situation — file attachments that should be inaccessible to anyone without a
live session, with no signed-URL leakage into browser history, chat logs, or a proxy's access log.

If your files can just be signed S3 URLs with a short TTL, do that instead; it is an order of magnitude less
code. But if legal or security says "an attachment URL must never be independently fetchable", the
SW-Range-request pattern is the cleanest way to satisfy that while still letting `<video>` seek natively. Note
the sharp edge: **your app is unusable in browsers with service workers disabled**, and Firefox private
browsing has historically been one of those.

---

## Decision 10 — Zero telemetry, zero third-party hosts, no CDN, self-hosted on their own AS

### What they did

There is no analytics, no crash reporting, no APM, no CDN, no third-party font host, and no ad tech anywhere
in this product. Static assets are served from Telegram's own nginx fleet inside their own autonomous system.

### The evidence

**Confirmed**, by exhaustive search of all 461 JS chunks and 453 source maps:

| Searched | Result |
|---|---|
| `sentry`, `Sentry` | **0 hits** |
| `google-analytics`, `gtag`, `googletagmanager`, `amplitude`, `mixpanel`, `segment`, `posthog` | **0 hits** |
| `workbox` | 0 hits (hand-written SW) |
| `react`, `react-dom`, `preact` | 0 hits |

**Confirmed** by live traffic: 737 HTTP responses across the authenticated session, of which 629 were
`web.telegram.org`, 102 were `blob:`, and 6 were t.me / telegram.me / telegram.dog deep-link aliases. **Zero
third-party hosts.** The only external hosts referenced anywhere in the bundle are
`translations.telegram.org` (language packs), `ss3.4sqi.net` (Foursquare venue-category icons, also
whitelisted in CSP `img-src`), and — inside the payments flow only — `api.stripe.com` and
`tgb.smart-glocal.com`.

**Confirmed** for hosting: A record `149.154.167.99`, AAAA `2001:67c:4e8:f004::9`, **AS62041 Telegram
Messenger Inc**, Amsterdam. `server: nginx/1.30.1`. `x-served-by: meta4240719` rotating per request across a
node pool. **No `cf-ray`, no `x-amz-cf-id`, no `x-cache`, no `age`, no `via`.** GoDaddy wildcard cert for
`*.web.telegram.org`, not Let's Encrypt.

### The why

Brand consistency with the privacy positioning. Telegram's FAQ sells "speed and security"; shipping a client
that phones home to Google would be self-refuting. Their own reuse policy is consistent — from
`core.telegram.org/api/obtaining_api_id`, unofficial clients are *"automatically put under observation"* to
prevent misuse. Telemetry is not neutral for this product; it is a promise they have chosen to keep at the
network layer. — *Strong inference; consistent with everything published, but never stated as a rule*

### What it cost them — and this is the interesting part

The engineering consequences are real and visible in our measurements:

1. **They are blind to production defects.** No error reporting anywhere. Combined with Teact having **no
   error boundaries** (Decision 3), a component that throws in production silently reuses its last good
   render and nobody ever finds out. We caught a live defect in one hour of walkthrough — a chat-header
   button whose accessible name is the raw i18n key `aria-label="AccDescrPollVoteDown"`, i.e. a translation
   lookup that returned the key. That has presumably shipped to a very large number of users. We also caught
   `console.error(undefined)` — an error path that swallows its own diagnostic.
2. **Bug reports are routed to a separate platform** (`bugs.telegram.org/c/4002`), which is why GitHub issues
   like #128 sit unattended for four years.
3. **Self-hosting costs them CDN engineering they have not done.** The delivery layer is measurably behind:
   `cache-control: max-age=3600` on **content-hashed immutable assets** (a revalidation storm every hour
   across ~490 assets, when these could be `max-age=31536000, immutable`); **gzip only**, no Brotli, no zstd
   — verified by sending `Accept-Encoding: br` and receiving identity; **no `alt-svc`**, so no HTTP/3; **no
   HSTS header on any response**. Brotli alone would cut roughly 20% off a 2.32 MiB transfer.

### Does it transfer

**Split it.** The "no third-party hosts" half transfers directly and is easy: self-host your fonts, do not
load anything from a public CDN, and keep your CSP `default-src 'self'`. For an internal corporate app that
is table stakes, not a stance — you get a smaller supply-chain surface and no third-party outage in your
critical path.

The "no telemetry at all" half **does not transfer, and you should invert it.** Telegram's blindness is a
price they pay for a brand promise; you have no such promise to make to your own employees, and you have a
duty to know when your app is broken for them. Run **self-hosted** error tracking (GlitchTip / Sentry
on-prem) and a small first-party metrics endpoint. Report crashes, unhandled rejections, failed sends, and
reconnect storms. Do *not* report message content or per-user behavioural analytics — that is where the
Telegram instinct is right. The rule to take is **"no data leaves the perimeter"**, not "no data".

And fix the delivery layer they did not: `immutable` caching on hashed assets, Brotli, HTTP/3, HSTS. All four
are configuration, not engineering.

---

## Decision 11 — Manual code splitting with named bundles and a size budget in CI

### What they did

No route-based splitting, no `React.lazy` sprayed around. Six hand-maintained barrel modules in
`src/bundles/` (`auth`, `main`, `extra`, `calls`, `stars`, `editor`), a tiny loader, and **151 `*.async.tsx`
shim files** that each own one lazy boundary. Bundle size is diffed in CI against a stored baseline, and
`[Size]` is a first-class commit tag.

### The evidence

**Confirmed.** `src/util/moduleLoader.ts`:

```ts
export enum Bundles { Auth, Main, Extra, Calls, Stars, Editor }
…
case Bundles.Extra: LOAD_PROMISES[Bundles.Extra] = import('../bundles/extra');
```

The consumer pattern is the whole of `MediaViewer.async.tsx`:

```tsx
const MediaViewerAsync: FC<OwnProps> = ({ isOpen }) => {
  const MediaViewer = useModuleLoader(Bundles.Extra, 'MediaViewer', !isOpen);
  return MediaViewer ? <MediaViewer /> : undefined;
};
```

The shim receives `isOpen`, so the chunk is fetched exactly when the feature is first needed —
`useModuleLoader` reads a synchronous memory cache first and only then triggers `loadModule(...).then(forceUpdate)`.

CI: `rollup-plugin-bundle-stats` with a baseline file for PR comparison, **plus a custom
`createWorkerBundleCollectorPlugin` that merges worker bundles into the report** so the 742 KB worker cannot
hide from the budget.

**Measured result.** Total deployment: 461 JS chunks, 5.28 MiB raw / 2.09 MiB gzip. Eager critical path:
29 assets, 763 KiB raw / **305.4 KiB gzip**. On the login screen only **6.5% of chunk files** load — but they
carry **29.6% of all JS bytes**. The big deferred chunks are correctly deferred: `main` (420 KB),
`useConnectionStatus` (480 KB), `Modal` (396 KB), `editor` (295 KB) are all absent until after auth.

### The why

The rubric put money on it (Decision 2), and it stuck as a habit. The `[Size]` commit tag and the CI baseline
are the mechanism by which a 2020 contest incentive is still shaping a 2026 codebase. — *Strong inference*

### What it cost them

Three things, and they are instructive.

1. **Chunk names are unreliable.** Rolldown derives the chunk name from the first/primary module, so
   `InputText-CnVXgAD5.js` carries **218,910 B** and is on the login critical path — a screen with **zero
   `<input>` elements** (measured). `Checkbox-Cxf2-dWf.js` contains the theme injector.
   `useConnectionStatus-DLbED6Q4.js` is 480 KB. The names are not content indicators, which makes the
   budget harder to reason about than it looks. That 219 KB text-input chunk on a QR screen is
   **barrel-file / shared-chunk bleed** — the cost of hand-maintained barrels. — *Inferred, flagged as such*
2. **Over-splitting at the tail.** 373 of 461 chunks (81%) are single-language `highlight.js` grammars,
   928,928 B raw / 433,732 B gzip. Great for the median user, but a code block in an exotic language costs an
   extra round trip on a `max-age=3600` cache.
3. **Manual maintenance.** Six barrels and 151 shims is a convention that has to be enforced by review
   forever.

### Does it transfer

**The discipline transfers; the paranoia does not.**

Worth stealing outright: **an explicit, reviewable lazy boundary per heavy feature** (one small file whose
whole job is "load this chunk when `isOpen` becomes true"), a **size budget diffed in CI against a baseline
with a hard fail**, and **counting worker bundles in that budget**. This is more predictable than implicit
route splitting and it means you always know what is in your first paint.

Not worth copying: the intensity. Your users are on a LAN with a warm cache. Our own repeat-visit measurement
shows what actually matters — on a warm reload this app transfers **2,975 bytes total, 0.4% of the cold load**,
and `loadEventEnd` halves (`screenshots/39-desktop-warm-reload-restored-chat-list-from-cache.png`). For an
internal app, **cold-load bytes are close to irrelevant and warm-load correctness is everything.** Set a
budget so you notice a 2 MB regression; do not split 373 syntax grammars.

---

## Decision 12 — Multi-tab as architecture, not as a feature

### What they did

State is scoped along **two independent dimensions** from the ground up: `byTabId` for per-tab UI state, and
`sharedState` (backed by a `SharedWorker`) for cross-tab settings. Exactly one tab is elected master and owns
the MTProto worker and the socket; every other tab proxies its API calls over a `BroadcastChannel`.

### The evidence

**Confirmed.** `GlobalState` carries both `byTabId: Record<number, TabState>` (a 1,141-line `tabState.ts`)
and `sharedState: SharedState` (`src/global/shared/sharedStateConnector.ts` → `new SharedWorker(...)`).

`AGENTS.md` makes it mandatory rather than optional:

> "Actions and selectors can accept a `tabId` parameter, so we don't lose tab context when working with
> multiple tabs. **`tabId` is required** if calling an action or selector that can accept it."

and the type machinery in `src/global/index.ts` makes the requirement structural: *"`Required` actions are
called from actions to ensure the `tabId` is always provided if needed."* On top of that sits a **custom
ESLint plugin, `eslint-plugin-tt-multitab`**, whose only job is enforcing this threading.

The runtime half:

```ts
const promise = isMasterTab ? makeRequest({…}) : makeRequestToMaster({ name: fnName, args });
```

with master election in `src/util/establishMultitabRole.ts` and `subscribeToMasterChange(...)` wired in
`src/index.tsx`. **Confirmed live:** `localStorage` contains `tt-multitab_1`, the election flag, and the
`sharedState.worker` is registered. Version reconciliation goes through the same channel — a tab whose
`appVersion` does not match hard-reloads itself.

Multi-tab shipped in **1.59.0 (2023-01-30)**, i.e. roughly two years after 1.0.0.

### The why

Structural necessity, not polish. An MTProto session is a single authenticated connection with server salts,
`msg_id` sequences and an update `pts`/`seq` stream that must be processed exactly once, in order. Two tabs
running two independent update pipelines against one session produce state corruption, not just wasted
sockets. — *Strong inference from the protocol requirements; **Confirmed** that the client is built this way*

### What it cost them

`tabId` threading through hundreds of actions and selectors — enough friction that it needed a bespoke lint
plugin to stay correct. And it was retrofitted: shipping this two years in is why the plugin exists at all.
Web K, which solved the same problem with a SharedWorker, has a documented hard limitation — Safari can only
run it in a single tab.

### Does it transfer

**The question transfers; the mechanism might not.** Decide on day one whether taskrgram's realtime
connection is per-tab or shared, because retrofitting is genuinely painful and this codebase is the proof.

For an internal app the calculus differs. Your realtime channel is probably a plain WebSocket to your own
server, and N sockets per user is usually fine — servers scale, and `BroadcastChannel` complexity is not
free. What you should copy regardless is the **state-shape discipline**: cleanly separate "state that belongs
to this tab" (open panel, scroll position, draft focus, modal stack) from "state that belongs to this user"
(settings, read cursors, theme). Even without a master tab, that separation is what makes two windows behave
sanely — and it is nearly free if you do it up front and expensive if you do not.

Note the desktop-first angle for taskrgram: users *will* have three windows open. Two of them will be the
same channel. Get read-state and notification suppression right across them.

---

## Decision 13 — Commit a 230 MB `dist/` to git and publish complete source maps

### What they did

The public repository contains the built artifacts. The release script commits the build:

```
"web:release:production": "npm i && npm run build:production && git add -A
   && git commit -a -m '[Build]' --no-verify && git push",
```

And production builds ship source maps (`build.sourcemap: true`), which are served publicly.

### The evidence

**Confirmed.** `dist/` is 230 MB in the clone. HEAD's commit subject is `[Build]`; every GitHub release body
is also `[Build]`. `--no-verify` skips the hooks.

**Confirmed.** **453 of 461 JS chunks ship a working `.map`**, every one resolving with HTTP 200:

```
GET /a/assets/index-IZ97MA_m.js.map     -> 200,   115,171 B
GET /a/assets/InputText-CnVXgAD5.js.map -> 200, 1,003,890 B
```

Extracting the `sources` arrays recovers **2,056 original file paths** — the complete module graph, including
internal codenames that were never meant to be a product name: `src/lib/vibecalls` (the WebRTC group-call
layer) and `src/components/gili` (the in-progress design-system layer). CSS maps are *not* served.

### The why

For `dist/`: **the public repo doubles as the deployment artifact store.** The release script is literally
build-and-push. — *Strong inference; the script is confirmed, the intent is inferred*

For source maps: for a GPL-3.0 project whose source is already public, publishing maps costs nothing and buys
better bug reports and community debugging. It is a coherent choice **in that context**. — *Possible*

### What it cost them

Repository bloat that every clone pays for; sources and published bundles versioned together so history is
dominated by build noise; `--no-verify` meaning the release commit skips whatever the hooks were protecting;
and a complete public map of the internal architecture including codenames. For an open-source client all of
that is acceptable. Two of the three would be unacceptable for a proprietary one.

### Does it transfer

**No, on both counts, and the reasons differ.**

`dist/` in git: no. Build artifacts belong in an artifact store or a container registry, and your CI should
produce them. This is the one decision in the audit that has no defensible version for you.

Source maps: **generate them, upload them to your error tracker, do not serve them publicly.** For a
proprietary internal app, public maps hand an attacker (including a curious insider) your complete module
graph, your internal codenames, and your unminified logic — including any client-side check you were hoping
was obscure. The correct pattern is `sourcemap: 'hidden'` (or equivalent): emit maps, strip the
`sourceMappingURL` comment, upload to Sentry/GlitchTip at deploy time. You keep readable stack traces and
publish nothing. Note that this only works if you actually have error tracking — see Decision 10.

---

## The through-line

Strip the specifics and one coherent philosophy is left, and it is more consistent than most shipped products
manage:

**The frame is the unit of correctness.** Everything else follows from that. If the frame budget is sacred,
then DOM reads and writes must be phase-separated (`fasterdom`), the separation must be enforced rather than
requested (`stricterdom`, `stylelint-high-performance-animation`), state updates must be schedulable and
therefore deferrable during animation (`beginHeavyAnimation`, `onFullyIdle`), high-frequency values must never
enter the render path at all (signals), the list reconciler must be tuned for the two lists that actually
matter (`teactFastList`), heavy computation must be off the main thread (the MTProto worker, the media worker
pool, the fastText worker), and the DOM must be bounded (the sliding window). Teact is not the philosophy —
Teact is the vehicle that happened to be available because a contest rule required one.

**Own the whole stack, and treat every dependency as a liability.** No React, no CDN, no analytics, no
Workbox, no HLS, no virtualization library, no error-reporting SaaS, their own nginx in their own AS, their
own charting library, their own Lottie renderer, their own framework. This is coherent with the brand and it
is what makes the codebase legible — there is exactly one way anything is done, and it is in the repo. It is
also why they have no error boundaries, no crash reports, and a four-year-old open accessibility issue.

**Make the cost visible.** `[Size]` and `[Perf]` as commit tags; bundle-size diffing in CI; DEBUG builds that
warn on renders over 7 ms, effects over 7 ms, and `useMemo` hit rates under 0.25, and dump per-component
render counts on double-click; a stale-global-write detector that throws; and — the strongest signal of all —
a *user-facing settings screen called "Resource-Intensive Processes"* where the expensive effects can be
switched off individually. This team does not pretend polish is free, either to themselves or to their users.

**Constraints, honestly labelled, produce the good parts.** The contest rubric produced the performance
culture. MTProto produced the worker boundary and the service-worker media pipeline. Variable-height,
media-rich rows produced the sliding window. Server-driven per-peer colours produced the runtime theme
injector. In every case the *constraint* is the reason, and where there is no constraint — two clients, a
committed `dist/`, 373 syntax-highlighting chunks — the decisions are markedly weaker.

### Where this philosophy is wrong for a 2026 internal corporate app

Bluntly: **most of it is optimising for a world you do not live in.**

Telegram's rubric was written for anonymous consumer users on unknown devices over unknown networks, where a
200 KB bundle difference was worth $500 and a broken 2FA flow cost $1,000. Your users are a known population,
on a corporate LAN, on machines you can inventory, who will open your app once on Monday and leave the tab
open until Friday. For them, cold-load bytes are close to irrelevant — the warm-reload measurement (2,975
bytes, 0.4% of cold) is the number that describes their actual experience. Spending engineering on the
contest-era size obsession is spending it on a metric your users will experience roughly once per deploy.

Four places the philosophy actively points the wrong way:

1. **Framework ownership is a luxury of a single-maintainer project with a decade of runway.** Teact costs a
   permanent senior engineer, no error boundaries, no devtools, and an onboarding tax on every hire — paid in
   exchange for scheduling control you can get from a 200-line scheduler on top of React. A five-person
   internal team cannot afford the maintenance and does not need the ceiling.
2. **"No telemetry" is a brand promise Telegram made and you did not.** Inherit "no data leaves the
   perimeter" and reject "no data". An internal app that cannot tell you it is broken for the Berlin office
   is worse than one that can, and you have no privacy story that self-hosted crash reporting violates.
3. **The animation ceiling is not your ceiling.** Telegram competes on delight against WhatsApp. You compete
   against Slack on *not being annoying*. Ten independently-toggleable animations is the right instinct
   pointed at the wrong target: ship one motion-reduction control wired to `prefers-reduced-motion` (which
   this app does not use at all) and put the saved effort into keyboard navigation, focus management, and
   screen-reader semantics — the exact three things this codebase's architecture makes hardest and its rubric
   never scored. Keeping inactive screens mounted inside `Transition` containers makes navigation feel
   instant *and* makes the accessibility tree ambiguous; we had to qualify every selector with `:visible`
   during this audit, and a screen reader has the same problem. For an internal tool with a legal
   accessibility obligation, that trade runs the other way.
4. **Feature surface.** This is a full-fidelity consumer client: payments, star gifts and gift *auctions*,
   stories, mini-apps, premium tiers, group video calls, a 32,000-character document editor in the composer.
   Gifts alone have 14 sub-folders. Roughly 10–15% of these 184,000 lines of component code correspond to
   anything an internal team chat needs. The architectural lessons are worth having; the scope is a warning.

The correct posture toward this codebase is the one its own author took toward React: **take the ideas, not
the expression.** The frame model, the worker boundary, the bounded window, the animation gating, the
performance-as-a-setting instinct, and the lint-encoded architecture are all excellent and all portable. The
framework, the two-client structure, the telemetry blackout, the committed build output, and the contest-era
size paranoia are not.
