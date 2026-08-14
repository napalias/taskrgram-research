# 09 — Authentication, Session Model and Security Posture

**Subject:** Telegram Web A, `https://web.telegram.org/a/`, version **12.0.38** (build 2026-08-11 15:24:14 UTC)
**Audit date:** 2026-08-14

---

## 0. Scope and rules of engagement — read this first

**This is an audit of publicly observable behaviour. It is not a penetration test.**

- **Nothing was exploited.** No vulnerability was triggered, chained, or weaponised.
- **No access control was bypassed.** No attempt was made to read another account's data, to access an endpoint we were not authorised to reach, or to escalate any privilege.
- **Every observation came from a normal, authenticated session** on an account created for this audit, whose owner authorised the test. The client was driven exactly as a user drives it — clicking, typing, scrolling, opening settings.
- **No credential material was read or recorded.** Storage was enumerated for **key names and value lengths only**. Auth-key bytes were never read, never logged, never copied off the machine. The lengths quoted below are `String.length` of the stored values, nothing more.
- **No traffic was decrypted.** MTProto frames were counted and sized from the browser's own WebSocket events. Their contents were never accessible to us and are not discussed as if they were.
- **We have no access to Telegram's server implementation.** Every server-side statement here is inference from client code, response headers, or Telegram's published documentation.

Where this document identifies a weakness, it does so to extract a **design lesson for taskrgram**. It contains no attack instructions, and the "what this means" sections are defensive.

**Confidence tags:** **Confirmed** (directly observed) · **Strong inference** (follows from confirmed facts) · **Possible** · **Unknown**.

---

## 1. The observed login flow

Every step below was walked in a real browser and screenshotted (`screenshots/01-…` through `screenshots/07-…`). **Confirmed** unless marked otherwise.

| Step | Observed behaviour |
|---|---|
| Entry state | **QR-code login is the default.** Copy: *"Log in to Telegram by QR Code / Open Telegram on your phone / Go to Settings > Devices > Add Device"*. Two secondary text buttons: `LOG IN BY PHONE NUMBER`, `LOG IN WITH PASSKEY`. See `screenshots/01-desktop-auth-qr-code-login-screen-initial.png`. |
| Phone form | `#sign-in-phone-code` (country selector, default `USA`), `#sign-in-phone-number` (prefilled `+1 `), and checkbox `#sign-in-keep-session`, labelled **"Keep me signed in", checked by default**. See `screenshots/02-desktop-auth-phone-number-entry-form.png`. |
| Country inference | Typing `+37063265395` rewrote the field to `+370 632 65395` and set the country to `Lithuania` **client-side, as you type, with no network request**. The country/prefix table ships in the bundle. See `screenshots/03-desktop-auth-phone-number-filled-lithuania-next-button.png`. |
| Code delivery | *"We've sent the code to the Telegram app on your other device."* Delivery was **in-app to an existing authorised session**, not SMS. See `screenshots/04-desktop-auth-code-verification-screen-awaiting-code.png`. |
| Resend | **There is no resend control on the code screen.** The only affordance is an edit (pencil) icon (`i.icon-edit`) that returns to the phone step; re-submitting the number re-sends. See `screenshots/05-desktop-auth-code-screen-no-resend-link-visible.png` and `screenshots/06-desktop-auth-code-screen-after-resend-request.png`. |
| Passkey | `auth.initPasskeyLogin` appears as a raw TL method string in the bundle and `LOG IN WITH PASSKEY` is present on **both** the QR and phone screens — WebAuthn login is shipped, not experimental. |
| Post-login | Straight to the chat list. **No page reload, no interstitial, URL unchanged** (`https://web.telegram.org/a/`). See `screenshots/07-desktop-auth-code-submitted-result.png`. |

Two details worth calling out as good product security, both **Confirmed**:

- **In-app delivery beats SMS.** Routing the login code to an already-authenticated device sidesteps SIM-swap and SS7 interception entirely for anyone who already has one session.
- **The login code is spoiler-masked in the chat-list preview** — an animated particle overlay covers the digits in the list, while the message bubble itself shows them in clear. That is a deliberate shoulder-surfing and screen-share mitigation, and it is the kind of small, cheap decision that signals a team thinking about real user context. Visible in `screenshots/08-desktop-main-layout-chat-list-and-service-chat-open.png`.

