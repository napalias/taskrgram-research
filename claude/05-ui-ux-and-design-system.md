# 05 — UI/UX and Design System: Telegram Web A (v12.0.38)

Desktop-first analysis of the interface layer: layout, tokens, components, motion, states. Every number below is either read out of the live DOM at a known viewport, resolved from `getComputedStyle` in a running authenticated session, or extracted from the shipped CSS/JS. Nothing is estimated from screenshots.

Confidence tags: **Confirmed** / **Strong inference** / **Possible** / **Unknown**.

---

## 1. Layout system

### 1.1 The three-column model

```
#root
├── noscript
├── #portals                     ← portal target for menus / modals / tooltips
├── script
├── div.Transition.full-height.is-auth
│   └── #Main.opacity-transition.fast.right-column-not-shown.right-column-not-open
│       ├── #LeftColumn.Transition      ← chat list · folders · search · settings
│       ├── #MiddleColumn.mask-image-disabled.ui-ready   ← header · messages · composer
│       └── #RightColumn                ← profile · media · management
└── svg                          ← global SVG defs (gradients, filters, clip paths)
```

**Confirmed** by direct DOM read. See `screenshots/08-desktop-main-layout-chat-list-and-service-chat-open.png` (light) and `screenshots/15-desktop-main-layout-dark-theme-chat-list-and-message-list.png` (dark).

Three observations that matter more than the tree itself:

1. `#root` **is `<body>`**. Telegram renders directly into `document.body`, not into a nested `<div id="app">`. **Confirmed** — `render(U(Dr,{}),document.getElementById('root'))` in the entry chunk, with `<body id="root">` in the HTML shell.
2. `#portals` is **pre-created in the static HTML**, before any JS runs. Menus, modals and tooltips are rendered there, outside the column tree.
3. State is expressed as **class names on containers**, not by conditional mounting: `right-column-not-shown`, `right-column-not-open`, `ui-ready`, `mask-image-disabled`, `opacity-transition fast`. The layout engine writes classes; CSS does the animation.

### 1.2 The measured geometry table

Column boxes read from `getBoundingClientRect()` at each viewport width with a channel open. Format is `width@x`, so a negative `x` means off-canvas.

| Viewport | LeftColumn | MiddleColumn | RightColumn | Mode |
|---|---|---|---|---|
| 1920 | 480@16 | 1424@496 | 464@1920 | 3-col, proportional |
| 1600 | 400@16 | 1184@416 | 384@1600 | 3-col, proportional |
| 1440 | 360@16 | 1064@376 | 344@1440 | 3-col, proportional |
| 1300 | 325@16 | 959@341 | 309@1300 | 3-col, proportional |
| **1276** | **319@16** | 941@335 | 303@1276 | last proportional width |
| **1275** | **421@16** | 838@437 | **408@1275** | first fixed-right width |
| 1100 | 363@16 | 721@379 | 408@1100 | fixed right column |
| 1024 | 338@16 | 670@354 | 408@1024 | fixed right column |
| **926** | **306@16** | 604@322 | 408@926 | last with visible left column |
| **925** | **424@-424** | **925@0** | 408@925 | left column goes off-canvas |
| 768 | 424@-424 | 768@0 | 408@768 | overlay mode |
| **601** | 424@-424 | 601@0 | 408@601 | last "tablet" width |
| **600** | **600@-600** | 600@0 | **600@660** | mobile: every column = viewport width |
| 375 | 375@-375 | 375@0 | 375@413 | mobile |

**Three breakpoints, located exactly: 1276/1275, 926/925, 601/600.** **Confirmed** by bisection.

Screenshots at every measured width: `screenshots/responsive-1920px-channel-view-desktop-layout.png`, `responsive-1600px-…`, `responsive-1440px-…`, `responsive-1280px-…`, `responsive-1100px-…`, `responsive-1024px-…`, `responsive-950px-…`, `screenshots/responsive-925px-channel-view-desktop-layout.png`, `responsive-900px-…`, `responsive-768px-…`, `responsive-600px-…`, `responsive-480px-…`, `responsive-375px-channel-view-desktop-layout.png`.

### 1.3 What actually changes at each breakpoint, and why

Deriving the rules from the numbers (**Confirmed** arithmetic on measured values):

| Range | Left column | Middle column | Right column |
|---|---|---|---|
| **≥ 1276 px** | `25vw` exactly (480/1920, 400/1600, 360/1440, 325/1300, 319/1276 all = 25.00%) | `100vw − left − 16` | `25vw − 1rem` — proportional |
| **926–1275 px** | `33vw` (421/1275, 363/1100, 338/1024, 306/926 all = 33.0%) | `100vw − left − 16` | **fixed 408 px** |
| **601–925 px** | off-canvas drawer, fixed **424 px** at `x = −424` | **full viewport** | fixed 408 px, overlays |
| **≤ 600 px** | off-canvas, **= viewport width** | = viewport width | = viewport width |

The mechanism is visible in the token space. **Confirmed:**

```css
#LeftColumn { --left-column-min-width: 16rem;                       /* 256 px floor */
              --left-column-max-width: calc(100vw - env(safe-area-inset-left)); }
.folders-sidebar-visible #LeftColumn { --left-column-max-width: calc(33vw - var(--folders-sidebar-width)); }
```

with `--folders-sidebar-width: 5rem` (80 px). So the "33vw" band is literally the folder-rail band: when the vertical folder sidebar is present the left column is allotted a third of the viewport *minus* the 80 px rail.

The right column's two lives are equally explicit. The `:root` token is `--right-column-width: 26.5rem` (424 px) with `--right-column-padding: 1rem` and `--right-column-content-width: calc(var(--right-column-width) - var(--right-column-padding))` — i.e. **408 px of content inside a 424 px panel**. But at ≥1276 px the live-resolved value is `--right-column-width: 25vw`, `--right-column-content-width: calc(25vw - 1rem)`. **Confirmed** — the live token dump at a 1600 px viewport returns `25vw`, not `26.5rem`.

That is the whole story of the second breakpoint in one token: **above 1276 px the right column is a percentage; below it, the `:root` fixed value takes over.**

### 1.4 What the fixed 408 px says about protecting the reading column

Follow the middle column across 1276 → 1275:

| Viewport | Middle column | Δ |
|---|---|---|
| 1276 | 941 px | — |
| 1275 | 838 px | **−103 px for a 1 px viewport change** |

At 1276 the layout is still trying to be proportional and the middle column is comfortable. At 1275 the design gives up on proportionality, hands the left column a full third and the right column a hard 408 px, and the reading column absorbs the entire difference in one step.

The constraint that explains this is `--messages-container-width: 47.5rem` = **760 px**. **Strong inference:** the message content itself is capped at 760 px regardless of column width, so between roughly 1275 and 1600 px the middle column is *wider than its content* and the extra space is inert margin. Once you accept that, giving the right column a fixed 408 px costs the reader nothing until the middle column drops below ~760 px — which happens at about 1200 px viewport width, right where the design has already switched to overlay behaviour for the right column.

The design is not protecting the *column*; it is protecting the **measure** — the 760 px line length. That is the correct thing to protect, and it is why a proportional-everything layout (the naive choice) reads worse at 2560 px than this one does.

Compare `screenshots/responsive-1280px-channel-view-desktop-layout.png` against `screenshots/responsive-1100px-channel-view-desktop-layout.png` to see the step.

### 1.5 The 16 px gutter and the floating-card chrome

**Confirmed:** the left column's `x` is **16 at every desktop width** — 1920, 1600, 1440, 1300, 1276, 1275, 1100, 1024, 926. It is never edge-to-edge.

That 16 px is `--middle-panel-inline-padding: 1rem`, and it is the visual signature of the whole app. The left column, the chat header, the composer, the audio player and the right column are **rounded, inset surfaces floating over a wallpaper** — not flush panels meeting at hairlines. The radii that make this work:

| Token | Value | Used for |
|---|---|---|
| `--border-radius-island` | `1.5rem` (24 px) | floating islands (audio player, panes) |
| `--border-radius-pane` | `1.5rem` (24 px) | header/footer panes |
| `--composer-pill-radius` | `1.5rem` (24 px) | the composer surface |
| `--border-radius-modal` | `2rem` (32 px) | modals |

with matched elevation:

| Token | Value |
|---|---|
| `--shadow-island` | `0 1px 4px 0 #0000000d` |
| `--shadow-pane` | `0 1px 5px -1px #00000036` |
| `--shadow-footer` | `0 1px 8px 1px #0000001f` |
| `--shadow-chat-list-panel` | `0 1px 8px 0 #0000001a` |

And the inner-radius discipline is tokenised rather than eyeballed — four separate components declare:

```css
--pane-content-radius: calc(var(--border-radius-pane) - .25rem);
```

i.e. content nested inside a 24 px pane gets a 20 px radius, computed, so the concentric curves stay parallel. **Confirmed** in `.AudioPlayer.full-width-player`, `.ChatReportPane`, `.GroupCallTopPane`, `.SVm1JjIT`, `.SetQDn1R`, `.dJU9jcVO`, `.o9pW489x`.

There is also a per-surface breakout convention:

```css
--surface-inline-padding: 1rem 1rem;
--surface-breakout-margin: -1rem -1rem;
```

so a child can cancel its parent's padding to bleed edge-to-edge (used by media inside bubbles: `.message-content.has-solid-background → --surface-breakout-margin: -.5rem`). **Confirmed.**

See `screenshots/41-desktop-dark-theme-channel-view-three-column-layout.png` — the floating-card effect is far more legible in dark mode, where the inset surfaces sit on `#0f0f0f` while the surfaces themselves are `#212121`.

### 1.6 Where the wallpaper comes from

The middle column's backdrop is a user-chosen wallpaper with its own controls (`Upload image`, `Set a color`, `Blurred`, `Pattern Intensity` = 75, "Colors, Gradients & Patterns") — `screenshots/13-desktop-settings-chat-wallpaper-picker-gallery.png`. Implementation is tokenised: `_patternBackground.module.scss`, `--pattern-color` (which reaction pills over media pick up), a tiling turbulence texture

```css
.ZnTs-RfL { --background-url: url(./turbulence_3x-Tnfp8p6G.png); --background-size: 256px; }
```

and animated gradient backgrounds (`gradientBackground.ts`, `useGradientBackground`). **Confirmed.**

**Relevance:** the wallpaper is what makes the floating-card chrome legible. If you copy the inset/rounded surfaces onto a flat single-colour background you get 16 px of unexplained dead space. Either commit to a textured/gradient backdrop or go flush.

---

## 2. Design tokens — the real system

### 2.1 Architecture

| Fact | Value | Confidence |
|---|---|---|
| Distinct custom properties defined anywhere in the CSS | **566** | Confirmed |
| Defined on `:root` (250 of them in one block in `index-CTuaTAxZ.css`) | **252** | Confirmed |
| Component-scoped selectors that define local variables | **577** | Confirmed |
| Declarations under `.theme-dark` / `html.theme-dark` in CSS | **150** — and these are *local* overrides (code-block syntax colours, individual modals), **not** the palette | Confirmed |
| Overrides on `html.theme-dark` for the base palette | **2** (`--lightningcss-light`, `--lightningcss-dark`) | Confirmed |
| Live-resolved variables on `:root` at runtime | **352** light / **354** dark | Confirmed |

Three tiers, and they are genuinely distinct:

1. **Global tokens** on `:root` — colour, radii, z-index, motion, typography, layout constants.
2. **Component tokens** — public contract of a component, e.g. `.Button` exposes `--button-text-color`, `--button-background-color`, `--button-active-background-color`, `--button-no-ripple-background-color`, `--ripple-color`. A *variant* is nothing but a class that rewrites those five.
3. **Private tokens** — prefixed `--_`, by convention internal to one component and not part of its contract: `--_size`, `--_font-size`, `--_color`, `--_background-color`, `--_height`, `--_progress`, `--_marquee-delay`, `--_scroller-right-gap`, `--_accent-color-rgb`, `--_color-outline`, `--_selected-color`, `--_dialog-padding`, `--_status-size`, `--_overflow-shift`, `--_y-shift`, `--_speed`, `--_duration-shift`, `--_shift-x`, `--_shift-y`. **Confirmed** across ~40 selectors.

