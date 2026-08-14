# 04 — Feature Inventory: Telegram Web A (v12.0.38)

Audit target: `https://web.telegram.org/a/`, build `12.0.38 A`, deployed 2026-08-11 15:24:14 UTC, walked live on 2026-08-14 with a real authenticated account on a 1600×1000 desktop viewport.

This document is an inventory, not a recommendation list — but every cluster ends with a one-line verdict for **taskrgram**, an internal team-chat app for a small team, desktop-first.

## How to read this document

Every feature carries two tags.

**Evidence tier** — how we know it exists:

| Tier | Meaning |
|---|---|
| **[A] Exercised** | We drove it in the live authenticated session and saw the result. Screenshot cited. |
| **[B] Present** | Visible in the live UI (a control, a settings row, a menu item) but not actuated. |
| **[C] Source-only** | Recovered from the shipped bundle, its source maps, i18n string tables, or CSS token names. Never seen rendered. |

**Confidence** — how sure we are the claim is true as stated: **Confirmed** / **Strong inference** / **Possible** / **Unknown**.

The two are independent: a tier-C finding can be Confirmed (a literal string in a shipped chunk is a fact), and a tier-A observation can be a Strong inference (we saw a behaviour but inferred its mechanism).

---

## 1. Authentication and account model

| Feature | Tier | Confidence | Detail |
|---|---|---|---|
| QR-code login as the **default** entry state | [A] | Confirmed | "Log in to Telegram by QR Code / Open Telegram on your phone / Go to Settings > Devices > Add Device". `screenshots/01-desktop-auth-qr-code-login-screen-initial.png` |
| Phone-number login | [A] | Confirmed | `#sign-in-phone-code` (country), `#sign-in-phone-number`, `#sign-in-keep-session` checked by default. `screenshots/02-desktop-auth-phone-number-entry-form.png` |
| Client-side country inference as-you-type | [A] | Confirmed | Typing `+37063265395` rewrote the field to `+370 632 65395` and set the country to `Lithuania` with **no network request**. `screenshots/03-desktop-auth-phone-number-filled-lithuania-next-button.png` |
| Code delivered in-app to an existing session, not SMS | [A] | Confirmed | "We've sent the code to the Telegram app on your other device." `screenshots/04-desktop-auth-code-verification-screen-awaiting-code.png` |
| **No resend control** on the code screen | [A] | Confirmed | Only affordance is `i.icon-edit` back to the phone step; re-submitting re-sends. `screenshots/05-desktop-auth-code-screen-no-resend-link-visible.png`, `screenshots/06-desktop-auth-code-screen-after-resend-request.png` |
| Passkey / WebAuthn login | [B] | Confirmed | `LOG IN WITH PASSKEY` button present on both the QR and phone screens; `auth.initPasskeyLogin` is a raw TL method string in the bundle. |
| Two-factor (SRP) with animated "monkey" mascot | [C] | Confirmed | `TwoFactorSetupMonkeyIdle-*.tgs`, `TwoFactorSetupMonkeyTracking-*.tgs` fetched at runtime; `lib/gramjs/client/2fa.ts`, `Password.ts`. |
| Local passcode lock (separate from the account password) | [B] | Confirmed | "Passcode Lock" row in Privacy and Security. `screenshots/25-desktop-settings-privacy-and-security-panel.png` |
| **Multi-account, 6 slots** | [C] | Confirmed | `account1` key in `localStorage`; account-switch menu items in CSS (`.account-menu-item`); 6-slot limit from source. |
| Login-code messages spoiler-masked in the chat-list preview | [A] | Confirmed | Animated particle overlay over the digits in the preview, clear text in the bubble. Deliberate shoulder-surfing mitigation. `screenshots/08-desktop-main-layout-chat-list-and-service-chat-open.png` |
| Session persisted with **no cookie at all** | [A] | Confirmed | `document.cookie` empty; MTProto auth keys in plaintext `localStorage` (`dc1_auth_key`, `dc2_auth_key`, `dc4_auth_key`, 514 chars each). CSRF is structurally inapplicable; XSS is catastrophic. |

**Relevance to an internal team app: essential, but implement almost none of this yourself.** Use your existing SSO/OIDC provider. The one idea worth lifting is *passkey-first, password-never*, and the one anti-pattern worth avoiding is holding a long-lived bearer credential in JS-readable storage — for an internal app use an `HttpOnly` cookie or a short-lived token in memory. The no-resend-button omission is a defect, not a design; add a resend with a visible cooldown.

---

## 2. Messaging core

| Feature | Tier | Confidence | Detail |
|---|---|---|---|
| Chat list with unread badges, previews, avatars, pinning | [A] | Confirmed | `screenshots/08-desktop-main-layout-chat-list-and-service-chat-open.png` |
| Message list with sender grouping, `edited` markers, per-post view counts | [A] | Confirmed | "2.8M 👁" on channel posts. `screenshots/19-desktop-public-channel-telegramtips-message-list-rich-media.png` |
| Date separators as floating pills ("Today", "July 23", "November 25, 2024") | [A] | Confirmed | Sticky, `--z-sticky-date: 9`. |
| Bounded sliding-window history loading | [A] | Confirmed | `.Message` node count went 29 → 29 → 29 → 89 → 89 across a 40,000 px scroll; heap fell 54.6 → 52.3 MB as the old slice was released. Not node recycling. `screenshots/23-desktop-message-list-scrolled-up-virtualization-test.png` |
| Scroll-to-bottom floating action button | [A] | Confirmed | Appears on scroll-up; `--z-scroll-down-button: 12`. |
| Pinned-message bar under the header | [A] | Confirmed | `screenshots/19-desktop-public-channel-telegramtips-message-list-rich-media.png` |
| Reply, quote-reply (partial-text quoting) | [B]/[C] | Confirmed | `Reply` in the context menu (tier A); `common/quote`, `.custom-shape .EmbeddedMessage` quote tokens (tier C). |
| Threads / comments on channel posts | [C] | Confirmed | `.CommentButton`, `.CommentButton_loading` component tokens; discussion-group management in source. |
| Drafts, scheduled messages, silent send, send-as | [C] | Confirmed | `CustomSendMenu`, `FormattedDateModal`, `scheduled` state slices. |
| Message selection mode + bulk toolbar | [B] | Confirmed | `Select` in the context menu; `.MessageSelectToolbar .item`, `--z-message-select-control/-area` tokens. |
| Forwarding, incl. forward-privacy and anonymous forwards | [B]/[C] | Confirmed | `Forward` in the context menu; `.Avatar.anonymous-forwards`. |
| Spoilers (animated particle mask) | [A] | Confirmed | Seen on login-code previews; `common/spoiler` in source. |
| Polls, quizzes, poll results | [C] | Confirmed | `.KibK-O8r` poll tokens (`--poll-option-media-width`, `--percent-column-width`), `middle/message/poll`, `right/PollResults`. A stray `aria-label="AccDescrPollVoteDown"` leaked an untranslated key into the live chat header. |
| Read receipts ("Seen by") | [C] | Confirmed | `SeenByModal`. |
| One-time / view-once media | [C] | Confirmed | `modals/oneTimeMedia`, `.icon-view-once`. |
| Sponsored messages inside channels | [C] | Confirmed | `.SponsoredMessage` has its own radius overrides and `--z` handling. |
| Saved Messages (self-chat) | [C] | Confirmed | `.Avatar.saved-messages`, `saved-dialog` MessageList variant. |
| Instant View (article reader) | [C] | Confirmed | `components/iv/`, `--iv-font-size-*` token family with a user-scalable `--iv-font-size-scale`. |
| Stories (ring on avatar, viewer, stealth mode) | [C] | Confirmed | `--color-avatar-story-unread-from/-to`, `--story-ribbon-height: 5.5rem`, `--z-story-viewer: 1150`. Not present for our fresh account. |