The **absent resend control** is the one usability-security friction point: a user whose code does not arrive must go back and re-submit, which is discoverable but not obvious. **Unknown** whether this is a deliberate rate-limit affordance or an omission; there is no client-side timer or cooldown UI to suggest either.

---

## 2. Session model

### 2.1 What is stored, measured directly

Enumerated in the live authenticated session (`document.cookie`, `localStorage`, `sessionStorage`, `indexedDB.databases()`, `caches.keys()`, `navigator.storage.estimate()`). **Confirmed.**

| Store | Contents |
|---|---|
| **Cookies** | **`document.cookie` is the empty string.** No session cookie, no CSRF token, no anything. |
| `sessionStorage` | **0 keys.** |
| `localStorage` | 8 keys — see table below |
| IndexedDB | `tt-data` v1, 322,094 B — the reduced global-state snapshot (see doc 03 §5) |
| Cache Storage | `tt-media`, `tt-media-avatars`, `tt-media-progressive`, `tt-lang-packs-v52`, `tt-assets` — 16,321,280 B |

localStorage, **names and value lengths only**:

| Key | Length (chars) | Role |
|---|---:|---|
| `dc1_auth_key` | 514 | MTProto auth key for DC 1 |
| `dc2_auth_key` | 514 | MTProto auth key for DC 2 |
| `dc4_auth_key` | 514 | MTProto auth key for DC 4 |
| `dc` | 1 | Current home datacentre id |
| `user_auth` | 28 | Legacy session descriptor — `{dcID, id, test}` per source |
| `account1` | 1,698 | Multi-account slot 1 session blob (`MULTIACCOUNT_MAX_SLOTS = 6`) |
| `tt-multitab_1` | 1 | Master-tab election flag |
| `tgme_sync` | 36 | Cross-client sync marker for the `t.me` `_websync_` beacon |

514 characters is consistent with a JSON-quoted 512-character hex string, i.e. a **256-byte (2048-bit) MTProto auth key** (**Strong inference** — we never read the values; the size and the source code both point the same way).

Source confirms the write path (`src/util/sessions.ts`): `storeSession()` writes one `dc${dcId}_auth_key` entry per datacentre plus a legacy `user_auth` / `dc` pair, all via `localStorage.setItem(...)` with `JSON.stringify` (**Confirmed in source**).

### 2.2 Three structural facts

1. **There is no ambient credential.** Authentication is not a token the browser attaches to requests; it is a symmetric key the application uses to *encrypt every message it sends*. Nothing about a request is authenticated by the browser on the application's behalf.
2. **Keys are held for several datacentres simultaneously.** Three were present after minutes of ordinary use (DC 1, 2, 4). The client is not migrating one credential between DCs — it holds a set, because MTProto auth keys are server-bound and *"encryption keys are not copied between DCs"* (Telegram's published documentation; the client's own `_switchDC` discards the key on migration, **Confirmed in source**).
3. **The credential is readable by any JavaScript running on the origin.** This is not a bug in the implementation; it is the only place a browser-based MTProto client can put a key it must use for symmetric encryption in userland code. There is no browser primitive that would let the page encrypt with a key it cannot read. (A non-extractable `CryptoKey` in IndexedDB would help against exfiltration-to-elsewhere but not against use-in-place by injected script; this client does not use one for the auth key.) **Strong inference.**

---

## 3. Why CSRF is structurally not a concept here — and why that is not the same as "safer"

### 3.1 The mechanism

Cross-site request forgery requires one thing: a credential the **browser** attaches automatically to a request the **attacker's page** causes. Cookies do that. `Authorization` headers set by your own JS do not.

In this client:

- there is **no cookie** (**Confirmed**: `document.cookie === ""`);
- the credential is a key used to *encrypt the message body*, so a request is authentic only if the sender could perform the encryption;
- an attacker page on another origin cannot read `localStorage` for `web.telegram.org` (same-origin policy), cannot make the browser attach the key, and cannot forge an MTProto frame without it;
- there is no HTTP endpoint that performs a state change on the basis of an ambient credential — the app's own `/progressive/` and `/download/` URLs are served by the **service worker**, never by the origin server.

So CSRF tokens, `SameSite`, double-submit patterns and origin checks are all simply **inapplicable**. **Confirmed** as an observation; **Strong inference** as a general statement about the model.

### 3.2 What replaces it — the actual threat model

Removing CSRF does not shrink the attack surface; it moves it. The properties a cookie-based design gets from the browser must now be provided by the application, and one of them cannot be provided at all.