The `--_` prefix is a real, cheap innovation. CSS custom properties have no visibility model; a leading underscore restores one by convention, and it means a component can be refactored internally without breaking a consumer that only ever touched the un-prefixed names. **Adopt this.**

### 2.2 Cascade layers, declared inline in the HTML

The **only** inline `<style>` in `index.html` is this, and it exists to solve one specific problem:

```css
@layer reset, variables, ui, components;
@layer ui {
  @layer tablist, spinner, button, input, layout;
}
```

**Confirmed.** The app ships 8 stylesheets on the login path and 25 in total, loaded asynchronously as route/feature chunks resolve. Without layers, *whichever stylesheet arrives last wins ties* — so the same UI could render differently depending on network ordering. Declaring the layer order up front, in the document, before a single stylesheet loads, fixes the precedence permanently. Chunked CSS then just declares `@layer ui { … }` (11 occurrences), `@layer variables`, `@layer reset`, `@layer component`.

This is the single most sophisticated CSS-architecture decision in the product and it is four lines long. **Strong inference on motive**, Confirmed on mechanism.

The sub-layering of `ui` into `tablist, spinner, button, input, layout` is a specificity ladder without specificity: `layout` beats `input` beats `button` beats `spinner` beats `tablist`, regardless of selector weight or file order.

There is one related artefact: `revert-layer` is used in anger — `.CalendarModal .day-button.selected { --button-text-color: revert-layer; }` — to pop a token back to the previous layer's value rather than restating it. **Confirmed.**

### 2.3 The finding: the dark palette is not in CSS at all

**Confirmed.** A static scrape of every stylesheet finds essentially no dark-mode palette — 2 `html.theme-dark` overrides on `:root`, both meaningless LightningCSS artefacts. The real palette lives in JavaScript.

`assets/initial-CskBLhZ6.js` holds a map of **78 `[light, dark]` tuples**:

```js
var Rr={"--color-primary":[`#3390EC`,`#8774E1`],
        "--color-primary-opacity":[`#50A2E91E`,`#8378DB1E`],
        "--color-background":[`#FFFFFF`,`#212121`],
        "--color-background-secondary":[`#F4F4F5`,`#0F0F0F`],
        "--color-background-own":[`#EEFFDE`,`#766AC8`], … }
```

and `assets/Checkbox-Cxf2-dWf.js` owns a `<style>` element it **rewrites on every theme change**:

```js
var Ht=document.createElement(`style`); document.head.appendChild(Ht);
function Kt(){ Ht.textContent = `
    html { ${qt(zt)} }
    html.theme-light { ${qt(Bt)} }
    html.theme-dark { ${qt(Vt)} }
  ` }
function qt(e){ return Array.from(e.entries()).map(([e,t])=>`--${e}: ${t};`).join(` `) }
```

Consequences, all **Confirmed**:

1. **This is why the CSP contains `style-src 'self' 'unsafe-inline'`.** Everything else about this app's CSP is exemplary — `script-src 'self' 'wasm-unsafe-eval'` with no `unsafe-inline` and no `unsafe-eval`, `object-src 'none'`, `base-uri 'none'`, `form-action 'none'`, and it correctly blocked our attempt to inject axe-core from a CDN. The one relaxation is bought entirely by the theming mechanism.
2. It enables **runtime-generated, server-driven palettes** that a static stylesheet cannot express — specifically per-peer accent colours: `Ut('color-peer-${e}', …)`, `Ut('color-peer-bg-${e}', …)`, `Ut('color-peer-gradient-${e}', …)`, with a 7-colour fallback baked in.
3. Theme switching is **animated**, not flipped: `src/util/switchTheme.ts` interpolates each variable with `colorjs.io` over `DURATION_MS = 200`, while adding `no-animations` to `<html>` for `ENABLE_ANIMATION_DELAY_MS = 500` to suppress every *other* transition, and swaps `<meta name="theme-color">` between `#212121` and `#fff`. Four variables are interpolated in RGB space rather than the default: `--color-text`, `--color-primary-shade`, `--color-text-secondary`, `--color-accent-own`.

### 2.4 …and the checked-in CSS palette has silently rotted

Because the `:root` colour block in the CSS is **never used** — the runtime `<style>` overrides it on first paint — nothing keeps it in sync. Comparing the static `:root` values against the values actually resolved in a live light-theme session finds **13 tokens that disagree**. **Confirmed.**

| Token | Value in the shipped CSS | Value actually applied |
|---|---|---|
| `--color-own-links` | `#ffffff` | **`#3390ec`** |
| `--color-list-icon` | `#ffffff` | **`#abafb1`** |
| `--color-background-compact-menu-hover` | `#000000b2` (70% black) | **`#00000011`** (6.7% black) |
| `--color-primary-shade` | `#2f84d9` | `#4a95d6` |
| `--color-text-meta-colored` | `#4fae4e` | `#4dcd5e` |
| `--color-accent-own` | `#4fae4e` | `#45af54` |
| `--color-message-reaction-own` | `#cef0ba` | `#c6eab2` |
| `--color-reply-own-hover` | `#dbf5cd` | `#d9f5ce` |
| `--color-reply-own-active` | `#c8ecbb` | `#c5ecbe` |
| `--color-reply-active` | `#e8e9ea` | `#e8e9e9` |
| `--color-background-own-selected` | `#d4ffab` | `#d0ffac` |
| `--color-text-secondary-apple` | `#8a8a90` | `#8e8e92` |
| `--color-forum-hover-unread-topic-hover` | `#dcdcdc` | `#e2e2e2` |

Three of these are not drift, they are *inversions* — `#ffffff` vs `#3390ec` is white-vs-blue, and a 70%-black hover overlay vs a 6.7%-black one is a 10× difference. If the JS injector ever failed to run, or a token were removed from the tuple map, the app would fall back to values that have not been correct for an unknown number of releases.

**Strong inference:** these 13 are the accumulated cost of having two sources of truth for one palette, with only one of them ever executed. There is a related symptom in the token *names*: three near-identical forum tokens coexist — `--color-forum-unread-topic-hover` (themed, used), `--color-forum-hover-unread-topic-hover` (themed, used) and `--color-forum-hover-unread-topic` (**theme-invariant `#e9e9e9`, no consumer found**, so in dark mode it would render as a light grey if anything ever read it).

**Lesson for taskrgram:** one source of truth. If the palette must be dynamic, generate the CSS from the same data at build time *and* inject it at runtime from that same data — never hand-maintain a second copy in a stylesheet you have already decided to override.

### 2.5 Light vs dark: what actually differs

Diffing 352 live-resolved light values against 354 dark ones:

| Category | Count | Share |
|---|---:|---:|
| **Identical in both themes** | **210** | 60% |
| **Different** | **142** | 40% |
| Present in dark only | 2 | — (`--lightningcss-light`, `--lightningcss-dark`, both empty strings — a LightningCSS `light-dark()` lowering artefact) |

**Every single one of the 142 differing tokens is a colour.** **Confirmed** by inspection of the full diff: 79 core UI colours, 56 peer-identity colours (`--color-peer-{7..20}` × `{base, bg, bg-active, gradient}`), 7 topic colours.

Not one spacing value, radius, z-index, duration, easing curve, font stack, font weight or layout constant changes between light and dark. The theme axis is **purely chromatic**, and the other 210 tokens are pure structure.

That is a stronger statement than it looks. It means a third theme costs exactly 142 colour decisions and zero layout work; it means a designer can be handed a 142-row spreadsheet; and it means any structural regression is theme-independent by construction. If you take one thing from this section into taskrgram, take this discipline: **make the theme axis colour-only, and enforce it.**

A second, subtler point: the 7 fallback peer colours (`--color-peer-0` … `--color-peer-6` = `#D45246 #F68136 #6C61DF #46BA43 #5CAFFA #408ACF #D95574`) and their `1a`/`2b` alpha variants are **identical in both themes**, while the server-supplied peers 7–20 have distinct dark values. **Strong inference:** the local fallback set was chosen to be theme-agnostic so it never needs a dark pass, while the server can ship theme-aware palettes for the richer set.

---

## 3. The token tables

All values are **live-resolved from a running session** unless marked otherwise. Light and dark side by side. Colours normalised to hex; `@a` denotes alpha.

### 3.1 Surfaces

| Token | Light | Dark |
|---|---|---|
| `--color-background` | `#ffffff` | `#212121` |
| `--color-background-secondary` | `#f4f4f5` | `#0f0f0f` |
| `--color-background-secondary-accent` | `#e4e4e5` | `#191919` |
| `--color-background-sidebar` | `#e4e4e5` | `#0f0f0f` |
| `--color-background-selected` | `#f4f4f5` | `#2c2c2c` |
| `--color-background-compact-menu` | `#ffffff @0.73` | `#212121 @0.87` |
| `--color-background-compact-menu-reactions` | `#ffffff @0.92` | `#212121 @0.87` |
| `--color-background-menu-separator` | `#000000 @0.10` | `#ffffff @0.10` |
| `--color-web-app-browser` | `#ffffff @0.73` | `#030303 @0.56` |
| `--color-toast-background` | `#202020 @0.8` | `#000000 @0.8` |

Note the **inversion of figure and ground**: in light mode the app surface is `#ffffff` and the *secondary* surface is darker (`#f4f4f5`); in dark mode the app surface is `#212121` and the secondary surface is *darker still* (`#0f0f0f`). Dark mode is not "light mode with the lightness flipped" — the surface hierarchy keeps the same direction (content surface lighter than chrome) rather than mirroring it. That is why `screenshots/41-desktop-dark-theme-channel-view-three-column-layout.png` reads as depth rather than as a photographic negative.

### 3.2 Text and lines

| Token | Light | Dark |
|---|---|---|
| `--color-text` | `#000000` | `#ffffff` |
| `--color-text-rgb` | `0,0,0` | `255,255,255` |
| `--color-text-secondary` | `#707579` | `#aaaaaa` |
| `--color-text-secondary-rgb` | `112,117,121` | `170,170,170` |
| `--color-icon-secondary` | `#707579` | `#aaaaaa` |
| `--color-text-meta` | `#686c72` | `#686c72` (identical) |
| `--color-text-lighter` | `#2e3939` | `#2e3939` (identical) |
| `--color-placeholders` | `#a2acb4` | `#a2acb4` (identical) |
| `--color-borders` | `#dadce0` | `#303030` |
| `--color-borders-input` | `#dadce0` | `#5b5b5a` |
| `--color-dividers` | `#c8c6cc` | `#3b3b3d` |
| `--color-gray` | `#c4c9cc` | `#717579` |
| `--color-list-icon` | `#abafb1` | `#a2a2a2` |
| `--color-chat-username` | `#3c7eb0` | `#e9eef4` |

`--color-text-meta` and `--color-placeholders` being **theme-invariant** is a latent contrast risk: `#686c72` on `#212121` is a low-contrast pairing for timestamps in dark mode. **Strong inference** (not measured in situ, but the arithmetic is unambiguous).

### 3.3 Accent and interactive

| Token | Light | Dark |
|---|---|---|
| `--color-primary` / `--accent-color` | `#3390ec` | **`#8774e1`** |
| `--color-primary-shade` | `#4a95d6` | `#7b71c6` |
| `--color-primary-shade-darker` | `#2b79c6` | `#2b79c6` (identical) |
| `--color-primary-shade-rgb` | `74,149,214` | `123,113,198` |
| `--color-primary-tint` | `#3390ec @0.10` | `#8774e1 @0.10` |
| `--color-primary-opacity` | `#50a2e9 @0.12` | `#8378db @0.12` |
| `--color-primary-opacity-hover` | `#50a2e9 @0.25` | `#8378db @0.25` |
| `--color-links` | `#3390ec` | `#8774e1` |
| `--color-active` | `#00c73e` (green) | **`#8774e1`** (purple) |
| `--color-chat-active` | `#3390ec` | `#766ac8` |
| `--color-chat-hover` / `--color-item-hover` | `#f4f4f5` | `#2c2c2c` |
| `--color-item-active` | `#ededed` | `#292929` |
| `--color-hover-overlay` | `#000000 @0.024` | `#ffffff @0.024` |
| `--color-interactive-element-hover` | `rgba(112,117,121,.08)` | `rgba(170,170,170,.08)` |