**Relevance to an internal team app: essential (core), skip (stories, sponsored, one-time media, view-once).** The reply/quote/thread triad is the highest-value part: for a work chat, threaded replies with a visible quoted excerpt is the single feature that keeps a busy channel readable. Polls are a genuine nice-to-have for standups and scheduling. Read receipts are politically loaded in a work context — ship them off by default or not at all.

---

## 3. The composer — a document editor wearing a chat input

This is the most surprising part of the product and the part most worth studying.

The input is **not** a `<textarea>` and **not** a bare `contenteditable`. The live DOM node is:

```html
class="form-control allow-selection NKP0M5xy ProseMirror ProseMirror-focused"
```

It is TipTap 3 on ProseMirror — **27 `@tiptap/*` packages and 13 `prosemirror-*` packages** in the source maps, with `src/util/tiptap` (17 files) as Telegram's adapter layer, shipped as its own `Bundles.Editor` chunk (`editor-CL7uxqfp.js`, 294,925 B).

### 3.1 Toolbar controls read from the live DOM [A] Confirmed

| Control | Note |
|---|---|
| `Open full editor` | Expands the composer into a near-full-height editing surface. The CSS confirms the geometry: `.composer-wrapper.rich-input-expanded` defines `--rich-editor-header-height: 2.5rem`, `--rich-editor-toolbar-height`, and `--rich-input-height: calc(var(--vh,1vh) * 100 - var(--rich-input-offset))`. |
| `Insert block` | Opens the block menu below. |
| `Lists` | Bulleted / Numbered / Checklist / List options. |
| `Table` | `prosemirror-tables` + `@tiptap/extension-table`. |
| `Add Link` | `RichEditorLinkModal`, `linkifyjs`. |
| `Code block` | `@tiptap/extension-code-block-lowlight` + `lowlight` + `highlight.js` (**375 files** in the source maps) with a Telegram fork `src/lib/hljs-tl`. Distinct light/dark syntax palettes, and a *third* palette for code inside your own (outgoing) bubbles. |
| `Equation` | `temml` — LaTeX to MathML. `Temml-Local-*.css` ships as its own stylesheet. |
| `Undo` / `Redo` | `prosemirror-history`. |

### 3.2 Block vocabulary found in the live block menu [A] Confirmed

`Bulleted list` · `Numbered list` · `Checklist` · `List options` · `Heading` · `Footer` · `Quote` · `Pullquote` · `Details` · `Divider`

Inline marks confirmed from the TipTap extension list [C]: `bold`, `italic`, `strike`, `underline`, `code`, `link`, `subscript`, `superscript`, plus `hard-break` and `horizontal-rule`.

`Details` (a collapsible disclosure block) and `Pullquote` are magazine-layout primitives, not chat primitives. `Footer` likewise. Together with `Heading` and `Table` this is unambiguously an **article authoring tool that happens to live in a chat composer**.

### 3.3 Long-form articles [A] Confirmed (in-product copy)

Copy read from @TelegramTips in the live session:

> "Rich Text Editor. You can create articles and long posts of **over 32,000 characters** in a visual editor that supports dozens of formatting options like tables, lists…"

> "The editor also allows you to **generate text and formatting using privacy-conscious AI tools**."

### 3.4 AI text generation [A]/[C] Confirmed (exists), Unknown (behaviour)

The in-product copy above is tier A. The implementation is tier C and consistent: `middle/composer/AiMessageEditorModal`, `global/actions/api/ai.ts`, `aiComposeTones`, `modals/aiTonePreview`, `summaryById` in global state, and a CSS token `.WebPage--ai-tone-emoji → --custom-emoji-size: 3rem`. We never invoked it, so what it produces, where it runs, and what "privacy-conscious" means operationally are **Unknown**.

### 3.5 Other composer surfaces

| Feature | Tier | Confidence |
|---|---|---|
| Emoji / sticker / GIF picker with mode switcher | [A] | Confirmed |
| Bot menu button and bot keyboards | [C] | Confirmed (`.bot-menu`, `--icon-width`, `--icon-gap` tokens) |
| Inline query results (`@bot query`) | [C] | Confirmed (`composer/inlineResults`) |
| Drag-and-drop file drop area | [C] | Confirmed (`DropArea`, `--z-drop-area: 55`) |
| Voice note recording (`opus-recorder` + `oggToWav` worker) | [C] | Confirmed |
| Round video message recording | [C] | Confirmed (`RoundVideoRecorder`, `--z-round-video-recorder: 1160`) |
| Send with Enter vs Ctrl+Enter, configurable | [B] | Confirmed (General Settings) |
| Automatic text replacements (`--`, `<<`, `>>`) | [B] | Confirmed (General Settings) |
| Embedded reply/edit preview strip above the input | [C] | Confirmed (`.ComposerEmbeddedMessage`, `--composer-embedded-message-duration: .15s`) |

**Relevance to an internal team app: the *idea* is essential; this *implementation* is a trap.** A team app needs code blocks with syntax highlighting, links, lists, checklists, quotes and bold/italic/inline-code — that is where the value is for engineers and PMs. Tables and headings are a nice-to-have if you also want the tool to host short specs. Pullquote, Footer, Details and LaTeX are **skip** for almost every internal team. Note the cost: the editor chunk alone is 295 KB and highlight.js contributes 375 source files. Ship a *narrow* TipTap schema, not `StarterKit` plus everything. The "Open full editor" affordance — same document, bigger canvas, no separate app — is the single best idea in this cluster and is cheap to copy.

---

## 4. Attachments

The `Add an attachment` menu, read live [A] Confirmed — `screenshots/17-desktop-composer-attachment-menu-open.png`:

| Item | Comment |
|---|---|
| `Photo or Video` | Conventional. Feeds the media editor (`ui/mediaEditor`, `ImageCropper`, `CropModal`). |
| `File` | Conventional. MIME sniffing via `file-type` / `media-typer` / `content-type`. |
| `Checklist` | **Not a file.** A structured, mutable to-do object sent as a message. CSS confirms it is checkbox-driven: `.todo-list .Checkbox .Checkbox-main:before → --color-borders-input: var(--secondary-color)`. |
| `Date` | **Not a file.** Inserts a date object; backed by `CalendarModal` / `HistoryCalendar` / `FormattedDateModal`. |
| `Article` | **Not a file.** Opens the long-form editor path described in §3. |

Three of five "attachments" are structured content types rather than uploads. That is a product statement: the attach button is where you add *any* non-plain-text payload, not where you add files.

Also confirmed [C]: attach-bots (`modals/attachBotInstall`, `AttachBotRecipientPicker`), maps/location (`modals/map`, external provider choice between Google / Bing / OSM / Apple, Foursquare venue-category icons from `ss3.4sqi.net`), contacts, and polls.

**Relevance to an internal team app: essential (Photo/Video, File, Checklist), nice-to-have (Date), skip (Article, attach-bots, location).** A shared checklist inside a channel message is worth more to a working team than any amount of rich text — it is the lightest possible task object and it lives where the conversation is. For taskrgram specifically this is the closest thing in the whole product to your core proposition; treat `Checklist` as a first-class message type, not an attachment.

---

## 5. Message actions and the context menu

Right-clicking a channel post produced exactly seven items [A] Confirmed — `screenshots/21-desktop-message-right-click-context-menu-with-reactions.png`:

`Reply` · `Copy Text` · `Copy Message Link` · `Download` · `Forward` · `Select` · `Report`

Two things about the presentation, both observed:

1. A **reaction strip is attached above the menu**, so react-vs-act is a single gesture rather than two menus.
2. Opening the menu **dims the entire application except the target message**, which stays at full brightness. This is a spotlight, not a flat scrim — see §5 of the UI/UX document.

Menu composition is context-dependent [C] Confirmed — the source carries pin, edit, delete, translate, report, "select all", schedule, and paid-reaction entries that a channel-post menu does not show.

**Relevance to an internal team app: essential.** `Copy Message Link` is the one item teams under-build and over-need: a stable, permalinkable message URL is what lets chat feed tickets, docs and incident timelines. Build it on day one. `Report` is skip for an internal tool; replace it with `Flag to admin` only if you have a moderation story.

---

## 6. Reactions

| Feature | Tier | Confidence | Detail |
|---|---|---|---|
| Emoji reactions on a message, chosen state | [A] | Confirmed | Reaction strip on the context menu; `.cBNtqIXS` token block defines `--reaction-background`, `--reaction-background-hover`, `--reaction-text-color` and a separate `.xStTFP-t` (chosen) variant. |
| Distinct reaction palettes for incoming vs own bubbles | [C] | Confirmed | `--color-message-reaction` / `--color-message-reaction-own` and hover/chosen variants — 8 dedicated tokens. |
| Reactions rendered *outside* the bubble over media | [C] | Confirmed | `.theme-light .Reactions.is-outside .message-reaction → --reaction-background: var(--pattern-color)` — the reaction pill picks up the wallpaper colour when it floats over an image. |
| Reactor list ("who reacted") | [C] | Confirmed | `.ReactorListModal`, `--custom-emoji-size: 1.5rem` for its emoji. |
| Custom emoji as reactions | [C] | Confirmed | Reaction picker has `--z-reaction-picker: 10200`, above the portal menu layer. |
| **Paid reactions** (Stars) | [C] | Confirmed | `.gvRkcCmp` reaction variant hard-codes `#ffbc2e` gold; `modals/paidReaction`. |
| Reaction animation effects, separately toggleable | [B] | Confirmed | "Reaction Effects" and "Sticker Effects" toggles in Animations and Performance. `screenshots/27-desktop-settings-animations-performance-expanded-granular-toggles.png` |
| Quick-reaction (double-click / hover) | [C] | Confirmed | `.Message .quick-reaction → --custom-emoji-size: 1.75rem`. |

**Relevance to an internal team app: essential.** Reactions are the cheapest possible ack primitive and they suppress an enormous amount of "+1" noise. Ship a small fixed set (6–8), make one of them a quick-reaction on hover, and support a "who reacted" popover. Skip paid reactions and custom-emoji reactions.

---

## 7. Stickers, emoji, GIFs, custom emoji

Observed live [A] Confirmed — `screenshots/16-desktop-composer-emoji-picker-panel-open.png` and `screenshots/43-desktop-dark-theme-emoji-picker-frosted-glass-panel.png`:

- A translucent, backdrop-blurred floating panel.
- Category strip: `Recently Used` · `Emoji & People` · `Animals and nature` · `Food and drink` · `Activity` · `Travel and places` · `Objects` · `Symbols` · `Flags`.
- Bottom mode switcher: emoji / favourites / stickers / GIF / backspace.
- Panel geometry is tokenised: `--symbol-menu-width: 24rem`, `--symbol-menu-height: 22.375rem`, `--symbol-menu-footer-height: 3rem`.

| Feature | Tier | Confidence | Detail |
|---|---|---|---|
| Animated `.tgs` stickers via **tlottie WASM in a worker** (not rlottie, not lottie-web) | [C] | Confirmed | `tlottie-HZNSEMV6.wasm` fetched at runtime; renders to `ImageBitmap` and blits to canvases. Device-aware quality: `HIGH_PRIORITY_QUALITY = (IS_ANDROID \|\| IS_IOS) ? 0.75 : 1`. |
| Shared-canvas mode for many small custom emoji | [C] | Confirmed | `useCoordsInSharedCanvas`, `isSharedCanvas` — dozens of inline emoji share one canvas rather than one canvas each. |
| Video stickers as WebM `<video>` | [C] | Confirmed | `StickerView.tsx` branches `isLottie` / `isVideo` / `isStatic` with `markVideoBroken` fallback. |
| Custom emoji inline in text, size-aware | [C] | Confirmed | `--custom-emoji-size` is overridden in **~40 distinct component scopes**, from `.75rem` in a badge to `8rem` in a hero. |
| Emoji-only messages scale up by count | [C] | Confirmed | 1 emoji → `7rem`, 2 → `5.625rem`, 3 → `5.25rem`, 4 → `4.5rem`, 5 → `3.75rem`, 6 → `3rem`, 7 → `2.25rem`. |
| Emoji statuses, collectible statuses | [C] | Confirmed | `StatusPickerMenu`, `SuggestedStatusModal`, `collectibleEmojiStatuses`. |
| GIF search and saved GIFs | [B] | Confirmed | GIF mode in the picker; `GIF` tab in the channel profile. |
| Sticker set management screen | [B] | Confirmed | `screenshots/33-desktop-settings-stickers-and-emoji-sets.png` |
| iOS emoji dataset bundled | [C] | Confirmed | `emoji-data-Cp82hEFa.js`, 55,591 B. |

