# Technology stack and evidence

## Frontend

| Technology | Exact evidence | Confidence |
|---|---|---|
| TypeScript | Public repo is TypeScript; TS/TSX source; version `12.0.38` matches runtime | Confirmed |
| Teact | Deployed `teact-CUUQAb7N.js`; imports from `lib/teact`; README describes React-paradigm reimplementation | Confirmed |
| Vite 8 / Rolldown | `vite@^8.2.1`, Vite config, deployed `rolldown-runtime-hePW80VL.js` | Confirmed |
| SCSS | `sass`, SCSS source, compiled CSS chunks | Confirmed |
| CSS Modules | Vite `css.modules` config and `.module.scss` components | Confirmed |
| CSS cascade layers | Inline `@layer reset, variables, ui, components` in HTML shell | Confirmed |
| Custom properties | Large light/dark/theme and layout token sets | Confirmed |
| FasterDOM/strict DOM scheduling | Deployed chunk and `requestMutation`/strict mode source | Confirmed |
| Custom icon font + SVG | `icons-*.woff2`, `Icon` components, SVG/inline SVG assets | Confirmed |
| Tiptap/ProseMirror | Package dependencies and rich-editor code | Confirmed |
| Lowlight | Package and code-language lazy modules | Confirmed |
| qr-code-styling | Deployed chunk and package dependency | Confirmed |
| Lottie/WASM | `TLottie` worker/chunk and vendored WASM implementation | Confirmed |

React 19 is present as a development dependency for types/tooling, but the deployed runtime and application source use Teact. Describing the app simply as “React” would be inaccurate.

## State and module architecture

- A global immutable-ish state object is organized by normalized chats, users, messages, settings, tabs, calls, stories, payments, and feature state.
- Action handlers are separated into API actions, API update reducers, and UI actions.
- Selectors isolate read models for memoized components.
- `withGlobal` injects derived state into Teact components.
- Async wrapper components call a bundle loader; primary bundles are `Auth`, `Main`, `Extra`, `Calls`, `Stars`, and `Editor`.
- Worker requests and updates are batched at tick end to reduce postMessage overhead.

## Network and API

The application does not introduce an app-specific REST/GraphQL layer for Telegram data.

```text
Teact component
  → global action
  → typed callApi(name, args)
  → dedicated module Worker
  → custom GramJS method
  → MTProto binary WebSocket
  → Telegram data center
```

Transport URLs are constructed as:

- `wss://<ip>:443/apiws`
- `wss://<ip>:443/apiws_test`
- `wss://<ip>:443/apiws_premium`
- HTTP POST equivalents at `/apiw1*` as fallback.

MTProto schema/types come from Telegram TL definitions. Update objects return through the worker and are routed into feature-specific API updaters.

## Storage and cache

| Store | Purpose | Confidence |
|---|---|---|
| localStorage | Account-slot session metadata, per-DC auth keys, shared settings, multi-tab markers | Confirmed source |
| IndexedDB `tt-data` | Client data/cache via `idb-keyval` | Confirmed source |
| IndexedDB `tt-passcode` | Optional AES-GCM encrypted session/global state | Confirmed source |
| Cache Storage | Hashed assets and first progressive media ranges | Confirmed source |
| In-memory normalized state | Current chats, messages, users, viewport slices, UI/modal state | Confirmed source |
| BroadcastChannel | Master-tab election, API forwarding, state/local DB sync, inter-client notifications | Confirmed source |

## Service worker

The module service worker handles:

- cache-first fingerprinted JS/CSS/fonts/images/WASM;
- network-first HTML and selected WASM;
- push events and notification clicks;
- Web Share Target payloads;
- local `/progressive/` range responses backed by worker-fetched Telegram media;
- local `/download/` responses;
- first-media-range caching with separate account cache names.

## External services

| Service | Role | Evidence | Confidence |
|---|---|---|---|
| Telegram data centers | Messages, users, media, settings, updates | MTProto implementation | Confirmed |
| `t.me`, `telegram.me`, `telegram.dog` | Web session synchronization | Runtime `_websync_` assets | Confirmed |
| Stripe | Card tokenization for supported native invoices | Direct POST to `api.stripe.com/v1/tokens` | Confirmed source |
| Smart Glocal | Card tokenization for supported invoices | Validated production/playground domains | Confirmed source |
| Foursquare image CDN | Venue category icons | CSP and map utility | Confirmed source |
| Google/Bing/OSM/Apple Maps | External map links | Map utility | Confirmed source |
| Fragment/Telegram ads/cocoon links | Feature-specific external navigation | Config/localization | Confirmed source |
| Third-party analytics | None observed | Assets, CSP, manifest, package, runtime inventory | Strong inference |

## Hosting and delivery

- `Server: nginx/1.30.1`.
- `x-served-by: meta...` suggests a Telegram edge node naming scheme.
- HTML and hashed assets observed with one-hour freshness.
- JS is gzip-compressed.
- Assets use content hashes and modulepreload.
- Source maps are generated by Vite and publicly reachable.
- Exact origin, CDN product, object storage, load balancer, deploy controller, and cloud are unknown.

## Testing and quality tooling in public source

- Vitest.
- Playwright.
- TypeScript compiler.
- ESLint and TypeScript ESLint.
- Stylelint with performance animation and whole-pixel rules.
- Bundle stats and visualizer plugins.
- GitHub Actions tooling.

These are source-confirmed capabilities, not proof that every check gates production deploys.
