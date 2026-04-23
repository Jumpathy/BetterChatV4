# BetterChat V4 — Codebase Overview

A modular Roblox chat system that replaces the default chat with rich text, markdown formatting, custom emojis, message reactions, bubble chat, context menus, and multilingual localization.

---

## Architecture Overview

```
Config/                         ← Custom emoji definitions + server bootstrap
Loader/MainModule/
  init.lua                      ← Server entry point
  shared/                       ← Shared utilities (both sides)
  server/                       ← Server-only logic
  client/                       ← Client UI + event handling
```

The system is split into a **server side** (channel management, filtering, message reconstruction) and a **client side** (UI rendering, input handling, bubble chat). They communicate via a thin networking abstraction over Roblox RemoteEvents/RemoteFunctions.

---

## File Reference

### Config

| File | Purpose |
|------|---------|
| `Config/init.lua` | Defines custom emoji mappings (e.g. `:troll:` → asset ID). No dependencies. |
| `Config/Loader/init.server.lua` | Server entry point. Requires MainModule and passes the config to it to start the system. |

---

### Shared Utilities (`shared/`)

These modules are used by both server and client.

#### `shared/network.lua`
Networking abstraction over RemoteEvents and RemoteFunctions. Hides raw Roblox networking behind a clean API so callers don't deal with RemoteEvent/RemoteFunction directly.

- **Server side**: `newEvent`, `newFunction`, `newUpdater`, `fireClients`, `update`
- **Client side**: `fire`, `invoke`, `onClientEvent`, `hookUpdater`
- The "updater" pattern (`newUpdater` / `hookUpdater`) is a bi-directional request-response: client subscribes and immediately requests current state, server pushes updates on change.

#### `shared/signal.lua`
Custom signal/event implementation similar to `RBXScriptSignal`. Batched, yield-safe, uses coroutine reuse.

- `Signal.new()` → `Connect`, `Fire`, `Wait`, `Once`, `DisconnectAll`
- Used throughout the codebase wherever Roblox's native events aren't sufficient (e.g. cross-module communication).

#### `shared/xml.lua`
Minimal XML parser. Converts XML strings into nested Lua tables. Used for parsing RichText tags inside messages.

#### `shared/md5.lua`
MIT-licensed MD5 hashing library. Present for integrity/ID purposes but not central to the main chat flow.

---

### Text Formatting (`shared/formatting/`)

#### `shared/formatting/richText.lua`
Core text processing module. Factory function — initialized with config, returns a richText object used by both server and client.

Key functions:
- `escape(text)` / `unescape(text)` — sanitize XML special characters
- `markdown(text)` — converts markdown to RichText XML (delegates to markdown.lua)
- `colorize(text, rgb)` — wraps text in a `<font color>` tag
- `strip(message)` — removes all XML tags (plain text extraction)
- `processForFilter(message, player)` — prepares a message for Roblox's text filter; handles emoji substitution and escaping
- `splitXmlAndText(input)` — splits a formatted string into alternating XML-tag and text segments

#### `shared/formatting/markdown.lua`
Converts markdown syntax to Roblox RichText XML. Supports:

| Syntax | Output |
|--------|--------|
| `**text**` | Bold |
| `*text*` / `_text_` | Italic |
| `***text***` | Bold + Italic |
| `__text__` | Underline |
| `~~text~~` | Strikethrough |
| `(R,G,B / text)` or `(#HEX / text)` | Colored text |
| `[color / text]` | Outlined (stroke) text |
| `<font / text>` | Custom font |

Implemented as a recursive depth-based formatter. Returns an object with a single `format(text)` method.

#### `shared/formatting/color.lua`
Color utility helpers.
- `hex_to_rgb` / `rgb_to_hex`
- `matchBrickcolor(name)` — fuzzy-matches a string to a Roblox BrickColor name. Pre-caches all 1032 BrickColors indexed by first letter for performance.

---

### Server Side (`server/`)

#### `Loader/MainModule/init.lua`
Main server initialization. Sets up the entire server-side chat infrastructure:

1. Configures `TextChatService` with a TextChannels folder
2. Creates the system `RBXSystem` channel
3. Initializes shared modules: signal, network, richText, md5
4. Exposes `getConfig` RemoteFunction → returns custom emoji config to clients
5. Sets up `typingIndicator` RemoteEvent
6. Creates the default "Main" channel (auto-join enabled)

#### `server/constructors/channel.lua`
Factory for creating chat channels. `channel.new(name, autoJoin)` creates a `TextChannel` with:

- `ShouldDeliverCallback` — intercepts each message before delivery, extracts the original text from metadata, runs it through the filter, and stores a reconstruction function so the client can restore formatting after filtering
- `sourceAdded` / `sourceRemoved` signals — track which players are in the channel
- Auto-join: connects `PlayerAdded` to add sources; removes sources on leave

Exposes two remote endpoints:
- `getChannelsIn(player)` — returns the list of channels the player belongs to
- `reconstruct(player, messageId, filteredText)` — called by clients after receiving a filtered message; returns the fully-formatted RichText version