**Relevance to an internal team app: nice-to-have (emoji, GIF), skip (animated stickers, custom emoji).** Native emoji plus a GIF picker costs you almost nothing. A WASM Lottie renderer, a worker pool, a shared-canvas compositor and per-device quality tuning is a multi-engineer-month investment to make cartoons move; there is no version of an internal team-chat business case that justifies it. The graduated emoji-only sizing (1 emoji → 112 px) is a delightful 20-line detail worth stealing outright.

---

## 8. Media viewer and the media pipeline

| Feature | Tier | Confidence | Detail |
|---|---|---|---|
| Media viewer is a **native `<dialog aria-modal="true">`** | [A] | Confirmed | `<dialog open id="MediaViewer" aria-modal="true" class="shown open">`. Escape closed it in one press, verified. `screenshots/44-desktop-media-viewer-native-dialog-element-open.png` |
| Zoom / pan | [C] | Confirmed | `mediaViewer/` source. |
| Video scrub-preview storyboard | [C] | Confirmed | `src/util/media/StoryboardParser.ts`. |
| Video streams via **Service-Worker-synthesised HTTP 206** | [A] | Confirmed | `206 GET /a/progressive/document5109473995049145023` observed at runtime. The SW invents Range responses so a native `<video>` can seek content that only exists behind MTProto. |
| MSE path, **Safari only** | [C] | Confirmed | `useStreaming.ts` bails unless `IS_SAFARI`. |
| No HLS, no DASH, no ffmpeg.wasm, no custom player | [C] | Confirmed | Zero hits for `hls`, `ffmpeg`. |
| All media arrives over the MTProto WebSocket, not HTTP | [A] | Confirmed | 660 frames received = 10,026,931 B against 310 sent = 85,464 B — a **117:1 asymmetry**. 102 of 737 HTTP responses had an empty host, i.e. `blob:` URLs. |
| Photo/video editor with crop | [C] | Confirmed | `ui/mediaEditor/`, `CropModal`, `AvatarEditable`. |
| Aggressive client-side media cache | [A] | Confirmed | 16.3 MB of 16.6 MB total storage was Cache Storage after a few minutes on one channel. TTL 5 days, hourly LRU sweep, `MEDIA_CACHE_MAX_BYTES = 512 * 1024`. |

**Relevance to an internal team app: essential (viewer), skip (the transport).** You have HTTP and a CDN; Telegram does not. Serve media by URL with signed links and `Range` support from your object store and you get streaming, seeking and caching for free. What *is* worth copying is the viewer being a real `<dialog>` — see the UI/UX document. Also worth copying: the 5-day TTL + hourly sweep on Cache Storage, which is ~40 lines and stops a chat client from silently consuming a gigabyte of a user's disk.

---

## 9. Voice and video messages

| Feature | Tier | Confidence | Detail |
|---|---|---|---|
| Voice notes, Opus-encoded in a worker | [C] | Confirmed | `opus-recorder`, `oggToWav` worker. |
| Waveform rendering | [C] | Confirmed | `generateWaveform.ts`, `waveform.ts`. |
| **Voice transcription** button | [C] | Confirmed | Two dedicated tokens: `--color-voice-transcribe-button` (`#e8f3ff` light / `#2a2a3c` dark) and `--color-voice-transcribe-button-own`. A token pair this specific means a shipped, styled control. |
| Round video messages | [C] | Confirmed | `RoundVideoRecorder`, `--z-round-video-recorder: 1160`. |
| Audio player as a floating "island" that shifts when the right column opens | [C] | Confirmed | `#Main.right-column-open .AudioPlayer.island-player → --player-shift-x: calc(var(--right-column-width) / -2)`. |
| Playback-speed control | [C] | Confirmed | `.AudioPlayer .playback-button.applied`. |
| Music metadata parsing (ID3 / Vorbis / MP4) | [C] | Confirmed | `music-metadata`, 91 files in the source maps. |

**Relevance to an internal team app: nice-to-have (voice notes), skip (round video, music player).** Voice notes are genuinely useful for async teams across time zones — but only if you also ship transcription, otherwise you have made your chat log unsearchable. If you cannot afford transcription, do not ship voice notes.

---

## 10. Calls

| Feature | Tier | Confidence | Detail |
|---|---|---|---|
| 1:1 audio/video calls | [C] | Confirmed | `calls/phone`, `lib/vibecalls/phone`, WebRTC + signalling messages, `sctpSignaling.ts`. |
| Group calls / voice chats | [C] | Confirmed | `calls/group`, `.AIHiBm95` defines a whole dark call-panel sub-theme (`--group-call-panel-color: #212121`, `--green-button-color`, `--blue-button-color`, `--purple-button-color`, `--gradient-blue`). |
| Screen sharing | [C] | Confirmed | `IS_SCREENSHARE_SUPPORTED`, `startSharingScreen`. |
| Noise suppression, speaker toggle, per-participant volume | [C] | Confirmed | `.participant-menu .volume-control.{low,medium,normal,high}` each with its own `--range-color`. |
| Video grid layout engine | [C] | Confirmed | `useGroupCallVideoLayout`. |
| Call header strip in the main layout | [C] | Confirmed | `.has-call-header → --call-header-height: 2rem`. |
| Calls loaded as an isolated chunk | [C] | Confirmed | `Bundles.Calls`, `calls-D3LYymlO.css` (14,857 B). |
| Privacy control for who may call you | [B] | Confirmed | In the ten visibility settings. |

**Relevance to an internal team app: skip.** You already pay for Zoom/Meet/Teams. Building WebRTC signalling, TURN infrastructure, and a video layout engine to replace a tool your team already has open is the clearest negative-ROI item in this entire inventory. Ship a "start a call" button that deep-links to your existing provider and put the link in the channel.

---

## 11. Chat types: channels, groups, forums/topics, "communities"