The accent hue **changes family between themes** — blue `#3390ec` becomes purple `#8774e1`. This is unusual and deliberate: `#3390ec` on `#212121` would be both harsh and low-contrast, so dark mode gets a desaturated violet instead. **Confirmed.** The knock-on is that `--color-active`, which is *green* `#00c73e` in light mode, becomes the same purple in dark — i.e. "active" stops being semantically green when the theme flips.

### 3.4 Message bubbles

| Token | Light | Dark |
|---|---|---|
| `--color-background-own` | `#eeffde` (pale green) | **`#766ac8`** (violet) |
| `--color-background-own-apple` | `#dcf8c5` | `#766ac8` |
| `--color-background-own-selected` | `#d0ffac` | `#6549d4` |
| `--color-accent-own` | `#45af54` | `#ffffff` |
| `--color-own-links` | `#3390ec` | `#ffffff` |
| `--color-message-meta-own` | `#4fae4e` | `#ffffff @0.53` |
| `--color-code-own` | `#3c7940` | `#ffffff` |
| `--color-reply-hover` | `#f4f4f4` | `#272727` |
| `--color-reply-own-hover` | `#d9f5ce` | `#8775da` |
| `--color-reply-own-active` | `#c5ecbe` | `#917dea` |

The own-bubble treatment is the clearest demonstration of the JS palette earning its keep: light mode uses a *tinted surface with coloured text on it* (green bubble, green metadata, blue links), dark mode uses a *saturated surface with white text on it* (violet bubble, white metadata, white links). Those are two different design strategies, not two values of one strategy, and they are expressed as pure token swaps.

Bubble geometry, **theme-invariant**:

| Token | Value | iOS/macOS override |
|---|---|---|
| `--border-radius-messages` | `.9375rem` (15 px) | `1rem` (16 px) |
| `--border-radius-messages-small` | `.375rem` (6 px) | `.5rem` (8 px) |
| `--message-text-size` | `16px` | — |
| `--message-meta-height` | `20px` | — |
| `--messages-container-width` | `47.5rem` (760 px) | — |

Grouping is handled entirely by corner-radius tokens rather than by extra elements: `.Message.own.first-in-group:not(.last-in-group)` sets `--border-bottom-right-radius: var(--border-radius-messages-small)`, and so on for eight combinations of `first-in-group` / `last-in-group` / `own`, plus a `has-appendix` case that zeroes the corner where the bubble tail attaches. **Confirmed.** Fifteen tiny rules replace what is usually a mess of conditional class logic in the component.

### 3.5 Semantic and identity colours

| Token | Light | Dark |
|---|---|---|
| `--color-error` | `#e53935` | `#e53935` |
| `--color-error-shade` | `#d33431` | `#d33431` |
| `--color-warning` | `#fb8c00` | `#fb8c00` |
| `--color-success` / `--color-green` | `#00c73e` | `#00c73e` |
| `--color-green-darker` | `#00a734` | `#00a734` |
| `--color-yellow` | `#fdd764` | `#fdd764` |
| `--color-marked` | `#fdd764 @0.50` | same |
| `--color-orange` | `#d08a31` | same |
| `--color-heart` | `#ff3c32` | same |
| `--color-stars` | `#ffaa00` | same |
| `--color-archive` / `--color-deleted-account` | `#9eaab5` | same |
| `--color-negative-progress` | `#ce4c47` | same |
| `--color-selection-highlight` | `#3993fb` | same |
| `--color-skeleton-background` | `#212121 @0.15` | same |
| `--color-skeleton-foreground` | `#e8e8e8 @0.20` | same |

Semantic colours are **entirely theme-invariant**. Error red is `#e53935` on white and on `#212121` alike. **Strong inference:** this is a deliberate simplification that trades some dark-mode contrast quality for the guarantee that "error is always this red". It is defensible for a small palette; it would not survive a WCAG audit at AA for small text on `#212121` (`#e53935` on `#212121` ≈ 4.0:1).

Gift rarity tiers (each with a matching 15%-alpha background):

| Tier | Colour | Background |
|---|---|---|
| Uncommon | `#40a920` | `#40a920 @0.15` |
| Rare | `#11aabe` | `#11aabe @0.15` |
| Epic | `#955cdb` | `#955cdb @0.15` |
| Legendary | `#bf7600` | `#bf7600 @0.15` |

Topic colours (forum topics), the one identity family that *is* themed:

| Token | Light | Dark |
|---|---|---|
| `--color-topic-blue` | `#2f7772` | `#6ff9f0` |
| `--color-topic-green` | `#44774a` | `#8eee98` |
| `--color-topic-red` | `#eb6858` | `#fb6f5f` |
| `--color-topic-rose` | `#9b576b` | `#ff93b2` |
| `--color-topic-violet` | `#8b5a96` | `#cb86db` |
| `--color-topic-yellow` | `#7f693b` | `#ffd67e` |
| `--color-topic-grey` | `#6c6c6c` | `#999999` |

Note the pattern: dark-theme topic colours are **much lighter and more saturated** than their light-theme counterparts — these are foreground colours on a dark surface, not surface colours. Correct, and the opposite of what a naive "darken everything" transform would produce.

Peer (sender) colours are a 4-token family per peer — `--color-peer-N`, `--color-peer-bg-N` (alpha `1a` = 10%), `--color-peer-bg-active-N` (alpha `2b` = 17%), `--color-peer-gradient-N` — for N = 0…25. The gradients are striped identity markers:

```css
--color-peer-gradient-14: repeating-linear-gradient(-45deg,
  #247bed 0px, #247bed 5px, #f04856 5px, #f04856 10px, #ffffff 10px, #ffffff 15px);
```

i.e. up to three-colour 5-px diagonal stripes used as the sender bar beside a quoted message. **Confirmed.**

### 3.6 Gradients

| Token | Value |
|---|---|
| `--premium-gradient` (`:root`) | `linear-gradient(84.4deg, #6c93ff -4.85%, #976fff 51.72%, #df69d1 110.7%)` |
| `--premium-gradient` (component scope, 6 places) | `linear-gradient(88.39deg, #6c93ff -2.56%, #976fff 51.27%, #df69d1 107.39%)` |
| `--premium-feature-background` | `linear-gradient(65.85deg, #6c93ff -0.24%, #976fff 53.99%, #df69d1 110.53%)` |
| `--stars-gradient` | `linear-gradient(90deg, #fa0 0%, #ffcd3a 100%)` |

The premium gradient is **redeclared at slightly different angles and stops in at least seven places** (`.Avatar`, `.Button`, `.StickerButton`, `._6pGLaJ5m`, `.oEaPoig5`, `.ua-K-wV3`, `:root`). **Strong inference:** copy-paste drift, not intent. This is exactly the failure mode a token system is supposed to prevent, and it happened anyway because the value was inlined per component instead of referenced.

### 3.7 Spacing and layout constants

There is **no `--space-1..8` scale**. Spacing is expressed as named, purposeful constants. **Confirmed** — this is a real design choice, not an omission.

| Token | Value (px @16) | Purpose |
|---|---|---|
| `--middle-panel-inline-padding` | `1rem` (16) | the gutter |
| `--middle-header-height` | `3rem` (48) | chat header |
| `--middle-header-gap` | `.5rem` (8) | gap under header |
| `--column-header-height` | `4rem` (64) | left/right column headers |
| `--chat-list-inline-inset` | `.5rem` (8) | chat row inset |
| `--right-column-width` | `26.5rem` (424) / `25vw` ≥1276 | |
| `--right-column-padding` | `1rem` (16) | |
| `--right-column-content-width` | `calc(width − 1rem)` = 408 | |
| `--messages-container-width` | `47.5rem` (**760**) | the measure |
| `--folders-sidebar-width` | `5rem` (80) | vertical folder rail |
| `--window-controls-width` | `0rem` browser / `5rem` under Tauri | native traffic lights |
| `--symbol-menu-width` | `24rem` (384) | emoji/sticker panel |
| `--symbol-menu-height` | `22.375rem` (358) | |
| `--symbol-menu-footer-height` | `3rem` (48) | |
| `--story-ribbon-height` | `5.5rem` (88) | |
| `--call-header-height` | `2rem` (32) | |
| `--left-column-min-width` | `16rem` (256) | |
| `--vh` | `1vh`, recomputed in JS | mobile viewport-unit fix |
| `--safe-area-{top,right,bottom,left}` | `env(safe-area-inset-*)` | |

Composer geometry is its own dense sub-system, and it is worth reading as a case study in "derive, don't hardcode":

| Token | Value |
|---|---|
| `--composer-height` | `3rem` (48) |
| `--composer-main-button-width/height` | `2.5rem` (40) |
| `--composer-pill-radius` | `1.5rem` (24) |
| `--composer-control-radius` | `1.25rem` (20) |
| `--composer-embed-radius` | `.5rem` (8) |
| `--composer-pill-padding` | `.25rem` (4) |
| `--composer-row-gap` | `.25rem` (4) |
| `--composer-top-gap` | `.5rem` (8) |
| `--composer-bottom-gap` | `1rem` (16) desktop / `.5rem` mobile |
| `--composer-icon-size` | `1.5rem` (24) |
| `--composer-icon-button-padding` | `.5rem` (8) |
| `--composer-text-size` | **`16px`** |
| `--composer-input-line-height` | `1.375` (`1.3125` for `#message-input-text`) |
| `--composer-min-bottom-reserve` | `calc(3rem + 1rem)` |
| `--composer-input-grow-duration` | `.1s` |
| `--composer-embedded-message-duration` | `.15s` |

and the derived values that make it hold together:

```css
--composer-input-padding-block:
  calc((var(--base-height,3rem) - var(--composer-text-size,1rem) * var(--composer-input-line-height)) / 2);

--_scroller-controls-width: calc(var(--action-button-size) + var(--composer-main-button-width));
--_scroller-gaps-width:     calc(2 * var(--composer-row-gap) + var(--composer-pill-padding) + .25rem);
--_scroller-right-gap:      calc(var(--_scroller-controls-width) + var(--_scroller-gaps-width));
```

The vertical padding that centres the text in the composer is **computed from the height, the text size and the line height**, so changing the font size does not require re-tuning the padding. Likewise the right-hand gap that keeps text from sliding under the send button is computed from the buttons' own widths. **Confirmed.** This is why the composer survives the Message Font Size slider without breaking.

The message list reserves space for the composer the same way:

```css
--message-list-min-reserve:    var(--composer-min-bottom-reserve);
--message-list-composer-gap:   var(--composer-top-gap);
--message-list-base-reserve:   calc(var(--message-list-min-reserve) + var(--message-list-composer-gap));
--message-list-bottom-inset:   var(--message-list-base-reserve);
--message-list-top-inset:      calc(header + player + panes);
```

with `.MessageList.no-footer { --message-list-bottom-inset: 0px }` for the channel case where a Join CTA replaces the composer. **Confirmed.**

### 3.8 Radii

| Token | Value | px |
|---|---|---|
| `--border-radius-default-tiny` | `.375rem` | 6 |
| `--border-radius-messages-small` | `.375rem` | 6 |
| `--border-radius-default-small` | `.625rem` | 10 |
| `--border-radius-button-tiny` | `.875rem` | 14 |
| `--border-radius-messages` | `.9375rem` | **15** |
| `--border-radius-default` | `1rem` | 16 |
| `--border-radius-button` | `1rem` | 16 |
| `--border-radius-toast` | `1rem` | 16 |
| `--composer-control-radius` | `1.25rem` | 20 |
| `--border-radius-island` | `1.5rem` | 24 |
| `--border-radius-pane` | `1.5rem` | 24 |
| `--composer-pill-radius` | `1.5rem` | 24 |
| `--border-radius-modal` | `2rem` | 32 |
| `--border-radius-forum-avatar` | `33.3333%` | squircle |
| `--radius` (Avatar) | `50%` | circle |

`15px` for message bubbles is the one value that is not on a 2-px grid — **Possible** that it is a legacy value carried from the mobile clients; the iOS override bumps it to a clean `1rem`.

