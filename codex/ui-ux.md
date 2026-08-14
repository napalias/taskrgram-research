# UI/UX architecture and design rationale

## Layout model

Telegram Web A models the application as three stateful columns plus an overlay plane.

| Width/state | Left | Middle | Right |
|---|---|---|---|
| Desktop >925 px | Static chat rail | Conversation workspace | Optional profile/settings pane |
| Tablet 601–925 px | Hideable/overlay rail | Primary workspace | Overlay/secondary pane |
| Mobile ≤600 px | Full-screen pane translated off-screen when chat opens | Full-screen conversation | Full-screen secondary pane |
| Short landscape | Mobile behavior up to 950×450 | Full-screen | Full-screen |
| Wide ≥1276 px | More inset/island treatment and percentage right column | Larger centered message area | 25vw right column |

Observed dimensions:

- At 1280×720 without an open chat, the left column was 320 px wide with 16 px outer inset; the middle area occupied the remaining 944 px.
- At 390×844 with a conversation open, each left/middle/right panel measured 390 px and used fixed/translated positioning; the composer was 374×48 px with 8 px side insets.

### Why this works

- Desktop users optimize for scanning and switching; persistent rails reduce navigation cost.
- Mobile users optimize for focus and thumb reach; a single visible pane avoids cramped split views.
- Reusing the same panel hierarchy across sizes reduces conceptual and implementation divergence.
- Fixed headers/composer and independently scrolling content preserve action availability during long histories.

## Information architecture

### Chat rail

Order of importance is visible in the composition:

1. account/menu;
2. stories;
3. search;
4. folder tabs;
5. virtualized chat list;
6. new-message FAB.

This prioritizes retrieve/switch/create—the dominant messaging tasks.

### Conversation

- Header: back affordance when required, identity/status, search/call/menu actions.
- Context panes: pinned messages, audio player, topic or selection state.
- Scrollable/windowed message history with sticky dates and floating jump controls.
- Composer: attachment, rich input, emoji/sticker/GIF, voice/video, send state.
- Overlay plane: menus, pickers, reactions, modals, media viewer, stories, calls.

### Settings

The settings overview groups items by user mental model rather than implementation:

- identity/account;
- appearance and keyboard;
- notifications;
- privacy/security;
- storage;
- organization;
- performance;
- stickers/emoji;
- language;
- devices;
- monetization/support/legal.

On wide screens, the overview remains visible while a nested pane opens. On mobile, the nested pane becomes a transition with back navigation. This provides spatial continuity on desktop and focused completion on mobile.

## Design tokens

The client uses CSS custom properties for a large token surface:

- Colors: background, secondary background, selected/hover, text tiers, borders, links, errors, success, message ownership, reactions, stories, gifts, premium.
- Geometry: 16 px default radius, 15 px message radius, 32 px modal radius, 24 px island/pane radius.
- Layout: 760 px/47.5rem message container, 424 px/26.5rem right column, 80 px folder rail, 64 px column header.
- Motion: 150–450 ms base transitions with different iOS and Android curves/durations.
- Safe areas: all four `env(safe-area-inset-*)` values.
- Z-index: named overlay tiers for modals, pickers, calls, stories, menus, toasts, and transient effects.
- Typography: Roboto by default; system stack on macOS; Vazirmatn for supported scripts; rounded/condensed/monospace variants.

### Why tokens are important here

Messaging UIs reuse the same concepts thousands of times. Tokens let Telegram recolor every message, menu, status, and control consistently for light/dark/custom themes and avoid bespoke component overrides. Named z-index layers are especially valuable in a product with many simultaneous overlays.

## Component patterns

Likely reusable component families, confirmed by runtime classes and public source:

- `Button`: default, primary, translucent, round, smaller, ripple.
- `ListItem`/`MenuItem`: icon + title + subtitle + trailing value/badge.
- `InputText`: floating label, error/success/focus states.
- `Checkbox`/radio/slider.
- `Avatar`: user/chat/story states and editable variant.
- `Transition`: fade/slide stacks with cleanup control.
- `Modal` and nested modal menus.
- `Header` and pane header primitives.
- `CustomScroll`, skeleton, spinner, empty placeholder, toast/notification.
- Virtualized chat/message/member/media lists.
- `Composer` primitives and rich-editor controls.
- Media, message, reaction, quote, profile, and picker families.

## Motion and feedback

- QR and login use progressive screen transitions rather than page reloads.
- Menus use backdrop + elevated rounded surfaces.
- Message sending, reactions, story opening, media expansion, and calls have specialized effects.
- Skeletons/spinners cover asynchronous data and bundle loading.
- The app exposes performance settings for interface animation, stickers/emoji, and autoplay.
- Platform-specific motion timing shows an intent to feel native instead of forcing one cross-platform curve.

### Trade-off

Motion reinforces continuity in a pane-based app, but the number of specialized effects increases GPU cost and QA complexity. The user-facing performance controls are a good mitigation. A separate reduced-motion path should be explicitly tested.

## Forms and validation

- Floating-label outlined fields supply focus, error, and success states.
- Phone input separates country selection and formatted number.
- Auth steps show one dominant action and secondary alternate login methods.
- Settings use labels/subtitles to explain consequences before entry.
- Dangerous actions are placed deeper in menus/panes and generally mediated by dialogs.

The public login's primary button used the full available width and a 48 px height on mobile. This improves touch accuracy and makes the next step unambiguous.

## Empty, loading, and error states

Source-confirmed patterns include:

- app-level `UiLoader` with page-specific loading state;
- skeleton rows and spinners;
- animated empty-state assets;
- connection-status overlays;
- retry/error dialogs and toasts;
- disabled composer controls when the context cannot act;
- offline/service-worker fallback behavior;
- unsupported-media fallbacks by platform;
- “no results,” banned, locked, and inactive account screens.

## Accessibility assessment

### Strengths

- Major icon buttons commonly have `aria-label` or title.
- Text inputs expose textbox roles and labels/placeholders.
- Checkboxes, sliders, menus, menuitems, and headings appear in the accessibility snapshot.
- 40–48 px common controls are touch-friendly.
- Focus-visible rules exist in source.
- Performance controls can reduce motion/media load.

### Weaknesses

- The inspected conversation view had no semantic `main`, `nav`, `aside`, or `header` landmarks.
- Icon-font glyphs leaked into accessibility snapshots as private-use characters.
- Several disabled composer buttons were unnamed.
- One action exposed a localization-key-like label rather than human language.
- The viewport disables user zoom.
- Screen-reader and keyboard task completion were not verified end to end.

## Design lessons for the internal project

1. Start with a formal pane state machine and breakpoint policy.
2. Build list virtualization and incremental updates before feature breadth.
3. Create token families for message ownership, interaction, semantic feedback, overlays, and motion.
4. Keep search and context switching more prominent than creation settings.
5. Use the same component/pane hierarchy across desktop and mobile, changing placement rather than concepts.
6. Keep rare capabilities lazy and contextual.
7. Make low-performance and reduced-motion modes first-class.
8. Improve on Telegram by using semantic landmarks, SVG icons hidden from screen readers, complete accessible naming, and zoom support.