| Type | Tier | Confidence | Detail |
|---|---|---|---|
| **Channel** (broadcast, join CTA replaces composer) | [A] | Confirmed | @TelegramTips, 11.19 M subscribers. `Join Channel` CTA pinned where the composer would be, plus a gift button beside it. `screenshots/19-desktop-public-channel-telegramtips-message-list-rich-media.png` |
| **Group** creation, two-step wizard | [A] | Confirmed | Step 1 "Add Members" (searchable picker, empty state "Sorry, nothing found."), step 2 "New Group / Group name" form, primary action a `FloatingActionButton` with `aria-label="Continue To Group Info"`. `screenshots/35-desktop-new-group-add-members-empty-contact-picker.png`, `screenshots/37-desktop-new-group-info-form-step2-name-and-photo.png`, `screenshots/38-desktop-new-group-info-form-name-field.png` |
| New-chat entry menu: `New Channel` / `New Group` / `New Message` | [A] | Confirmed | `screenshots/34-desktop-new-message-menu-new-channel-group-options.png` |
| **Forums / topics** | [C] | Confirmed | `left/main/forum/ForumPanel`, `right/{CreateTopic,EditTopic,ManageTopic}`, `chats.topicsInfoById`. CSS is rich: `.Avatar.forum → --radius: var(--border-radius-forum-avatar)` (33.3333%, a squircle rather than a circle), `--z-forum-panel: 13`, seven `--color-topic-*` hues, `--color-forum-hover-unread-topic` states. |
| Group management: admins, permissions, invite links, join requests, discussion group, reaction settings | [C] | Confirmed | `right/management`, `modals/{inviteViaLink,chatInvite,leaveGroup,deleteMember,rank}`. |
| Similar channels, personal channels | [B] | Confirmed | `Similar Channels` is a live tab in the channel profile. |
| Channel statistics (`lovely-chart`) | [C] | Confirmed | `.lovely-chart--container` has a full light/dark sub-palette. |
| Boosts, giveaways, monetisation | [C] | Confirmed | `modals/boost`, `GiveawayModal`, `AboutMonetizationModal`. |
| **"Communities"** | — | **Unknown** | No feature by that name appears in the live UI, the settings tree, the CSS token space or the recovered source tree of this build. The nearest constructs are channel + linked discussion group, forums with topics, and shareable folders (`modals/chatlist`). If "communities" is expected in taskrgram, it must be designed, not copied. |

**Relevance to an internal team app: essential (groups, topics), nice-to-have (channels), skip (boosts, giveaways, monetisation, statistics).** The channel/group split maps cleanly onto announce-only vs discussion, which every team needs. **Forums/topics is the feature to study hardest**: a topic is a named sub-thread with its own unread state inside one group, which is exactly how a small team avoids the "one channel per micro-topic" sprawl that eventually kills a Slack workspace. The squircle forum avatar and per-topic colour are cheap identity cues that make topics scannable.

---

## 12. Folders

Read live from Settings → Chat Folders [A] Confirmed — `screenshots/30-desktop-settings-chat-folders-configuration.png`:

| Control | Detail |
|---|---|
| Folder list with "Sort chats into folders" description | Empty state: **"You have no folders yet"**. |
| **`Show Folder Tags`** | Coloured tags on chat rows indicating folder membership. |
| **`Tabs View: "Tabs on the left" \| "Tabs at the top"`** | The folder rail can be a vertical icon rail or a horizontal tab strip. |

The vertical rail is tokenised at `--folders-sidebar-width: 5rem` (80 px), and it *changes the left column's own sizing rule*:

```css
.folders-sidebar-visible #LeftColumn { --left-column-max-width: calc(33vw - var(--folders-sidebar-width)); }
```

Related [C] Confirmed: shareable folders (`modals/chatlist`), `util/folderManager.ts` (a denormalised, incrementally-maintained index of folder membership and unread counts, `UPDATE_THROTTLE = 500`), `util/folderIconMap.ts`, and a custom-emoji folder icon picker (`.settings-folders-icon-picker → --custom-emoji-size: 1.5rem`).

**Relevance to an internal team app: essential.** A team of 30 people will have 40 channels within a year and the flat list stops working at about 15. Two implementation notes worth stealing: (1) offering *both* rail-left and tabs-top is a real accessibility/preference win for near-zero cost, since the folder set is the same data rendered two ways; (2) `folderManager` maintaining a denormalised membership index outside the render path is why the chat list stays fast — a naive implementation re-filters every chat on every update.

---

## 13. Search

### 13.1 Global search — faceted [A] Confirmed

Tabs read live from the search screen (`screenshots/18-desktop-global-search-results-chats-channels-messages.png`):

`Chats` · `Channels` · `Apps` · `Posts` · `Media` · `Links` · `Files` · `Music` · `Voice`

Results are sectioned into **"Chats and Contacts" / "Global Search" / "Messages"**, i.e. local-then-remote, and public channel results carry subscriber counts.

The `.LeftSearch` component gets its own token scope (`--left-search-inset: 1rem`, `--fade-mask-top: 0`), and search-result rows carry a hover/focus placeholder colour override (`.LJYDZ1gw .SearchInput → --color-placeholders: var(--color-text-secondary)`).

### 13.2 In-chat search [A] Confirmed

A dedicated panel opens over the message list (`screenshots/24-desktop-in-chat-search-panel-open.png`), stacked at `--z-local-search: 12`. Match highlighting has explicit tokens (`--color-selection-highlight: #3993fb`, and in dark mode `.theme-dark .message-content .matching-text-highlight → --color-text: #000` so highlighted text flips to dark-on-bright).

Also [C] Confirmed: search by sender, by date (`HistoryCalendar`), and saved-messages/topic-scoped search.

**Relevance to an internal team app: essential.** Faceting by type is the highest-leverage part — "the PDF someone posted in March" is the single most common real query in a work chat, and a Files tab answers it in one click where a text search never will. Ship at minimum `Messages / Files / Links / People`. The local-then-remote sectioning is also worth copying: it makes the first result appear instantly from cache while the server query is still in flight.

---

## 14. Profile / right column

Tabs exposed on a channel profile, read live [A] Confirmed — `screenshots/22-desktop-right-column-channel-profile-info-panel.png` and `screenshots/42-desktop-dark-theme-right-column-profile-media-grid.png`:

`Posts` · `Gifts` · `Media` · `Files` · `Links` · `Music` · `GIF` · `Voice` · `Similar Channels`

Structurally the right column is a `Transition` container (`#RightColumn>.Transition → --slide-background-color: var(--color-background-secondary)`), so profile → media → member list are slides within one panel rather than separate routes. Secondary tabs are deliberately delayed: `right/Profile.tsx` carries the comment *"@optimization Used to delay first render of secondary tabs while animating"*.

**Relevance to an internal team app: essential, in reduced form.** `Media / Files / Links` scoped to a single channel is the answer to "where did we put that thing" and it is a pure re-query of data you already have. Cut `Gifts`, `GIF`, `Similar Channels`, and probably `Music`. Add `Members` and `Pinned`.

---

## 15. Settings — the complete tree

Read from the live UI [A] Confirmed — `screenshots/11-desktop-settings-main-menu-list.png`:

```
Account                     Username, Bio, Birthday
General Settings            Wallpaper, Theme, Keyboard
Notifications               Sounds, Calls, Badges
Privacy and Security        Last Seen, Password, Blocked Users
Data and Storage            Media Download, Cache
Chat Folders                Sort chats into folders
Animations and Performance  Animations, Transitions, Autoplay
Stickers and Emoji          Stickers, Emoji, Reactions
Language                    English, Translate Messages
Active Sessions             Sessions, Automatically terminate  [4]
Telegram Premium
My Stars                    0
My Gram                     0
Send a Gift
Ask a Question / Telegram FAQ / Privacy Policy
```