| Threat | Cookie-based web app | This client |
|---|---|---|
| **XSS on the origin** | Session cookie marked `HttpOnly` is unreadable by injected script; attacker is limited to acting *within* the page, and the credential itself does not leave | Credential is in `localStorage`, readable by any script on the origin. Successful XSS means the **long-lived, multi-DC auth key set** is available to the injected code — full account takeover, persisting beyond the browser session, independent of the user's device |
| **Malicious/compromised browser extension** with storage permission for the origin | Can read non-`HttpOnly` cookies and DOM; `HttpOnly` session cookie remains out of reach | Extension storage access to the origin exposes the auth keys directly |
| **Filesystem access to the browser profile** (shared machine, stolen laptop without full-disk encryption, backup, forensic image) | Cookie jar is also on disk — comparable exposure, but session cookies are typically shorter-lived and server-revocable per request | The Local Storage LevelDB in the profile contains the keys in plaintext unless the optional passcode lock is enabled (§4) |
| **CSRF** | Real; needs `SameSite`, tokens, origin checks | **Not applicable** |
| **Session fixation** | Real; needs rotation on privilege change | Not applicable in the same form — the key is established by DH, not accepted from input |
| **Token theft in transit** | Mitigated by TLS + `Secure` flag | Mitigated by TLS *and* by MTProto's own encryption layer beneath it — this is genuinely stronger than a bearer token, which is replayable if ever observed |
| **Server-side revocation** | Delete server session record; next request fails | Supported and exposed to users (§5): terminating a session invalidates the auth key server-side |
| **`HttpOnly` protection** | Available | **Structurally impossible.** The application must be able to read the key to use it |

The honest summary: **this design trades one well-understood, well-tooled threat (CSRF) for a worse one (XSS becomes total and durable account compromise).** Telegram's mitigations for that trade are consistent and visible — zero third-party script origins, a CSP with no `'unsafe-inline'` for scripts and no `'unsafe-eval'`, and no ad or analytics tags anywhere (**Confirmed**, §6). They have made XSS hard rather than making it survivable, because survivable is not on the menu. An internal app has a better option available (§8).

---

## 4. The optional passcode lock, and its key derivation

Telegram Web A ships a local **Passcode Lock** (visible in the live UI under Privacy and Security — `screenshots/25-desktop-settings-privacy-and-security-panel.png`). When enabled, the persisted session and state are AES-GCM encrypted rather than stored in the clear, using a separate IndexedDB store `tt-passcode`; when locked, only a scrubbed state is written (`clearGlobalForLockScreen`).

The key derivation, quoted from the public source (`src/util/passcode.ts`) — **Confirmed in source**:

- key material is a **single SHA-256 pass** over the passcode;
- the salt is a **hardcoded constant string**, `'harder better faster stronger'`, identical for every installation;
- `IV_LENGTH = 12` for AES-GCM.

Stated factually as a design lesson: a single unsalted-in-practice hash provides **no work factor**. Password-based key derivation exists to make each guess expensive (PBKDF2 with a high iteration count, scrypt, Argon2id) and to make precomputation across users useless (a random per-installation salt). Neither property is present here. The consequence is that the passcode raises the bar against a casual shoulder-surfer or someone who opens the laptop lid, and does **not** meaningfully protect the encrypted blob against an adversary who has already obtained it and is willing to spend compute. Users are likely to choose a short numeric passcode, which compounds it.

**Confidence:** **Confirmed** as a property of the published source at HEAD `d915b1b9` (version 12.0.38). **Strong inference** that the deployed bundle behaves identically — the deployed version string, build timestamp and the 2,056 source paths recovered from public source maps all match that tree — but we did **not** enable passcode lock in the live session, so this is not Confirmed at runtime.

**Also worth being clear about what the feature is and is not.** Passcode lock is an at-rest protection for a shared or lost device. It does not protect a running session, it does not protect against script executing on the origin, and it is not end-to-end encryption of anything.

---

## 5. Multi-device session management, as observed

Settings → Active Sessions (`screenshots/29-desktop-settings-active-sessions-device-list.png`). Verbatim from the live UI — **Confirmed**:

```
THIS DEVICE
Chrome 140
Telegram Web 12.0.38 A, macOS
- United States

Chrome 151         Telegram Web 12.0.38 A, macOS          Klaipėda, Lithuania
Chrome 150         Telegram Web 12.0.38 A, macOS          Klaipėda, Lithuania
Samsung Galaxy S8  Telegram Android 12.9.1, Android 9 P (28)  Klaipėda, Lithuania
```

The Settings root row read `Active Sessions — Sessions, Automatically terminate` with a badge showing `4`, matching the four live sessions. **We did not open the auto-terminate sub-screen, so the configured interval is Unknown.** For comparison, an adjacent Privacy setting for account auto-deletion was observed set to *"If away for 18 months"*.

Three observations that matter for anyone building the equivalent screen:

1. **The client self-reports its own device metadata.** Our session was headless Chromium on Linux with the user agent spoofed to Chrome 140 / macOS — and Active Sessions faithfully reported **"Chrome 140 … macOS"** while the app's own `<body>` carried the class `is-linux`, derived from a different signal. **Confirmed**, and it is a clean demonstration that device labels in a session list are **client-asserted strings, not verified facts**. Geolocation ("United States", "Klaipėda, Lithuania") is the one field that is not client-controlled — it is derived server-side from the connecting IP (**Strong inference**).
2. **App family and version are part of the identity** — `Telegram Web 12.0.38 A` distinguishes Web A from Web K and from native clients, which is exactly what a user needs to recognise a session.
3. **Termination is real revocation.** Per Telegram's documented model, terminating a session invalidates the corresponding auth key server-side; the affected client's next RPC fails and it must re-authenticate. We did **not** terminate a session during this audit, so the runtime behaviour is **Unknown to us** and asserted only from documentation.

---

## 6. Delivery-layer posture: CSP, HSTS, source maps

### 6.1 The CSP, verbatim

Delivered as `<meta http-equiv="Content-Security-Policy">` in `index.html`. There is **no CSP response header**. **Confirmed.**

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

**It is enforced, and we have direct evidence.** During the accessibility measurement run, our own tooling tried to inject `axe-core` from a public CDN and the browser refused it:

```
Refused to load the script 'https://cdnjs.cloudflare.com/ajax/libs/axe-core/4.10.2/axe.min.js'
because it violates the following Content Security Policy directive:
"script-src 'self' 'wasm-unsafe-eval' https://t.me/_websync_ …"
```

(**Confirmed.** That was our audit tooling being correctly blocked, not an attack.)

### 6.2 Directive-by-directive trade-offs

| Directive | Reading |
|---|---|
| `script-src 'self' 'wasm-unsafe-eval' + three t.me sync endpoints` | **Strong.** No `'unsafe-inline'`, no `'unsafe-eval'`, no CDN. `'wasm-unsafe-eval'` is required by the four WASM modules — tlottie (400,534 B), fastText (1,122,181 B), and the Opus encoder/decoder. That relaxation permits WebAssembly compilation, not arbitrary JS `eval`; it is a much narrower grant than `'unsafe-eval'` and is the correct directive for this need. |
| `style-src 'self' 'unsafe-inline'` | **The one real weakening, and it is self-inflicted by the theming architecture.** The design system is JS-driven: 78 `[light, dark]` token tuples are interpolated with `colorjs.io` over a 200 ms transition and written into a runtime `<style>` element by `switchTheme.ts`. Injecting a `<style>` element at runtime requires `'unsafe-inline'` unless it carries a nonce. So dark mode costs the entire style-injection protection — a real cost, since `'unsafe-inline'` styles enable a family of data-exfiltration and UI-redress techniques. **Confirmed** mechanism, **Strong inference** that theming is the specific reason (the vite config comment and the token injector both point there). |
| `connect-src 'self' wss://*.web.telegram.org blob: http: https:` | **Effectively unrestricted.** The specific `wss://*.web.telegram.org` grant is decorative once `http:` and `https:` are present. Motivated by mini-apps/in-app browser, link previews, user-selectable map providers, and payment tokenisation to `api.stripe.com` / `tgb.smart-glocal.com` — a general-purpose client genuinely cannot enumerate the origins it will talk to. The trade-off is that CSP provides **no exfiltration constraint** here, which matters precisely because §3.2 established that XSS would be catastrophic. |
| `frame-src` with 11 wallet URL schemes | TON Connect wallet deep-linking (`tonkeeper-tc:`, `mytonwallet-tc:`, `safepal-tc:`, …). Notably, no TON JS SDK exists in the bundle graph — the schemes are for handing off to installed wallet apps. |
| `img-src … https://ss3.4sqi.net/img/categories_v2/` | Foursquare venue-category icons — the single non-Telegram image origin, for location messages. |
| `object-src 'none'`, `base-uri 'none'`, `form-action 'none'` | All three correct and tight. `form-action 'none'` is unusual and welcome. |
| **Missing: `frame-ancestors`** | A `<meta>`-delivered CSP **cannot** set `frame-ancestors` — the directive is ignored outside a header. Clickjacking protection is instead provided by the `x-frame-options: deny` **header**, present on every response observed. **Confirmed.** This is the correct compensating control, but it is worth naming as the reason a header-delivered CSP would be better. |
| **Missing: `report-uri` / `report-to`** | Also unavailable via `<meta>`. There is no CSP violation telemetry — consistent with the app's total absence of any reporting endpoint, but it means violations in the field are invisible. |

