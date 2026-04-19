# Orbit Desktop Client

## Technology Stack

The Orbit desktop client is built with **Tauri v2** (Rust backend + OS-native WebView) and **Vue 3** (frontend), using the team's own **VUI** component library (`@dolanske/vui`). Tauri provides a lightweight, cross-platform shell with a binary size an order of magnitude smaller than Electron, while Vue 3 and VUI enable rapid development with shared tooling across all Hivecom projects. The Rust backend handles performance-critical work: IRC protocol parsing, file I/O, audio device management, and IPC.

- For the full rationale and Tauri vs. Electron comparison, see [ADR: Tauri vs. Electron](../06-decisions/01-adr-tauri-vs-electron.md).
- For the Vue framework selection and alternatives considered (Leptos, Svelte, Quasar), see [ADR: Vue Alternatives](../06-decisions/02-adr-vue-alternatives.md).

## Key Features

### Text Chat

- IRC connection management: connect, auto-reconnect with exponential backoff, TLS required.
- Channel list with optional hierarchical rendering (see [Channel Organization](#channel-organization) below).
- Message history fetched via IRCv3 `chathistory` on channel join.
- Rich rendering: inline link previews, image thumbnails, emoji (Unicode + custom per-server), basic Markdown (bold, italic, code, strikethrough).
- Message amending and retracting (rendered from `+orbit/msg-amend` and `+orbit/msg-retract` tags).
- Unread indicators and mention highlights.

### Channel Organization

IRC channels are flat. There is no server-side hierarchy, no folder concept, no metadata. Orbit keeps it that way - but uses **dot notation** as a client-side rendering convention to give communities hierarchical organization for free.

If a channel name contains dots (e.g., `#dev.frontend`, `#dev.backend`), the client interprets dots as directory separators and renders the channels in a collapsible tree structure in the sidebar. Channels without dots remain at the top level.

Example rendering for a server with channels `#general`, `#announcements`, `#dev.frontend`, `#dev.backend`, `#dev.infrastructure`, `#gaming.lfg`, `#gaming.strategy`, `#gaming.clips`:

```
# general
# announcements
> dev
  # frontend
  # backend
  # infrastructure
> gaming
  # lfg
  # strategy
  # clips
```

Key constraints:

- **No protocol change.** The IRC server sees flat channel names as always. `#dev.frontend` is just a channel name with a dot in it.
- **No server-side enforcement.** No metadata is stored, no configuration is required. The hierarchy exists only in the client's rendering logic.
- **Opt-in by naming convention.** Communities that don't use dots see a flat channel list. Communities that adopt dot notation get automatic folder grouping. Operators create the hierarchy simply by naming channels with dot-separated prefixes.
- **Nesting depth is unbounded but discouraged.** `#dev.frontend.react` would render as a nested subfolder. More than two levels deep is a smell - keep it shallow.

### Voice & Video

- Satellite selector: server Satellites shown with verified badge, BYON Satellites shown with community label.
- Join/leave voice sessions - "Join voice" means picking a Satellite and joining or creating a session in the current channel. See [Satellite](../02-components/02-satellite.md) for the session model.
- Visual participant list showing who is in the active session.
- Mute/deafen controls.
- Per-user volume adjustment.
- Voice activity indicators.
- Push-to-talk support (configurable keybind).
- Webcam video: toggle on/off per user.
- Video layout: grid view for small groups, speaker-focused view for larger sessions.
- Bandwidth adaptation: LiveKit's simulcast/SVC handles varying connection quality automatically.

### System Integration

- System tray icon with unread/mention badge counts.
- Desktop notifications (OS-native) for mentions and DMs.
- Audio device selection (input/output) in settings.
- Notification preferences (per-channel mute, DM-only mode).

## Custom URI Scheme - `orbit://`

### Ground Control Links

| URI                                                     | Behavior                                              |
|---------------------------------------------------------|-------------------------------------------------------|
| `orbit://server.example.com/`                           | Connect to server, show channel list                  |
| `orbit://server.example.com/channel-name`               | Connect and navigate to `#channel-name`               |
| `orbit://server.example.com/channel-name?voice=true`    | Connect, navigate to `#channel-name`, auto-join voice |

Channel names omit the `#` prefix in the URI path. The client prepends `#` when joining - this avoids URL-encoding issues with the `#` character (which is a fragment delimiter in URIs).

### Satellite Standalone Links (`satellite://`)

| URI                                                     | Behavior                                              |
|---------------------------------------------------------|-------------------------------------------------------|
| `satellite://sat1.example.com/room-id`                  | Connect directly to a Satellite and join room         |
| `satellite://node-url/room-id?name=Display+Name`        | Connect with a display name hint                      |

The `satellite://` scheme is registered separately from `orbit://` and is dedicated exclusively to direct Satellite connections. The host is the Satellite's hostname (e.g., `sat1.example.com`).

### Invite Model

Since Orbit is decentralized, there is no central invite service. An `orbit://` link *is* the invite - sharing the link is sharing the invite. The server operator controls access via IRC channel modes (`+i` for invite-only, `+k` for key-protected channels). The `orbit://` URI is essentially a deep link, not a magic token.

### Platform Registration

- **Linux**: `.desktop` file in `~/.local/share/applications/` with `MimeType=x-scheme-handler/orbit;x-scheme-handler/satellite`. Registered via `xdg-mime`.
- **Windows**: Registry key under `HKEY_CLASSES_ROOT\orbit` pointing to the Orbit executable with `%1` argument. Add a second registry key under `HKEY_CLASSES_ROOT\satellite` pointing to the same executable.
- **macOS**: `CFBundleURLTypes` entry in `Info.plist` with scheme `orbit`. Add a second `CFBundleURLTypes` entry with scheme `satellite`.

On invocation, the app launches (or focuses if already running) and routes to the specified server/channel or Satellite session. If the client is not installed, `orbit://` links are inert - there is no web fallback in the MVP (the full web client could serve as a fallback in a post-MVP update).

## DNS SRV Resolution

DNS SRV resolution is handled by the desktop client's Rust resolver. For the full resolution algorithm and SRV record definitions, see [DNS & Service Discovery](../05-infrastructure/01-domain-discovery.md).

## Memory Discipline

| Concern            | Strategy                                                                                                                                   |
|--------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Chat history       | Paginated loading. Keep at most 200 messages per channel in the DOM. Older messages are evicted and re-fetched on scroll-up via `chathistory`. |
| Image rendering    | Images are proxied and resized by the Rust backend (or server-side) before display. No raw multi-MB images loaded into the WebView.        |
| Large IPC payloads | File downloads and bulk history loads are served via Tauri's custom protocol handler (`tauri://`), not JSON-serialized IPC.                |
| Layout thrashing   | Resize events are debounced aggressively (200ms minimum). This works around a known WebKitGTK memory leak on Linux triggered by rapid resize cycles. |
| Audio buffers      | Managed entirely in the Rust backend via `cpal`. No Web Audio API overhead in the WebView.                                                 |

## Offline Messages and Reconnection

Ergochat includes built-in bouncer/always-on functionality that handles offline message delivery without additional infrastructure.

### Always-On Mode

When enabled, Ergochat keeps the user's session alive on the server even when no client is connected. The user remains in all joined channels and accumulates messages. This is configured server-side (`accounts.multiclient.always-on`) and can be user-toggled.

Orbit deployments SHOULD enable Ergochat's always-on mode for registered users. This ensures that users receive all messages sent while they were offline, and that their channel memberships are preserved across disconnections.

### Reconnection Flow

When the Orbit client reconnects after a disconnection:

1. The client authenticates via SASL.
2. The client issues `CHATHISTORY TARGETS` to discover which channels and DMs have new messages since the last known timestamp.
3. For each target with new messages, the client issues `CHATHISTORY LATEST` to fetch missed messages.
4. Messages are deduplicated against any already-displayed messages (using `msgid`).
5. Unread indicators and mention highlights are updated.

### Partial Disconnection

Because Ground Control and Satellite are independent, one can drop while the other stays live:

- **IRC drops, Satellite stays up**: Voice/video continues uninterrupted. Ephemeral Satellite chat continues. The client displays a warning ("Text chat reconnecting...") and attempts to re-establish the IRC connection with exponential backoff. No voice session interruption occurs.
- **Satellite drops, IRC stays up**: Text chat continues. The client displays "Voice session disconnected" and optionally attempts to rejoin the same Satellite room. Other participants see the user leave the voice session.
- **Both drop**: The client reconnects to IRC first (primary), then rejoins the Satellite session if one was active.

### Message Outbox

Messages composed during an IRC disconnection are held in a client-side outbox. They are sent when the connection is re-established and only cleared from the outbox after the server acknowledges delivery (confirmed by the server returning a `msgid` for the message). If delivery fails after reconnection, the user is notified.