### 15.1 General Settings [A] Confirmed

`screenshots/12-desktop-settings-general-theme-wallpaper-options.png`, `screenshots/13-desktop-settings-chat-wallpaper-picker-gallery.png`, `screenshots/14-desktop-settings-general-dark-theme-selected.png`

| Control | Values observed |
|---|---|
| Message Font Size | Slider, default **16** (matches `--message-text-size: 16px`) |
| Chat Wallpaper | Gallery picker + `Upload image`, `Set a color`, `Reset to default`, **`Blurred`** toggle, **`Pattern Intensity`** = 75, "Colors, Gradients & Patterns" |
| Theme | `Light` / `Dark` / **`System`** |
| Time format | 12-hour / 24-hour |
| Send with | `Enter` / `Ctrl+Enter` |
| Automatic text replacements | "Convert --, << and >> as you type" |

### 15.2 Animations and Performance [A] Confirmed

Covered in depth in the UI/UX document — it is the most decision-relevant screen in the product. Summary: a three-stop slider `Power Saving | Nice and Fast | Lots of Stuff`, then a section literally headed **"Resource-Intensive Processes"** with three expandable groups and **ten individually-toggleable interface animations**, with the two `backdrop-filter` effects broken out separately. `screenshots/26-desktop-settings-animations-and-performance-toggles.png`, `screenshots/27-desktop-settings-animations-performance-expanded-granular-toggles.png`

### 15.3 Stickers and Emoji [B] Confirmed

Installed sticker sets, emoji sets, reaction configuration. `screenshots/33-desktop-settings-stickers-and-emoji-sets.png`

### 15.4 Account [B] Confirmed

Username, Bio, Birthday.

**Relevance to an internal team app: essential (theme, font size, send-key), nice-to-have (wallpaper), skip (stickers/emoji management).** Font size and send-key are the two settings users actually change and both are one-line. Note what is *absent* from this settings tree and would be mandatory in an internal tool: no status/away message, no working-hours or focus mode, no per-channel mute schedule.

---

## 16. Privacy and security — all ten visibility controls

Read live [A] Confirmed — `screenshots/25-desktop-settings-privacy-and-security-panel.png`:

**Ten (in fact twelve) separate visibility settings:**

| # | Setting |
|---|---|
| 1 | Phone number |
| 2 | Last seen (and online status) |
| 3 | Profile photos |
| 4 | Bio |
| 5 | Birthday |
| 6 | Gifts |
| 7 | Forward link (whether forwards of your messages link back to you) |
| 8 | Calls |
| 9 | Voice and video messages |
| 10 | Who can message you |
| 11 | Who can add you (to groups) |
| 12 | "Show 18+ Content" |

**Plus, on the same screen:**

- Blocked Users
- Passcode Lock
- Two-Step Verification
- **Passkeys**
- **"Window title bar → Show chat name"** — a desktop-app affordance leaking into the web client (see §24)
- Account auto-deletion: "If away for 18 months"

Each visibility setting is Everybody / My Contacts / Nobody with per-user exception lists [C] Confirmed.

**Relevance to an internal team app: nice-to-have, radically simplified.** In an internal tool the directory is the org chart; most of these controls are meaningless. The three that survive are **last-seen/online visibility** (genuinely contentious in a work context, and the one people ask for), **who can DM you**, and **blocked users**. The pattern worth copying is not the ten settings but the *shape*: one uniform Everybody/Team/Nobody control with an exception list, rendered by one component, so adding a new visibility axis is a data change and not a UI project. Explicitly skip 18+ content, birthday, gifts, forward-link.

---

## 17. Notifications

Read live [A] Confirmed — `screenshots/31-desktop-settings-notifications-sounds-badges.png`:

| Control | Detail |
|---|---|
| Web Notifications | Master toggle |
| **Offline Notifications** | Separate toggle — push while the tab is closed, distinct from in-page notifications |
| Sound volume | 0–10 slider |
| Per-category toggles | Private chats / Groups / Channels, each with notification + **message preview** sub-toggles |
| Badges | Unread badge behaviour |
| Calls | Call-notification behaviour |

Supporting evidence [A]: `notification.mp3` (11,180 B) is fetched **on the login screen**, before any notification can possibly exist — a small waste, but it means the sound is warm on first message. `--z-notification: 1700`, `.Notification → --color-toast-action: var(--color-primary)`.

**Relevance to an internal team app: essential.** Separating "offline/push" from "in-app" is the right axis and most teams get it wrong by conflating them. The **message-preview** sub-toggle per category is the detail to copy: people screen-share, and a preview-suppressing switch that is independent of the notification switch is what makes a chat client safe to have open in a meeting. Add what Telegram lacks: mute schedules and per-channel overrides.

---

## 18. Data and storage

Read live [A] Confirmed — `screenshots/28-desktop-settings-data-and-storage-cache-controls.png`:

**Auto-download matrices** — three independent matrices, each with four chat-type columns:

| Media type | Contacts | Other Private Chats | Group Chats | Channels |
|---|---|---|---|---|
| Photos | ✓/✗ | ✓/✗ | ✓/✗ | ✓/✗ |
| Videos and GIFs | ✓/✗ | ✓/✗ | ✓/✗ | ✓/✗ |
| Files | ✓/✗ | ✓/✗ | ✓/✗ | ✓/✗ |

Plus a **maximum file size cap** ("up to 10MB") and **"Clear Media Cache"**.

Underlying measured behaviour [A] Confirmed: five Cache Storage buckets are provisioned on the *unauthenticated* login screen (`tt-media`, `tt-media-avatars`, `tt-media-progressive`, `tt-lang-packs-v52`, `tt-assets`), four of which cannot be populated until a session exists. `tt-assets` grows from 1 to 42 entries between cold load and reload — the service worker precaches the shell after first paint.

**Relevance to an internal team app: nice-to-have, drastically reduced.** A 3 × 4 matrix is twelve decisions to ask a user to make about a thing they do not think about. On desktop with corporate bandwidth this is close to pointless. Ship one control — "Download media automatically: always / on Wi-Fi / never" — plus **"Clear cache", which is essential** and which teams routinely forget until a support ticket arrives about a 2 GB browser profile.

---

## 19. Active sessions

Read live [A] Confirmed — `screenshots/29-desktop-settings-active-sessions-device-list.png`. The "THIS DEVICE" row verbatim:

```
Chrome 140
Telegram Web 12.0.38 A, macOS
- United States
```

Other live sessions on the same account, showing the cross-platform identity string:

```
Chrome 151          Telegram Web 12.0.38 A, macOS       Klaipėda, Lithuania
Chrome 150          Telegram Web 12.0.38 A, macOS       Klaipėda, Lithuania
Samsung Galaxy S8   Telegram Android 12.9.1, Android 9 P (28)   Klaipėda, Lithuania
```

