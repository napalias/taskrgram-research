# 08 — Accessibility

**Target:** `https://web.telegram.org/a/` — Telegram Web A `12.0.38 A`, measured 2026-08-14.
**Raw data:** `/home/claude/audit/perfout/a11y.json` (axe-core output, computed styles, tab-order trace); screenshots in the sibling `screenshots/` folder.

---

## 1. Scope and honesty statement — read this before quoting anything

**What was done:**

- **axe-core 4.13.0**, injected locally via `page.evaluate()` and run against the live unauthenticated login screen. (CDN injection was blocked by the site's own CSP — a correct, tight CSP and a positive security finding; axe was installed from npm and injected through CDP, which is not subject to CSP.)
- **Manual keyboard probing** in headless Chromium: 14 consecutive `Tab` presses with the focused element, its role, `tabindex`, computed `outline`/`box-shadow`/`background-color` and bounding rect recorded at each step.
- **Computed-style and pixel-diff measurement** of the focus indicator.
- **DOM inspection** of the authenticated UI (landmarks, `aria-label`s, dialog semantics, the `.Transition` mount pattern).
- **Source reading** of the public repository for the a11y utilities and conventions that exist.

**What was NOT done, and what that invalidates:**

- **No screen reader was used. No NVDA, no JAWS, no VoiceOver, no Narrator, no Orca.** Headless Chromium has no assistive-technology stack attached.
- Therefore **every claim about what a screen-reader user actually hears is Unknown unless it is a direct DOM observation.** "This element has `aria-label="X"`" is Confirmed. "A screen reader announces X" is not — announcement depends on the AT, its browser mode, its verbosity settings, and the accessibility tree Chrome computes, none of which we observed.
- **No testing with voice control, switch access, screen magnification, or Windows High Contrast Mode.**
- **No testing at 200 % or 400 % browser zoom** (WCAG 1.4.4 / 1.4.10 reflow) — the `user-scalable=no` finding in §2 is from the markup, not from a reflow test.
- **The authenticated audit was not exhaustive.** Most of the automated evidence is from the login screen, which has exactly **two interactive elements**. The main chat UI has hundreds. Absence of a finding below is **not** evidence of absence.

**Confidence scale:** **Confirmed** (directly observed) · **Strong inference** (measured inputs + short reasoning) · **Possible** · **Unknown**.

---

## 2. axe-core results — 4 violations

Login screen, axe-core 4.13.0. **Confirmed.**

| Impact | Rule | Nodes | WCAG | Detail |
|---|---|---:|---|---|
| **SERIOUS** | `color-contrast` | **2** | 2 AA, **1.4.3** | Both login CTAs: `#3390ec` on `#ffffff` = **3.31:1** at 16 px normal weight. Requires **4.5:1**. Affects `Log in by phone number` **and** `Log in with Passkey` — i.e. **every primary action on the screen fails contrast.** |
| MODERATE | `landmark-one-main` | 1 | best-practice | Document has **no `<main>` landmark**. |
| MODERATE | `region` | 3 | best-practice | The `<h1>`, the `<ol>` instruction list and the `.AnimatedSticker` block are **not contained by any landmark**. |
| MODERATE | `meta-viewport` | 1 | 2 AA, **1.4.4** | `<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1.0, user-scalable=no, shrink-to-fit=no, viewport-fit=cover">` — **pinch-zoom disabled**. See §6. |

Supporting manual inventory of the same screen (**Confirmed**):

| Check | Result |
|---|---|
| Landmarks (`main`/`nav`/`header`/`footer`/`aside`/`[role]`) | **0 — none at all** |
| Headings | **1**, correct: `<h1>Log in to Telegram by QR Code</h1>` |
| `<html lang>` | `"en"` — present and correct |
| `<title>` | `"Telegram"` — generic; does not describe the current state |
| `<input>` / `<a>` / `<img>` on this screen | 0 / 0 / 0 |
| `<canvas>` (Lottie plane animation) | 1 — **no `aria-label`, no `role`**; decorative but unlabelled |
| Elements with `[role="button"]` | **0** |
| Elements carrying `tabindex` | **0** (no positive tabindex anywhere) |

**Note on scale:** four violations across a two-control screen is a thin sample. The `color-contrast` failure is the one that matters, because `#3390ec` is `--color-primary` — the token used app-wide for links, active states and primary text. **Strong inference:** the same 3.31:1 failure recurs everywhere that token is used as *text on a light surface*, which the login screen shows is at least the whole auth flow. We did not enumerate the authenticated surface.

---

## 3. The focus indicator — a precise, fixable finding that axe did not catch

Computed styles on a login button, unfocused vs focused after `Tab`. **Confirmed.**

| Property | Unfocused | Focused |
|---|---|---|
| `outline` | `rgb(51,144,236) none 0px` | `rgb(51,144,236) none 0px` — **still `none`** |
| `box-shadow` | `none` | `none` |
| `border` | `0px none` | `0px none` |
| `background-color` | `rgba(0,0,0,0)` | **`rgba(74,149,214,0.08)`** |

There is **no outline and no box-shadow**. The *only* focus affordance is an **8 %-alpha blue background tint**. A pixel diff confirms it does render — **96.3 % of the button's pixels change** between `screenshots/a11y-button-unfocused.png` and `screenshots/a11y-button-focused.png` — but "many pixels changed" and "a perceivable indicator" are different claims.

Composited, `rgba(74,149,214,0.08)` over `#ffffff` gives **`#F0F7FC`**. Applying the WCAG relative-luminance formula:

**Contrast of the focus indicator against the unfocused surrounding white ≈ 1.08:1.**

The relevant success criteria:

- **WCAG 2.2 SC 2.4.13 Focus Appearance (AAA)** — the focus indicator area must have a contrast ratio of at least **3:1** between its focused and unfocused states. (Note: SC 2.4.11 is *Focus Not Obscured (Minimum)*, a different criterion; the AA case here rests on 1.4.11 Non-text Contrast.)
- **WCAG 2.1 SC 1.4.11 Non-text Contrast (AA)** — 3:1 for visual indicators of UI component state.
- **WCAG 2.0 SC 2.4.7 Focus Visible (A)** — a keyboard-operable interface must have a **visible** focus indicator mode.

**1.08:1 against a required 3:1.** At that ratio the indicator is effectively invisible on a typical monitor at typical brightness, and certainly so for a low-vision user. **Judgement:** this fails 1.4.11 (AA) and 2.4.13 (AAA) on the measurement, and is at serious risk under 2.4.7 — 2.4.7 is qualitative ("visible"), and we did not run a human perception test, so we state the measurement rather than declare a Level A failure outright.

Two further facts sharpen it:

- **axe did not flag this.** axe does not evaluate `:focus-visible` appearance at all. This is a finding *beyond* the four automated violations — and a demonstration that automated coverage is not a floor you can rest on.
- There are **zero `:focus` or `:focus-visible` CSS rules matching these buttons** in any stylesheet (queried across all `document.styleSheets`). The tint comes from a generic `.Button` background rule that happens to apply on focus. **Strong inference: the focus state was never designed. It is an accident of a hover-ish background rule.**

See `screenshots/a11y-focus-after-tabs.png` for the state after the tab sequence.

**Fix, concretely:** a `:focus-visible { outline: 2px solid <token>; outline-offset: 2px; }` rule with a token chosen to hit ≥3:1 against *both* the light and dark surface it can appear on. That is a few lines, applies globally through the `.Button` primitive, and would resolve the most severe keyboard finding in this audit.

---

## 4. What is done right — stated as plainly as the failures

These are **Confirmed** and are genuinely good. A team copying this app should copy these too.

- **Real `<button>` elements.** Both CTAs are `<button type="button" class="Button auth-button default primary text">`. There are **0** `div[role="button"]` on the screen. Native semantics means keyboard activation, correct role exposure, and Enter/Space handling for free — no re-implementation, nothing to get wrong.
- **A clean, minimal, trap-free tab order.** 14 consecutive `Tab` presses produced exactly:

  ```
   1. <button> LOG IN BY PHONE NUMBER   rect (620,574) 360x48
   2. <button> LOG IN WITH PASSKEY      rect (620,622) 360x48
   3. <body>   (focus left the document → browser chrome)
   4-14. cycles 1 → 2 → body, repeatedly
  ```

  Two focusable elements, visited top-to-bottom in DOM order matching visual order, **no keyboard trap, no positive `tabindex` anywhere**. Focus correctly escapes to browser chrome and comes back. This is the boring correct answer and plenty of production apps get it wrong.
- **A native `<dialog aria-modal="true">` for the media viewer.** Observed in the authenticated UI: `<dialog open id="MediaViewer" aria-modal="true" class="shown open">`. Using the platform dialog gets top-layer stacking, a real backdrop, focus containment and Escape-to-close from the browser — but **only when it is opened via `showModal()`**, which we did not establish. **Escape closing it in one press is Confirmed**; the remaining guarantees are **Strong inference**, conditional on `showModal()`. See `screenshots/44-desktop-media-viewer-native-dialog-element-open.png`. **Noted inconsistency:** the rest of the app's overlays go through a `#portals` div instead, so the media viewer is the exception rather than the rule. Defensible, but it means the guarantees the `<dialog>` gives you for free are not uniform across the app.
- **Meaningful `aria-label`s on most icon-only buttons** in the authenticated UI — e.g. `Go back`, `Add an attachment`, `Continue To Group Info`. Icon buttons having accessible names at all is the baseline many apps miss; this one mostly clears it.
- **`<html lang="en">` present and correct**, and a single correct `<h1>` per screen.
- **A11y utilities exist and are used as a kit, not ad hoc.** The source carries `trapFocus.ts`, `focusEditableElement.ts`, `focusNoScroll.ts`, `captureEscKeyListener.ts`, `scrollLock.ts`, `useVirtualBackdrop.ts`, `useKeyboardListNavigation.ts`, `useSelectWithEnter.ts`, `useInputFocusOnOpen.ts`, with `role` and `dir` explicitly allowlisted in the renderer's attribute handling. ARIA/roles are present across the primitive kit (`Button`, `Menu`, `MenuItem`, `Modal`, `ListItem`, `TabList`, `InputText`, `Draggable`).
- **The localisation convention reserves an accessibility prefix** — `Acc` — for a11y-only strings, i.e. accessible names are treated as translatable first-class content rather than hardcoded English. (Which makes §5 all the more unfortunate.)
- **A user-controlled reduce-motion surface.** The Animations and Performance panel offers a 3-stop slider plus ten individually-toggleable interface animations, including the two `backdrop-filter` effects broken out separately (`screenshots/26-desktop-settings-animations-and-performance-toggles.png`, `screenshots/27-desktop-settings-animations-performance-expanded-granular-toggles.png`). **Gap, though:** the source shows **no `prefers-reduced-motion` media query usage** — the app has its own setting instead of honouring the OS-level one. A user who has already told their operating system they get motion sickness must tell this app again. For taskrgram, do both: honour `prefers-reduced-motion` as the default, and let the setting override it.

---

## 5. The concrete defect in the authenticated UI: a raw i18n key as an accessible name

Observed live in the chat header. **Confirmed:**

```html
aria-label="AccDescrPollVoteDown"
```

`AccDescrPollVoteDown` is a **localisation key**, not a string. The lookup returned the key instead of a translation, and the key went straight into the accessible name.

**What this costs:** visually, nothing — the control is an icon and looks fine. To a screen-reader user, the control is announced as the literal token "AccDescrPollVoteDown" (**Strong inference** — the DOM attribute is Confirmed; the announcement is inferred, since we ran no AT). It is unusable and unguessable.

**The class of bug, and why it matters more than one button:**

- A missing translation in *visible* text is caught by anyone who looks at the screen, in any language, immediately.
- A missing translation in an **accessible name is invisible to everyone who can see**. It survives design review, QA, screenshot diffing, and dogfooding. The only people it reaches are the people least likely to be in the room.
- So the failure rate of `aria-label` translations is structurally higher than for visible strings — the feedback loop that fixes visible strings does not exist for these.

Note that this project has **stronger-than-usual** i18n machinery: types are generated from the strings file (`npm run lang:ts` → `LangKey`), so a *missing or renamed* key is a compile error rather than a runtime `???`. The bug still shipped — meaning it is a runtime lookup path (a key present in the type system but absent from the loaded language pack, or a fallback that returns the key), not a typo. **Type generation is necessary but not sufficient.**

**Prevention, for taskrgram — cheap and mechanical:**

1. **A CI check that no accessible name matches the key pattern.** Crawl the app's main states in a headless browser, collect every element's computed accessible name, and fail the build if any matches your key regex — e.g. `^[A-Z][A-Za-z0-9]*$` with no whitespace, or a prefix convention like `^Acc[A-Z]`. This is ~30 lines on top of the a11y test run you should have anyway.
2. **Make the i18n fallback loud, not silent.** In dev and staging, a missing key should render `⚠MISSING:Key⚠` — visible in both the visual and accessible layers — rather than returning the key verbatim, which is exactly what makes this bug quiet.
3. **Lint accessible-name props at the source.** Fail on `aria-label={"literal"}` and on any `aria-label` whose value is not produced by the translation function.
4. **Include accessible names in translation-completeness reporting**, so a language pack at "98 % translated" cannot be 98 % of visible strings and 40 % of `Acc*` keys.

---

## 6. `user-scalable=no` — zoom and reflow

**Confirmed** in the shipped markup:

```html
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1.0, user-scalable=no, shrink-to-fit=no, viewport-fit=cover">
```

`user-scalable=no` together with `maximum-scale=1.0` instructs the browser to disable pinch-zoom. axe flags it under **WCAG 2 AA SC 1.4.4 Resize Text**.

Honest qualifications:

- **Modern iOS Safari and Android Chrome ignore `user-scalable=no`** and permit pinch-zoom anyway, precisely because of this accessibility problem. So the *practical* impact on current mobile browsers is smaller than the rule's severity suggests. **Possible**, not Confirmed — we did not test on a real mobile browser.
- It is still wrong to ship. It signals intent to block zoom, it is honoured by some webviews and older engines, and it is a one-token fix with no upside. There is no legitimate reason for it in a desktop-first app.
- **Related but distinct — and untested:** SC 1.4.10 Reflow requires content to work at 320 px-equivalent width without two-dimensional scrolling, and SC 1.4.4 requires text to scale to 200 % without loss of content. We measured **layout** down to 320 px (no horizontal scrollbar at any tested width — `screenshots/responsive-480px-login-screen.png` and the wider variants) but we **did not test browser zoom at 200 % or 400 %**, which is the actual test. Reflow compliance is **Unknown**.
- One measured layout risk: the login QR block is **fixed at 280 × 280 px and never scales**. At a 320 px viewport that leaves 20 px of side margin; below ~300 px it would begin to clip. Not reachable in the tested range, and there is no fluid fallback. **Strong inference**, extrapolated from measured box metrics.

Also note the document sets `translate="no"` / `class="notranslate"` plus `<meta name="google" content="notranslate">`. Machine translation is actively suppressed. That is a defensible product decision for a messenger, but for an internal tool it removes an assistive strategy some users rely on — worth an explicit decision rather than an inherited default.

---

## 7. The structural tension: `.Transition` keeps inactive screens mounted

**Confirmed observation.** In the authenticated UI, `#LeftColumn` is a `.Transition` container that holds the previous and next screens in the DOM **simultaneously**, so it can slide between them. The practical consequence during this audit: **every `querySelector` for a left-column control matched 3+ elements, only one of which was visible.** `aria-label="Go back"` matched multiple nodes. Every selector in the audit harness had to be qualified with `:visible`.

State is expressed as **class names on containers** (`right-column-not-shown`, `right-column-not-open`, `ui-ready`, `mask-image-disabled`) rather than by conditional mounting — CSS does the animation work against a DOM that stays put.

**Why this is good for performance:** transitions become cheap and cancellable; there is no mount/unmount cost mid-animation; the Teact scheduler can defer updates during the transition without a half-built tree on screen. It is a coherent, deliberate design and it is a large part of why the app feels smooth.

**The likely accessibility consequence — Strong inference, NOT Confirmed:**

- Multiple copies of the same control, with the same accessible name, exist in the accessibility tree at once.
- Chrome prunes elements that are `display: none` or `visibility: hidden` from the tree, and honours `aria-hidden` and the `inert` attribute. If off-screen screens are hidden by **transform/opacity/off-canvas positioning** rather than by one of those mechanisms, **they remain in the tree** — visible to AT, focusable by Tab, and reachable by screen-reader virtual-cursor navigation.
- The observed left column at narrow widths sits at `x = -424` (off-canvas by transform, per the measured geometry), which is exactly the pattern that stays in the tree.
- **Expected symptoms if so:** a screen-reader user hears "Go back, button" three times; Tab lands on invisible controls; the virtual cursor wanders into a screen that is not on screen. This is a classic and well-documented failure mode for slide-transition patterns.

**We mark this Strong inference and not Confirmed for two specific reasons:** (1) we ran **no assistive technology**, so we did not observe the announcement or the virtual-cursor behaviour; (2) we did not enumerate, per hidden screen, whether `aria-hidden`, `inert`, `visibility: hidden` or `display: none` is applied — the app may well handle it correctly on some or all paths. **Determining this is a 30-minute test with a real screen reader and it should be the first thing anyone verifies.**

**Corroborating context (Confirmed):** upstream GitHub issue **#128, "Improve screen reader accessibility," was opened 2022-04-05 and remains open with no assignee, no label, and no milestone** — over four years. The reporter is an actual screen-reader user, and notably calls `telegram-tt` "by far the most accessible Telegram web app" while listing: chat-list navigation problems (initials and name read as separate lines, with only the name clickable), confusing message/reply presentation, and message senders not being announced. There is **no published accessibility statement for either Telegram web client**, and bug reports are routed to Telegram's Suggestions Platform rather than GitHub — which plausibly explains why the issue sits unattended rather than indicating deliberate refusal.

**The honest framing of the trade-off:** the architecture that produces the animation quality — a custom framework, a bounded sliding-window DOM, signals that update state *without re-rendering components*, and screens that stay mounted so transitions are cheap — is **precisely** the architecture that makes a stable accessibility tree and correct ARIA live regions hard. Signals bypassing render is a performance feature and an a11y hazard in the same mechanism: DOM text updated outside the render pass will not announce unless a live region is explicitly wired. **This is not a case of a team that forgot about accessibility. It is a case of a design whose central performance idea has an accessibility cost that has to be paid deliberately, and largely has not been.**

---

## 8. Prioritised remediation — for taskrgram, not for Telegram

Telegram is a consumer product built by a very small team optimising for animation quality on a hostile network. **taskrgram is an internal corporate tool, and the calculus is different in ways that are worth being blunt about.**

### 8.1 Why an internal tool often has the *harder* obligation

- **You cannot choose your users.** A consumer app's blind users can switch to a competitor; your blind employee cannot switch employers to read the standup thread. If the tool is how work is coordinated, an inaccessible tool is an inaccessible workplace.
- **Employment law reaches internal tools directly.** In most jurisdictions, an employer's obligation to provide reasonable accommodation applies to the tools it requires employees to use — this is typically a stronger and more directly enforceable duty than public-accommodation web rules, and it does not depend on the tool being public-facing.
- **Procurement will ask.** If taskrgram is ever sold, licensed, or deployed at a customer, in the EU (EN 301 549 / European Accessibility Act) or to any US federal or federally-funded body (Section 508), you will be asked for a **VPAT / Accessibility Conformance Report**. Retrofitting the evidence for one is dramatically more expensive than accumulating it as you build. "It's internal" is not a durable exemption — internal tools get demoed, spun out, and acquired.
- **Small user base, concentrated harm.** Consumer scale means a 0.5 % failure rate is a large absolute number and shows up in support volume. At 200 employees, one affected person is 0.5 % and may generate no ticket at all — they will just quietly struggle. **Low user counts hide accessibility problems; they do not reduce them.**

*(Not legal advice. Flag jurisdictional specifics for counsel before making commitments.)*

### 8.2 The list

Ordered by (harm prevented × confidence) ÷ effort.

| # | Item | Evidence from this audit | Effort |
|---|---|---|---|
| 1 | **Design a real `:focus-visible` indicator** — 2 px outline + offset, verified ≥3:1 against every surface it can land on, applied at the `Button`/`ListItem` primitive so it is universal | Measured **1.08:1** vs the required 3:1; **zero** `:focus` rules matched the CTAs | Hours |
| 2 | **Fix the primary-action contrast at the token level.** Split "brand fill" from "brand text on light surface" — one token cannot be both | **3.31:1** measured on both CTAs; `#3390ec` is `--color-primary`, used app-wide | Hours (tokens) + a design pass |
| 3 | **Audit every hidden-but-mounted screen for `inert` / `aria-hidden`.** If you adopt the `.Transition` pattern, this is the price of admission — and it must be enforced by the transition component itself, not by each caller | Duplicate controls in the DOM confirmed; AT consequence Strong inference | Days |
| 4 | **Add landmarks: one `<main>`, plus `<nav>` for the chat list and `<aside>` for the detail panel.** Every screen, including auth | **0 landmarks** on the login screen; `landmark-one-main` + `region` violations | Hours |
| 5 | **Remove `user-scalable=no` and `maximum-scale=1.0`** | Confirmed in markup; WCAG 1.4.4 | Minutes |
| 6 | **CI check: no accessible name may match the i18n key pattern** | `aria-label="AccDescrPollVoteDown"` shipped to production | Hours |
| 7 | **Wire ARIA live regions for incoming messages, send confirmation, and connection state** — `aria-live="polite"` for arrivals, `assertive` only for errors | Not testable here; **Unknown**. But signals-bypassing-render (§7) means DOM updated outside the render pass **will not announce** unless explicitly wired | Days, and needs AT testing to tune |
| 8 | **Make `<title>` state-descriptive** — `"Log in — taskrgram"`, `"#eng-platform (3 unread) — taskrgram"`. Screen-reader users navigate windows and tabs by title | `<title>` is the constant string `"Telegram"` | Hours |
| 9 | **Honour `prefers-reduced-motion` as the default**, with the in-app setting as an override | Source shows a rich animation-settings panel but **no `prefers-reduced-motion` usage** | Hours |
| 10 | **Label or hide decorative canvases/animations** — `aria-hidden="true"` on purely decorative ones | 1 unlabelled `<canvas>` with no `role` on the login screen | Minutes |
| 11 | **Use native `<dialog aria-modal="true">` for *all* modals**, not just the media viewer | Media viewer does it right; `#portals` overlays do not | Days |
| 12 | **Publish an internal accessibility statement and keep a VPAT current from v1** | Neither Telegram client has one | Ongoing |

### 8.3 Testing plan

Four layers, cheapest first. None of them substitutes for the one below it.

**1. axe in CI — every PR, blocking.**

- `@axe-core/playwright` against a fixed set of app states: login, chat list, open thread, composer focused, modal open, settings, empty states, error states. Fail the build on any `serious` or `critical` violation; report `moderate` without blocking at first, then ratchet.
- Add the accessible-name-vs-i18n-key assertion (§5) to the same run.
- **Expect axe to catch roughly a third of real issues.** It found 4 here and completely missed the 1.08:1 focus indicator, which is the worse defect. Treat a green axe run as "no obvious regressions", never as "accessible".

**2. Keyboard-only pass — every release, ~30 minutes, no special tooling.**

- Unplug the mouse. Complete each core journey end to end: log in, find a channel, read a thread, reply, attach a file, open and close a modal, open and close the media viewer, search, change a setting.
- Check at each step: is focus visible at all times; does Tab order match visual order; is there any trap; does Escape close what is open; does focus return to the invoking control when an overlay closes; does focus move to new content when a screen changes.
- This is the highest-yield-per-minute test in the whole plan and it needs no expertise.

**3. One real screen-reader pass per release — the layer this audit could not do.**

- Minimum: **NVDA + Firefox on Windows** (the most common real-world pairing) and **VoiceOver + Safari on macOS**. Both are free/built-in.
- Same journeys as the keyboard pass, plus: are duplicate controls announced (§7); is an incoming message announced without stealing focus; are the chat list and message list navigable by heading/landmark/list shortcuts; is the composer's state (formatting, attachments) conveyed.
- Budget a half day. **Then budget for the finding that the first pass will be humbling** — this is normal, and the second pass is much faster.
- Where you have the option, **pay a screen-reader user to do this**, once per quarter if not per release. The issue-#128 reporter's feedback is more precise and more useful than any sighted developer's simulation of it, and it took them one session to produce.

**4. Contrast enforced at the design-token level, not at review time.**

- Every colour pair in the token system gets a computed contrast ratio in a test that runs in CI. Adding a token or changing a value that drops a text pair below 4.5:1 (or a UI/focus pair below 3:1) fails the build.
- Run it against **both** themes. This app's primary palette is injected at runtime from JS — 78 `[light, dark]` tuples in a runtime `<style>` element — so a static CSS scrape finds almost no dark-mode variables and a naive contrast linter would silently check only half the product. **Whatever your theming mechanism, make sure your contrast check sees the same values the browser does.**
- Name tokens by role, not by appearance: `--text-on-surface-brand` versus `--fill-brand` cannot be confused into a 3.31:1 button the way one `--color-primary` can.

---

## 9. Confidence summary

| Claim | Confidence |
|---|---|
| 4 axe violations, incl. `color-contrast` 3.31:1 on both login CTAs | **Confirmed** |
| No `<main>`, zero landmarks on the login screen | **Confirmed** |
| `user-scalable=no` present in the shipped viewport meta | **Confirmed** |
| Focus indicator is an 8 %-alpha tint at ≈1.08:1; no outline, no box-shadow, no matching `:focus` rules | **Confirmed** (measured + derived via the WCAG luminance formula) |
| That indicator fails SC 1.4.11 (AA) / 2.4.13 (AAA) | **Strong inference** (measurement is Confirmed; SC application is judgement) |
| Native `<button>`s, clean 2-stop trap-free tab order, no positive tabindex | **Confirmed** |
| `<dialog aria-modal="true">` for the media viewer; Escape closes it | **Confirmed** (Escape verified interactively) |
| `aria-label="AccDescrPollVoteDown"` present in the authenticated DOM | **Confirmed** |
| What a screen reader announces for that control | **Strong inference** — no AT was used |
| `.Transition` leaves duplicate controls mounted in the DOM | **Confirmed** |
| That those duplicates reach the accessibility tree and confuse AT | **Strong inference** — not verified with AT, and per-screen hiding mechanism not enumerated |
| Upstream issue #128 open since 2022-04-05, unassigned | **Confirmed** |
| Modern mobile browsers ignore `user-scalable=no` | **Possible** — not tested here |
| Reflow behaviour at 200 % / 400 % browser zoom | **Unknown** — not tested |
| Accessibility of the authenticated UI beyond the specific items above | **Unknown** — not systematically audited |
| Behaviour with voice control, switch access, magnification, or High Contrast Mode | **Unknown** — not tested |
