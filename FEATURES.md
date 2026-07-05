# Orbit Feature Map

> The spec describes the complete target design. This map is the index of what that design covers, component by component. Build status, release planning, and work in flight live on the project board, not in this repo.
>
> For full details on any item, follow the spec link in the rightmost column.

## Context

Discord, Slack, and Teams are the platforms most people use as a reference point when evaluating Orbit. Orbit covers a large portion of what those platforms do, some things differently by design, and excludes a handful on purpose.

A full assessment, category by category with explicit gap calls, is in the [Platform Comparison](spec/01-product/04-platform-comparison.md). Read that first if your primary question is "how does this stack up against what we use today."

The short version:

| | Orbit |
|---|---|
| Text chat, DMs, history | + |
| Replies, threads, retractions | + |
| Message editing | - (waiting on an IRC standard; client-side edit tags as fallback) |
| Message reactions | + (client-side via tags) |
| Full-text search | + (via Ergo Postgres/SQLite history backends) |
| Group voice, video, screen share | + |
| P2P 1:1 calls | + |
| Files and inline media | + |
| Presence and rich status | + |
| Desktop + web + PWA | + |
| Embedded client | + (iframe) |
| Mobile native app | + (Tauri v2 Mobile) |
| Push notifications | + (native `draft/webpush`) |
| OIDC / SSO / MFA | + |
| Anonymous guest access | + |
| IRC bot ecosystem | + |
| Webhook / REST integration API | + (webhook bridge + REST gateway) |
| Self-hosting, open protocol | + |
| Federation | - (not a goal for now; blocked on upstream server linking) |
| Role hierarchies | * |
| Enterprise compliance tools | - |
| Application platform | - |

**Legend:** + covered by the design, - not covered (the row says why where it matters), * model difference (covered differently by design)

Orbit does not fork the IRC server. The Uplink is any stock IRCv3 server (Ergo is the reference), and Orbit conforms to IRCv3: whatever Ergo implements, the Orbit client supports. The one text feature IRC has not standardized yet is message editing, which Orbit defers until a standard lands; if none has by the time editing matters, the fallback is client-side edit message tags.

For information about enterprise tools, see [Platform Comparison](spec/01-product/04-platform-comparison.md) and [Scope](spec/01-product/03-scope.md).

## Uplink (IRC Text Layer)

| Feature | Details | Spec |
|---------|---------|------|
| IRCv3 transport | Stock Ergochat with 17 required extensions (message-tags, chathistory, sasl, echo-message, away-notify, etc.) | [uplink](spec/02-architecture/03-uplink.md) |
| Channel path notation | Slash-separated names (`#dev/frontend`) rendered as collapsible tree - pure client-side | [uplink](spec/02-architecture/03-uplink.md) |
| Message retractions | Server-enforced via `REDACT` / `draft/message-redaction` (shipped in stable Ergo) - tombstone in Orbit, NOTICE fallback for basic IRC clients | [uplink](spec/02-architecture/03-uplink.md) |
| Message replies | Standard IRCv3 `+draft/reply` tag referencing a `msgid` - inline excerpt if original is in buffer; interoperates with other IRC clients | [uplink](spec/02-architecture/03-uplink.md) |
| Threads | Client-managed sub-channels (`#parent/t-<msgid>`). IRC clients can `/join` directly | [uplink](spec/02-architecture/03-uplink.md) |
| Message editing | Not yet an IRC standard; deferred until Ergo or IRCv3 ships one, with client-side edit tags as the fallback | [uplink](spec/02-architecture/03-uplink.md) |
| Channel renaming | Optional, capability-gated (`draft/channel-rename`); blocked on registered channels, so `display-name` metadata is the rename path for established channels | [uplink](spec/02-architecture/03-uplink.md) |
| DMs | Standard IRC `PRIVMSG` with operator-configured retention and always-on mode | [messaging](spec/02-architecture/10-messaging.md) |
| Group private conversations | Invite-only private channels (`+s +i`) | [messaging](spec/02-architecture/10-messaging.md) |
| Presence | Online / Away / Offline via `away-notify` + `extended-monitor` + `draft/pre-away` | [messaging](spec/02-architecture/10-messaging.md) |
| Rich status & avatars | `draft/metadata-2` keys: `orbit.avatar`, `display-name`, `orbit.status` | [messaging](spec/02-architecture/10-messaging.md) |
| Richer presence | Presence history (last online), structured status states, server-managed avatars; not yet designed in the spec | [messaging](spec/02-architecture/10-messaging.md) |
| Read markers | `draft/read-marker` - server-side per-channel read position, synced across devices | [messaging](spec/02-architecture/10-messaging.md) |
| Full-text search | Available via Ergo Postgres/SQLite history backends + indexer | [uplink](spec/02-architecture/03-uplink.md) |
| `+orbit/*` tag namespace | Sat invites, P2P offers/answers, thread signals, file metadata (replies use the standard `+draft/reply`) | [tags](spec/03-implementation/02-tags.md) |
| Tag trust model | Server-asserted vs. client-asserted data; client enforcement rules for spoofing, flooding, unverified senders | [tags and trust](spec/02-architecture/04-tags-and-trust.md) |

