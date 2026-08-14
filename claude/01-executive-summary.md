# Executive summary

**Subject:** Telegram Web A — `https://web.telegram.org/a/`, v12.0.38, built 2026-08-11
**Audited:** 2026-08-14, public surface plus one authorised authenticated session
**For:** the taskrgram internal team-chat build, desktop-first

---

## The one-paragraph version

Telegram Web A is a **362,996-line TypeScript single-page application built on a custom
in-house React clone**, talking to Telegram's datacenters over **binary MTProto inside an
obfuscated WebSocket** — not over any REST or GraphQL API — with a **dedicated worker**
owning the protocol, a **service worker acting as a local HTTP media server** so native
`<video>` can stream files that have no URL, and **zero third-party services of any kind**:
no analytics, no CDN, no error reporting, no fonts, no cookies. It is an unusually
coherent piece of engineering whose every major decision traces back to a 2019–2020
Telegram coding contest that judged entries on *"speed, size of the apps and attention to
detail."* Most of what makes it impressive is worth stealing. Almost none of the specific
technology choices are worth copying.

---

## Ten findings that matter

**1. There is no REST API, and that changes everything.**
The client speaks MTProto — TL-serialised binary RPC over an authenticated session —
directly from the browser. We measured 1,003 WebSocket events across six datacenter
endpoints in one short session. **Confirmed.** For taskrgram this is the single biggest
divergence: you will have a normal HTTP API, which makes roughly a third of Telegram's
client complexity simply not your problem.

**2. Media comes down the protocol socket, not from a CDN.**
310 frames / 85 KB sent vs 660 frames / **10,026,931 bytes** received — a 117:1
asymmetry. Photos, video and stickers arrive as MTProto file chunks and are handed to the
page as `blob:` URLs. **Confirmed.** The service worker then synthesises HTTP 206 Range
responses at `/a/progressive/document<id>` so a native `<video>` element can seek and
stream. That last trick is genuinely clever and **is** worth stealing for auth-gated media.

**3. Zero third parties. Zero telemetry.**
Across 737 responses the only hosts contacted were `web.telegram.org` and three `t.me`
aliases. No Sentry, no Google Analytics, no Amplitude, no CDN, no font provider.
Self-hosted on their own AS62041 in Amsterdam, nginx 1.30.1. **Confirmed.** This is a
brand promise implemented as an engineering constraint — and it means they fly blind on
real-user performance data.

**4. The custom framework's origin is a contest rule, not a benchmark.**
Teact (~2,800 LOC) exists because the contest demanded near-zero dependencies. The
widely-repeated story that they benchmarked React and found it too slow **is not
supported by any source we could find** — the Teact-vs-React benchmark postdates the
framework by two years and publishes no numbers. What Teact actually buys is control over
*when* rendering happens: a RAF-driven, strictly ordered update pass that **defers itself
entirely while a heavy animation is running**. That idea transfers. The framework does not.

**5. Performance is a user-facing product surface.**
Settings contains an "Animations and Performance" screen with a three-stop slider
(*Power Saving / Nice and Fast / Lots of Stuff*) and a section headed, literally,
**"Resource-Intensive Processes"** — with ten individually-toggleable interface
animations. Notably, **Context Menu Blur** and **Message Blur** are broken out *separately
from the animations they accompany*, because backdrop-filter is the expensive part. This
is the clearest available evidence of how this team thinks. **Confirmed** —
`screenshots/27-desktop-settings-animations-performance-expanded-granular-toggles.png`.

**6. The message list is not virtualized, and that's correct.**
Measured under scroll on an 11M-subscriber channel: 29 DOM message nodes → 89 → capped at
89, with `scrollHeight` growing 20,317 → 61,266 px and JS heap going *down* 54.6 → 52.3 MB
as older slices released. **Confirmed.** It is a bounded sliding window over loaded
slices, not react-window-style recycling. You keep native scrollbars, native find-in-page
and variable-height rows; you give up unbounded history on screen. For a chat log that is
the right trade, and it is far less code.