Broadcasts reactions:
- `reactToMessage(player, messageId, reaction)` — receives a reaction from one client, broadcasts it to all others

---

### Client Entry Points

#### `client/Chat.lua`
Builds the entire chat UI hierarchy programmatically (no Roblox Studio design file). Creates:

- `Main` frame (400×300) — the chat window
- `Scroller` — ScrollingFrame for the message list
- `Chatbar` — text input area with an ActionIcon button
- `Channelbar` — channel selector (hidden when only one channel exists)
- `Autofill` — dropdown for emoji/text autocomplete
- `ContextMenu` — right-click / long-press menu (Reply, Edit, Delete, React)

#### `client/chat/startup.client.lua`
LocalScript entry point. Requires `chat/init.lua`, creates the UI from `Chat.lua`, parents it to `PlayerGui`, and shows the welcome message.

#### `client/chat/init.lua`
Wires up all client components into a single `chatVariables` object that is passed to every submodule. Loads:

- `chatbar`, `autofill`, `channelBar`, `chatWindow`, `bubbleChat` components
- `localization`, `network`, `customEmojiHandler`
- `messageCreators` and `coreConnections`

Exposes `:systemMessage(text)` for injecting system-level messages (e.g. friend join notifications).

---

### Client Components (`client/chat/components/`)

#### `components/chatbar.lua`
Text input management.

- Dynamic height: grows as the user types multi-line messages
- 185 character limit
- Live markdown preview as you type
- Typing indicator: fires `typingIndicator` true on focus, auto-cancels after 5 seconds of no typing, fires false on blur
- Enter to send; sends via the current `TextChannel`
- Supports **action context** (Reply / Edit): `:assignActionImage(imageId, context)` sets the ActionIcon and stores what action will happen on send; `:unassign()` clears it

#### `components/channelBar.lua`
Channel selector strip. Adjusts layout when shown/hidden to avoid overlapping the scroller.

#### `components/chatWindow.lua`
Visual state management for the message area.
- Hover: shows scrollbar thumb, dims window slightly
- Focus (chatbar active): darkens background
- All transitions use smooth tweens

#### `components/autofill.lua`
Autocomplete dropdown. Extension-based — loads handlers from `extensions/autofills/`.

- Dynamically generates buttons from a list of options
- Keyboard navigation: Up/Down arrows, Tab, Right arrow to confirm
- Scrolls selected button into view
- Pattern matching: captures text between `:` delimiters and sends to emoji extension

---

### Message System (`client/chat/messageCreators/`)

#### `messageCreators/defaultMessage.lua`
Core message rendering logic. `new(messageData, isReceived)` creates a message UI bubble.

**Bubble types:**
- `DefaultBubble` — standard message with avatar, username, text
- `ReplyStartBubble` — first message in a reply chain (shows quoted text above)
- `ReplyingToPreviousBubbleChain` — follow-up message to same sender in a reply chain

**Message grouping:** consecutive messages from the same sender within 50 seconds are grouped (no repeated avatar/name).

**Context menu actions:**
- Reply — sets chatbar action icon, stores `replyingTo` context
- Edit — replaces message text with a live TextBox for editing
- Delete — removes message from UI
- React — sends reaction emoji via `reactToMessage` network call

Tracks messages by ID and GUID for edits, reactions, and reconstruction.

---

### Client Modules (`client/chat/modules/`)

#### `modules/coreConnections.lua`
Bridges Roblox's CoreGui chat API with the custom chat. Creates BindableEvents/BindableFunctions and registers them with `SetCore`/`GetCore` so that game scripts using the standard Roblox chat API continue to work.

Supported surface area: `ToggleVisibility`, `SetVisible`, `FocusChatBar`, `ChatBarDisabled`, `ChatMakeSystemMessage`, `ChatWindowPosition`, `ChatWindowSize`, `IsFocused`, and more.

#### `modules/eventHandler.lua`
Central message event dispatcher.

1. Subscribes to `getChannelsIn` updater → gets list of channels
2. Connects `MessageReceived` on each channel
3. On each message: invokes `reconstruct` to get the fully-formatted text, determines if it's a new message or an edit, routes to the correct message creator
4. Auto-scrolls to the newest message
5. Shows `(Edited)` label on edited messages
6. Handles incoming `reactToMessage` events and updates reaction UI

#### `modules/mouse.lua`
Mouse tracking via Heartbeat. Accounts for GuiInset (topbar height offset).

- `mouse:detectInObject(object, wrapper)` — returns a signal and connection that fire when the mouse enters/exits an object
- `mouse:getPosition()` — current mouse position

---

### Custom Emoji System (`modules/customEmojiHandler/`)

#### `customEmojiHandler/init.lua`
Renders messages containing custom emojis by mixing `TextLabel` and `ImageLabel` elements inline.

- `setText(label, text, message)` — main entry point; breaks text into segments and renders each
- `breakdown(label, text)` — splits text at emoji markers
- `createImage(label, imageName)` — creates a sized ImageLabel for an emoji
- Uses a `UIListLayout` for horizontal arrangement