## Satellite (Voice & Video)

| Feature | Details | Spec |
|---------|---------|------|
| Single-node LiveKit SFU | Voice, video, screen sharing with Opus + VP9 codecs | [satellite](spec/02-architecture/05-satellite.md) |
| Group voice sessions | Create → invite via `+orbit/sat-invite` TAGMSG → join. Multiple concurrent sessions per channel | [satellite](spec/02-architecture/05-satellite.md) |
| P2P 1-on-1 calls | Direct WebRTC handshake via `+orbit/p2p-offer`/`+orbit/p2p-answer` - 2 IRC messages only | [satellite](spec/02-architecture/05-satellite.md) |
| BYOS (Bring Your Own Satellite) | Users can add their own Satellite URL; displayed as "Community" (no verified badge) | [satellite](spec/02-architecture/05-satellite.md) |
| Session permissions | Creator = admin. Mute, kick, password, lock/unlock, allow-list, moderator delegation | [satellite](spec/02-architecture/05-satellite.md) |
| Knocking | Rejected users can knock; creator admits/rejects with 60s timeout | [satellite](spec/02-architecture/05-satellite.md) |
| Password-protected sessions | Password set at creation, sent to token service on join, never over IRC | [satellite](spec/02-architecture/05-satellite.md) |
| Standalone mode | `satellite://` URI - no IRC needed. Quick calls, embedded voice on websites | [satellite](spec/02-architecture/05-satellite.md) |
| Ephemeral session chat | LiveKit data channels; not persisted | [satellite](spec/02-architecture/05-satellite.md) |
| Self-hosted STUN/TURN | `coturn` default; STUNner for K8s | [satellite](spec/02-architecture/05-satellite.md) |
| DNS SRV discovery | `_satellite._tcp` + `/info` endpoint; verified badge for DNS-discovered nodes | [satellite](spec/02-architecture/05-satellite.md) |
| Satellite Gateway | Thin routing layer in front of a pool of LiveKit nodes; transparent to clients | [satellite](spec/02-architecture/05-satellite.md) |
| K8s autoscaling | HPA autoscaling, preStop drain hooks, STUNner for NAT traversal | [satellite (implementation)](spec/03-implementation/03-satellite.md) |
| AV1 codec support | Codec option alongside VP9 | [satellite (implementation)](spec/03-implementation/03-satellite.md) |

## Depot (File Storage)

| Feature | Details | Spec |
|---------|---------|------|
| Thin storage gateway | Bespoke authority in front of storage; raw S3 alone cannot do OIDC attribution, per-app quotas, app API keys, or recipient scoping | [depot](spec/02-architecture/06-depot.md) |
| Storage backend toggle | Configurable flag: S3-compatible (MinIO, AWS S3, R2, B2) OR local filesystem | [depot](spec/02-architecture/06-depot.md) |
| Credential modes | Toggleable accepted credentials: anonymous and/or OIDC JWT and/or user-minted API keys | [depot](spec/02-architecture/06-depot.md) |
| Upload flow | Pre-signed direct upload (S3 backend) or proxied upload (filesystem backend); share URL with `+orbit/file` tag | [depot](spec/02-architecture/06-depot.md) |
| Identity-scoped controls | Per-user quotas, deletion by uploader, and audit trail when an identity is present (OIDC or API key) | [depot](spec/02-architecture/06-depot.md) |
| API key uploads | User-minted API keys enable CLI / cURL / ShareX-style uploads | [depot](spec/02-architecture/06-depot.md) |
| Recipient-scoped DM uploads | Orbit-specific capability scoping an upload to a DM recipient | [depot](spec/02-architecture/06-depot.md) |
| E2E encrypted file uploads | Client-side encryption before Depot upload; not yet designed in the spec | [depot](spec/02-architecture/06-depot.md) |
| Inline previews | Orbit clients render images, audio, video; IRC clients see plain URL | [depot](spec/02-architecture/06-depot.md) |
| DNS SRV discovery | `_depot._tcp`; missing Depot → file sharing hidden in UI | [depot](spec/02-architecture/06-depot.md) |