Also present: per-session terminate, terminate-all, and **"Automatically terminate old sessions"** with a period selector.

**Relevance to an internal team app: essential.** Cheap to build, disproportionately reassuring, and the first thing a security reviewer asks for. Include app version and approximate location, and make "terminate all other sessions" a single obvious button.

---

## 20. Language and translation

Read live [A] Confirmed — `screenshots/32-desktop-settings-language-selection-list.png`:

- Full interface language list: English, Arabic, Belarusian, Catalan, Chinese Simplified, Chinese Traditional, Croatian, Czech, Dutch, Finnish, French, German, Hebrew, Hungarian, … (RTL languages included).
- **"Show Translate Button"** — per-message translate affordance.
- **"Translate Entire Chats"** — whole-conversation translation.

Implementation notes [C] Confirmed and worth knowing:

- Strings live on the Telegram Translation Platform; the repo carries only `fallback.strings` + generated `initialStrings`, and language packs are fetched live from `https://translations.telegram.org/en/weba`.
- Types are **generated from the strings file** (`npm run lang:ts` → `LangKey`, `LangVariable`), so a missing or renamed key is a **compile error**, not a runtime `???`.
- Runtime formatting uses native `Intl.PluralRules`, `Intl.DisplayNames`, `Intl.NumberFormat`, `Intl.ListFormat`.
- Language detection for the translate button uses **fastText compiled to WASM** (`fasttext-wasm-zrRkeJ3U.wasm`, fetched at runtime) — i.e. language ID happens on-device.
- Lang packs cached in IndexedDB (`langpack-` prefix) plus Cache Storage (`tt-lang-packs-v52`); only the master tab fetches, then broadcasts.
- RTL is first-class: `lang.isRtl`, dedicated `slideRtl` / `slideOptimizedRtl` transitions, and an RTL-aware marquee direction token.

Notable contradiction [C] Confirmed: `<html lang="en" translate="no" class="notranslate">` plus `<meta name="google" content="notranslate">` — the app suppresses *browser* translation while shipping its own.

**Relevance to an internal team app: nice-to-have (i18n), skip (message translation) unless your team is genuinely multilingual.** The one practice to copy regardless of whether you localise: **generate the key type from the string table so missing keys break the build.** It costs an afternoon and eliminates an entire class of shipped bug — as demonstrated by this very app leaking `aria-label="AccDescrPollVoteDown"` into the live chat header, which is exactly the failure that type generation prevents.

---

## 21. Premium, Stars, Gram, gifts, mini-apps

| Cluster | Tier | Confidence | Detail |
|---|---|---|---|
| Telegram Premium | [B] | Confirmed | Settings row; `main/premium/{PremiumMainModal,previews}`, `PremiumLimitReachedModal`, `src/limits.ts`. `--premium-gradient: linear-gradient(84.4deg, #6c93ff -4.85%, #976fff 51.72%, #df69d1 110.7%)` is re-declared in **six** component scopes. |
| My Stars (balance 0) | [B] | Confirmed | `--stars-gradient: linear-gradient(90deg, #fa0 0%, #ffcd3a 100%)`, `--color-stars: #fa0`, `.Button.stars` variant with `#ffb727` (and a dark override `#cf8920`). |
| My Gram (balance 0) | [B] | Confirmed | Settings row only; no further evidence in the CSS token space. |
| Send a Gift | [B] | Confirmed | Four rarity tiers each with a colour + background pair: uncommon `#40a920`, rare `#11aabe`, epic `#955cdb`, legendary `#bf7600`. `modals/gift/` has **14 sub-folders** (auction, craft, info, locked, message, offer, preview, recipient, resale, status, transfer, upgrade, value, withdraw). |
| Gift auctions and resale | [C] | Confirmed | `giftAuctionByGiftId`, `.message-content.gift,.message-content.auction → --max-width: 18rem`. |
| Paid messages, paid reactions | [C] | Confirmed | `common/paidMessage`, `PAID_MESSAGES_PURPOSE`, `--paid-stars-padding`. |
| Payments: Stripe + Smart Glocal | [C] | Confirmed | `https://api.stripe.com/v1/tokens` and `https://tgb.smart-glocal.com/cds/v1/tokenize/card` are the only third-party endpoints in the entire bundle graph outside `*.telegram.org`. |
| TON wallet deep-linking | [C] | Confirmed | CSP `frame-src` whitelists **11 wallet URL schemes** (`tonkeeper-tc:`, `mytonwallet-tc:`, `safepal-tc:`, `bitkeep:`, `imtokenv2:`, `bybitapp:`, `echooo:`, `bnc:`, `nicegram-tc:`, `tonkeeper-pro-tc:`) — yet **no TON SDK is bundled**. Deep-link/iframe only. |
| **Mini-apps** (bot Web Apps) in an in-app browser | [C] | Confirmed | `modals/browser` with `BrowserModal` + `MinimizedBrowserModal` (minimisable in-app web view), `botAppPermissionsById`, `trustedBotIds`, `modals/urlAuth`, `modals/emojiStatusAccess`, `modals/locationAccess`. `--color-web-app-browser` is a themed token (`#FFFFFFBB` light / `#0303038F` dark). |
| **Mini-app theme contract: 14 tokens** exposed to third-party bot apps | [C] | Confirmed | `bg_color`, `text_color`, `hint_color`, `link_color`, `button_color`, `button_text_color`, `secondary_bg_color`, `header_bg_color`, `accent_text_color`, `section_bg_color`, `section_header_text_color`, `subtitle_text_color`, `destructive_text_color`, `section_separator_color`. |

**Relevance to an internal team app: skip (all monetisation), nice-to-have (mini-apps).** Premium/Stars/Gram/gifts/auctions is a consumer-economy surface with zero internal analogue; it is also, by folder count, one of the largest parts of this codebase. The genuinely transferable idea is the **mini-app theme contract**: a fixed, tiny, documented set of 14 colour tokens handed to embedded third-party UI so it inherits the host theme. If taskrgram ever embeds an internal dashboard, a Grafana panel or a deploy tool as a tab, publish exactly that kind of contract — 14 tokens, not 566.

---

## 22. Passkeys

| Feature | Tier | Confidence | Detail |
|---|---|---|---|
| Passkey login from the auth screen | [B] | Confirmed | `LOG IN WITH PASSKEY` on both the QR and phone screens. `screenshots/01-desktop-auth-qr-code-login-screen-initial.png` |
| Passkey management in settings | [B] | Confirmed | "Passkeys" row under Privacy and Security. |
| Implementation | [C] | Confirmed | `modals/passkey`, `SettingsPasskeys`, `util/browser/passkeys.ts`, `gramjsBuilders/passkeys.ts`, and the TL method `auth.initPasskeyLogin` as a literal string in the bundle. |