#### `customEmojiHandler/cache.lua`
Object pool for `TextLabel` and `ImageLabel` instances. Avoids creating new instances on every render.

- `cache:store(object)` — returns object to pool
- `cache:get(className)` — retrieves from pool or creates new

#### `customEmojiHandler/lineDetector.lua`
Parses a formatted message string into a list of word segments, each tagged with its formatting (bold, italic, color, stroke) and any embedded emoji. Used by `customEmojiHandler/init.lua` to lay out inline content.

---

### Bubble Chat (`components/bubbleChat/`)

#### `bubbleChat/init.lua`
Connects bubble chat to the main message and typing indicator systems. Creates one bubble controller per player character.

#### `bubbleChat/system/init.lua`
Main bubble chat initializer. Configuration:

| Setting | Value |
|---------|-------|
| Easing | Bounce |
| Max bubbles | 4 per player |
| Fadeout time | 10 seconds |
| Text size | 16 |
| Background | RGB(20,20,20) |
| Max visibility distance | 40 studs |

Returns a billboard system with `create(part)` to attach bubbles to a character part.

#### `bubbleChat/system/stack.lua`
Manages the vertical stack of bubbles above a player's head.

- `stack:push(ui)` — adds bubble, triggers position tween for all existing bubbles
- `stack:remove(ui)` — removes bubble
- `stack:fade(ui, duration)` — fades out a bubble (called when MaxMessages is exceeded)
- `stack:update()` — recalculates and tweens all bubble Y positions
- Only the newest bubble shows the caret (speech bubble tail)

#### `bubbleChat/system/billboard/init.lua`
Per-player `BillboardGui` controller.

- Raycast-based visibility: sets `AlwaysOnTop` based on whether the player has line-of-sight to the speaker
- Color theming via `setColors()`
- Typing indicator: `setTypingIndicatorVisible(state)`
- `createMessage(text, message)` — adds a new bubble to the stack

#### `bubbleChat/system/billboard/ui.lua`
Creates bubble UI elements: rounded speech bubbles sized to text, the caret tail, and the animated typing indicator (three dots with staggered tweens).

#### `bubbleChat/system/utility.lua`
Thin TweenService wrapper used by the bubble animation system.

---

### Extensions (`client/chat/extensions/`)

#### `extensions/autofills/emojis/init.lua`
Emoji autocomplete extension. Implements the autofill interface:
- `beginsWith`: `:` character
- `endsWith`: `:` character (or mid-word for partial match)
- `onCapture(text)`: searches the emoji list and returns matching options with images

Merges standard Unicode emojis with custom emojis from config.

#### `extensions/autofills/emojis/emojiList.lua`
Database of 100+ standard emojis, indexed by first letter for fast lookup.

- `list.search(text)` — fuzzy search by name prefix
- `list:findDirect(text)` — exact name lookup

---

### Localization (`client/chat/localization/`)

#### `localization/master.lua`
Localization entry point. Fetches a `Translator` from the `LocalizationTable` based on the player's locale, falls back to `en-us`.

- `module:localize(key, default, ...)` — get a translated string with fallback
- `module:getWelcomeMessage()` — welcome message with "System" prefix
- `module:getChatbarPlaceholder()` — chatbar hint text
- `module:getMessagePrefix(base)` — "System" / "Team" prefixes

#### `localization/table/manager.lua`
Loads all language translation tables and merges them into a single `LocalizationTable`. Supports 50+ locales including `en-us`, `fr-fr`, `de-de`, `ja-jp`, `zh-cn`, `ru-ru`, `es-es`, and many more.

---

## Data Flow

### Sending a Message
```
User types → chatbar.lua (markdown preview)
  → FocusLost → TextChannel:SendAsync (with metadata)
    → Server ShouldDeliverCallback (filter + store reconstruction fn)
      → MessageReceived on client
        → eventHandler invokes reconstruct()
          → Server returns formatted RichText
            → defaultMessage.lua renders bubble
              → bubbleChat shows above head
```

### Typing Indicator
```
User focuses chatbox → fire typingIndicator(true)
  → 5s timeout → fire typingIndicator(false)
    → Server broadcasts to all clients
      → bubbleChat/billboard shows/hides typing dots
```

### Emoji Reaction
```
User right-clicks message → ContextMenu appears
  → Selects reaction → fire reactToMessage(messageId, emoji)
    → Server broadcasts to all clients
      → eventHandler updates reaction UI on message bubble
```

### Channel Subscription
```
Client connects → hookUpdater getChannelsIn
  → Server fires current channel list
    → eventHandler connects MessageReceived on each channel
      → New channels pushed when player joins one
```

---

## Network API

| Name | Type | Direction | Purpose |
|------|------|-----------|---------|
| `getConfig` | RemoteFunction | Client → Server | Fetch custom emoji config |
| `reconstruct` | RemoteFunction | Client → Server | Restore formatted message after filter |
| `getChannelsIn` | Updater | Bi-directional | Channel list subscription |
| `typingIndicator` | RemoteEvent | Server → Clients | Player typing state |
| `reactToMessage` | RemoteEvent | Client → Server → Clients | Emoji reactions |