Local radius overrides are used to retheme whole regions: `.tEuug5Du { --border-radius-default: .625rem }`, `.aTWb9GK8 { --border-radius-default: 1.25rem }`, `.vLryWdvD { --border-radius-default: 0 }`, `.zUiUfZVD { --modal-border-radius: 1.75rem }`. **Confirmed** — a component asks for `--border-radius-default` and an ancestor decides what that means.

### 3.9 Z-index scale

**Theme-invariant.** Two disjoint bands, which is itself the design.

**Band 1 — within-column stacking (−1 … 55):**

| Token | Value |
|---|---|
| `--z-below` | `-1` |
| `--z-resize-handle` | `2` |
| `--z-media-viewer-head`, `--z-video-player-controls` | `3` |
| `--z-register-add-avatar` | `5` |
| `--z-chat-ripple` | `6` |
| `--z-chat-float-button` | `calc(6 + 1)` = 7 |
| `--z-message-select-area` | `8` |
| `--z-message-select-control`, `--z-sticky-date` | `9` |
| `--z-scroll-notch`, `--z-story-ribbon`, `--z-country-code-input-group` | `10` |
| `--z-left-header`, `--z-middle-header`, `--z-middle-footer` | `11` |
| `--z-scroll-down-button`, `--z-local-search` | `12` |
| `--z-forum-panel`, `--z-message-context-menu` | `13` |
| `--z-message-highlighted` | `14` |
| `--z-message-effect` | `15` |
| `--z-menu-backdrop` | `20` |
| `--z-menu-bubble` | `21` |
| `--z-animation-fade` | `50` |
| `--z-drop-area` | `55` |

**Band 2 — app-level overlays (900 … 12000):**

| Token | Value |
|---|---|
| `--z-right-column` | `900` |
| `--z-right-column-menu` | `950` |
| `--z-header-menu-backdrop` | `980` |
| `--z-header-menu` | `990` |
| `--z-resize-grip` | `1000` |
| `--z-reaction-interaction-effect` | `1100` |
| `--z-story-viewer` | `1150` |
| `--z-symbol-menu-mobile` | `calc(1150 + 1)` = 1151 |
| `--z-round-video-recorder` | `1160` |
| `--z-modal-low-priority` | `1400` |
| `--z-media-viewer` | `1500` |
| `--z-modal` | `1510` |
| `--z-confetti` | `1600` |
| `--z-modal-menu` | `1600` |
| `--z-notification` | `1700` |
| `--z-ui-loader-mask` | `2000` |
| `--z-lock-screen` | `3000` |
| `--z-symbol-menu-modal` | `5000` |
| `--z-portal-menu` | `10000` |
| `--z-reaction-picker` | `10200` |
| `--z-modal-confirm` | `10500` |
| `--z-overlay-effects` | `12000` |

Three things are right about this and one is wrong.

Right: (a) every stacking decision is **named**, so `z-index: 13` never appears in a rule and a reviewer can see that the forum panel and the message context menu are deliberately peers; (b) the two bands prevent local stacking from ever colliding with global overlays; (c) relationships are expressed as arithmetic (`calc(var(--z-story-viewer) + 1)`, `calc(var(--z-chat-ripple) + 1)`) so the invariant survives a renumbering.

Wrong: the scale has **grown by accretion at the top**. `10000 → 10200 → 10500 → 12000` are the fingerprints of four separate "this needs to be above that" incidents. **Strong inference.** Start with generous gaps (100, 200, 300…) or you end up here.

### 3.10 Motion tokens

**Theme-invariant.** The complete set:

| Token | Value | Meaning |
|---|---|---|
| `--layer-transition` | `.3s cubic-bezier(.33, 1, .68, 1)` | layer push/pop (ease-out-cubic) |
| `--layer-transition-behind` | `.3s cubic-bezier(.33, 1, .68, 1)` | the outgoing layer |
| `--slide-transition` | `.3s cubic-bezier(.25, 1, .5, 1)` | screen slide (ease-out-quart) |
| `--top-stack-transition` | `.45s cubic-bezier(.25, 1, .5, 1)` | the message-list stack, deliberately slower |
| `--select-transition` | `.2s ease-out` | selection state |
| `--chat-transform-transition` | `.2s ease-out` | chat-row transforms |
| `--mask-glide-transition` | `.15s` | scroll-fade masks |
| `--composer-embedded-message-duration` | `.15s` | reply/edit strip |
| `--composer-input-grow-duration` | `.1s` | input auto-grow |
| `--pane-slide-transition` | `0s` when `body.no-page-transitions` | kill switch |

Component-level, notable:

```css
.FloatingActionButton { --transition: transform .25s cubic-bezier(.34, 1.56, .64, 1),
                                      background-color .15s, color .15s, opacity .15s; }
```

The `1.56` control point is an **overshoot** — the FAB springs past its final scale and settles. It is the only spring curve in the system. **Confirmed.**

```css
#MiddleColumn .Composer > .Button {
  --composer-button-transition: border-radius .15s, opacity var(--select-transition),
                                transform var(--select-transition), background-color .15s, color .15s; }
#MiddleColumn .messages-layout { --slide-transition: var(--top-stack-transition); }
.rGm45f2Q { --color-transition: .25s ease-in-out; --state-transition: .25s cubic-bezier(.29,.81,.27,.99); }
.OBvwfQLp, ._2Si5iy8m { --color-transition: .15s; --transform-transition: .25s ease-in-out; }
```

Menu entry scale is a token, not a keyframe constant:

| Selector | `--animation-start-scale` |
|---|---|
| `.Menu .bubble` | `.85` |
| `.CountryCodeInput .bubble` | `.95` |
| `.aTWb9GK8` (a picker bubble) | `.5` |
| `body.no-context-menu-animations .Menu .bubble` | **`1 !important`** |

That last line is the entire "disable context-menu animation" setting: setting the start scale to 1 makes the scale animation a no-op without touching a keyframe or a component. **Confirmed.** Elegant.

Platform overrides — a rare case where motion is *not* invariant, but the axis is **platform**, not theme:

```css
:root body.is-ios     { --layer-transition: .65s cubic-bezier(.22, 1, .36, 1);
                        --layer-transition-behind: .65s cubic-bezier(.33, 1, .68, 1);
                        --slide-transition: .45s cubic-bezier(.25, 1, .5, 1); }
:root body.is-android { --slide-transition: .35s cubic-bezier(.16, 1, .3, 1); }
```

iOS transitions run **more than twice as long** (650 ms vs 300 ms) with a much steeper ease-out. **Confirmed.** This is deliberate platform mimicry — matching UIKit's interactive-push feel — and it is achieved with three token overrides rather than a per-platform animation module.

Also: `--layer-blackout-opacity: .1` — the dimming applied to the layer *behind* a pushed layer is only 10%, not the 40–50% a typical scrim uses.

### 3.11 Typography

**Theme-invariant.** Five families:

| Token | Stack |
|---|---|
| `--font-family` | `"Roboto", -apple-system, BlinkMacSystemFont, "Apple Color Emoji", "Segoe UI", Oxygen, Ubuntu, Cantarell, "Fira Sans", "Droid Sans", "Helvetica Neue", sans-serif` |
| `--font-family-rounded` | `"Nunito", "Roboto", "Helvetica Neue", sans-serif` |
| **`--font-family-numbers-rounded`** | **`ui-rounded, "Numbers Rounded", "Roboto", "Helvetica Neue", sans-serif`** |
| `--font-family-monospace` | `"Cascadia Mono", "Roboto Mono", "Droid Sans Mono", "SF Mono", "Menlo", "Ubuntu Mono", "Consolas", monospace` |
| `--font-family-condensed` | `"Roboto Condensed", "Roboto", "Helvetica Neue", sans-serif` |

Platform and language overrides:

```css
body.is-ios, body.is-macos {
  --font-family: system-ui, -apple-system, BlinkMacSystemFont, "Roboto", "Apple Color Emoji", "Helvetica Neue", sans-serif;
  --font-family-rounded: ui-rounded, "Nunito", "Roboto", "Helvetica Neue", sans-serif;
  --font-weight-semibold: 600;                     /* 500 everywhere else */
}
html[lang=fa], html[lang=fa] body { --font-family: "Vazirmatn", "Roboto", …; }
```

**`--font-family-numbers-rounded` is the most interesting token in the system.** It leads with the CSS generic `ui-rounded` — the platform's own rounded UI face (SF Rounded on Apple) — then falls back to a *custom-named* family `"Numbers Rounded"`, then to Roboto. **Strong inference on purpose:** this is a dedicated face for badge counts, unread counters and timers — numerals only, rounded, tabular-feeling — so that a "12" in an unread pill has the same optical weight as a "1" and the pill does not jitter as the count changes. Only **two `.woff2` files** load on the app shell (22,672 B total, **Confirmed**), so the custom face is either subset to digits or the `ui-rounded` generic is doing the work on most platforms.

That is a level of typographic care most chat apps never reach — and it costs one token.

Weights, sizes:

| Token | Value |
|---|---|
| `--font-weight-normal` | `400` |
| `--font-weight-medium` | `500` |
| `--font-weight-semibold` | `500` (**600** on iOS/macOS) |
| `--message-text-size` | `16px` (user-adjustable, default 16) |
| `--composer-text-size` | `16px` |
| `--message-meta-height` | `20px` |
| `--composer-input-line-height` | `1.375` / `1.3125` |

There are only **three weights and effectively two of them differ** (semibold == medium == 500 outside Apple platforms). **Strong inference:** a deliberate restriction so the UI reads consistently across the Roboto/system-ui split, where a shared 600 would render very differently.

Instant View (article reading) gets its own scale with a user-scalable multiplier:

| Token | Value |
|---|---|
| `--iv-font-size-scale` | `1` |
| `--iv-font-size-title` | `calc(1.5625rem × scale)` = 25 px |
| `--iv-font-size-subtitle` | `calc(1.375rem × scale)` = 22 px |
| `--iv-font-size-heading2` | `calc(1.1875rem × scale)` = 19 px |
| `--iv-font-size-body` | `calc(1.125rem × scale)` = 18 px |
| `--iv-font-size-compact` | `calc(1.0625rem × scale)` = 17 px |
| `--iv-font-size-small` | `calc(1rem × scale)` = 16 px |

Reading text is 18 px against chat's 16 px, and one variable scales the whole article. **Confirmed.**

Emoji sizing is typographic too: `--emoji-size: 1.25em` and `--custom-emoji-size: var(--emoji-size)` — **relative to the surrounding text**, so emoji scale with the font-size slider automatically. Inline custom emoji have a floor: `--custom-emoji-size: max(1.25em, 20px)`. And emoji-only messages get a graduated scale by count: 1 → `7rem`, 2 → `5.625rem`, 3 → `5.25rem`, 4 → `4.5rem`, 5 → `3.75rem`, 6 → `3rem`, 7 → `2.25rem`. **Confirmed.**

### 3.12 Elevation

| Token | Light | Dark |
|---|---|---|
| `--shadow-island` | `0 1px 4px 0 #0000000d` | same |
| `--shadow-pane` | `0 1px 5px -1px #00000036` | same |
| `--shadow-footer` | `0 1px 8px 1px #0000001f` | same |
| `--shadow-chat-list-panel` | `0 1px 8px 0 #0000001a` | same |
| `--color-default-shadow` | `#727272 @0.25` | `#101010 @0.61` |
| `--color-light-shadow` | `#727272 @0.17` | `#000000 @0.25` |

The four *named* shadows are theme-invariant black; the two *colour* tokens are themed and get **darker and more opaque** in dark mode (`#727272 @0.25` → `#101010 @0.61`), because a grey shadow is invisible on `#212121`. **Confirmed.** Mixed strategy — the composed shadows should arguably reference the colour tokens and do not.

Elevation is deliberately shallow: the largest blur in the system is 8 px and every shadow has a 1 px y-offset. There is no Material-style elevation ladder. Depth is communicated by **surface colour and radius**, with shadow as a hairline separator. On `screenshots/22-desktop-right-column-channel-profile-info-panel.png` the right column reads as "in front" because of its background colour and rounded edge, not because of a drop shadow.

---

## 4. Component architecture