**Trade-off of `<meta>` delivery overall:** it ships with the HTML (no server configuration to drift), applies to everything the parser sees after it, and is genuinely simpler to keep in sync with the build — the CSP is generated in `vite.config.ts` and injected. The costs are concrete: no `frame-ancestors`, no reporting, and no protection for anything the browser processes before the meta tag is parsed.

### 6.3 HSTS

**No `strict-transport-security` header was observed on any response** from `web.telegram.org` (**Confirmed** across many requests to `/a/`, `/a/assets/*`, `/`, `/k/`).

The trade-off, stated fairly: HSTS protects against protocol-downgrade and cookie-injection attacks on a hostile network, and its absence is a real gap on a first, unprotected visit. The mitigations that do exist here are partial — TLS is used everywhere in practice, `x-frame-options: deny` is set, and MTProto adds its own encryption layer beneath TLS, so even a successful TLS strip does not expose message content to a network attacker (**Strong inference**). What it *would* expose is the static app delivery path, which is the more valuable target anyway: an attacker who can serve modified JavaScript on the origin owns the auth keys per §3.2. **Whether `web.telegram.org` is on the browser HSTS preload list was not checked and is Unknown** — that would change the practical severity substantially, and any conclusion here should wait on that check.

### 6.4 Public source maps

**453 of 461 JS chunks ship a working `.map`, all served with HTTP 200**, exposing **2,056 original source paths** including internal codenames (`src/lib/vibecalls`, `src/components/gili`). CSS maps are not served. Whether `sourcesContent` is populated was not checked (**Unknown**). **Confirmed** otherwise.

Trade-off: for a **GPLv3 project whose full source is already on GitHub**, publishing maps costs nothing and buys real debuggability — a user can file a meaningful bug report, and Telegram's own engineers get readable stack traces from a production build. This is a defensible, even good, decision *in context*. For a proprietary internal product the calculus inverts: maps hand an attacker your module graph, internal naming, and unminified logic for free, while your own team can get the same benefit by uploading maps privately to an error-tracking service instead of serving them.

### 6.5 Positives worth recording

- **Zero third-party origins at runtime.** 737 HTTP responses in the authenticated session: 629 `web.telegram.org`, 102 `blob:`, and 6 to the three `t.me`/`telegram.me`/`telegram.dog` sync aliases. **No analytics, no crash reporting, no CDN, no font provider, no ad tech, anywhere in the bundle graph.** (**Confirmed.**) This single property does more for the app's security posture than any header, because it means the XSS surface is code Telegram wrote.
- **No `api_id` / `api_hash` string literals in the bundle** — they are referenced only as instance properties, injected at build time (**Confirmed**; where they come from at runtime is **Unknown**).
- **No DC IP addresses hardcoded** — transport URLs are templated from server-supplied config (**Confirmed**).
- **`x-frame-options: deny` on every response** (**Confirmed**).
- **WebAuthn passkeys shipped** for login, alongside SRP-based two-step verification (**Confirmed**: `auth.initPasskeyLogin` in the bundle, `LOG IN WITH PASSKEY` in the UI, `SettingsPasskeys` and `util/browser/passkeys.ts` in source).
- **A feature gate before anything runs.** `compatTest.js` checks 14 APIs (including `crypto.subtle`, `BigInt`, `navigator.locks`, `BroadcastChannel`) and replaces the page with a static "unsupported browser" table rather than half-booting a client whose crypto primitives are missing (**Confirmed**).

---

## 7. Findings summary

