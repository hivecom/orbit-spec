# Clients

The Orbit client family: desktop, web app/PWA, embedded, and mobile. One Vue
component tree serves all of them; the divergence between environments is
isolated behind a platform adapter. This page covers the client architecture
- stacks, contexts, channel semantics, caching, and connection models. The
per-surface user experience lives in
[Product - Experience](../01-product/02-experience.md); URI schemes,
reconnection sequences, embed and theming mechanics, and concrete limits live
in [Implementation - Clients](../03-implementation/08-clients.md).

## The Client Family

- **Desktop** - Tauri v2 (Rust backend + OS-native WebView) with Vue 3 and
  the team's VUI component library. The Rust backend handles
  performance-critical work: IRC protocol parsing, file I/O, audio device
  management via cubeb, and IPC. Tauri's binary is an order of magnitude
  smaller than Electron's - see
  [ADR: Tauri vs. Electron](decisions/01-adr-tauri-vs-electron.md) and
  [ADR: Vue Alternatives](decisions/02-adr-vue-alternatives.md).
- **Web app / PWA** - the same Vue application deployed to a static host,
  installable as a PWA with an offline shell and background outbox sync.
- **Embedded** - the web app loaded in an iframe with an embed mode
  parameter. See [Embedded Client](#embedded-client).
- **Mobile** - Tauri v2's iOS and Android targets, compiling the same Rust
  backend and the same shared frontend into a native app. See
  [Mobile](#mobile).

There are effectively two builds: the Tauri binary (desktop and mobile
targets) and the web app (full, PWA, and embedded presentations).

## The Platform Adapter

All clients share the entire component tree; the only divergence is a thin
platform adapter - a contract of capability ports the shared core calls for
anything that differs between environments (notifications, tray, audio
devices, deep links, DNS resolution, file transfer, the history cache). A
port an environment can't provide is null, and the core degrades explicitly
rather than scattering platform checks.

Components and stores never call platform APIs directly; they go through the
injected adapter. That keeps the component tree environment-agnostic, keeps
the core headlessly testable against a mock adapter, and means each new
target is a new adapter rather than a fork. The adapter lives in its own
package, so the core can only import what the contract exposes.

The port contract, per-environment implementations, and the capability
matrix live in [Implementation - Platform](../03-implementation/06-platform.md).

## Channel Organization

IRC channels are flat ([Uplink - Channels](03-uplink.md#channels)). Orbit
clients use path notation as a client-side rendering convention: a channel
name containing slashes (`#dev/frontend`) renders in a collapsible tree,
grouped under its path prefix; channels without slashes stay top-level.

Constraints:

- **No protocol change.** The server sees flat channel names; a slash is just
  a character in the name.
- **No server-side enforcement.** The hierarchy exists only in the client's
  rendering logic.
- **Opt-in by naming convention.** Communities that don't use slashes see a
  flat list; operators create hierarchy by naming channels with
  slash-separated prefixes.
- **Nesting depth is unbounded but discouraged.** More than two levels deep
  is a smell.

The edge-case rendering rules (leading, trailing, and consecutive slashes)
live in [Implementation - Clients](../03-implementation/08-clients.md).

### Subchannel Authorization

Naming conventions mean anyone can create `#dev/malicious` to impersonate the
`#dev` tree. Orbit clients close this with a subchannel allowlist stored in
the parent channel's metadata: the parent's operators list the authorized
child segment names, and clients check each slash-notation channel against
its parent's list when rendering.

- Listed child: rendered normally.
- Unlisted child, or no allowlist on the parent: shown with an unverified
  indicator in the channel list and header.

Because only channel operators can set metadata on a channel, the allowlist
is operator-controlled - a user can't register an impersonating channel and
then authorize it themselves. The check is recursive for deeper paths: every
level of the chain must authorize the next, and a missing allowlist anywhere
marks the channel unverified. For unjoined parents the client fetches the
allowlist lazily; for joined parents it arrives in the join burst.

Channel display metadata follows the same trust rules as user metadata:
operator-controlled but not server-verified, cosmetic only, never a
capability or permission claim
([Services](08-services.md#metadata-is-not-an-identity-signal)).

## Invite Model

Orbit is decentralized, so there is no central invite service. An `orbit://`
link is the invite - sharing the link is sharing the invite. The server
operator controls access via IRC channel modes (invite-only, key-protected);
the URI is a deep link, not a magic token. Operators sharing links publicly
SHOULD publish equivalent web app URLs alongside, since the custom scheme is
inert without the client installed - the web app accepts the same parameters
and is a full fallback. URI grammar, platform registration, and the fallback
mapping live in [Implementation - Clients](../03-implementation/08-clients.md).

## Local History Cache

Every Orbit client keeps a persistent, on-device copy of the message history
it has seen: the durable archive underneath the in-memory live window, and
distinct from it.

The IRC history model is unusually cache-friendly, which is why this design
holds together:

- **Records are immutable and append-only.** A delivered message never
  changes; the only post-delivery mutations are retraction tombstones (never
  a content edit) and, once editing standardizes upstream, edits. It's a
  write-once store with rare overlay events, not a cache-coherence problem.
- **There's a stable server-assigned dedup key.** Every message carries a
  server `msgid` and an authoritative `server-time`, so the same line merges
  into one cached record whether it arrives live, as an echo, or in a history
  replay. Lines without an ID (presence events, replays from servers that
  omit the tag) get a deterministic synthetic key and dedupe by content
  signature.
- **Delivery is already batched.** History replays arrive pre-chunked by the
  server.

The cache shifts the load profile off Uplink. Ergo's retention is
operator-configured and intentionally bounded - the source of truth, not a
long-term archive - and every history request costs the server CPU, disk,
and a round-trip. With a local cache, scrollback is served from disk,
reconnection fetches only the delta past the newest cached message, messages
that aged out of server retention stay readable on every device that saw
them, and the cache is the foundation for client-side search. The cache
never weakens the trust model: it stores what the server delivered, keyed by
server-asserted identity and message IDs - a performance and availability
layer, not a new source of authority.

The cache is a capability port on the platform adapter. The backing store
differs by environment, and the asymmetry is intentional:

- **Web app / PWA**: IndexedDB. Browser-managed, quota-bounded, best-effort -
  the browser can evict it under pressure, so the web cache is an accelerator
  and recent-history archive with the server as durable fallback.
- **Desktop and mobile**: SQLite via the Rust backend. Bounded only by disk,
  no surprise eviction - effectively a complete personal archive. A user who
  wants a permanent searchable archive runs the desktop client.
- **Embedded**: IndexedDB if available, otherwise a null port and an
  ephemeral in-memory buffer.

The cache writer is owned at app scope, subscribed to the message stream for
all joined targets - never by a view component, so what's persisted is
independent of what's rendered. Users get a storage management surface:
per-buffer stats, eviction (oldest-first within a target, never
cross-target), export, and a configurable per-buffer cap.

Known limits: the web cache can be evicted by the browser; and the cache
can't recover content the server never delivered or that was retracted.

Schema, seeding and paging algorithms, lifecycle, and environment quotas live
in [Implementation - Local Cache](../03-implementation/07-local-cache.md).

## Memory Discipline

The renderer holds a bounded live window: a hard cap on messages mounted in
the DOM, with older messages evicted and served from the local cache, falling
back to server history only when the cache is exhausted. Large images are
proxied and resized before display rather than loaded raw into the WebView.
Bulk payloads (file downloads, history loads) bypass JSON IPC on desktop.
Audio buffers are managed entirely in the Rust backend, keeping Web Audio
overhead out of the WebView. Resize handling is debounced to work around a
known WebKitGTK leak on Linux. The concrete caps and values live in
[Implementation - Clients](../03-implementation/08-clients.md).

## Connection Model

Ergo's always-on mode keeps a registered user's session alive server-side
across disconnections, preserving channel membership and accumulating
messages ([Messaging - Always-On](10-messaging.md#always-on-mode)). On
reconnect, the client discovers which targets have new messages and fetches
only the delta, deduplicating by message ID. The exact reconnection sequence
lives in [Implementation - Clients](../03-implementation/08-clients.md).

### Partial Disconnection

Uplink and Satellite are independent, so one can drop while the other stays
live:

- **IRC drops, Satellite stays up**: voice continues uninterrupted; the
  client shows a text-reconnecting warning and re-establishes IRC with
  exponential backoff.
- **Satellite drops, IRC stays up**: text continues; the client reports the
  voice session lost and optionally attempts to rejoin.
- **Both drop**: reconnect IRC first, then rejoin the Satellite session if
  one was active.

### Message Outbox

Messages composed during a disconnection are held in a client-side outbox,
sent on reconnection, and cleared only after the server acknowledges delivery
by returning a message ID. If delivery fails after reconnection, the user is
notified. On the web, the PWA flushes the outbox via background sync when
connectivity returns.

## Rate Limiting

Guest and client rate limiting is handled by Ergo's built-in flood protection
and per-IP connection limits, configured by the operator. No separate service
exists, and the same limits apply to every connecting client - desktop, web,
embedded, or third-party IRC.

## Embedded Client

The embedded client is the web app running with an embed mode parameter - a
constrained presentation layer, not a separate build. There is no separate
bundle, no separate deployment, and no version drift: embedders load the same
deployed web app, so they receive the same fixes, performance work, and
WebRTC stack automatically. Route-based code splitting keeps the embedded
surface from downloading features it never navigates to.

In embed mode the full sidebar, server browser, and settings are hidden; a
compact single-channel view renders, with an "Open in Orbit" escape hatch
that tries the desktop client first and falls back to the full web app.
Voice and video work normally, subject to the embedding page granting the
iframe media permissions. The embed snippet, code-splitting mechanics, and
theming parameters live in
[Implementation - Clients](../03-implementation/08-clients.md); the
constrained experience and guest UX live in
[Product - Experience](../01-product/02-experience.md).

## Mobile

Mobile is a first-class client with two supported paths.

The PWA already works on mobile with no additional development: Android
supports the full install flow, and iOS supports PWA installation and push
notifications since 16.4, with Background Sync and some service worker
behaviors more restricted than Android. Users who don't want a native app
are fully served by it.

For app store presence and deeper OS integration, the native path is Tauri
v2's mobile targets: the same Rust backend and the same shared frontend
compile to iOS and Android apps, with a thin mobile entrypoint in the
monorepo and a mobile platform adapter that reuses the desktop adapter and
overrides only what differs. This preserves the single-codebase guarantee - a
fix in the shared core propagates to desktop, web, and mobile simultaneously.

Two alternatives were considered and rejected. Capacitor is more mature on
mobile today, but it would introduce a second native abstraction layer
alongside Tauri, splitting the platform adapter and plugin ecosystem. Native
Swift/Kotlin maximizes platform integration but means a separate frontend
codebase per platform, which a small team can't sustain; pure native clients
are something the community can build on Orbit's open protocol
([Product - Scope](../01-product/03-scope.md)).

Known risks, stated as facts:

- Tauri's mobile targets are younger than its desktop targets, and the iOS
  and Android WebView layers have different capabilities and rough edges.
- Mobile UX patterns (gestures, bottom navigation, pull-to-refresh) need
  their own layout shell and navigation model beyond the desktop layout.
- Background voice connectivity is hard: both platforms aggressively kill
  background processes, and keeping a call alive while backgrounded requires
  platform-specific handling (foreground services on Android, VoIP
  entitlements on iOS). If Tauri can't expose the needed hooks, a thin
  native audio wrapper is the fallback.
- Push notifications require the delivery layer
  ([Messaging - Notifications](10-messaging.md#notifications)); without it,
  the mobile app has no push, and everything else works.

The mobile design is thinner than desktop's because the detailed design work
hasn't happened yet; this section records the direction and its constraints,
not a finished design.
