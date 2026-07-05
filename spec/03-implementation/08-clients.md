# Clients

Client implementation reference: URI schemes and OS registration, the reconnection sequence,
memory values, rendering edge cases, PWA configuration, and embedding. The client family, the
platform adapter concept, channel organization semantics, and the cache design live in
[Clients architecture](../02-architecture/11-clients.md); the per-surface experience lives in
[Experience](../01-product/02-experience.md).

## Custom URI Schemes

### `orbit://` Links

| URI                                                     | Behavior                                              |
|---------------------------------------------------------|-------------------------------------------------------|
| `orbit://server.example.com/`                           | Connect to server, show channel list                  |
| `orbit://server.example.com/channel-name`               | Connect and navigate to `#channel-name`               |
| `orbit://server.example.com/channel-name?voice=true`    | Connect, navigate to `#channel-name`, auto-join voice |
| `orbit://server.example.com/%26channel-name`            | Connect and navigate to `&channel-name`               |
| `orbit://server.example.com/%2Bchannel-name`            | Connect and navigate to `+channel-name`               |
| `orbit://server.example.com/%21channel-name`            | Connect and navigate to `!channel-name`               |

**Channel prefix rules:** When the URI path contains a bare channel name (no prefix), the client
assumes `#` - this is the common case and avoids URL-encoding issues with the `#` character
(which is a fragment delimiter in URIs). To target a channel with a different IRC prefix (`&`,
`+`, `!`), the prefix MUST be explicitly included in the path using percent-encoding (`%26`,
`%2B`, `%21` respectively). The client detects a percent-encoded prefix and uses it verbatim
instead of prepending `#`.

An `orbit://` link *is* the invite - the invite model is in
[Clients architecture](../02-architecture/11-clients.md).

### Satellite Standalone Links (`satellite://`)

| URI                                                     | Behavior                                              |
|---------------------------------------------------------|-------------------------------------------------------|
| `satellite://sat1.example.com/room-id`                  | Connect directly to a Satellite and join room         |
| `satellite://node-url/room-id?name=Display+Name`        | Connect with a display name hint                      |