## Identity & Auth

| Feature | Details | Spec |
|---------|---------|------|
| SASL authentication | PLAIN, SCRAM-SHA-256, ANONYMOUS (guests) - standard Ergochat | [identity](spec/02-architecture/09-identity.md) |
| IRC channel modes | `+o` (Operator), `+v` (Voiced), `+b` (Banned), default user - no custom roles | [identity](spec/02-architecture/09-identity.md) |
| Verified vs. unverified display | Clients visually distinguish authenticated users from guests via badges/styling | [identity](spec/02-architecture/09-identity.md) |
| Anonymous embedded guests | `SASL ANONYMOUS`, `anon-*` nicknames, no account creation needed | [identity](spec/02-architecture/09-identity.md) |
| Graceful degradation (no IdP) | NickServ/SASL handles auth; Satellite participants all unverified; Depot can't verify identity | [identity](spec/02-architecture/09-identity.md) |

## Transponder (OIDC Identity Provider)

| Feature | Details | Spec |
|---------|---------|------|
| Provider role, not Orbit software | Transponder is the OIDC provider role; any compliant provider (Keycloak, Authentik, Zitadel, Supabase) fills it. Orbit does not build it | [transponder](spec/02-architecture/07-transponder.md) |
| Shared identity layer | Single OIDC issuer consumed by Uplink, Satellite, Depot. One auth, one JWT, verified everywhere | [transponder](spec/02-architecture/07-transponder.md) |
| Ergo JWT verification paths | Auth-script bridge (`SASL PLAIN`, any provider/algorithm, JWKS-based) as the general path; native `accounts.jwt-auth` via `IRCV3BEARER` for RS256/EdDSA/HMAC providers (static key; no ECDSA/ES256, no JWKS) | [transponder](spec/02-architecture/07-transponder.md) |
| OIDC client auth flow | Authorization Code + PKCE; browser-based login; operator controls UX (password, SSO, passkeys, MFA) | [transponder](spec/02-architecture/07-transponder.md) |
| Multi-server identity | Per-domain JWTs; each domain = own identity domain | [transponder](spec/02-architecture/07-transponder.md) |
| `_transponder._tcp` DNS SRV | Identity provider discovery via DNS | [infrastructure](spec/02-architecture/12-infrastructure.md) |
| `/.well-known/orbit/oidc` | OIDC issuer URL discovery for web clients | [infrastructure](spec/02-architecture/12-infrastructure.md) |
| NickServ migration path | Import accounts → disable NickServ → configure JWT verification (bridge or native `jwt-auth`) | [transponder](spec/02-architecture/07-transponder.md) |

## Federation

| Feature | Details | Spec |
|---------|---------|------|
| Server-to-server linking | Not present in stock Ergo; the dependency federation is blocked on. Not a reason to fork | [federation](spec/02-architecture/14-federation.md) |
| Cross-org federation | Independent OIDC providers; trust chain across organizations; research, no committed design | [federation](spec/02-architecture/14-federation.md) |

## Clients

