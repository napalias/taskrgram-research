# Telegram Web A — architecture and UI/UX audit

Reverse-engineering audit of **Telegram Web A** (`https://web.telegram.org/a/`), app
version **12.0.38**, deployment built 2026-08-11. Produced 2026-08-14 for the
**taskrgram** project — an internal team-chat app that should feel comparable, not be a
clone.

Desktop is treated as the primary target throughout, per the brief. Mobile and
responsive behaviour are documented but secondary.

---

## Start here

| If you want… | Read |
|---|---|
| The 10-minute version | [`01-executive-summary.md`](01-executive-summary.md) |
| What to actually build | [`11-recommendations-for-taskrgram.md`](11-recommendations-for-taskrgram.md) |
| Why they made these choices | [`06-design-decisions-and-rationale.md`](06-design-decisions-and-rationale.md) |
| The UI implementation in detail | [`05-ui-ux-and-design-system.md`](05-ui-ux-and-design-system.md) |

## All documents

| File | Contents |
|---|---|
| [`01-executive-summary.md`](01-executive-summary.md) | Overview, headline findings, confirmed-vs-inferred summary, the ten things that matter |
| [`02-tech-stack.md`](02-tech-stack.md) | Full stack with evidence strings — Teact, Vite/Rolldown, GramJS, TipTap, WASM modules, hosting, and an explicit "what is *not* there" |
| [`03-runtime-architecture.md`](03-runtime-architecture.md) | Process topology, MTProto transport, service worker as media server, state persistence, multi-tab, caching, pagination. Mermaid diagrams |
| [`04-features.md`](04-features.md) | Complete feature inventory by domain, each tagged by evidence tier, each with a relevance verdict for an internal team app |
| [`05-ui-ux-and-design-system.md`](05-ui-ux-and-design-system.md) | Layout system, measured breakpoints, design tokens light vs dark, component architecture, motion, states, patterns to steal and avoid |
| [`06-design-decisions-and-rationale.md`](06-design-decisions-and-rationale.md) | Thirteen decisions, each as what → evidence → why → cost → does it transfer. Includes myth-correction |
| [`07-performance.md`](07-performance.md) | Measured load timings, byte budget, CPU-bound analysis, caching, runtime behaviour under scroll. Methodology caveats up front |
| [`08-accessibility.md`](08-accessibility.md) | axe results, focus-indicator measurement, the i18n-key leak defect, structural tensions, remediation plan for taskrgram |
| [`09-auth-session-and-security.md`](09-auth-session-and-security.md) | Login flow, the cookieless session model, why CSRF is structurally absent and what replaces it, CSP, what to do differently |
| [`10-evidence-log.md`](10-evidence-log.md) | 186-row master evidence table: source, verbatim observation, interpretation, confidence. Plus claims we deliberately did not make |
| [`11-recommendations-for-taskrgram.md`](11-recommendations-for-taskrgram.md) | Licensing analysis first, then P0/P1/P2 build guidance, steal-this / do-not-copy lists, suggested stack, phased plan, open questions |
| [`screenshots/`](screenshots/) | 72 screenshots, each named for exactly what it shows |

---

## How this audit was done

**Scope.** Public pages plus one authenticated session on a throwaway account whose
owner authorised the test. Nothing was exploited. No authentication, rate limit, CAPTCHA
or access control was bypassed. No data was modified. Every observation came from
passive inspection and normal browser interaction.

**Method.** Four parallel evidence streams, cross-checked against each other:

1. **Live authenticated walkthrough** — headless Chromium driven via Playwright at a
   desktop viewport of 1600×1000 @2× DPR, with every network response, WebSocket frame
   and console message logged. Logged in by phone number with a relayed login code.
   72 screenshots.
2. **Asset forensics** — every JS, CSS and WASM file the deployment serves, downloaded
   and analysed: 461 JS files, exact byte sizes, library fingerprints, and the 453
   publicly-served source maps that expose 2,056 original source paths.
3. **Public source analysis** — the GPL-3.0 repository the client is built from
   (362,996 LOC of TS/TSX), read directly for architecture, and used to confirm or
   refute what the black-box evidence suggested.
4. **Rationale research** — the client's origin, the contest that produced it, and the
   authors' own statements, with every substantive claim cited to a URL.

**Instrumented performance runs** were done separately with their own methodology and
caveats, documented in `07-performance.md`.

## Reading the confidence tags

Every substantive claim carries one of four tags:

- **Confirmed** — directly visible in source, headers, requests, or runtime behaviour we
  observed ourselves.
- **Strong inference** — supported by multiple independent signals.
- **Possible** — plausible, not sufficiently verified.
- **Unknown** — cannot be determined from the access we had.

Where a widely-repeated story about this codebase turned out to be unsupported, it is
flagged as such rather than repeated. `06-design-decisions-and-rationale.md` has a
dedicated section on this.

## Known limits of this audit

- All latency numbers came from a headless browser on a datacenter IP, through a
  TLS-terminating relay that forced HTTP/1.1. They are **not** production-representative.
  Structural findings are unaffected.
- No real screen reader (NVDA / JAWS / VoiceOver) was used, so accessibility claims about
  assistive-technology behaviour are marked Unknown unless directly observed.
- We have no visibility into Telegram's server-side implementation, and make no claims
  about it.
- The test account was new and empty, so features requiring real conversation history,
  contacts, calls or payments were inspected in the UI rather than fully exercised.