| # | Finding | Confidence | Severity in Telegram's context | Severity if copied into taskrgram |
|---|---|---|---|---|
| 1 | MTProto auth keys stored in plaintext `localStorage`, one per DC | **Confirmed** (names + lengths; source confirms the write path) | High but largely unavoidable for a browser MTProto client | **Unacceptable** — a corporate app has better options (§8) |
| 2 | `HttpOnly`-equivalent protection is structurally impossible | **Strong inference** | Inherent to the protocol | Avoidable entirely |
| 3 | Passcode-lock KDF is a single SHA-256 with a hardcoded salt | **Confirmed in source**; **Strong inference** for the deployed build | Medium — an at-rest control that underdelivers | Would be a straightforward finding; use Argon2id/PBKDF2 or WebAuthn PRF |
| 4 | `style-src 'unsafe-inline'` required by runtime theme injection | **Confirmed** | Medium | Avoidable with a nonced or constructed stylesheet (§8) |
| 5 | `connect-src` effectively unrestricted (`http:` `https:`) | **Confirmed** | Medium — no exfiltration constraint | Avoidable; an internal app knows its egress origins |
| 6 | No HSTS header observed | **Confirmed** (preload-list status **Unknown**) | Medium, pending preload check | Trivially fixable |
| 7 | CSP delivered by `<meta>`, so no `frame-ancestors`, no reporting | **Confirmed** | Low — `x-frame-options: deny` compensates | Send a header instead |
| 8 | Complete production source maps served publicly | **Confirmed** | Low — the source is GPL and already public | Do not do this |
| 9 | `cache-control: max-age=3600` on content-hashed immutable assets | **Confirmed** | Performance, not security | Fix at launch |
| 10 | Device labels in Active Sessions are client-asserted (spoofed UA reported verbatim) | **Confirmed** | Low, but users read them as facts | Label unverified fields as such |
| 11 | No resend control on the code screen | **Confirmed** | Low, usability-security friction | Provide an explicit cooldown timer instead |

---

## 8. What this means for taskrgram

An internal corporate team-chat app has a **different threat model and far better tools**. Most of what constrains Telegram here does not constrain you.

### 8.1 Session and credential handling — do this differently

1. **Use `HttpOnly` cookies, or an in-memory access token refreshed via an `HttpOnly` cookie.** You control both ends and you speak HTTPS/JSON, so you can keep the long-lived credential out of JavaScript's reach entirely. Concretely: `Set-Cookie: session=…; HttpOnly; Secure; SameSite=Lax; Path=/`, short-lived access token in memory only, refresh token in an `HttpOnly; SameSite=Strict` cookie scoped to the refresh path. **Never `localStorage` for anything long-lived.** This single decision converts "XSS equals permanent account takeover" into "XSS equals damage for the duration of the page's lifetime", which is a categorically better failure mode.
2. **Accept that CSRF comes back, and handle it properly.** `SameSite=Lax` as the baseline, an `Origin`/`Sec-Fetch-Site` check on every state-changing route, and a token for anything that must accept cross-site navigation. This is well-trodden, boring, and cheap — much cheaper than defending a readable credential.
3. **Better still, do not build authentication at all.** Federate to the corporate IdP over OIDC and let it own MFA, device policy, conditional access, joiner/leaver, and revocation. Telegram had to build phone verification, SRP 2FA, passkeys, QR device-linking and a session manager because it has no IdP to delegate to. You do.
4. **Bind sessions to a device where it is cheap.** WebAuthn passkeys (which Telegram already ships) or a platform-bound key make credential theft substantially less useful. If you keep any client-side secret at all, hold it as a **non-extractable `CryptoKey`** in IndexedDB rather than a readable string, and be clear-eyed that this stops exfiltration, not use-in-place.
5. **Make revocation real and instant.** Server-side session records with a "terminate" action, propagated to the client's socket within seconds. Copy the Active Sessions screen as a product idea — it is genuinely good — but treat the browser/OS strings as **unverified client input**, and say so in the UI. Server-derived fields (IP, geo, first-seen, last-seen) are the trustworthy ones.
6. **If you build a local lock screen, derive keys properly.** Argon2id (or PBKDF2 with a modern iteration count) with a random per-installation salt, or skip password-derived keys and use WebAuthn PRF. And be honest in the UI about what it protects — at-rest data on a shared machine, not a live session.

### 8.2 Delivery layer — do this at launch, not later