The `satellite://` scheme is registered separately from `orbit://` and is dedicated exclusively
to direct Satellite connections. The host is the Satellite's hostname (e.g., `sat1.example.com`).
The standalone session flow is in [Satellite](03-satellite.md#standalone-session-flow).

### Platform Registration

- **Linux**: `.desktop` file in `~/.local/share/applications/` with `MimeType=x-scheme-handler/orbit;x-scheme-handler/satellite`. Registered via `xdg-mime`.
- **Windows**: Registry key under `HKEY_CLASSES_ROOT\orbit` pointing to the Orbit executable with `%1` argument. Add a second registry key under `HKEY_CLASSES_ROOT\satellite` pointing to the same executable.
- **macOS**: `CFBundleURLTypes` entry in `Info.plist` with scheme `orbit`. Add a second `CFBundleURLTypes` entry with scheme `satellite`.

On invocation, the app launches (or focuses if already running) and routes to the specified
server/channel or Satellite session.

### Web Fallback for `orbit://` Links

If the Orbit client is not installed, `orbit://` links are inert in most browsers. To avoid dead
links when sharing publicly, operators SHOULD publish equivalent `https://` links alongside
`orbit://` links. The Orbit web app accepts the same parameters as URL query strings and serves
as a full fallback:

| `orbit://` link | `https://` equivalent |
|---|---|
| `orbit://irc.example.com/general` | `https://app.example.com/?server=irc.example.com&channel=general` |
| `orbit://irc.example.com/general?voice=true` | `https://app.example.com/?server=irc.example.com&channel=general&voice=true` |

The web app handles these parameters on startup identically to how the desktop client handles the
`orbit://` URI. Users who open the `https://` link and don't have the desktop client installed
get the full web app experience. Users who do have the desktop client installed can use the "Open
in Desktop" link shown in the web app to switch over.

Sharing the `https://` link is the recommended approach for public-facing community promotion
(websites, social media, README files). The `orbit://` link is more appropriate for in-app
sharing between users who are expected to have the client installed.

## Reconnection Flow

When the Orbit client reconnects after a disconnection:

1. The client authenticates via SASL.
2. The client issues `CHATHISTORY TARGETS` to discover which channels and DMs have new messages since the last known timestamp.
3. For each target with new messages, the client issues `CHATHISTORY LATEST` to fetch missed messages.
4. Messages are deduplicated against any already-displayed messages (using `msgid`).
5. Unread indicators and mention highlights are updated.

With a populated [local cache](07-local-cache.md), step 3 fetches only the delta after the cached
tip. The partial-disconnection and outbox models are in
[Clients architecture](../02-architecture/11-clients.md).

## Memory Discipline

| Concern            | Strategy                                                                                                                                   |
|--------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Chat history       | Paginated loading. Keep at most 200 messages per channel in the DOM (the *live window*). Older messages are evicted from the DOM and served from the on-device [local cache](07-local-cache.md), which falls back to `chathistory` only when the cache is exhausted. |
| Image rendering    | Images are proxied and resized by the Rust backend (or server-side) before display. No raw multi-MB images loaded into the WebView.        |
| Large IPC payloads | File downloads and bulk history loads are served via Tauri's custom protocol handler (`tauri://`).                |
| Layout thrashing   | Resize events are debounced aggressively (200ms minimum). This works around a known WebKitGTK memory leak on Linux triggered by rapid resize cycles. |
| Audio buffers      | Managed entirely in the Rust backend via `cubeb`. No Web Audio API overhead in the WebView.                                                |

## Channel Rendering Edge Cases

Slash-notation channel names render as a collapsible tree (the convention and its semantics are
in [Clients architecture](../02-architecture/11-clients.md)). The following channel name patterns
have defined rendering behavior:

| Channel name | Rendering | Rationale |
|---|---|---|
| `#/hidden` | Top-level channel named `/hidden` | Leading slash produces an empty prefix; the empty prefix is discarded and the channel is treated as top-level |
| `#dev/` | Top-level channel named `dev/` | Trailing slash produces an empty suffix; the channel is treated as top-level with no children |
| `#dev//frontend` | Rendered as `dev > frontend` | Consecutive slashes are collapsed to a single separator |
| `#general` | Top-level channel | No slash, no grouping |
| `#dev/frontend/react` | Rendered as `dev > frontend > react` | Three levels deep; valid but discouraged |
| `#v2.0-release` | Top-level channel named `v2.0-release` | Dots in channel names are not hierarchy separators - only slashes are. Channel names with dots are unambiguous. |

These are client-side rendering decisions only. The IRC server sees the channel names as-is. No
channel name is rejected or modified by the client - only its position in the rendered tree
changes.

## Channel Rename Handling

Native renaming via `draft/channel-rename` is a narrow, capability-gated case; the model and its
caveats are in [Uplink architecture](../02-architecture/03-uplink.md). A client that does choose
to expose native renaming should:

- Negotiate `draft/channel-rename` during the CAP exchange, so the server delivers the single rename
  event rather than the leave/rejoin fallback.
- Treat the server's rename notification as an in-place migration of local state: carry the existing
  channel's messages, member list, modes, topic, and list-mode entries onto the new name, and re-key
  everything else that is keyed by channel name - read markers, cached metadata, the active-channel
  pointer, and any persisted auto-join target - so nothing resets or double-counts.
- Resolve a channel's registration status before offering the action, and present native renaming
  only where the server can actually honour it. Everywhere else, present `display-name` editing
  instead and explain why, rather than letting the user attempt a rename the server will reject.
- Surface the standardized failure replies (name already in use, cannot rename, insufficient
  privilege) to the user, since the operation round-trips through the server and can fail after the
  request is sent.

The `display-name` metadata key that covers the routine rename intent is defined in
[Uplink](01-uplink.md#channel-metadata-keys).

## Thread Rendering Mechanics

The thread creation sequence is in [Uplink](01-uplink.md#thread-creation-sequence); the thread
UX is in [Experience](../01-product/02-experience.md). The client mechanics:

- Messages that have received a `+orbit/msg-thread` signal display a thread indicator; clicking it
  opens the thread panel. The client JOINs the thread channel (if not already joined) and fetches
  history via `chathistory`. The original message is shown at the top, followed by all thread
  replies in chronological order.
- The client handles the JOIN, `+s` mode, TOPIC, TAGMSG (on creation), and PRIVMSG operations
  transparently - the user sees a reply input, not IRC primitives.
- Clients that have joined a thread channel receive new replies via normal IRC message delivery.
  Being in the channel is the subscription.
- The authoritative reply count comes from `chathistory` on the thread channel, fetched when the
  user opens the thread panel. No client-side approximation from TAGMSGs.
- Thread channels are hidden from the Orbit channel list and server browser, and the plain-text
  fallback PRIVMSG (`↳ Thread started by ...`) is suppressed from the chat view in favor of the
  indicator.

## PWA Configuration

The web app ships as a Progressive Web App using
[`vite-plugin-pwa`](https://vite-pwa-org.netlify.app/), which wraps Workbox and handles service
worker generation, manifest injection, and asset precaching automatically. What the PWA gets the
user is described in [Experience](../01-product/02-experience.md).

PWA configuration in `vite.config.ts`:

```ts
VitePWA({
  registerType: 'autoUpdate',
  manifest: {
    name: 'Orbit',
    short_name: 'Orbit',
    theme_color: '#1a1a2e',
    icons: [ /* ... */ ],
    display: 'standalone',      // no browser chrome when installed
    start_url: '/',
    scope: '/',
  },
  workbox: {
    // Precache the app shell; runtime-cache API responses separately
    globPatterns: ['**/*.{js,css,html,woff2,svg,png}'],
  },
})
```

The app shell (HTML, JS, CSS, fonts) is precached by the service worker, so repeat visits load
the shell from cache before the network responds. The message outbox (messages composed during a
disconnection) is flushed via the Background Sync API when connectivity is restored, using the
same outbox logic as the desktop client.

The PWA install flow, offline shell, and background sync are handled entirely by
`vite-plugin-pwa` and Workbox. There is no custom service worker to maintain. The installed PWA
and the Tauri desktop client can coexist; the Tauri binary adds the platform-native integrations
(tray, `orbit://` URI scheme, Rust audio management).

## Embedding

The embedded client is the web app running with `?mode=widget` (or
`?mode=widget&server=...&channel=...`) passed as URL parameters. The app reads these on startup
and activates a constrained presentation layer - there is no separate bundle, no separate
deployment, and no version drift between the embedded client and the full web app. What the
constrained presentation shows is in [Experience](../01-product/02-experience.md).

```html
<!-- Recommended: iframe -->
<iframe
  src="https://app.example.com/?mode=widget&server=irc.example.com&channel=general"
  width="400"
  height="600"
  allow="microphone; camera"
></iframe>
```

Voice and video work normally when embedded. The iframe just needs `allow="microphone; camera"`
from the embedder - no special configuration beyond the standard iframe permission attributes.

### Code Splitting

The embedded client loads the same web app bundle as the full client, but only renders a
single-channel view. To avoid loading code for features it never uses (server browser, settings,
full voice UI, etc.), the app uses Vite's route-based code splitting.

When `mode=widget` is active on startup, a lazy boundary is established at the application shell
level. Features outside the embedded surface - the server browser, settings panels, the full
Satellite session management UI - are placed behind dynamic `import()` calls and are never
fetched unless the user navigates to them. In embedded mode, they are never navigated to, so they
are never downloaded.

This makes the initial embedded bundle significantly smaller than the full web app load. The
trade-off is a slightly more complex chunk graph in the Vite build, but `vite-plugin-pwa`'s
Workbox precaching handles this automatically - only the chunks that embedded mode actually needs
are precached for its initial load path.

### Theming

Embedders can customize the appearance to match their site by passing VUI CSS custom property
overrides as URL query parameters. The client reads these on startup and applies them over VUI's
default token values before rendering.

```html
<iframe
  src="https://app.example.com/?mode=widget&server=irc.example.com&channel=general&theme[accent]=3b82f6&theme[light-mode]=dark"
  width="400"
  height="600"
  allow="microphone; camera"
></iframe>
```

**Supported theme parameters:**

| Parameter | Default | Type |
|---|---|--|
| `theme[color-mode]` | `system` | `light`, `dark`, `system` |
| `theme[light-accent]` | `#93EFD8` | `CSSHexColor` |
| `theme[dark-accent]` | `#93EFD8`| `CSSHexColor` |
| `theme[radius]` | `8px`| `CSSValue` | 

Values are interpreted as hex color strings (without the `#`) for color tokens, and as CSS values
for non-color tokens.

Only the tokens listed above are exposed as theme parameters. Full VUI token override is not
supported - embedders who need deeper customization should self-host the web app and modify the
VUI theme directly.

## DNS SRV Resolution

DNS SRV resolution is handled by the desktop client's Rust resolver. For the full resolution
algorithm and record definitions, see [Deployment](09-deployment.md).

## Cross-References

- [Clients architecture](../02-architecture/11-clients.md) - client family, adapter concept, channel organization, invite model, outbox
- [Local Cache](07-local-cache.md) - the durable history layer under the live window
- [Satellite](03-satellite.md) - standalone links and session flows
- [Deployment](09-deployment.md) - discovery and resolution
