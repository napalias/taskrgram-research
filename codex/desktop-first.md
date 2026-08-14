# Desktop-first product blueprint

## Recommendation

Build the first release as a desktop web application optimized for mouse, keyboard, large histories, file handling, and persistent context. Preserve a clean panel state model so mobile can be added later, but do not let mobile constraints weaken desktop information density or keyboard workflows.

## Recommended shell

```text
┌──────────────┬───────────────────────────────┬──────────────────────┐
│ Navigation   │ Conversation                  │ Context              │
│ 304–360 px   │ flexible, min 480 px          │ 320–424 px optional  │
│              │                               │                      │
│ Account      │ Header                        │ Profile              │
│ Search       │ Pinned/context area           │ Members              │
│ Filters      │ Virtualized message history   │ Shared files         │
│ Chat list    │ Composer                      │ Search/results       │
└──────────────┴───────────────────────────────┴──────────────────────┘
```

### Behavior

- Keep the navigation column permanently visible above approximately 925 px.
- Allow the user to resize the navigation column within sensible bounds.
- Open profile, members, files, and search in an optional right column instead of replacing the conversation.
- Center the readable message region inside a wider middle column; do not stretch message bubbles across the entire screen.
- Keep the conversation header and composer fixed while message history scrolls independently.
- Restore column widths, selected chat, open context panel, scroll anchor, and draft after reload.

## Desktop MVP

### Essential

1. Authentication and secure session management.
2. Searchable, virtualized chat list.
3. Conversation history with incremental loading in both directions.
4. Plain-text composer with drafts, reply, edit, delete, and attachments.
5. Delivery/read state and live incoming updates.
6. Desktop notifications with per-chat mute controls.
7. File drag-and-drop, paste from clipboard, upload progress, and downloads.
8. Conversation search and jump to result/date.
9. Profile/member/shared-file context panel.
10. Keyboard navigation and command palette.

### Defer until the core is reliable

- Stories.
- Animated stickers and elaborate visual effects.
- Calls and screen sharing.
- Rich document editing.
- Payments, gifts, and premium systems.
- Bots or embedded mini apps.
- Multiple accounts.
- Complex themes and wallpaper editors.

## Keyboard model

Desktop productivity should be a first-class feature rather than an accessibility afterthought.

| Shortcut | Suggested action |
|---|---|
| `Cmd/Ctrl+K` | Focus global search or command palette |
| `Cmd/Ctrl+F` | Search inside the current conversation |
| `Cmd/Ctrl+N` | Start a new conversation |
| `Cmd/Ctrl+,` | Open settings |
| `Esc` | Close the top overlay or context panel |
| `Alt/Option+↑/↓` | Move between chats |
| `Cmd/Ctrl+Shift+M` | Mute/unmute current chat |
| `↑` in an empty composer | Edit the latest outgoing message |
| `Enter` | Send, when configured |
| `Shift+Enter` | New line |

Provide visible shortcut hints in menus and a discoverable shortcut reference. Never make mouse-only context menus the sole path to an action.

## Component hierarchy

```text
DesktopAppShell
├─ NavigationRail
│  ├─ AccountMenu
│  ├─ GlobalSearch
│  ├─ ChatFilters
│  └─ VirtualChatList
├─ ConversationWorkspace
│  ├─ ConversationHeader
│  ├─ ContextBannerStack
│  ├─ VirtualMessageList
│  ├─ JumpControls
│  └─ Composer
├─ ContextPanel
│  ├─ Profile
│  ├─ Members
│  ├─ SharedContent
│  └─ ConversationSearch
└─ OverlayRoot
   ├─ Menus
   ├─ Dialogs
   ├─ Pickers
   └─ MediaViewer
```

Keep the shell, protocol/data layer, and feature panels as separate deep modules. The conversation should consume a stable view model rather than knowing how transport or persistence works.

## State model

Separate authoritative synchronized data from temporary desktop UI state:

```text
server data
├─ users
├─ conversations
├─ messages by conversation
├─ memberships and permissions
└─ delivery/read state

local durable state
├─ drafts
├─ notification preferences
├─ pane widths
├─ selected filters
└─ last-open conversation and scroll anchor

ephemeral UI state
├─ open menus/dialogs
├─ selected messages
├─ drag/drop state
├─ active searches
└─ upload progress
```

Use normalized entities and bounded viewport slices. A long-running desktop session must not place every message from every visited conversation in the DOM or an ever-growing active array.

## Desktop-specific details

### Files

- Support drag enter/leave/drop over the conversation.
- Show explicit file-versus-media treatment before upload.
- Preserve upload progress if the user switches chats.
- Make completed downloads discoverable and revealable in the operating system where supported.
- Validate size and type in both client and server.

### Notifications

- Ask for browser notification permission only after explaining the value.
- Deduplicate notifications when the app is focused or another tab is active.
- Route notification clicks to the exact conversation/message.
- Support mute durations and preview privacy.

### Multiple windows and tabs

- Start with one authoritative connection owner and synchronize secondary tabs.
- Define failover when the owner closes or sleeps.
- Prevent duplicate sends and duplicated desktop notifications.
- Use idempotency keys for outgoing messages.

### Resizing

- Left pane: suggested 304–360 px, minimum 256 px.
- Right pane: suggested 360–424 px, collapsible.
- Middle workspace: minimum approximately 480 px.
- Below the minimum combined width, close the right panel before collapsing the left panel.

## Accessibility requirements

- Use `nav`, `main`, `aside`, and `header` landmarks for the three columns.
- Provide a skip link to the active conversation and another to the composer.
- Implement logical keyboard order independent of visual z-index.
- Give every icon-only action an accessible name and tooltip.
- Preserve browser zoom through 400%.
- Respect reduced motion and forced-colors modes.
- Announce incoming messages without reading every background-chat update.
- Keep focus in the originating control when a non-modal panel opens; trap focus only in true modals.

## Performance budgets

Suggested initial budgets for the internal product:

| Milestone | JavaScript transfer | Target behavior |
|---|---:|---|
| Login shell | ≤150 KB | Interactive quickly on normal corporate hardware |
| First chat | ≤350 KB cumulative | Chat list and conversation usable |
| Attachments/search | Lazy | Loaded on first intent |
| Long session | Bounded memory | Old viewports and media released |

Track message-list render time, update-to-paint latency, search latency, reconnect time, upload throughput, worker failures, and memory after several hours.

## Suggested implementation phases

### Phase 1 — shell and data spine

- Desktop three-column shell.
- Authentication/session boundary.
- Connection Worker or typed API client.
- Normalized entities and incremental update stream.
- Virtualized chat and message lists.

### Phase 2 — dependable messaging

- Drafts, send, edit, delete, reply.
- Delivery/read state and reconnect reconciliation.
- Attachments and progress.
- Conversation search.
- Notifications.

### Phase 3 — desktop productivity

- Full keyboard model and command palette.
- Resizable/persistent panes.
- Shared-content and member panels.
- Bulk selection and moderation actions, where relevant.
- Multi-tab ownership and failover.

### Phase 4 — optional breadth

- Calls, richer media, reactions, bots, advanced editor, and mobile layouts according to actual internal demand.

## What to borrow from Telegram Web A

- Persistent chat rail and optional context column.
- Normalized update-driven state.
- Worker isolation for heavy protocol or cryptographic work.
- Virtualized lists and offset pagination.
- Independent panel transitions and scroll state.
- Service-worker support for notifications and downloads.
- User-configurable send key and animation/performance controls.

## What to simplify or improve

- Use a conventional backend-for-frontend and secure HttpOnly cookies if direct MTProto-style connectivity is unnecessary.
- Keep the MVP bundle and feature surface much smaller.
- Use semantic landmarks and SVG icons from the start.
- Retain browser zoom and build keyboard navigation before secondary features.
- Prefer intent-based lazy loading over unconditional delayed warming.
- Establish durable observability and idempotency before adding calls, commerce, or visual effects.