Inferred from the live DOM, the CSS token scopes, and the recovered source tree. The primitive kit lives in `src/components/ui/` (114 files): `Button`, `ListItem`, `Menu`, `MenuItem`, `MenuSeparator`, `Modal`, `Transition`, `InfiniteScroll`, `Portal`, `RippleEffect`, `Spinner`, `TabList`, `Draggable`, `RangeSlider`, `Loading`, `DropdownMenu`, `FloatingActionButton`, `Skeleton`, plus `mediaEditor/`, `textInput/`, `placeholder/`.

### 4.1 Button — variants as token rewrites

Live variants observed in the DOM (**Confirmed**):

| Live class string | Where |
|---|---|
| `Button auth-button default primary text` | both login CTAs |
| `Button smaller translucent round` | header icon buttons |
| `FloatingActionButton revealed` | "Continue To Group Info" in the group wizard |
| `has-ripple` | list rows and menu items |

The full variant set from the CSS, each defined **purely by rewriting five tokens**: `primary`, `secondary`, `danger`, `green`, `gray`, `dark`, `stars`, `text`, `text.primary`, `text.secondary`, `text.danger`, `adaptive`, `translucent`, `translucent-primary`, `translucent-white`, `translucent-black`, `translucent-bordered`, `transparentBlured`, `bluredStarsBadge`, `tiny`, `tiny.pill`, `smaller`, `round`, `loading`.

The contract:

```css
.Button          { --button-text-color: white; --button-background-color: transparent; }
.Button.primary  { --ripple-color: #00000014;
                   --button-text-color: var(--color-white);
                   --button-background-color: var(--color-primary);
                   --button-active-text-color: var(--color-white);
                   --button-active-background-color: rgba(var(--color-primary-shade-rgb), .9);
                   --button-no-ripple-background-color: var(--color-primary-shade-darker); }
.Button.adaptive { --ripple-color: var(--accent-background-active-color);
                   --button-text-color: var(--accent-color);
                   --button-background-color: var(--accent-background-color);
                   --button-active-background-color: var(--accent-background-active-color); }
```

Two things worth stealing:

1. **`--button-no-ripple-background-color`** exists as a first-class token — the pressed colour when ripple is disabled. Every variant defines both a ripple colour and a non-ripple pressed colour, so turning off ripples degrades to a correct static press state rather than to nothing. That is the mark of a team that treats the motion-off path as a real design, not a fallback.
2. **`.Button.adaptive`** inherits from `--accent-color` / `--accent-background-color` rather than naming a colour. Drop it inside `.peer-color-7` and it becomes that peer's colour with no props. This is contextual theming through the cascade, and it is why the app can tint entire regions (own-message subtrees, per-peer quote bars, the call panel) without threading props.

`.Button.loading .Spinner { --spinner-size: 1.8125rem }` — the child spinner is sized by the parent through a token rather than by a prop. The same pattern appears in ~15 places (`--spinner-size` is set by `.Checkbox.loading`, `.Radio.loading`, `.SearchInput .Loading`, `.CommentButton_loading`, `.Loading`, `.AmlHA0pP`, …). **Confirmed.**

### 4.2 ListItem

```css
.ListItem .ListItem-button                                             { --ripple-color: #00000014; }
.ListItem:not(.disabled):not(.is-static) .ListItem-button:hover,
.ListItem:not(.disabled):not(.is-static) .ListItem-button:focus-visible { --background-color: var(--color-chat-hover); }
.ListItem.focus                                                        { --background-color: var(--color-chat-hover); }
.ListItem.has-menu-open .ListItem-button                               { --background-color: var(--color-chat-hover); }
.ListItem:not(.has-ripple):not(.is-static) .ListItem-button:active,
body.no-page-transitions .ListItem .ListItem-button:active             { --background-color: var(--color-item-active) !important; }
```

Note `:hover` and `:focus-visible` are given **the same** treatment, and `has-menu-open` is treated as a hover state — so a row with its context menu open stays visually anchored. **Confirmed.** The modifier vocabulary (`disabled`, `is-static`, `has-ripple`, `focus`, `has-menu-open`, `selected`) is consistent across `ListItem` and `Chat`.

`.Chat` is a `ListItem` specialisation and shows how far the token approach goes — a selected chat row rewrites **eight** colour tokens at once so every descendant flips to white without any descendant knowing:

```css
.Chat.selected:not(.forum) .ListItem-button {
  --color-text: var(--color-white); --color-text-meta-colored: var(--color-white);
  --color-text-meta: var(--color-white); --color-text-secondary: var(--color-white);
  --color-error: var(--color-white); --color-list-icon: var(--color-white);
  --color-chat-username: var(--color-white);
  --background-color: var(--color-chat-active) !important; }
```

Even the verified-badge SVG participates: `--color-fill: #fff; --color-checkmark: var(--color-primary)` — the checkmark inverts so it stays visible against the now-white badge. **Confirmed.** That is a genuinely well-considered detail.

The avatar badge maintains an outline that matches whatever surface it currently sits on, through `--_color-outline`, updated in five states (`.Chat .avatar-badge`, `:hover`, `.has-menu-open`, `.selected`, `.selected:not(.forum)`).

### 4.3 Radio, Checkbox, Switch — the `gili` layer

There are **two** primitive layers in the tree. `src/components/ui/` is the legacy kit; `src/components/gili/` (21 files, aliased `@gili/*`) is a newer, self-contained design-system layer: `primitives/{Checkbox,Switch,Radio,IconBackdrop}`, `layout/{Island,Surface,Control,Interactive}`, `templates/{SwitchField,CheckboxField}`, `modal/Modal`. **Strong inference:** an in-progress migration to a cleaner primitive set alongside the old one.

The `gili` control layout is done with **CSS Grid template areas driven by tokens**, which is unusual and good:

```css
.pYHFVT8p          { --control-grid-template-areas: "input before label after";
                     --control-grid-columns: auto auto 1fr auto; --control-gap: 1rem; }
.pYHFVT8p.PnRlFtk7 { --control-grid-template-areas: "before label after input";
                     --control-grid-columns: auto 1fr auto auto; }
.pYHFVT8p:has(> ._1GWvQCDI) { --control-grid-template-areas: "input before label after"
                                                             "input before desc  after"; }
```

Four states — control-leading vs control-trailing, with and without a description line — expressed as four token values on one grid. The `:has()` selector detects the description child. **Confirmed.** A checkbox row, a radio row, a switch row and a switch-with-subtitle row are all one component.

Switch track colour is likewise a token chain with semantic overrides:

```css
._5wY4L9f0                     { --switch-track-color: var(--ui-border-color, var(--color-borders-input)); }
.f9IlicSg:checked + ._5wY4L9f0 { --switch-track-color: var(--ui-accent-color, var(--color-primary)); }
.wwsvbTY5 ._5wY4L9f0                     { --switch-track-color: var(--color-error); }
.wwsvbTY5 .f9IlicSg:checked + ._5wY4L9f0 { --switch-track-color: var(--color-green); }
```

The last two invert the semantics (off = red, on = green) for a destructive-context switch, by class, with no component change. **Confirmed.**

Where the two layers meet is visible in the shipped CSS: `.component-theme-dark` is a **third** dark palette (`--color-background: #212121`, `--color-background-secondary: #0f0f0f`, `--color-background-own: #766ac8`, …) scoped to a class rather than to `html.theme-dark`, presumably so a `gili` component can be rendered dark inside a light context. **Possible** — we did not observe it in use.

### 4.4 Menu / MenuItem

```css
.Menu .bubble       { --offset-y: calc(100% + .5rem); --offset-x: 0; --animation-start-scale: .85; }
.MenuItem           { --ripple-color: #00000014; }
.HeaderMenuContainer .Menu .bubble { --offset-y: calc(100% + 1rem); }
.SymbolMenu .bubble { --offset-y: 3.5rem; }
.TKm7uBfD .Menu .bubble { --offset-y: 3.5rem; --offset-x: 1rem; }
.x5xbqz9R           { --offset-y: 3.25rem !important; --offset-x: var(--folders-sidebar-width) !important; }
```

Positioning is `--offset-x` / `--offset-y` tokens on a `.bubble` element, with per-context overrides — including one that offsets by the folder-rail width so a menu opened from the rail clears it. Underneath, `@floating-ui/dom` handles collision. **Confirmed.**

Menus render into `#portals` (`--z-portal-menu: 10000`) with a backdrop at `--z-menu-backdrop: 20` and the bubble at `--z-menu-bubble: 21` within it.

### 4.5 Modal

Modal sizing is entirely token-driven, per modal, via hashed classes:

| Selector | Tokens |
|---|---|
| `.ARSaAYKH` | `--modal-max-width: 26.25rem` |
| `._7OttD-eB`, `.zUiUfZVD` | `--modal-max-width: 35rem` |
| `.L8XYzHt7` | `--modal-max-width: 44rem` |
| `.eCOBpcEm` | `--modal-max-width: 100dvw` |
| `._4mTvM22v` | `--modal-max-height: min(39.3125rem, 80dvh)`, `--modal-header-height: 1rem` |
| `.hvYD4rD1`, `.zUiUfZVD` | `--modal-max-height: min(92dvh, 50rem)` |
| `.bDq-yyYY` | `--modal-max-height: min(96dvh, 62rem)` |
| `.WyLmghib` | `--modal-max-height: 100dvh` |
| `.zUiUfZVD` | `--modal-header-height: 3.5rem`, `--modal-content-block-padding: .5rem`, `--modal-border-radius: 1.75rem`, `--modal-background-color: var(--color-background-secondary)`, `--modal-scroll-fade-size: 0rem` |

`dvh` (dynamic viewport height) is used throughout rather than `vh` — correct for mobile browser chrome. **Confirmed.** `--modal-scroll-fade-size` and `.sxU91IyC:has(.DfZgX-2g) { --modal-scroll-fade-size: 1rem }` show the scroll-fade mask is conditional on content, detected with `:has()`.

### 4.6 Transition — and the trade-off it represents

This is the most consequential component in the app and the one with the clearest cost.

`src/components/ui/Transition.tsx` supports **16 named animations**: `none`, `slide`, `slideRtl`, `slideFade`, `zoomFade`, `zoomBounceSemiFade`, `slideLayers`, `fade`, `pushSlide`, `reveal`, `slideOptimized`, `slideOptimizedRtl`, `semiFade`, `slideVertical`, `slideVerticalFade`, `slideFadeAndroid`. Each has a matching `--slide-background-color` token so the incoming pane is opaque during the slide:

```css
.Transition-pushSlide, .Transition-pushSlideBackwards,
.Transition-slideFadeAndroid, .Transition-slideFadeAndroidBackwards,
.Transition-slideLayers, .Transition-slideLayersBackwards { --slide-background-color: var(--color-background); }
#RightColumn > .Transition { --slide-background-color: var(--color-background-secondary); }
#LeftColumn                { --slide-background-color: transparent; }
```

It drives classes **imperatively** (`addExtraClass` / `removeExtraClass` / `setExtraStyles` from the DOM layer), waits on `waitForAnimationEnd` / `waitForTransitionEnd` with `FALLBACK_ANIMATION_END = 1000`, and forces a reflow through `requestForcedReflow` at the one point where it is genuinely required. `will-change` is applied through the extra-styles channel rather than declared statically — i.e. only for the duration of an animation. **Confirmed.**

**The finding: inactive screens stay mounted.** In the live session, *every* `querySelector` for a control in the left column matched **3 or more elements**, only one of which was visible. The same for `aria-label="Go back"`. `#LeftColumn` is a `.Transition` container that keeps the previous and next screens in the DOM simultaneously so it can slide between them. Every selector in this audit had to be qualified with `:visible`. **Confirmed.**

The trade-off, stated plainly:

| Gained | Paid |
|---|---|
| Transitions are cheap — nothing mounts or unmounts mid-animation, so there is no render cost at the moment of the gesture | The **accessibility tree contains multiple copies of the same control**, all with the same accessible name. A screen-reader user encounters three "Go back" buttons. |
| Transitions are **cancellable and reversible** — swipe back halfway and release, and the old screen is still there, fully rendered | Any automation, test, or extension that queries by role or label gets ambiguous results |
| Scroll position and form state survive navigation for free | DOM size and memory grow with navigation depth |
| No flash of unstyled/empty content on return | `document.activeElement` can end up on an off-screen element unless focus is managed explicitly |

