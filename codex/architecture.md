# Runtime architecture

## Browser startup

1. nginx/edge returns a small HTML shell.
2. `redirect.js` and `compatTest.js` establish routing/browser support.
3. The hashed module entry loads Teact, global state initialization, styles, session utilities, and service-worker setup.
4. The client loads cached/local state, initializes localization, coordinates other tabs, elects a master tab, and renders `App`.
5. `App` selects auth, main, lock, or inactive screen from local/session/global state.
6. The main or auth bundle loads asynchronously behind `UiLoader`.

## Worker and protocol boundary

The UI never imports a server SDK into every component. Components dispatch global actions. API actions call a typed `callApi` connector. The connector sends batched messages to a dedicated module Worker. The Worker owns GramJS, Telegram TL constructors, crypto, sockets, retries, and update decoding.

Important properties:

- one Worker on the elected master tab;
- other tabs forward API requests over BroadcastChannel;
- callbacks/progress receive generated IDs;
- ArrayBuffers are transferred where possible;
- worker updates are batched before postMessage;
- decoded GramJS classes are converted into plain client API objects;
- feature-specific update reducers merge normalized state.

## Transport

Primary:

```text
wss://<telegram-dc-address>:443/apiws[_test][_premium]
subprotocol: binary
binaryType: arraybuffer
```

Fallback:

```text
POST https://<telegram-dc-address>:443/apiw1[_test][_premium]
body: binary MTProto payload
```

The WebSocket wrapper uses a 3-second initial connection timeout, exponential growth up to 30 seconds, an async mutex for receive ordering, and a binary queue. These are public-source client details; production DC selection and failover policy remain unknown.

## State model

The global tree is split into stable domains such as:

- auth/config/passcode;
- users/chats/peers;
- messages, threads, topics, search;
- settings/shared settings;
- stories/reactions/symbols;
- calls;
- payments/stars/gifts;
- management/statistics;
- per-tab UI/modal state.

Selectors produce component-facing views. Actions update the tree, and `withGlobal` subscribes Teact components. The model supports optimistic mutation and reconciliation with streamed API updates.

## Routing

The app uses a static base path and hash-based navigation. Numeric chat IDs and optional thread/list type are encoded in the fragment. Deep-link parameters can also appear in the fragment, including web-auth tokens. Browser history utilities layer push/replace state over the SPA.

Advantages:

- no origin rewrite rules for every chat;
- static hosting remains simple;
- chat links are addressable without server rendering;
- refresh can restore conversation context.

Trade-off: hash formats are less conventional and require careful migration/version handling.

## Persistence and multi-tab behavior

- Account-slot session records store main DC, test flag, user ID, and per-DC auth keys in localStorage.
- Legacy session keys are still supported.
- IndexedDB stores application data and optional encrypted passcode state.
- BroadcastChannel carries global diffs, local DB updates, API calls/responses, and language refreshes.
- Master-tab election avoids duplicate sockets and centralizes update ownership.
- Other Telegram web clients are detected through an inter-client channel.

## Service worker and caching

The service worker is not only an offline asset cache. It is a local transport adapter:

- Fingerprinted assets: cache first.
- HTML and selected WASM: network first with a three-second cache fallback.
- Progressive media: accepts browser Range requests, requests parts from an active client, and returns 206 responses.
- Downloads: bridged through a local route.
- Push/share: translates browser events into application actions.
- Progressive cache: caches at most the initial portion of a file, using account-specific cache names.

This is a deep module: browser media elements see ordinary URLs/range responses while MTProto media fetching stays behind the worker/client boundary.

## Pagination and virtualization

Message history, chat lists, search, stories, topics, participants, gifts, contacts, and profile media use offset IDs and bounded viewport slices. `useInfiniteScroll` maintains a visible subset and invokes loaders near boundaries. Message actions calculate `offsetId` and `addOffset` for forward, backward, and around-anchor loads.

This reduces DOM size and memory but requires careful anchor restoration, outlying-message handling, prepend compensation, and reconciliation when live updates arrive.

## Feature bundles

| Bundle | Responsibility |
|---|---|
| Initial/core | Global init, primitives, auth entry, protocol bootstrap |
| Auth | Code, password, registration |
| Main | Core chat shell and common messaging |
| Extra | Settings, search, rich modals, media viewer, management, many rare flows |
| Calls | Phone/group call UI and support code |
| Editor | Tiptap rich composer and syntax support |
| Stars | Commerce, gifts, auctions, transactions |

The main component schedules calls and editor bundle warming after delays. Other async wrappers load bundles when their feature becomes active.

## Authentication and authorization boundary

The browser authenticates the MTProto session and retains auth keys. There is no evidence of a conventional web-session cookie for the SPA. The server remains responsible for authorization.

For an internal implementation, avoid copying this part unless direct protocol connectivity is necessary. A backend-for-frontend with secure cookies can substantially reduce browser secret exposure and simplify rate limiting, audit logging, and role enforcement.

## Hosting and deployment inference

Confirmed delivery facts:

- nginx 1.30.1;
- static content-hashed chunks;
- gzip;
- one-hour HTTP cache headers;
- public source maps;
- Telegram edge identifier header.

Most likely deployment shape:

```text
build pipeline → immutable asset set + short-lived HTML
              → Telegram-controlled static origin
              → nginx/edge nodes
              → browser
```

Alternative: nginx may itself be the origin and the `meta` node may be an internal cache rather than a distinct CDN. Public evidence cannot decide.