| Feature | Details | Spec |
|---------|---------|------|
| Desktop client (Tauri v2 + Vue) | Full-featured: IRC, voice/video, file uploads, system tray, notifications, `orbit://` deep links | [clients](spec/02-architecture/11-clients.md) |
| Web app / PWA | Same Vue codebase; WebSocket to Ergochat; installable PWA with offline shell | [clients](spec/02-architecture/11-clients.md) |
| Embedded client | Same web app in an iframe; compact single-channel view; theming via URL params | [clients](spec/02-architecture/11-clients.md) |
| Mobile PWA | The web app installs as a PWA on Android and iOS (Safari 16.4+) | [clients](spec/02-architecture/11-clients.md) |
| Mobile native app (Tauri v2 Mobile) | Native iOS/Android builds; same Rust backend + Vue frontend | [clients](spec/02-architecture/11-clients.md) |
| Rich rendering | Link previews, image thumbnails, emoji, basic Markdown (bold, italic, code, strikethrough) | [clients](spec/02-architecture/11-clients.md) |
| Voice UI | Satellite selector, participant list, mute/deafen, per-user volume, push-to-talk, grid/speaker layouts | [clients](spec/02-architecture/11-clients.md) |
| Message outbox | Messages held during disconnection, sent on reconnect, cleared after server ack | [clients](spec/02-architecture/11-clients.md) |
| Memory discipline | Paginated chat (200 msgs/channel in DOM), image resizing via Rust, debounced resizes | [clients](spec/02-architecture/11-clients.md) |
| Local history cache | On-device persistent cache (IndexedDB on web, SQLite on desktop); cache-first prefill, progressive rendering, sliding window; offloads `chathistory` from Uplink | [local cache](spec/03-implementation/07-local-cache.md) |
| Storage management | Settings -> Storage: per-buffer stats, eviction, JSON export, configurable per-buffer cap | [local cache](spec/03-implementation/07-local-cache.md) |
| Client-side search index | Local inverted index / SQLite FTS5 over the on-device cache; answers most searches offline and offloads the server's history backend | [local cache](spec/03-implementation/07-local-cache.md) |
| Chat list virtualization | Mount only the visible message range to push effective DOM cost below the live-window cap for very large buffers | [local cache](spec/03-implementation/07-local-cache.md) |
| `orbit://` + `satellite://` URI schemes | Deep links to server/channel/voice; standalone Satellite links | [clients (implementation)](spec/03-implementation/08-clients.md) |
| Screen sharing | Share screen in voice sessions | [satellite](spec/02-architecture/05-satellite.md) |
| Multi-server client | Connect to multiple Orbit servers simultaneously | - |
| Custom emoji / stickers | Reactor component, a KV store backed by Depot for emoji, sticker, and reaction assets; not yet designed in the spec | - |

## Extensions & Integrations

| Feature | Details | Spec |
|---------|---------|------|
| IRC bot ecosystem | Any IRC bot library, any language, zero Orbit-specific API | [extensions](spec/02-architecture/15-extensions.md) |
| Bot documentation & examples | Documentation + example bots (Rust, Python, JS) | [extensions](spec/02-architecture/15-extensions.md) |
| Webhook bridge | Lightweight service connecting to Uplink as IRC client; fires HTTP POST callbacks | [extensions](spec/02-architecture/15-extensions.md) |
| REST API gateway | JSON HTTP endpoints wrapping common IRC commands | [extensions](spec/02-architecture/15-extensions.md) |
| Formal API | OpenAPI spec, auth scoping, rate limiting, developer docs portal | [extensions](spec/02-architecture/15-extensions.md) |
| Orbit Extension API | Client-side plugins via `+orbit-ext/<name>/*` tag sub-namespace; pairs with IRC bots | [extensions](spec/02-architecture/15-extensions.md) |
| Custom role / permission system | Extend beyond IRC modes via Extension API + bots | [extensions](spec/02-architecture/15-extensions.md) |

## E2E Encryption

| Feature | Details | Spec |
|---------|---------|------|
| E2E DMs | Double Ratchet (Signal Protocol) via `libsignal-protocol`; `+orbit/e2e` tag | [e2e](spec/02-architecture/13-e2e.md) |
| Key transport | X3DH key agreement via identity keys published through `draft/metadata-2` | [e2e](spec/02-architecture/13-e2e.md) |
| E2E P2P calls | Secure by construction - DTLS-SRTP mandatory, fingerprints verified at handshake | [e2e](spec/02-architecture/13-e2e.md) |
| Cross-device key sync | Option A: Depot backup; Option B: P2P transfer; Option C: per-device keys (Signal model); choice undecided | [e2e](spec/02-architecture/13-e2e.md) |

## Push Notifications

| Feature | Details | Spec |
|---------|---------|------|
| Web Push / `draft/webpush` | Native in stock Ergo (v2.15.0+); DMs/mentions to offline users, mentions of offline channel members | [push](spec/03-implementation/11-push.md) |
| Privacy-first payloads | No message content - sender name and channel only | [messaging](spec/02-architecture/10-messaging.md) |
| Multi-backend mobile delivery | FCM (Android), APNs (iOS), UnifiedPush (Google-free Android) | [push](spec/03-implementation/11-push.md) |