**Relevance to an internal team app: essential — but delegate it.** WebAuthn belongs in your IdP, not in your chat app. What the chat app should copy is the *placement*: passkey login offered as a peer of the primary method on the first screen, not buried in an "advanced" flow.

---

## 23. Multi-tab

| Fact | Tier | Confidence |
|---|---|---|
| `tt-multitab_1` key exists in `localStorage` from the very first unauthenticated load | [A] | Confirmed |
| One elected master tab owns the MTProto connection and the worker; others proxy | [C] | Confirmed |
| Cross-tab coordination over `BroadcastChannel` + a `SharedWorker` | [C] | Confirmed |
| Per-tab UI state (`byTabId`) is architecturally separated from shared global state | [C] | Confirmed |
| Version mismatch between tabs triggers a hard reload of the stale tab | [C] | Confirmed — `case 'requestGlobal': { let {appVersion:t}=e; if(t!=='12.0.38'){window.location.reload(); return} … }` |
| `navigator.locks` is a hard compatibility requirement (feature-gated in `compatTest.js`) | [C] | Confirmed |
| A custom ESLint plugin (`tt-multitab`) enforces `tabId` threading through actions | [C] | Confirmed |

**Relevance to an internal team app: essential to decide early, cheap only if decided early.** People keep chat open in three tabs. The authors' own summary is the right one: *"Multi-tab is an architecture decision, not a feature… Retrofitting this later is very painful."* You may not need a SharedWorker, but you do need to answer, before you write the state layer: which state is per-tab (open chat, scroll position, draft focus) and which is shared (unread counts, connection, notification suppression)? Getting that split wrong produces duplicate notifications and unread counts that fight each other.

---

## 24. Desktop (Tauri) coupling

The web bundle **carries the Rust-desktop bridge**. From the source maps [C] Confirmed:

`@tauri-apps/api` plus plugins `shell`, `notification`, `updater`, `process` — 10 files.

This surfaces in the shipped CSS as a platform class:

```css
body.is-tauri { --custom-cursor: default; --window-controls-width: 5rem; }
```

`--window-controls-width` is `0rem` in the browser and `5rem` (80 px) in the desktop shell, reserving space for native traffic-light/window buttons in the header. And the **"Window title bar → Show chat name" toggle appears in the *web* Privacy and Security screen** [A] Confirmed — a desktop-only setting leaking into the web UI, i.e. one settings tree serving both shells with imperfect gating.

Runtime self-update is also shared: `version.txt?<epoch-ms>` is polled and compared against the hardcoded `12.0.38`, setting `isAppUpdateAvailable` [A] Confirmed (request observed live).

Observed platform classes on `<body>` in our session [A]: `is-pointer-env with-message-blur is-linux`. The full known set includes `is-ios`, `is-macos`, `is-android`, `is-tauri`, `is-linux`, `is-pointer-env`, and each one changes real tokens — e.g. `body.is-ios` switches `--border-radius-messages` from `.9375rem` to `1rem` and `--font-weight-semibold` from 500 to 600.

**Relevance to an internal team app: nice-to-have.** One codebase, two shells, distinguished by body classes and a handful of tokens, is a genuinely good pattern and far cheaper than Electron-plus-a-fork. If you ever want a desktop build for global hotkeys, tray unread counts and native notifications, design for it now by keeping platform variance in CSS custom properties rather than in component branches. The leaking title-bar setting is the warning: gate platform-specific settings on the platform class, not on hope.

---

## 25. What is measurably *absent*

Worth recording, because absences are design decisions too. All [C] Confirmed by exhaustive bundle search.

| Absent | Consequence |
|---|---|
| React / Preact | Custom framework (Teact). |
| webpack | Vite + **Rolldown**; the entire bundler runtime is **716 bytes**. |
| Sentry or any crash reporter | No error telemetry at all. |
| Google Analytics, gtag, Amplitude, Mixpanel, Segment, PostHog | **Zero third-party analytics.** Confirmed at runtime too: 737 HTTP responses, **zero third-party hosts** except three `t.me`/`telegram.me`/`telegram.dog` deep-link beacons. |
| Any CDN, font provider, or ad network | Self-hosted `nginx/1.30.1`, round-robin `x-served-by: meta424xxxx` node pool. |
| lodash, moment, dayjs, axios, zod | No general-purpose utility layer. |
| Workbox | Hand-written service worker (6 files). |
| ffmpeg.wasm, HLS, DASH | Native `<video>` + SW Range responses only. |
| A TON SDK | Despite 11 wallet schemes in CSP. |

**Relevance to an internal team app: adopt the analytics restraint selectively.** Zero telemetry is a positioning choice Telegram can afford and you probably cannot — you will want crash reporting at minimum. But the *supply-chain* discipline is copyable and valuable: a chat client with no lodash, no moment, no axios and no analytics vendor has a dramatically smaller attack surface and a dramatically shorter dependency-review meeting.

---

## 26. Minimum viable feature set for taskrgram

Everything above, reduced to what a small internal team actually needs, in build order.

**Tier 1 — without these you do not have a product**

1. Channels (announce) and groups (discussion), with **topics inside groups**.
2. Message list with sender grouping, date pills, edited markers, and a bounded sliding-window history loader (§2 — 29→89 nodes, not a virtualiser).
3. Composer with a **narrow** rich-text schema: bold, italic, inline code, link, bullet/numbered list, **checklist**, quote, **code block with syntax highlighting**.
4. Reply with visible quoted excerpt.
5. Reactions, small fixed set, with a hover quick-reaction.
6. Context menu: Reply, Copy Text, **Copy Message Link**, Download, Forward, Select, Edit, Delete, Pin.
7. Faceted search: Messages / Files / Links / People, local-then-remote.
8. Notifications with the offline/in-app split and an independent **message-preview** toggle per category.
9. Media viewer as a native `<dialog>`.
10. Light/dark/system theme.

**Tier 2 — the difference between usable and good**

11. Folders, with both rail-left and tabs-top presentations.
12. Right-column Media / Files / Links / Pinned / Members per channel.
13. Checklist as a first-class message type.
14. Active sessions with terminate-all.
15. Message font size and Enter-vs-Ctrl+Enter.
16. Clear cache, plus a TTL/LRU sweep on the media cache.
17. Voice notes **only if** paired with transcription.

**Tier 3 — deliberately not building**

Calls · stories · stickers and animated emoji · gifts, Stars, premium, payments · sponsored messages and ads · translation · TON, wallets, mini-app marketplace · the ten-axis privacy matrix · the 3 × 4 auto-download matrix · article/pullquote/details/LaTeX authoring · statistics and boosts.

The honest summary of the ratio: **an internal team app needs roughly 10–15% of this surface**, and roughly 80% of the engineering value in the other 85% is in the *infrastructure* (worker-owned protocol, phase-batched DOM, animation gating, multi-tab election, SW media) rather than the features.
