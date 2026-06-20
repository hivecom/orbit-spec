# Orbit Desktop Client

## Technology Stack

The Orbit desktop client is built with **Tauri v2** (Rust backend + OS-native WebView) and **Vue 3** (frontend), using the team's own **VUI** component library (`@dolanske/vui`). Tauri provides a lightweight, cross-platform shell with a binary size an order of magnitude smaller than Electron, while Vue 3 and VUI enable rapid development with shared tooling across all Hivecom projects. The Rust backend handles performance-critical work: IRC protocol parsing, file I/O, audio device management, and IPC.

- For the full rationale and Tauri vs. Electron comparison, see [ADR: Tauri vs. Electron](../0A-decisions/01-adr-tauri-vs-electron.md).
- For the Vue framework selection and alternatives considered (Leptos, Svelte, Quasar), see [ADR: Vue Alternatives](../0A-decisions/02-adr-vue-alternatives.md).

## Key Features

### Text Chat

- IRC connection management: connect, auto-reconnect with exponential backoff, TLS required.
- Channel list with optional hierarchical rendering (see [Channel Organization](#channel-organization) below).
- Message history fetched via IRCv3 `chathistory` on channel join.
- Rich rendering: inline link previews, image thumbnails, emoji (Unicode + custom per-server), basic Markdown (bold, italic, code, strikethrough).
- Message retractions (rendered from the server `REDACT` command via `draft/message-redaction`). Message editing is post-Uplink.
- Unread indicators and mention highlights.

### Channel Organization

IRC channels are flat. There is no server-side hierarchy, no folder concept, no metadata. Orbit keeps it that way - but uses **path notation** as a client-side rendering convention to give communities hierarchical organization for free.

If a channel name contains slashes (e.g., `#dev/frontend`, `#dev/backend`), the client interprets slashes as directory separators and renders the channels in a collapsible tree structure in the sidebar. Channels without slashes remain at the top level.

Example rendering for a server with channels `#general`, `#announcements`, `#dev/frontend`, `#dev/backend`, `#dev/infrastructure`, `#gaming/lfg`, `#gaming/strategy`, `#gaming/clips`:

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

- **No protocol change.** The IRC server sees flat channel names as always. `#dev/frontend` is just a channel name with a slash in it.
- **No server-side enforcement.** No metadata is stored, no configuration is required. The hierarchy exists only in the client's rendering logic.
- **Opt-in by naming convention.** Communities that don't use slashes see a flat channel list. Communities that adopt path notation get automatic folder grouping. Operators create the hierarchy simply by naming channels with slash-separated prefixes.
- **Nesting depth is unbounded but discouraged.** `#dev/frontend/react` would render as a nested subfolder. More than two levels deep is a smell - keep it shallow.

### Edge Cases

The following channel name patterns have defined rendering behavior:

| Channel name | Rendering | Rationale |
|---|---|---|
| `#/hidden` | Top-level channel named `/hidden` | Leading slash produces an empty prefix; the empty prefix is discarded and the channel is treated as top-level |
| `#dev/` | Top-level channel named `dev/` | Trailing slash produces an empty suffix; the channel is treated as top-level with no children |
| `#dev//frontend` | Rendered as `dev > frontend` | Consecutive slashes are collapsed to a single separator |
| `#general` | Top-level channel | No slash, no grouping |
| `#dev/frontend/react` | Rendered as `dev > frontend > react` | Three levels deep; valid but discouraged |
| `#v2.0-release` | Top-level channel named `v2.0-release` | Dots in channel names are not hierarchy separators - only slashes are. Channel names with dots are unambiguous. |

These are client-side rendering decisions only. The IRC server sees the channel names as-is. No channel name is rejected or modified by the client - only its position in the rendered tree changes.

### Subchannel Authorization

Slash-notation channels use client-side naming conventions, which means anyone can create `#dev/malicious` to impersonate the `#dev` tree. Orbit clients address this with a **subchannel allowlist** stored in the parent channel's `draft/metadata-2` metadata.

The parent channel operator sets a `subchannels` metadata key (a comma-separated list of authorized child segment names):

```
METADATA #dev SET subchannels :frontend,backend,infrastructure
```

When Orbit clients render the channel list, they check each slash-notation channel against its parent's `subchannels` allowlist:

- If the parent channel has a `subchannels` key and the child segment is listed, the channel is rendered normally.
- If the parent has a `subchannels` key but the child segment is **not** listed, or if the parent has no `subchannels` key at all, the channel is shown with an **unverified indicator** (a visual warning in the channel list and channel header).

Because only channel operators can set `draft/metadata-2` keys on a channel (see [Uplink Overview - Ergochat Configuration](../02-components/01-uplink/01-overview.md)), the allowlist is operator-controlled. A user cannot register `#dev/malicious` and then add themselves to `#dev`'s allowlist - they would need operator access to `#dev` to do so.

This check is recursive for deeper paths: `#a/b/c` requires both `#a`'s `subchannels` to include `b` and `#a/b`'s `subchannels` to include `c`. A missing allowlist at any level in the chain marks the channel as unverified.

The `subchannels` key is fetched lazily for unjoined parent channels via `METADATA <parent> GET subchannels`. For joined parent channels, the metadata arrives in the join burst automatically.

### Channel Metadata Keys

Beyond subchannel authorization, Orbit clients use `draft/metadata-2` to store display metadata on channels. These keys are set by channel operators via the channel settings UI and are used for richer channel display throughout the client.

| Key | Content | Example |
|---|---|---|
| `display-name` | Friendly human-readable channel name (separate from the IRC channel name) | `Frontend` |
| `avatar` | URL of the channel's avatar image | `https://depot.example.com/avatars/ch-abc123/avatar.webp` |
| `color` | Accent color for the channel (hex, without `#`) | `3b82f6` |
| `homepage` | URL associated with the channel (project page, docs, etc.) | `https://example.com/docs` |
| `markdown` | Rich channel description, stored as Markdown | `Welcome to **#dev/frontend**...` |
| `subchannels` | Comma-separated list of authorized direct child segment names | `frontend, backend, infra` |

These keys follow the same trust rules as user metadata: they are set by clients and relayed by the server without validation. Because only operators can set metadata on a channel, the values are operator-controlled but not server-verified. Clients SHOULD sanitize URLs and MUST NOT treat any metadata value as a capability or permission claim - display metadata is cosmetic only. See [Tag Trust Model - Metadata Is Not an Identity Signal](../02-components/05-services.md#metadata-is-not-an-identity-signal) for the general rule.

### Voice & Video

- Satellite selector: server Satellites shown with verified badge, BYOS (Bring Your Own Satellite) Satellites shown with community label.
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

### Uplink Links

| URI                                                     | Behavior                                              |
|---------------------------------------------------------|-------------------------------------------------------|
| `orbit://server.example.com/`                           | Connect to server, show channel list                  |
| `orbit://server.example.com/channel-name`               | Connect and navigate to `#channel-name`               |
| `orbit://server.example.com/channel-name?voice=true`    | Connect, navigate to `#channel-name`, auto-join voice |
| `orbit://server.example.com/%26channel-name`            | Connect and navigate to `&channel-name`               |
| `orbit://server.example.com/%2Bchannel-name`            | Connect and navigate to `+channel-name`               |
| `orbit://server.example.com/%21channel-name`            | Connect and navigate to `!channel-name`               |

**Channel prefix rules:** When the URI path contains a bare channel name (no prefix), the client assumes `#` - this is the common case and avoids URL-encoding issues with the `#` character (which is a fragment delimiter in URIs). To target a channel with a different IRC prefix (`&`, `+`, `!`), the prefix MUST be explicitly included in the path using percent-encoding (`%26`, `%2B`, `%21` respectively). The client detects a percent-encoded prefix and uses it verbatim instead of prepending `#`.

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

On invocation, the app launches (or focuses if already running) and routes to the specified server/channel or Satellite session.

### Web Fallback for `orbit://` Links

If the Orbit client is not installed, `orbit://` links are inert in most browsers. To avoid dead links when sharing publicly, operators SHOULD publish equivalent `https://` links alongside `orbit://` links. The Orbit web app accepts the same parameters as URL query strings and serves as a full fallback:

| `orbit://` link | `https://` equivalent |
|---|---|
| `orbit://irc.example.com/general` | `https://app.example.com/?server=irc.example.com&channel=general` |
| `orbit://irc.example.com/general?voice=true` | `https://app.example.com/?server=irc.example.com&channel=general&voice=true` |

The web app handles these parameters on startup identically to how the desktop client handles the `orbit://` URI. Users who open the `https://` link and don't have the desktop client installed get the full web app experience. Users who do have the desktop client installed can use the "Open in Desktop" link shown in the web app to switch over.

Sharing the `https://` link is the recommended approach for public-facing community promotion (websites, social media, README files). The `orbit://` link is more appropriate for in-app sharing between users who are expected to have the client installed.

## DNS SRV Resolution

DNS SRV resolution is handled by the desktop client's Rust resolver. For the full resolution algorithm and SRV record definitions, see [DNS & Service Discovery](../05-infrastructure/01-domain-discovery.md).

## Memory Discipline

| Concern            | Strategy                                                                                                                                   |
|--------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Chat history       | Paginated loading. Keep at most 200 messages per channel in the DOM (the *live window*). Older messages are evicted from the DOM and served from the on-device [local cache](04-local-cache.md), which falls back to `chathistory` only when the cache is exhausted. |
| Image rendering    | Images are proxied and resized by the Rust backend (or server-side) before display. No raw multi-MB images loaded into the WebView.        |
| Large IPC payloads | File downloads and bulk history loads are served via Tauri's custom protocol handler (`tauri://`).                |
| Layout thrashing   | Resize events are debounced aggressively (200ms minimum). This works around a known WebKitGTK memory leak on Linux triggered by rapid resize cycles. |
| Audio buffers      | Managed entirely in the Rust backend via `cubeb`. No Web Audio API overhead in the WebView.                                                |

## Offline Messages and Reconnection

Ergochat includes built-in bouncer/always-on functionality that handles offline message delivery without additional infrastructure.

### Always-On Mode

When enabled, Ergochat keeps the user's session alive on the server even when no client is connected. The user remains in all joined channels and accumulates messages. This is configured server-side (`accounts.multiclient.always-on`) and can be user-toggled.

Orbit deployments MUST enable Ergochat's always-on mode for registered users. This ensures that users receive all messages sent while they were offline, and that their channel memberships are preserved across disconnections.

### Reconnection Flow

When the Orbit client reconnects after a disconnection:

1. The client authenticates via SASL.
2. The client issues `CHATHISTORY TARGETS` to discover which channels and DMs have new messages since the last known timestamp.
3. For each target with new messages, the client issues `CHATHISTORY LATEST` to fetch missed messages.
4. Messages are deduplicated against any already-displayed messages (using `msgid`).
5. Unread indicators and mention highlights are updated.

### Partial Disconnection

Because Uplink and Satellite are independent, one can drop while the other stays live:

- **IRC drops, Satellite stays up**: Voice/video continues uninterrupted. Ephemeral Satellite chat continues. The client displays a warning ("Text chat reconnecting...") and attempts to re-establish the IRC connection with exponential backoff. No voice session interruption occurs.
- **Satellite drops, IRC stays up**: Text chat continues. The client displays "Voice session disconnected" and optionally attempts to rejoin the same Satellite room. Other participants see the user leave the voice session.
- **Both drop**: The client reconnects to IRC first (primary), then rejoins the Satellite session if one was active.

### Message Outbox

Messages composed during an IRC disconnection are held in a client-side outbox. They are sent when the connection is re-established and only cleared from the outbox after the server acknowledges delivery (confirmed by the server returning a `msgid` for the message). If delivery fails after reconnection, the user is notified.