## Server Discovery

| Feature | Details | Spec |
|---------|---------|------|
| Public directory (`orbit.directory`) | Self-service registration; operator submits domain → directory verifies ownership | [infrastructure](spec/02-architecture/12-infrastructure.md) |
| Client integration | Built-in "Browse Communities" feature; search by name, tags, region, language | [infrastructure](spec/02-architecture/12-infrastructure.md) |

## Infrastructure

| Feature | Details | Spec |
|---------|---------|------|
| Reference Docker Compose | Ergochat + Caddy (always); LiveKit, MinIO, coturn optional. Single `docker compose up` | [deployment](spec/03-implementation/09-deployment.md) |
| TLS everywhere | No plaintext, ever. Let's Encrypt for public; self-signed for dev only | [deployment](spec/03-implementation/09-deployment.md) |
| DNS SRV + well-known discovery | `_satellite._tcp`, `_depot._tcp`; `/.well-known/orbit/services.json` for web clients | [infrastructure](spec/02-architecture/12-infrastructure.md) |
| Monorepo structure | `apps/desktop/`, `apps/web/`, `packages/core/`, `packages/platform/`; pnpm workspaces + vite-plus | [monorepo](spec/03-implementation/10-monorepo.md) |
| CI pipeline | Test `packages/core` → build web + desktop → release artifacts for Linux/Windows/macOS | [monorepo](spec/03-implementation/10-monorepo.md) |
| Health checks & monitoring | Health endpoints for all components; Uptime Kuma recommended | [deployment](spec/03-implementation/09-deployment.md) |
| Backup guidance | Ergochat SQLite, MinIO buckets, Depot metadata, config files | [deployment](spec/03-implementation/09-deployment.md) |

## IRC Compatibility

| Feature | Orbit Client | Modern IRC Client | Basic IRC Client |
|---------|-------------|-------------------|--------------------|
| Text chat | Yes | Yes | Yes |
| History | Yes | Yes | Limited |
| Retractions | Tombstone | Message removed | NOTICE fallback |
| Threads | Thread panel UI | `/join` thread channel | `/join` thread channel |
| Voice / video | Yes | - | - |
| File uploads | Inline preview | Plain URL | Plain URL |
| Avatars | Rendered | Via `METADATA GET` | - |
| Read markers | Synced | If cap supported | - |
| Bots | First-class | First-class | First-class |

> Full compatibility profile: [Protocol Posture](spec/02-architecture/02-protocol-posture.md)

## Research

Active research tracks with no ship commitment. Each requires a standalone prototype and benchmark before any decision is made.

### Media over QUIC / Iroh

| Feature | Details | Spec |
|---------|---------|------|
| QUIC-native media transport | Replace WebRTC with MoQ using the Iroh Rust networking stack for native clients | [moq-iroh](spec/0R-research/01-moq-iroh.md) |
| WebTransport fallback | Web clients would use WebTransport over HTTP/3 instead of WebRTC | [moq-iroh](spec/0R-research/01-moq-iroh.md) |

**Blockers:** MoQ is still IETF draft stage; Iroh untested at scale; Safari does not support WebTransport; Iroh↔WebTransport bridging architecture unspecified; maintaining two media transports increases complexity significantly.

### Leptos / WASM Frontend

| Feature | Details | Spec |
|---------|---------|------|
| Rust to WASM frontend | Rewrite desktop frontend from Vue to Leptos; shared Rust types across IPC bridge | [leptos-wasm](spec/0R-research/02-leptos-wasm.md) |

**Blockers:** Immature ecosystem; WASM debugging harder than JS; high onboarding cost; the embedded client must remain JS-based regardless (WASM bundles too large for it); requires **>30% memory reduction** over Vue to justify the effort. Depends on the client being feature-complete for a fair comparison.

### Linux Gaming Overlay - Tier 1 (Layer-Shell / X11)

| Feature | Details | Spec |
|---------|---------|------|
| Wayland overlay | `wlr-layer-shell` on wlroots-based compositors (Sway, Hyprland, river) | [linux-overlay](spec/0R-research/03-linux-overlay.md) |
| X11 overlay | Composite overlay window via Xlib with input passthrough | [linux-overlay](spec/0R-research/03-linux-overlay.md) |

