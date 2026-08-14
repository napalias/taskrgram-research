# Performance, accessibility, SEO, and quality

## Measurement boundary

These are passive measurements from the audit environment, not a controlled Lighthouse lab run. Curl timings include network and server conditions at one moment. Asset inventory reflects features exercised during the session. No claim is made about global user percentiles.

## Loading and bundle measurements

| Resource | Raw bytes | Compressed bytes | Audit fetch time |
|---|---:|---:|---:|
| HTML shell | 5,806 | not separately measured | ~0.13 s |
| Entry JS | 28,951 | 12,418 | ~0.17–0.23 s |
| Main JS | 420,528 | 148,175 | ~0.32–0.46 s |
| Calls support JS | 52,407 | not measured compressed | ~0.27 s |
| Editor JS | 294,925 | 110,610 | ~0.30–0.42 s |
| Extra JS | 796,897 | 290,444 | ~0.36–0.53 s |
| Protocol worker JS | 742,096 | 240,922 | ~0.36–0.50 s |
| Base CSS | 92,523 | 31,503 | ~0.22–0.29 s |
| Main CSS | 61,287 | 15,591 | ~0.27 s |
| Extra CSS | 209,067 | 57,950 | ~0.26–0.38 s |

The sizes should not be summed as cold-start bytes. `extra`, `editor`, calls, and feature chunks load by state, interaction, or idle warming. After login plus settings/conversation exploration, the page asset inventory contained 55 scripts, 23 stylesheets, four fonts, nine images, four other resources, and five inline SVGs.

## Positive performance architecture

- Very small HTML shell.
- Native ESM, modulepreload, and content-hashed chunks.
- Dedicated protocol and media workers.
- Worker message/update batching.
- FasterDOM read/write scheduling and optional strict checks.
- Memoized Teact components and normalized selectors.
- Windowed/infinite lists.
- Feature bundle boundaries and async wrappers.
- Cache-first service-worker behavior for fingerprinted assets.
- Network-first HTML for update freshness.
- Progressive media range bridge with bounded first-range caching.
- Image thumbnails/Canvas blur to show content before full media.
- User controls for animations and autoplay.
- No observed third-party analytics in the critical path.

## Performance risks

1. The protocol worker is large even before feature UI.
2. The `extra` bundle is broad and large; one uncommon action may unlock a large payload.
3. The rich editor is expensive for users who only send plain text.
4. Fixed-time idle warming can spend data/CPU on unused features.
5. A one-hour HTTP cache is short for content-hashed assets.
6. Public source maps add bandwidth only when requested, but should not be preloaded.
7. Animated stickers, stories, video, calls, canvas effects, and large chat histories create a demanding low-end-device matrix.
8. Multi-tab state synchronization reduces duplicate sockets but adds correctness and serialization costs.

## Caching

Observed asset response:

```text
Cache-Control: max-age=3600
Content-Encoding: gzip
ETag: W/...
X-Frame-Options: deny
```

Recommendation:

- HTML/version pointers: short-lived, network-first.
- Content-hashed JS/CSS/fonts/images: `public, max-age=31536000, immutable`.
- Source maps: private error-monitor upload unless deliberate public source is a product policy.
- Track service-worker hit rate and cache migration failures separately from HTTP cache performance.

## Runtime quality

- Final inspected authenticated conversation: zero captured console errors and zero warnings.
- This does not cover calls, media upload, payments, stories, mini apps, or failure modes.
- Screenshot capture of the heavy authenticated view timed out in the audit browser, while DOM and interactions remained responsive. This is an audit-tool limitation, not enough evidence of an application bug.

## Accessibility

### Confirmed positives

- Buttons for primary actions generally expose labels such as Open menu, Back, Search, attachment, emoji, voice, and navigation.
- Native/ARIA roles appeared for textbox, checkbox, slider, menu, and menuitem.
- Inputs use labels/placeholders and visible focus styling.
- Common controls are 40–48 px.
- Headings are present in auth/settings/chat structures.
- Source contains focus-visible styles and platform/touch adaptations.

### Confirmed concerns

1. No `main`, `nav`, `aside`, or `header` semantic elements were present in the inspected conversation DOM.
2. Private-use icon-font glyphs appeared as accessibility text noise.
3. Several disabled composer controls had no accessible name.
4. One control exposed a localization-key-like name instead of user-facing language.
5. The viewport declares `maximum-scale=1.0` and `user-scalable=no`.
6. The application uses many generic `div` elements with `role=button`; native controls would provide more robust semantics.

### Not measured

- WCAG contrast ratios across every theme/wallpaper.
- Complete keyboard-only navigation and focus order.
- Screen-reader task completion.
- 200–400% browser zoom/reflow.
- Reduced-motion behavior with OS preference.
- Forced-colors/high-contrast mode.
- Live-region behavior for incoming messages and connection state.

## SEO and metadata

Confirmed public shell metadata:

- title and description;
- canonical URL;
- Open Graph and Twitter card fields;
- icons and manifest;
- mobile/Apple web-app capabilities;
- theme color;
- no-JavaScript fallback video/text.

The app's useful content is authenticated and client-rendered, so search indexing beyond the product shell is neither expected nor desirable. The canonical/meta setup is adequate for brand/link previews.

## Mobile usability

Strengths:

- full-width single-pane focus;
- 48 px primary actions/composer;
- safe-area tokens;
- large touch targets and FAB;
- explicit back behavior;
- responsive QR/auth layout;
- short-landscape detection;
- platform-specific motion.

Risks:

- disabled pinch zoom;
- dense composer icon rows when many features are active;
- viewport/virtual-keyboard complexity;
- right/left pane transitions and scroll restoration require extensive device testing;
- rich media can strain memory on older mobile browsers.

## Recommended verification plan for the internal project

1. Define budgets for cold shell, first chat, first attachment, and 30-minute session.
2. Test P75/P95/P99 sync and render times on low-end Android and older iOS.
3. Add automated accessibility checks plus manual NVDA/VoiceOver/TalkBack passes.
4. Run zoom/reflow at 200% and 400%.
5. Measure virtual-list anchor stability under incoming updates.
6. Simulate dropped/reordered worker messages and socket reconnects.
7. Test Cache Storage and IndexedDB quota/eviction.
8. Exercise multi-tab master failover.
9. Validate all icon-only controls have stable names.
10. Ship performance controls and reduced motion from the first release.