Telegram mitigates the last point with `trapFocus.ts`, `focusNoScroll.ts`, `captureEscKeyListener.ts`, `useVirtualBackdrop.ts` and `useInputFocusOnOpen.ts`, and the source does apply ARIA across the primitive kit. But it does **not** appear to apply `inert` or `aria-hidden` to the outgoing pane, which is what would resolve the ambiguity cleanly.

**Verdict for taskrgram: adopt the pattern, fix the gap.** Keeping both panes mounted is the right call for a chat app with a left-column drill-down. Add one line — `inert` on the non-active pane — and you keep every benefit and lose the accessibility cost. `inert` also removes the elements from hit-testing and from tab order, which fixes the focus problem at the same time.

### 4.7 Avatar

```css
.Avatar { --color-user: var(--color-primary); --radius: 50%; --_size: 0px;
          --_font-size: max(calc(var(--_size) / 2 - 4px), .5rem); }
.Avatar.size-mini        { --_font-size: max(calc(var(--_size) / 2 - 2px), .625rem); }
.Avatar.forum            { --radius: var(--border-radius-forum-avatar); }   /* 33.3333% squircle */
.Avatar.message-bubble   { --radius: 0; }
.Avatar.deleted-account, .Avatar.hidden-user { --color-user: var(--color-deleted-account); }
.Avatar.saved-messages, .Avatar.replies-bot-account, .Avatar.anonymous-forwards { --color-user: var(--color-primary); }
.Avatar, .ProfilePhoto   { --color-user: var(--accent-color); }
```

The initials font size is **derived from the avatar size with a floor** — one formula covers every size from 16 px to 160 px, and no size variant needs its own type rule. `--_size` is a private token the component writes from JS. **Confirmed.** This is the single cleanest example of the private-token convention doing real work.

### 4.8 `#portals` and the portal target

Pre-created in the static HTML, before any JS. **Confirmed.** Everything overlay-ish renders there: menus, modals, tooltips, the reaction picker. This is why the z-index Band 2 values are so large — portalled content is a sibling of the app root, not a descendant, so it must out-stack the entire column tree.

---

## 5. Interaction and motion

### 5.1 Ripple

A dedicated `RippleEffect` component, driven entirely by tokens. `--ripple-color` is set in **at least 14 distinct scopes**: `.ListItem .ListItem-button` and `.MenuItem` and `.EmbeddedMessage` and `.MessageSelectToolbar .item` all use `#00000014` (8% black); dark surfaces use `#ffffff14`; `.Button.danger` uses `rgba(var(--color-error-rgb), .16)`; `.Button.adaptive` and `.SponsoredMessage__button` use `var(--accent-background-active-color)` so the ripple inherits the peer/context accent.

`--z-chat-ripple: 6` with `--z-chat-float-button: calc(6 + 1)` — the ripple has a reserved stacking slot inside the chat row, and the float button is defined *relative to it*.

The graceful-degradation path is first-class: `.ListItem:not(.has-ripple):not(.is-static) .ListItem-button:active` and `body.no-page-transitions .ListItem .ListItem-button:active` both fall back to `--background-color: var(--color-item-active) !important`. **Confirmed.** Ripple off ⇒ instant press colour, not nothing.

### 5.2 The context-menu spotlight

Right-clicking a message **dims the whole application except the target message**, which stays at full brightness. This is a spotlight, not a scrim. **Confirmed** — see `screenshots/21-desktop-message-right-click-context-menu-with-reactions.png`.

The mechanism is visible in the token space: `--z-message-highlighted: 14` sits **above** `--z-message-context-menu: 13`, and `--z-menu-backdrop: 20` / `--z-menu-bubble: 21` are a separate pair. So the target message is promoted above the dimming layer while the rest of the list stays below it. The reaction strip attaches above the menu at `--z-reaction-picker: 10200`. **Strong inference** on the exact composition; Confirmed on the visual result and the token values.

Why it is better than a scrim: a flat overlay tells you "a menu is open"; a spotlight tells you "a menu is open **about this**". With a 40-item message list on screen and a menu positioned near the cursor, that disambiguation is not decorative.

### 5.3 Backdrop-filter frosted panels

Three surfaces use real `backdrop-filter` blur: the emoji/sticker panel, context menus, and message bubbles. Evidence:

- Live body class **`with-message-blur`**, observed. **Confirmed.**
- `body:not(.no-menu-blur) .aTWb9GK8 { --color-background: var(--color-background-compact-menu); }` and the same for `.x5xbqz9R` — the *translucent* background is only used when blur is enabled; with blur off, the menu falls back to the opaque `--color-background`. **Confirmed.**
- The dedicated translucent palette: `--color-background-compact-menu` = `#ffffff @0.73` / `#212121 @0.87`, `--color-background-compact-menu-reactions` = `#ffffff @0.92` / `#212121 @0.87`, and `--color-background-compact-menu-hover`.
- `.iOBYBaMj { --blured-background-color: #c4c9cc42 }` and `.Button.transparentBlured { --button-background-color: #ffffff1a }`.

See `screenshots/43-desktop-dark-theme-emoji-picker-frosted-glass-panel.png` and `screenshots/16-desktop-composer-emoji-picker-panel-open.png`.

The important detail is that **translucency and blur are coupled and both are switchable**: turning off blur does not leave you with a semi-transparent, unreadable panel — it swaps the background token to an opaque one. Most implementations get this wrong.

### 5.4 The two-phase enter/exit pattern

Observed on `#Main`, verbatim:

```
#Main.opacity-transition.fast.right-column-not-shown.right-column-not-open
```

**Two separate classes for one closed state.** **Confirmed.** The tell of a deliberate two-phase model:

| Class | Meaning (Strong inference) |
|---|---|
| `right-column-not-open` | logical state — the user has not requested the panel |
| `right-column-not-shown` | render state — the panel is not occupying space / not painted |

Opening runs *shown* first (mount, allocate width, start from an off-screen transform) then *open* on the next frame (animate in). Closing runs *open* off first (animate out) and *shown* off only after the transition ends. One boolean cannot express "closing but still visible", which is exactly the state you need a class for if CSS is doing the animation.

The complementary token machinery is there too:

```css
#Main.right-column-open .AudioPlayer.island-player { --player-shift-x: calc(var(--right-column-width) / -2); }
.MiddleHeader   { --header-transform-transition: transform var(--layer-transition); }
._7zgs9ckg      { --island-transform-transition: transform var(--layer-transition); }
body.no-right-column-animations .MiddleHeader { --header-transform-transition: transform 0s; }
body.no-right-column-animations ._7zgs9ckg    { --island-transform-transition: transform 0s; }
body.no-right-column-animations .AudioPlayer.island-player { --player-transform-transition: transform 0s; }
```

The floating audio player and the header **shift left by half the right column's width** when it opens, so the centred elements stay centred in the *remaining* space rather than jumping. And the whole behaviour is disabled by one body class rewriting three duration tokens to `0s`. **Confirmed.**

### 5.5 State as class names

The comprehensive body/root class vocabulary observed or found:

| Class | Axis |
|---|---|
| `is-pointer-env` | mouse vs touch |
| `is-linux`, `is-ios`, `is-macos`, `is-android`, `is-tauri` | platform |
| `theme-light`, `theme-dark` | theme |
| `notranslate` | translation suppression |
| `with-message-blur` | performance setting (opt-**in**) |
| `no-page-transitions`, `no-message-sending-animations`, `no-media-viewer-animations`, `no-message-composer-animations`, `no-context-menu-animations`, `no-menu-blur`, `no-right-column-animations` | performance settings (opt-**out**) |
| `no-animations` | transient, 500 ms during a theme switch |
| `keyboard-visible` | mobile keyboard |
| `cursor-grabbing`, `cursor-ew-resize` | drag state |
| `folders-sidebar-visible` | layout mode |
| `has-call-header` | layout mode |
| `ui-ready`, `mask-image-disabled` | render lifecycle |

`body.keyboard-visible { --composer-min-bottom-reserve: calc(var(--composer-height) + var(--composer-bottom-gap-mobile)); }` — the on-screen keyboard changes the composer's reserved space through a token, and the message list's bottom inset follows automatically because it is derived from that token.

