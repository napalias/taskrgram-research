# Feature inventory

This inventory combines runtime-confirmed surfaces with features present in the version-matched public source. “Source-confirmed” means implemented in the public `12.0.38` client but not necessarily exercised against this fresh account.

## Authentication and account

- QR login with instructions and animated Telegram-branded QR.
- Phone-number login with country selector, number formatting, and keep-signed-in option.
- Login by passkey.
- Code confirmation, two-step password, and new-account registration states.
- Add-account slots using `?account=N`.
- Optional local passcode lock.
- Active-session/device management and automatic inactive-session termination policy.
- Profile editing: name, bio, birthday, username, avatar, and personal channel.

## Chat discovery and organization

- Search-first chat rail.
- Chat list with avatars, timestamp, message preview, unread/reaction/mention states, draft and status indicators.
- Stories ribbon/toggler.
- Archived chats.
- Chat folders with folder tags and left/top tab placement.
- Contacts and global/public-post search.
- New message and peer pickers.
- Multi-account switcher.

## Messaging and conversation

- Private chats, groups, channels, saved messages, bots, and service chats.
- Reply, quote, forward, edit, delete, pin, selection, context actions, and scheduled messages.
- Thread/topic and forum support.
- Message search, shared-media search, jump to date, mentions, reactions, and bottom navigation.
- Rich composer with formatting, links, code, code blocks, tables, headings, lists, details, superscript/subscript, and Markdown conversion.
- Drafts, send-as, bot commands/keyboards, inline bots, custom send menu, and prepared messages.
- Polls and to-do-list UI.
- Emoji, stickers, GIFs, custom emoji, quick reactions, effects, and paid reactions.
- Voice messages and round-video recording.
- Attachments, drag/drop, compression choice, one-time media, maps/location, contacts, files, and media.
- Message translation, language detection, code highlighting, and math rendering.

## Media

- Image/video/audio/document rendering with thumbnails and progressive loading.
- Media viewer, gallery transitions, download manager, and streaming/range requests.
- Canvas blur/thumbnail effects and animated sticker/Lottie rendering through workers/WASM.
- Audio player and waveform processing.
- Local cache clearing and automatic-download controls by media and chat category.

## Calls and real-time presence

- One-to-one voice/video calls.
- Group calls and participant lists.
- Active-call header, ringing/connect/busy/end sounds, rate-call modal.
- WebRTC and signaling support in the public source.
- Online/last-seen state and typing/update streams.

## Communities and publishing

- Group/channel profiles, member management, admin tools, permissions, invite links, requests, and moderation/report flows.
- Forum topics and topic creation/editing.
- Stories, story viewer/editor-related flows, privacy, reactions, archive, stealth mode, and statistics.
- Channel/message/story statistics.
- Sponsored messages and ad reporting/about surfaces.

## Bots, web apps, and embedded content

- Bot chats, bot commands, inline results, attach-menu bots, games, mini apps, trusted-bot confirmation, and browser modal.
- URL authentication and deep links.
- Instant View/rich content renderer.
- Web-view framing controlled by CSP and safety/confirmation modals.

## Monetization and commerce

- Telegram Premium discovery and limits.
- Stars balance, transactions, subscriptions, payments, paid reactions, and gifts.
- Gift purchase, transfer, resale, upgrade, crafting, auctions, offers, and status display in source.
- Invoice/receipt and checkout UI.
- Conditional Stripe and Smart Glocal card tokenization; Apple Pay and Google Pay credential types exist in the MTProto schema.

## Settings and personalization

- Light, dark, and system theme; custom chat wallpapers.
- 12/24-hour clock.
- Enter versus Cmd/Shift+Enter send behavior.
- Automatic text replacement.
- Notification categories, message previews, sounds, volume, badges, call notifications, and push/offline support.
- Privacy controls for phone, last seen, photo, bio, birthday, gifts, forwarding, calls, voice/video messages, messages, and group adds.
- Two-step verification, blocked users, passkeys, sensitive-content toggle, and account-deletion inactivity interval.
- Animation, transition, sticker/emoji animation, and media autoplay controls.
- Sticker suggestions, dynamic pack order, custom emoji, and quick reaction.
- Language packs, translate button, and full-chat translation.

## PWA and desktop integration

- Install prompt and dynamic manifest.
- Service-worker asset cache, push notifications, notification clicks, Web Share Target, progressive media, and downloads.
- File System/browser APIs where available.
- Tauri desktop packaging with updater, shell, notification, process, tray, window, and deep-link support.

## Interaction model worth reusing internally

1. Chat switching stays in a persistent rail on desktop.
2. Conversation and profile/settings are adjacent panes, not full page routes.
3. On mobile, the same panes become a back-stack of full-screen views.
4. Creation uses prominent contextual affordances: New Message FAB, composer, attachment menu.
5. Rare or risky actions live in menus and confirmation modals.
6. Long lists are windowed and page incrementally.
7. Update streams mutate normalized local state, enabling optimistic and live UI.
8. Device-heavy behavior is user-adjustable rather than silently degraded.