**Scope:** Narrow - speaker indicator (avatar + speaking ring) and optional webcam PiP. No chat, no notifications.

**Blockers:** Wayland compositor fragmentation (`wlr-layer-shell` not supported by GNOME Mutter or KDE KWin); dual X11+Wayland maintenance burden; must add <0.5ms per frame.

### Linux Gaming Overlay - Tier 2 (Vulkan Implicit Layer)

| Feature | Details | Spec |
|---------|---------|------|
| Vulkan implicit layer | Custom `.so` intercepting `vkQueuePresentKHR` to composite speaker indicators onto game framebuffer | [vulkan-overlay](spec/0R-research/04-vulkan-overlay.md) |

**Blockers:** Multi-driver compatibility (AMD RADV/AMDVLK, NVIDIA proprietary, Intel ANV); DXVK/VKD3D-Proton interop; webcam video texture upload; crashing layer = crashing game (zero tolerance). **Depends on Tier 1 working first.**

## Open Questions

Decisions still pending that affect scope or design. See [Open Questions](spec/OPEN-QUESTIONS.md) for full context.

| Question | Context |
|----------|---------|
| ~~Embedded guest voice permissions~~ | Resolved: guests can speak by default. Configurable per Satellite instance by the operator. |
| ~~File upload limits~~ | Resolved: configurable per deployment. Storage-layer enforcement with operator-configurable limits. |
| Message history retention | Default retention period? Per-channel configurability? Storage implications of unlimited retention? |
| Satellite `/info` shape | What should the metadata endpoint return beyond name/region/capacity/version? Codecs? Max participants? TURN hints? |
| ~~BYOS trust UX~~ | Resolved: the domain's DNS records are the network's authoritative Satellite source; anything unsanctioned requires first-connect confirmation (SSH host key model). |
| Satellite token bootstrapping | Is a public join key sufficient? (Probably yes - same trust model as public TeamSpeak/Mumble.) |
| Multi-node sessions | Can a voice session span multiple LiveKit nodes? Probably not. What happens on node failure? |

## Architecture Decisions

Key decisions already made. See the [ADRs](spec/02-architecture/decisions/) for full context.

| Decision | Outcome | Spec |
|----------|---------|------|
| Desktop framework | **Tauri v2** over Electron - ~10-15 MB binary, ~30-50 MB idle RAM | [ADR-01](spec/02-architecture/decisions/01-adr-tauri-vs-electron.md) |
| Frontend framework | **Vue 3 + Vite + VUI** over Leptos, Svelte, Quasar | [ADR-02](spec/02-architecture/decisions/02-adr-vue-alternatives.md) |
| Protocol | **IRC (Ergochat / IRCv3)** - 30 years of tooling, bot ecosystem, component independence | [vision](spec/01-product/01-vision.md) |
| No IRC server fork | **Do not fork the IRC server.** Run a stock IRCv3 server (Ergo reference) and conform to IRCv3. Push, OIDC, metadata, and redaction are already native; editing waits on IRC standardization. Forking would break IRC compatibility and make features Orbit-only | [protocol posture](spec/02-architecture/02-protocol-posture.md) |
| Federation | **Not a goal for now** - requires server-to-server linking absent from stock Ergo; not a fork reason | [federation](spec/02-architecture/14-federation.md) |
| Media transport | **WebRTC (LiveKit SFU)** - MoQ/Iroh stays a research track | [satellite](spec/02-architecture/05-satellite.md) |
| Identity provider | **Any OIDC-compliant provider** (Keycloak, Authentik, Zitadel, Supabase) - Transponder is a role, not Orbit-built software; the auth-script bridge is the general-purpose JWT verification path (any provider/algorithm), with native Ergo `accounts.jwt-auth` as an option for RS256/EdDSA/HMAC providers | [transponder](spec/02-architecture/07-transponder.md) |
| E2E boundary | **1:1 only** - if a server mediates, no E2E. Channels and group voice explicitly excluded | [messaging](spec/02-architecture/10-messaging.md) |
| Mobile strategy | **PWA and Tauri v2 Mobile from one codebase**; native Swift/Kotlin left to community | [clients](spec/02-architecture/11-clients.md) |

## Out of Scope

Genuine product exclusions (application platform, walled-garden servers, group E2E, enterprise compliance tooling, script-tag embedding, native per-platform clients, forking the IRC server) live in [Scope](spec/01-product/03-scope.md) with the reasoning.