**Assessment.** This approach is fast (no React re-render to toggle a visual state), debuggable (the entire app state is legible in DevTools' element inspector) and cheap (the CSS was going to exist anyway). The cost is that it is untyped — nothing prevents `no-page-transition` (singular) from being written and silently doing nothing — and it is invisible to component tests. **Adopt it for cross-cutting, low-frequency, presentation-only state; do not use it for anything a component needs to read.**

---

## 6. The Animations and Performance settings screen

The most decision-relevant screen in the entire product, and the clearest evidence of how this team thinks about motion. **Confirmed** by direct reading — `screenshots/26-desktop-settings-animations-and-performance-toggles.png` and `screenshots/27-desktop-settings-animations-performance-expanded-granular-toggles.png`.

### 6.1 The full control list, verbatim

A three-stop slider, labelled *"Choose the desired animations amount"*:

```
Power Saving  |  Nice and Fast  |  Lots of Stuff
```

Then a section literally headed **"Resource-Intensive Processes"**, with three expandable groups:

**Interface Animations**

| Control |
|---|
| Page Transitions |
| Message Sending Animation |
| Media Viewer Animations |
| Message Composer Animations |
| Context Menu Animation |
| **Context Menu Blur** |
| **Message Blur** |
| Right Column Animation |
| **Dust-effect deletion** |
| **Text Streaming** |

**Stickers and Emoji**

| Control |
|---|
| Allow Animated Emoji |
| Loop Animated Stickers |
| Reaction Effects |
| Sticker Effects |

**Media Autoplay**

| Control |
|---|
| Autoplay GIFs |
| Autoplay Videos |

Sixteen switches plus a three-stop preset. The presets map to `ANIMATION_LEVEL_MIN` / `MED` / `MAX`, with `ANIMATION_LEVEL_CUSTOM` when the user touches an individual switch; `ANIMATION_LEVEL_DEFAULT = ANIMATION_LEVEL_MED`. **Confirmed.**

### 6.2 What this screen reveals

**1. Blur is broken out from the animations it accompanies.** `Context Menu Blur` and `Message Blur` are their own switches, sitting next to `Context Menu Animation` rather than inside it. That is only worth doing if you know that `backdrop-filter` is categorically more expensive than a transform or an opacity fade — which it is: it forces the compositor to sample and blur everything behind a moving element on every frame. Separating them means a user on a weak GPU can keep the *motion* (which reads as responsiveness) and drop the *blur* (which reads as polish). Most apps offer one "reduce animations" switch and throw both away together.

Note also which of the two is opt-**in**: the body class is `with-message-blur`, not `no-message-blur`, while every other setting is `no-*`. **Strong inference:** message blur is off by default and the user opts in; the rest are on by default and the user opts out. That ordering is a cost judgement encoded in a class name.

**2. "Resource-Intensive Processes" is a literal heading in a consumer product.** Not "Advanced", not "Performance", not "Reduce Motion". The team chose to tell users, in the UI, that these features cost something. That is an unusual amount of respect for the user's machine, and it reframes the switches from "make it uglier" to "spend less".

**3. There are ten separately-toggleable interface animations.** Ten switches means ten places in the codebase where an animation is gated on a flag — and the CSS confirms the gates are real and cheap:

```css
body.no-page-transitions              { --pane-slide-transition: 0s; }
body.no-page-transitions .x-4xZYr-    { --translate-y: 0 !important; }
body.no-context-menu-animations .Menu .bubble { --animation-start-scale: 1 !important; }
body.no-right-column-animations .MiddleHeader { --header-transform-transition: transform 0s; }
body:not(.no-menu-blur) .aTWb9GK8     { --color-background: var(--color-background-compact-menu); }
```

Every gate is a **token rewrite**, not a component branch. This is the pay-off of the token architecture: sixteen user-facing performance switches cost roughly sixteen CSS rules, not sixteen conditional render paths.

**4. Two entries name effects that do not exist in most chat apps.** `Dust-effect deletion` (a message disintegrating into particles — `src/util/particles.ts`) and `Text Streaming` (progressive character-by-character reveal, presumably for AI-generated text — `.message-content .text-loading` has a shimmer with `--background-gradient-size: 20rem` and `--animation-color: var(--color-skeleton-background)`). A team that ships a particle-dissolve *and* immediately puts a switch next to it understands both sides of the trade.

**5. Autoplay is grouped with animation, not with data.** GIF and video autoplay appear here rather than in Data and Storage. **Strong inference:** the team classifies them as a *rendering* cost (decode + composite on every frame) rather than a *bandwidth* cost. On a desktop with a fast connection, that is the correct classification.

### 6.3 What it says about the team

There is no `prefers-reduced-motion` media query anywhere in the stylesheets. **Confirmed.** That is a genuine gap — a user with an OS-level reduced-motion preference gets nothing automatically. But the reason is visible: this team decided a single OS boolean was too blunt an instrument for sixteen distinct costs, and built a control panel instead.

**The right answer for taskrgram is both.** Honour `prefers-reduced-motion` as the *default* for the preset slider, then let the user override per-effect. That takes one media query on top of a system you were going to build anyway.

The measured backdrop for all of this: on a 4×-throttled CPU, total blocking time went from **72 ms to 753 ms** — a 10.5× degradation — while FCP moved only 10%. **Confirmed.** The main thread is where this app's cost lives, and this settings screen is the team's answer.

---

## 7. Native platform usage — and an inconsistency

### 7.1 The media viewer is a real `<dialog>`

Read from the live DOM (**Confirmed**):

```html
<dialog open id="MediaViewer" aria-modal="true" class="shown open">
```

Escape closed it in a single press, verified. `screenshots/44-desktop-media-viewer-native-dialog-element-open.png`.

Using the platform element buys, for free and correctly:

| Behaviour | Otherwise you write |
|---|---|
| Top-layer stacking | a z-index that must beat every other z-index forever |
| `::backdrop` | a scrim element plus its own stacking |
| Escape to close | a keydown listener with capture/scope management |
| Focus trapping | a focus-trap implementation with sentinel nodes |
| Inertness of the page behind | `aria-hidden` or `inert` juggling on the app root |
| Focus restoration on close | storing and restoring `document.activeElement` |
| Correct `aria-modal` semantics for AT | manual role/`aria-modal` bookkeeping |

### 7.2 …but everything else goes through `#portals`

Menus, modals, tooltips and the reaction picker render into the pre-created `<div id="portals">`, stacked manually with `--z-portal-menu: 10000`, `--z-reaction-picker: 10200`, `--z-modal-confirm: 10500`, `--z-overlay-effects: 12000`, and trapped manually with `trapFocus.ts`, `useVirtualBackdrop.ts` and `captureEscKeyListener.ts`. **Confirmed.**

So the app contains **two overlay systems**: one native and correct, one hand-rolled. The hand-rolled one is where the z-index accretion in §3.9 comes from, and where the focus-management utility modules come from.

**Assessment: `<dialog>` should be the default, and the inconsistency is a maturity artefact.** `showModal()` reached baseline availability across all engines in early 2022; the portal system predates it. But there are two legitimate reasons not to have migrated everything: (a) `<dialog>`'s top layer is strictly LIFO, which makes the "modal opens a menu opens a picker" cases in this app awkward, and non-modal `show()` does not participate in the top layer at all; (b) `::backdrop` cannot be animated as flexibly as a portalled scrim in older engines, and this app animates its backdrops (the spotlight in §5.2 would be difficult to build on `::backdrop`).

**Recommendation for taskrgram: `<dialog showModal()>` for every true modal — confirmations, settings sheets, image viewers — and a portal only for popovers and menus** (where the modern answer is the Popover API plus anchor positioning, not a hand-rolled portal). You get correct semantics for the 90% case and keep the escape hatch for the spotlight-style effects.

### 7.3 Other native-platform wins in this app

| Used | Where |
|---|---|
| `<dialog aria-modal>` | media viewer |
| Native `<button>` for real buttons | login CTAs are `<button type="button">`; **zero `[role="button"]` divs** on the screen. **Confirmed** by measurement |
| Native scrolling everywhere (no pixel virtualisation) | correct scrollbars, correct find-in-page, correct anchor restoration |
| Service Worker Range responses (HTTP 206) | native `<video>` seeking on non-HTTP content |
| `env(safe-area-inset-*)` | four dedicated tokens |
| `dvh` units | modal max-heights |
| `:has()` | control-layout variants, modal scroll-fade |
| `@layer` | the whole cascade order |
| CSS `light-dark()` (lowered by LightningCSS) | `--lightningcss-light` / `--lightningcss-dark` artefacts |
| `Intl.PluralRules` / `DisplayNames` / `NumberFormat` / `ListFormat` | all i18n formatting |
| `navigator.locks`, `BroadcastChannel`, `SharedWorker` | multi-tab election |
| `OffscreenCanvas` in a worker | thumbnail blurring |
| `ui-rounded` generic font family | numeric badges |

That is an unusually modern platform baseline, enforced by `compatTest.js`, which feature-gates 14 APIs at load and shows a hardcoded unsupported-browser page on failure. **Confirmed.**

---

## 8. States: loading, empty, error, skeleton

### 8.1 Loading

| Mechanism | Detail |
|---|---|
| **Spinner** | Eight pre-rendered SVG data-URI spinners as tokens: `--spinner-{white,white-thin,blue,dark-blue,black,green,gray,yellow}-data`. Each is a complete `<svg>` base64-encoded into a `url()`. Size is set by the *parent* via `--spinner-size`, in ~15 scopes: `1rem` (dropdown link), `1.25rem` (checkbox/radio/poll), `1.5rem` (search, comment button, composer), `1.75rem` (connection status), `1.8125rem` (loading button), `2rem`, `2.75rem` (full-page). **Confirmed.** |
| **Shimmer** | `.message-content .text-loading { --background-gradient-size: 20rem; --animation-color: var(--color-skeleton-background); }` with a dark-mode swap to `--color-skeleton-foreground`. Skeleton colours are theme-invariant translucent greys: `#212121 @0.15` and `#e8e8e8 @0.20`. |
| **Progress** | `.media-inner .message-media-last-progress { --_progress: 0%; --_color: var(--color-primary); }`, `--positive-progress` / `--negative-progress` / `--multiplier: calc(1 / var(--positive-progress) - 1)` for circular progress arcs. |
| **Connection status** | `#ConnectionStatusOverlay > .Spinner { --spinner-size: 1.75rem }` — a dedicated overlay for transport state. Exercised involuntarily: our session logged `Socket zws4… closed. Code: 1006` and `Error: TIMEOUT`, retried, and continued. **Confirmed** that the reconnect path works. |
| Dedicated stylesheets | `Skeleton-DT32WtCE.css` (15,436 B) and `Loading-CzX-7jrR.css` (175 B) ship separately. |

The zero-JS loading state is `--z-ui-loader-mask: 2000` — a masking layer above the app but below modals, removed when `#MiddleColumn` gains `ui-ready`. **Confirmed** from the live class list.

### 8.2 Empty

Observed verbatim (**Confirmed**):

| String | Where |
|---|---|
| **"You have no folders yet"** | Settings → Chat Folders, before any folder exists. `screenshots/30-desktop-settings-chat-folders-configuration.png` |
| **"Sorry, nothing found."** | New Group → Add Members, with an empty contact list. `screenshots/35-desktop-new-group-add-members-empty-contact-picker.png` |

Both are plain, sentence-case, no illustration, no CTA. `NoMessages` is a dedicated component and it does get an animated sticker (`.NoMessages .topic-icon { --custom-emoji-size: 3rem }`), so the empty-state treatment is graded: cheap text for utility surfaces, animated sticker for the primary surface.

**Assessment.** "Sorry, nothing found." is the weaker of the two — an apology instead of an action. The strong pattern for a team app is *state + next step*: "No folders yet. Create one to group your channels." Telegram gets this right in the folders case (the description line "Sort chats into folders" doubles as guidance) and wrong in the picker case.

### 8.3 Error

Error styling is heavily tokenised — `--color-error: #e53935`, `--color-error-shade: #d33431`, `--color-error-rgb`, `.Button.danger` with its own five-token block, `.rV7it5ZO { --secondary-bg: var(--color-error) }`, `._4PYsxjmB`, `._53pKJZFl { --ui-border-color: var(--color-error); --ui-accent-color: var(--color-error) }` for whole error-state form regions, and `.wwsvbTY5` for inverted-semantics switches. **Confirmed.**

But: **no application-level JS exception was thrown during the entire authenticated walkthrough.** 162 console events, all of them transport (WebSocket 1006 closures, `Error: Not connected`, `Error: TIMEOUT`) or GPU (software-WebGL fallback warnings in our headless environment). **Confirmed.** We never saw a rendered error state.

Two real defects were found:

1. `aria-label="AccDescrPollVoteDown"` on a live chat-header button — an untranslated i18n **key leaking into an accessible name**. Invisible to sighted users; a screen reader announces the control as "AccDescrPollVoteDown". **Confirmed.**
2. `console.error(undefined)` in the load path — an error handler logging an undefined value, i.e. a path that swallows its own diagnostic. **Confirmed.**

The source layer has no error boundaries at all (a consequence of the custom framework). **Strong inference:** a rendering exception anywhere would take down the tree. For taskrgram on React, put an error boundary at the column level, so a broken right-column tab does not take the message list with it.

### 8.4 Skeleton and placeholder strategy — vector paths, not blurhash

This is one of the best engineering decisions in the product. **Confirmed.**

**Mechanism 1 — vector path thumbnails.** Telegram's `strippedThumb` is a compact byte-encoded SVG path. The client expands it with a lookup table into an inline SVG:

```ts
const TEMPLATE = '<?xml version="1.0" …><svg … viewBox="0 0 {{width}} {{height}}" …>'
               + '<path fill-opacity="0.1" d="{{path}}" /></svg>';
const LOOKUP = 'AACAAAAHAAALMAAAQASTAVAAAZaacaaaahaaalmaaaqastava.az0123456789-,';
```

Cost: a string substitution. No decode, no canvas, no worker, no main-thread blocking. The result is a **silhouette** of the image at 10% opacity — it conveys shape (a person, a document, a chart) where blurhash conveys only average colour blobs.

**Mechanism 2 — OffscreenCanvas blur in a worker,** for cases where a real mini-thumbnail exists:

```ts
const FAST_BLUR_ITERATIONS = 2;
const isFilterSupported = 'filter' in ctx;
if (isFilterSupported) { ctx.filter = `blur(${radius}px)`; }
ctx.drawImage(imageBitmap, -radius * 2, -radius * 2, width + radius * 4, height + radius * 4);
if (!isFilterSupported) { fastBlur(ctx, 0, 0, width, height, radius, FAST_BLUR_ITERATIONS); }
```

Note the overdraw (`-radius * 2`, `+radius * 4`) to avoid transparent edges from the blur kernel, and the hand-written `fastBlur` fallback for contexts without `filter`. Hooks: `useBlur`, `useBlurSync`, `useCanvasBlur`, `useOffscreenCanvasBlur`, `useAverageColor`.

Plus `getAppendixColorFromImage()` — the bubble tail (the little pointer on a message bubble) samples the **corner pixel of the image** so its colour matches the media it is attached to. `.Message .message-content.has-appendix-thumb .svg-appendix { --background-color: #ccc }` is the fallback. **Confirmed.** That is a genuinely obsessive detail.

**Why this beats blurhash for a chat app:** blurhash requires a base-83 decode plus a DCT reconstruction per image on the main thread. In a message list that is loading 30 images at once during a scroll, that is 30 synchronous decodes exactly when the main thread is busiest. A path-byte SVG is a template fill and costs nothing; the canvas blur happens **in a worker**. The main thread does zero placeholder work in either path.

**Adopt this.** If your backend can produce a compact vector silhouette, do that. If not, at minimum move blurhash decoding into a worker with `OffscreenCanvas` — it is the same amount of work in the right place.

### 8.5 Scroll-affordance masks

A small but complete state system for "there is more content in this direction":

```css
.custom-scroll.with-scrollable-hint, .custom-scroll-x.with-scrollable-hint {
  --scrollable-hint-mask-size: var(--scrollable-hint-size, 2.5rem);
  --scrollable-hint-cutoff: 0rem;
  --scrollable-hint-{top,bottom,left,right}-mask-outset: var(--scrollable-hint-mask-size); }
.MessageList { --message-list-fade-floor: #0000003d;
               --message-list-top-fade-ramp: 1.5rem;
               --message-list-bottom-fade: var(--message-list-min-reserve); }
.M6ZdO77E { --fade-mask-height: 2.625rem; }
.aJ0cuUN4 { --fade-mask-color: var(--color-background-secondary); }
```

Four independent directional masks, sized from one token, with a colour that follows the surface. Combined with `--mask-glide-transition: .15s` for the fade in/out as scroll position changes. **Confirmed.** The message list's top and bottom fades are tied to the *composer's* reserved space, so the fade always ends exactly where the composer begins.

---

## 9. Patterns worth stealing

Ranked by value-to-effort for taskrgram.

**1. Cascade layers declared inline in the HTML, before any stylesheet loads.**

```css
@layer reset, variables, ui, components;
@layer ui { @layer tablist, spinner, button, input, layout; }
```

Four lines. Permanently removes "which stylesheet won" from your debugging vocabulary in a code-split app. There is no reason not to do this on day one, and retrofitting it later means auditing every specificity hack you wrote in the meantime.

**2. Make the theme axis colour-only, and prove it.** 210 of 352 tokens are identical between light and dark; **all 142 that differ are colours**. Zero spacing, radii, z-index, duration or type tokens change. Write a test that asserts this. It turns "add a third theme" from a project into a spreadsheet.

**3. Component variants as token rewrites, not style branches.** `.Button.primary` sets five variables and nothing else. Twenty-four variants, one component, one set of layout rules. Combine with `.Button.adaptive`, which reads `--accent-color` from its context — so an entire subtree can be recoloured by an ancestor with no props threaded.

**4. The `--_` private-token convention.** CSS variables have no visibility model; a leading underscore restores one. `--_size`, `--_font-size`, `--_progress` are internal; everything else is contract. Free, and it makes component refactors safe.

**5. Derive geometry, don't hardcode it.**

```css
--composer-input-padding-block:
  calc((var(--base-height,3rem) - var(--composer-text-size,1rem) * var(--composer-input-line-height)) / 2);
--pane-content-radius: calc(var(--border-radius-pane) - .25rem);
--_font-size: max(calc(var(--_size) / 2 - 4px), .5rem);
```

Vertical centring computed from height/size/line-height; concentric radii computed from the parent's; avatar initials computed from the avatar size with a floor. This is why the font-size slider does not break the composer.

**6. Every performance switch is a token rewrite.** Sixteen user-facing motion settings cost ~sixteen CSS rules, not sixteen component branches. `body.no-context-menu-animations .Menu .bubble { --animation-start-scale: 1 !important; }` is the entire feature.

**7. Design the motion-off state, don't just remove motion.** Every button variant defines both `--ripple-color` **and** `--button-no-ripple-background-color`. Blur-off swaps the translucent background for an opaque one rather than leaving a see-through panel. The degraded path is a design, not an absence.

**8. Protect the measure, not the column.** `--messages-container-width: 47.5rem` (760 px) caps the reading width independently of the column, which is what makes the fixed-408 px right column free. Decide your measure first; let the columns fall out of it.

**9. The context-menu spotlight instead of a flat scrim.** Dim everything except the target. In a list of forty similar rows, this is the difference between "a menu is open" and "a menu is open about *this*". Costs one z-index promotion.

**10. Two-phase enter/exit as two classes.** `right-column-not-shown` + `right-column-not-open`. One boolean cannot express "closing but still visible". If CSS is doing your animation, you need the second class.

**11. Placeholder work belongs off the main thread.** Vector path silhouettes cost a string substitution; blur happens in a worker on an `OffscreenCanvas`. Never decode thirty placeholders synchronously during a scroll.

**12. `<dialog showModal()>` as the default modal.** Top layer, `::backdrop`, Escape, focus trap, focus restore, `aria-modal` — all free and all correct. Telegram gets this right exactly once (the media viewer) and hand-rolls the rest; you can get it right everywhere.

**13. A dedicated numeric font family for counts.** `--font-family-numbers-rounded: ui-rounded, "Numbers Rounded", …`. One token; badge counts stop jittering and start looking designed.

**14. Graduated emoji-only sizing.** 1 emoji → 112 px, falling to 36 px at 7. Twenty lines, disproportionate delight.

**15. Generate i18n key types from the string table.** A missing key becomes a compile error rather than a rendered `AccDescrPollVoteDown` — a failure this very app ships.

**16. A 14-token theme contract for embedded third-party UI.** If taskrgram ever embeds a dashboard or an internal tool as a panel, publish a tiny fixed contract (`bg_color`, `text_color`, `hint_color`, `link_color`, `button_color`, …), not your whole token space.

---

## 10. Patterns to avoid

**1. Two sources of truth for one palette.** The `:root` colour block in the CSS is dead code — overwritten on first paint by the JS injector — and **13 of its values have silently diverged**, including `--color-own-links` (`#ffffff` vs the real `#3390ec`) and `--color-background-compact-menu-hover` (70% black vs the real 6.7%). If you need runtime theming, generate the static CSS from the same data at build time. Never hand-maintain a copy of something you have decided to override.

**2. `style-src 'unsafe-inline'` as the price of theming.** This app's CSP is otherwise excellent — `script-src 'self' 'wasm-unsafe-eval'` with no `unsafe-inline`, `object-src 'none'`, `base-uri 'none'`, `form-action 'none'` — and it correctly blocked a CDN script injection during this audit. The single relaxation exists solely because the theme system writes `<style>` text at runtime. Use `CSSStyleSheet.replaceSync()` on a constructed stylesheet, or set the properties via `element.style.setProperty()`, and keep the strict CSP.

**3. Inactive screens left in the accessibility tree.** Keeping both panes mounted for cheap, cancellable transitions is correct. Leaving three identical "Go back" buttons exposed to assistive technology is not. Add `inert` to the non-active pane — one attribute fixes AT, tab order and hit-testing together. (Every selector in this audit needed `:visible`; that is the smell.)

**4. A z-index scale grown by accretion.** `10000 → 10200 → 10500 → 12000` is four separate "this must be above that" incidents fossilised in a token file. Start with generous gaps and a documented band structure.

**5. Inlined design values that a token was supposed to own.** `--premium-gradient` is redeclared at slightly different angles and stops in **seven** places (84.4deg/−4.85%/51.72%/110.7% in `:root`, 88.39deg/−2.56%/51.27%/107.39% in six component scopes). The token system did not prevent the drift because the value was copied instead of referenced.

**6. Token-name rot.** Three near-identical forum tokens coexist — `--color-forum-unread-topic-hover`, `--color-forum-hover-unread-topic-hover`, and `--color-forum-hover-unread-topic` (theme-invariant `#e9e9e9`, no consumer found, would render light-grey in dark mode). Delete unused tokens; do not let a name space accumulate near-synonyms.

**7. Theme-invariant semantic and meta colours.** `--color-error: #e53935` is identical on `#ffffff` and `#212121` (≈4.0:1 on the dark surface). `--color-text-meta: #686c72` and `--color-placeholders: #a2acb4` are likewise theme-invariant, so timestamps and placeholders are low-contrast in dark mode. Simplification is fine until it fails WCAG; theme your semantic colours.

**8. A focus indicator that does not exist.** Measured on the focused login button: `outline: none`, `box-shadow: none`, `border: 0px none`. The **only** focus affordance is an 8%-alpha blue background tint, which computes to `#F0F7FC` — a contrast of **1.08:1** against white, against WCAG 2.2 SC 1.4.11's requirement of **3:1**. There are **zero** `:focus` / `:focus-visible` rules matching these buttons in any stylesheet. axe did not flag it (axe does not evaluate focus appearance), which is exactly why it survived. See `screenshots/a11y-button-unfocused.png` and `screenshots/a11y-button-focused.png`.

**9. Both primary CTAs failing colour contrast.** `#3390ec` on `#ffffff` = **3.31:1** at 16 px normal weight, against a 4.5:1 requirement — on `Log in by phone number` *and* `Log in with Passkey`. The brand blue is not usable as small text on white. Use `--color-primary-shade-darker` (`#2b79c6`, ≈4.9:1) for text and keep `#3390ec` for fills.

**10. `user-scalable=no`.** `<meta name="viewport" content="…maximum-scale=1.0, user-scalable=no, shrink-to-fit=no, viewport-fit=cover">` disables pinch-zoom, a WCAG 2 AA failure (SC 1.4.4) that blocks low-vision users. There is no good reason for this on a desktop-first app.

**11. Zero landmarks.** No `<main>`, no `<nav>`, no `<header>`, no `[role]` anywhere on the login screen — the `<h1>`, the instruction list and the animation are all orphaned. In a three-column app, landmarks are how a screen-reader user navigates between columns at all. `<nav>` for the left column, `<main>` for the middle, `<aside>` for the right, `<h1>` for the current chat name: four elements, enormous payoff.

**12. No `prefers-reduced-motion` anywhere.** Sixteen motion switches and not one line honouring the OS preference. A user who has already told their operating system they get motion sickness has to find and configure a settings screen. Honour the media query as the *default* for your preset, then allow per-effect override.

**13. A 295 KB rich-text editor in a chat composer.** TipTap with 27 extensions plus 13 ProseMirror packages plus highlight.js (375 files) plus temml plus lowlight, to support Pullquote, Details and LaTeX in a chat input. Ship a narrow schema; the marginal block types are close to free to *specify* and very expensive to *load*.

**14. Untranslated keys reaching the DOM.** `aria-label="AccDescrPollVoteDown"` is live in production. The fix is structural, not vigilance: generate `LangKey` from the string table so the compiler catches it (this app already does that for the *new* localisation path — the leak is from the legacy `oldLangProvider` path it is migrating away from).

**15. A settings screen that leaks another platform's controls.** "Window title bar → Show chat name" appears in the **web** Privacy and Security screen; it is meaningless without the Tauri shell (`body.is-tauri`, `--window-controls-width: 5rem`). One codebase for two shells is a good pattern; gate platform-specific settings on the platform class, not on hope.

---

## Appendix: confidence ledger

**Confirmed** — measured directly and reproducibly: all column geometry and breakpoints (§1.2–1.5); all live-resolved token values and the light/dark diff (§2.5, §3); the 566/252/577 token counts and the cascade-layer declaration (§2.1–2.2); the JS theme injector and its 78 tuples (§2.3); the 13-token CSS/JS drift (§2.4); the live DOM structure, class strings and `<dialog>` (§1.1, §4, §7); the context-menu item list and spotlight behaviour (§5.2); the complete Animations and Performance control list (§6.1); the empty-state strings (§8.2); all accessibility measurements — 3.31:1 CTA contrast, 1.08:1 focus indicator, zero landmarks, tab order, `user-scalable=no` (§10).

**Strong inference** — reasoning from confirmed evidence: the 760 px measure as the reason for the fixed 408 px right column (§1.4); the purpose of `--font-family-numbers-rounded` (§3.11); the meaning of the two-phase `not-shown`/`not-open` split (§5.4); the premium-gradient drift being copy-paste rather than intent (§3.6); the z-index accretion history (§3.9); `--color-forum-hover-unread-topic` being dead (§2.4); dark-mode contrast risk on theme-invariant meta colours (§3.2); the absence of `inert` on inactive Transition panes (§4.6).

**Possible** — plausible but unverified: `15px` message radius as a legacy mobile value (§3.8); `.component-theme-dark` existing for dark-in-light component rendering (§4.3).

**Unknown** — not determinable from this audit: the exact composition of the spotlight dim (which element carries the dimming layer); the runtime behaviour of the AI composer; whether `"Numbers Rounded"` is a shipped subset font or a name that never resolves.

**Environment caveats affecting nothing structural:** all sessions ran through a TLS-terminating local proxy with HTTP/2 disabled, from a datacenter IP, in headless Chromium without a GPU. This inflates every latency number and produced the WebGL and WebSocket-1006 console noise. It does **not** affect box metrics, resolved token values, DOM structure, class names, byte counts, or the accessibility measurements — all of which were byte-identical across runs.