7. **Send CSP as a response header**, not a `<meta>` tag. You get `frame-ancestors 'none'` and `report-to` for free, both of which Telegram cannot have.
8. **Avoid `style-src 'unsafe-inline'` by construction.** Telegram needs it because a JS theme engine injects a `<style>` element at runtime. You can get the same dark mode with a **static stylesheet keyed off `data-theme` on `<html>`** — the tokens are known at build time — and, if you truly need runtime palette generation (per-user accent colours), inject it through a **nonced `<style>` element** with the nonce minted per response. (A constructed stylesheet via `new CSSStyleSheet()` + `adoptedStyleSheets` is the other commonly cited route; verify it against your target browsers and your CSP before relying on it.)
9. **Enumerate `connect-src`.** You know your API origin, your object store, and your error tracker. List them. An internal app has no excuse for `https:`.
10. **HSTS with a long max-age, `includeSubDomains`, and preload.** One header, no downside for an internal domain.
11. **Do not publish source maps.** Upload them to your error tracker behind auth. Same debuggability, none of the disclosure.
12. **`cache-control: max-age=31536000, immutable` on content-hashed assets, and enable Brotli.** Telegram does neither; their warm reload is already excellent (0.4% of cold bytes) and would be better still.

### 8.3 What genuinely transfers

- **Zero third-party origins is achievable and enormously valuable.** No analytics tag, no font CDN, no session-replay vendor. It makes CSP writable, data governance trivial, and the XSS surface entirely your own code. For an internal tool, this is realistic in a way it rarely is for consumer products.
- **In-app / second-channel delivery of login codes**, in preference to SMS. If you federate to an IdP, you get this for free via push-based MFA.
- **Masking secrets in previews and notifications.** The spoiler overlay on the login code in the chat-list preview is a two-line idea that prevents a real class of leak — and an internal chat app will carry secrets in messages (credentials, customer data) where the same treatment applies to notification previews and screen sharing.
- **A user-visible session manager with one-click termination.**
- **Granular, discoverable privacy and notification controls** (`screenshots/25-desktop-settings-privacy-and-security-panel.png`, `screenshots/31-desktop-settings-notifications-sounds-badges.png`) — the shape of these screens is worth borrowing wholesale.
- **A hard feature gate before boot** (`compatTest.js`), so an unsupported browser gets a clear message instead of a half-initialised client.
- **The build-time CSP.** Generating the policy in the bundler config, next to the code that determines what the app loads, is why theirs is as tight as it is. Do the same — just emit it as a header.

### 8.4 What explicitly does not transfer

- The cookieless, key-in-`localStorage` session model. It is a consequence of speaking MTProto from a browser, not a security design to admire.
- The service-worker media server (doc 03 §4). You will have real URLs.
- Client-side heavy cryptography, and therefore `'wasm-unsafe-eval'` in your CSP.
- Multi-DC credential sets and DC migration handling.

---

## 9. What we could not verify

1. **Anything server-side.** Revocation semantics, rate limiting, code-attempt lockouts, flood-wait thresholds, and how auth keys are stored and invalidated at Telegram. All **Unknown**.
2. **Whether the deployed bundle is byte-equivalent to the public repo.** Version string, build timestamp and 2,056 recovered source paths all agree, which is strong — but it is not a reproducible-build proof. **Strong inference**, not Confirmed.
3. **The passcode-lock code path at runtime.** Read in source; never enabled during the session.
4. **Session termination behaviour.** Never exercised.
5. **The auto-terminate interval** and the contents of that sub-screen. Never opened.
6. **HSTS preload-list membership** for `web.telegram.org`. Not checked; it materially affects the severity of §6.3.
7. **`sourcesContent` in the published source maps** — we read the `sources` arrays, not whether full original source is embedded.
8. **Passkey login end to end.** The affordance and the TL method were observed; the WebAuthn ceremony was not performed.
9. **Two-step verification (SRP), account deletion, and multi-account slots.** Present in UI and source; not exercised.
10. **The `_websync_` cross-client sync mechanism** (`tgme_sync`, 36 chars; three cross-origin beacons to `t.me`, `telegram.me`, `telegram.dog`). We observed the beacons and the localStorage key; **we did not investigate what is synchronised, and no security conclusion should be drawn from this without more work.**
11. **Any credential value.** Deliberately never read. Every statement about key contents in this document is inferred from length and from published source.