**7. Three breakpoints, located exactly — and the middle one is the interesting one.**
1276/1275 px, 926/925 px, 601/600 px. Above 1276 all three columns scale proportionally.
Between 926 and 1275 the right column **stops scaling and becomes a fixed 408 px** — the
design switches from filling the window to protecting the reading column. Below 926 the
left column goes off-canvas; below 601 every column is exactly viewport-width.
**Confirmed by measurement**, not read from a stylesheet.

**8. The dark theme isn't in the CSS.**
566 CSS custom properties, cascade layers declared inline in the HTML to fix async-CSS
ordering — and then the dark palette turns out to be **78 `[light, dark]` tuples living in
JavaScript**, injected into a runtime `<style>` element. That is what forces
`style-src 'unsafe-inline'` into an otherwise tight CSP. It bought them per-peer accent
colours; it cost them a CSP directive and left the checked-in `:root` block as dead code
that has **silently drifted in 13 places**. **Confirmed.**

**9. Credentials sit in plaintext localStorage, and there are no cookies at all.**
`dc1_auth_key`, `dc2_auth_key`, `dc4_auth_key` — 514 chars each — plus `user_auth`, with
`document.cookie` completely empty. **Confirmed.** CSRF is structurally not a concept
here, which is not the same as being safer: any XSS or malicious extension is full account
takeover, and `HttpOnly` protection is impossible by construction. taskrgram is an
internal app with a normal HTTP backend and should simply not do this.

**10. It is CPU-bound, not byte-bound.**
Under a 4× CPU throttle, total blocking time went **72 ms → 753 ms (10.5×)** while FCP
moved only 10%. Warm reload transfers **0.4%** of cold bytes. Byte-shaving is the wrong
axis to optimise here — and it is probably the wrong axis for you too, on a corporate LAN.
**Confirmed** (with the methodology caveats in `07-performance.md`).

---

## What they got right, in one list

- Protocol/transport client isolated in a worker; only plain DTOs cross the boundary.
- A strict two-phase measure→mutate DOM discipline, **enforced at runtime in dev and by
  custom lint rules** — architecture encoded as tooling rather than as documentation.
- Expensive work gated on animation and idle state, so motion never competes with data.
- A bounded, migration-versioned persisted cache rather than "serialise the whole store".
- Warm-start caching that is close to ideal.
- Native platform primitives where they exist — the media viewer is a real
  `<dialog aria-modal="true">`, and Escape closes it for free.
- Performance and privacy exposed as product surfaces users can actually control.

## What they got wrong, or wrong for you

- Writing their own framework. Every advantage it delivers is reproducible on React,
  Preact or Solid at a fraction of the onboarding cost.
- Auth keys in plaintext localStorage; a passcode KDF that is a single SHA-256 with a
  hardcoded salt.
- 453 production source maps published, exposing 2,056 source paths.
- `cache-control: max-age=3600` on **content-hashed immutable assets** — the one header
  that undermines an otherwise excellent caching story.
- A 230 MB `dist/` committed to git.
- 373 of 461 JS chunks are single-language syntax-highlighting grammars — 81% of the
  files for a rounding error of the value.
- Accessibility: 4 axe violations including 3.31:1 contrast on both login CTAs, no
  landmarks at all, `user-scalable=no`, a focus indicator at **1.08:1** against the 3:1
  WCAG 2.2 requires, and a raw i18n key (`AccDescrPollVoteDown`) leaking into an
  `aria-label`. The upstream screen-reader issue has been open since April 2022.

---

## The bottom line for taskrgram

Read this codebase as **reference architecture, and do not paste from it** — it is
GPL-3.0-or-later, held by an individual author rather than Telegram. Internal web-only use
does not trigger distribution obligations (there is no AGPL network clause), but shipping
any desktop binary would. Get that confirmed by whoever owns legal; we are not lawyers.
Full analysis in `11-recommendations-for-taskrgram.md`.

Then build the boring version: a real framework, a normal HTTP + WebSocket API, HttpOnly
session cookies, and roughly **10–15% of Telegram's feature surface**. Spend the time you
save on the four things Telegram is actually good at — a responsive frame budget,
disciplined DOM writes, a bounded persisted cache, and treating motion as something with a
measurable cost that users are allowed to turn off.
