# Orbit Feature Map

> Quick-reference of what ships in **MVP**, what follows in **NEXT**, and what remains an active **research** track.
>
> For full details on any item, follow the spec link in the rightmost column.

## Legend

Each section heading indicates the scope - **MVP**, **NEXT**, or **Research**. Items that cross scope boundaries are noted inline with a parenthetical. Open questions are prefixed with `OPEN`.

---

## MVP

Everything below ships in the first usable release. The MVP is deliberately narrow: text chat with history, group voice, file storage, anonymous web widget, and a lightweight desktop client.

### Uplink (IRC Text Layer)

| Feature | Details | Spec |
|---------|---------|------|
| IRCv3 transport | Stock Ergochat with 17 required extensions (message-tags, chathistory, sasl, echo-message, away-notify, etc.) | [01-uplink/01-overview](spec/02-components/01-uplink/01-overview.md) |
| Channel path notation | Slash-separated names (`#dev/frontend`) rendered as collapsible tree - pure client-side | [01-uplink/01-overview](spec/02-components/01-uplink/01-overview.md) |
| Message retractions | Server-enforced via `REDACT` / `draft/message-redaction` - tombstone in Orbit, NOTICE fallback for basic IRC clients | [01-uplink/01-overview](spec/02-components/01-uplink/01-overview.md) |
| Message replies | `+orbit/msg-reply` tag referencing a `msgid` - inline excerpt if original is in buffer | [01-uplink/01-overview](spec/02-components/01-uplink/01-overview.md) |
| Threads | Client-managed sub-channels (`#parent/t-<msgid>`). IRC clients can `/join` directly | [01-uplink/01-overview](spec/02-components/01-uplink/01-overview.md) |
| DMs | Standard IRC `PRIVMSG` with operator-configured retention and always-on mode | [01-uplink/03-dms](spec/02-components/01-uplink/03-dms.md) |
| Group private conversations | Invite-only private channels (`+s +i`) | [01-uplink/03-dms](spec/02-components/01-uplink/03-dms.md) |
| Presence | Online / Away / Offline via `away-notify` + `extended-monitor` + `draft/pre-away` | [01-uplink/04-presence](spec/02-components/01-uplink/04-presence.md) |
| Rich status & avatars | `draft/metadata-2` keys: `orbit.avatar`, `display-name`, `orbit.status` | [01-uplink/04-presence](spec/02-components/01-uplink/04-presence.md) |
| Read markers | `draft/read-marker` - server-side per-channel read position, synced across devices | [01-uplink/04-presence](spec/02-components/01-uplink/04-presence.md) |
| `+orbit/*` tag namespace | Sat invites, P2P offers/answers, thread signals, reply refs, file metadata | [01-uplink/02-tags](spec/02-components/01-uplink/02-tags/01-namespace.md) |
| Tag trust model | Server-asserted vs. client-asserted data; client enforcement rules for spoofing, flooding, unverified senders | [01-uplink/02-tags](spec/02-components/01-uplink/02-tags/02-trust-model.md) |

### Satellite (Voice & Video)

| Feature | Details | Spec |
|---------|---------|------|
| Single-node LiveKit SFU | Voice, video, screen sharing with Opus + VP9 codecs | [02-satellite](spec/02-components/02-satellite.md) |
| Group voice sessions | Create → invite via `+orbit/sat-invite` TAGMSG → join. Multiple concurrent sessions per channel | [02-satellite](spec/02-components/02-satellite.md) |
| P2P 1-on-1 calls | Direct WebRTC handshake via `+orbit/p2p-offer`/`+orbit/p2p-answer` - 2 IRC messages only | [02-satellite](spec/02-components/02-satellite.md) |
| BYOS (Bring Your Own Satellite) | Users can add their own Satellite URL; displayed as "Community" (no verified badge) | [02-satellite](spec/02-components/02-satellite.md) |
| Session permissions | Creator = admin. Mute, kick, password, lock/unlock, allow-list, moderator delegation | [02-satellite](spec/02-components/02-satellite.md) |
| Knocking | Rejected users can knock; creator admits/rejects with 60s timeout | [02-satellite](spec/02-components/02-satellite.md) |
| Password-protected sessions | Password set at creation, sent to token service on join, never over IRC | [02-satellite](spec/02-components/02-satellite.md) |
| Standalone mode | `satellite://` URI - no IRC needed. Quick calls, embedded voice on websites | [02-satellite](spec/02-components/02-satellite.md) |
| Ephemeral session chat | LiveKit data channels; not persisted | [02-satellite](spec/02-components/02-satellite.md) |
| Self-hosted STUN/TURN | `coturn` default; STUNner for K8s | [02-satellite](spec/02-components/02-satellite.md) |
| DNS SRV discovery | `_satellite._tcp` + `/info` endpoint; verified badge for DNS-discovered nodes | [02-satellite](spec/02-components/02-satellite.md) |

### Depot (File Storage)

| Feature | Details | Spec |
|---------|---------|------|
| S3-compatible storage | MinIO, AWS S3, R2, B2 - thin API (~200-300 lines) | [03-depot](spec/02-components/03-depot.md) |
| Pre-signed upload flow | Client → `POST /upload/presign` → direct S3 upload → share URL with `+orbit/file` tag | [03-depot](spec/02-components/03-depot.md) |
| OIDC mode | JWT verification, per-user quotas, file deletion by uploader, audit trail | [03-depot](spec/02-components/03-depot.md) |
| Open mode | No auth, rate limiting + size caps only - for small trusted communities | [03-depot](spec/02-components/03-depot.md) |
| Inline previews | Orbit clients render images, audio, video; IRC clients see plain URL | [03-depot](spec/02-components/03-depot.md) |
| DNS SRV discovery | `_depot._tcp`; missing Depot → file sharing hidden in UI | [03-depot](spec/02-components/03-depot.md) |

### Identity & Auth

| Feature | Details | Spec |
|---------|---------|------|
| SASL authentication | PLAIN, SCRAM-SHA-256, ANONYMOUS (guests) - standard Ergochat | [03-identity/01-auth](spec/03-identity/01-authentication.md) |
| IRC channel modes | `+o` (Operator), `+v` (Voiced), `+b` (Banned), default user - no custom roles | [03-identity/02-permissions](spec/03-identity/02-permissions.md) |
| Verified vs. unverified display | Clients visually distinguish authenticated users from guests via badges/styling | [03-identity/02-permissions](spec/03-identity/02-permissions.md) |
| Anonymous web widget users | `SASL ANONYMOUS`, `guest-*` nicknames, no account creation needed | [03-identity/01-auth](spec/03-identity/01-authentication.md) |
| Graceful degradation (no IdP) | NickServ/SASL handles auth; Satellite participants all unverified; Depot can't verify identity | [03-identity/01-auth](spec/03-identity/01-authentication.md) |

### Clients

| Feature | Details | Spec |
|---------|---------|------|
| Desktop client (Tauri v2 + Vue) | Full-featured: IRC, voice/video, file uploads, system tray, notifications, `orbit://` deep links | [04-clients/01-desktop](spec/04-clients/01-desktop.md) |
| Web app / PWA | Same Vue codebase; WebSocket to Ergochat; installable PWA with offline shell | [04-clients/02-web-app](spec/04-clients/02-web-app.md) |
| Embedded widget | Same web app in `?mode=widget` iframe; compact single-channel view; theming via URL params | [04-clients/03-widget](spec/04-clients/03-widget.md) |
| Platform adapter | Thin TypeScript interface (`tauri.ts` / `web.ts`) selected at build time via `VITE_TARGET` | [04-clients/02-web-app](spec/04-clients/02-web-app.md) |
| Rich rendering | Link previews, image thumbnails, emoji, basic Markdown (bold, italic, code, strikethrough) | [04-clients/01-desktop](spec/04-clients/01-desktop.md) |
| Voice UI | Satellite selector, participant list, mute/deafen, per-user volume, push-to-talk, grid/speaker layouts | [04-clients/01-desktop](spec/04-clients/01-desktop.md) |
| Message outbox | Messages held during disconnection, sent on reconnect, cleared after server ack | [04-clients/01-desktop](spec/04-clients/01-desktop.md) |
| Memory discipline | Paginated chat (200 msgs/channel in DOM), image resizing via Rust, debounced resizes | [04-clients/01-desktop](spec/04-clients/01-desktop.md) |
| `orbit://` + `satellite://` URI schemes | Deep links to server/channel/voice; standalone Satellite links | [04-clients/01-desktop](spec/04-clients/01-desktop.md) |
| Screen sharing | Share screen in voice sessions | [02-satellite](spec/02-components/02-satellite.md) |
| Multi-server client | Connect to multiple Orbit servers simultaneously | - |

### Infrastructure

| Feature | Details | Spec |
|---------|---------|------|
| Reference Docker Compose | Ergochat + Caddy (always); LiveKit, MinIO, coturn optional. Single `docker compose up` | [05-infra/02-deployment](spec/05-infrastructure/02-deployment.md) |
| TLS everywhere | No plaintext, ever. Let's Encrypt for public; self-signed for dev only | [05-infra/02-deployment](spec/05-infrastructure/02-deployment.md) |
| DNS SRV + well-known discovery | `_satellite._tcp`, `_depot._tcp`; `/.well-known/orbit/services.json` for web clients | [05-infra/01-discovery](spec/05-infrastructure/01-domain-discovery.md) |
| Monorepo structure | `apps/desktop/`, `apps/web/`, `lib/core/`, `lib/platform/` | [05-infra/03-monorepo](spec/05-infrastructure/03-monorepo.md) |
| CI pipeline | Test `lib/core` → build web + desktop → release artifacts for Linux/Windows/macOS | [05-infra/03-monorepo](spec/05-infrastructure/03-monorepo.md) |
| Health checks & monitoring | Health endpoints for all components; Uptime Kuma recommended | [05-infra/02-deployment](spec/05-infrastructure/02-deployment.md) |
| Backup guidance | Ergochat SQLite, MinIO buckets, Depot metadata, config files | [05-infra/02-deployment](spec/05-infrastructure/02-deployment.md) |

### Transponder (OIDC Identity Provider)

| Feature | Details | Spec |
|---------|---------|------|
| Shared identity layer | Single OIDC issuer consumed by Uplink, Satellite, Depot. One auth, one JWT, verified everywhere | [04-transponder](spec/02-components/04-transponder.md) |
| Auth-script bridge | ~50-100 line stateless service translating Ergochat's `auth-script` into JWT verification | [04-transponder](spec/02-components/04-transponder.md) |
| OIDC client auth flow | Authorization Code + PKCE; browser-based login; operator controls UX (password, SSO, passkeys, MFA) | [04-transponder](spec/02-components/04-transponder.md) |
| Multi-server identity | Per-domain JWTs; each domain = own identity domain | [04-transponder](spec/02-components/04-transponder.md) |
| `_transponder._tcp` DNS SRV | Identity provider discovery via DNS | [05-infra/01-discovery](spec/05-infrastructure/01-domain-discovery.md) |
| `/.well-known/orbit/oidc` | OIDC issuer URL discovery for web clients | [05-infra/01-discovery](spec/05-infrastructure/01-domain-discovery.md) |
| NickServ migration path | Import accounts → disable NickServ → configure auth-script | [04-transponder](spec/02-components/04-transponder.md) |
| Legacy IRC client support | Auth-script bridge accepts both JWTs and plain passwords (ROPC grant) | [04-transponder](spec/02-components/04-transponder.md) |

### IRC Compatibility

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

> Full compatibility profile: [04-compatibility-profile](spec/01-architecture/04-compatibility-profile.md)

---

## NEXT

Features planned for post-MVP releases, roughly ordered by expected priority.

### Uplink Fork

| Feature | Details | Spec |
|---------|---------|------|
| Message editing | Atomic server-canonical state via `+orbit/msg-amend` - no IRCv3 standard exists today | [01-uplink/01-overview](spec/02-components/01-uplink/01-overview.md) |
| Message reactions | Server-enforced reaction support | [01-architecture/02-philosophy](spec/01-architecture/02-philosophy.md) |
| Full-text search | Server-side message search | [01-architecture/02-philosophy](spec/01-architecture/02-philosophy.md) |
| Integrated push notifications | Uplink-native - leverages session state, online/offline status, DM recipients | [06-next/04-push](spec/06-next/04-push-notifications.md) |
| Server-to-server linking | Foundation for federation (Layer 1) | [06-next/01-federation](spec/06-next/01-federation.md) |

### Federation

| Feature | Details | Spec |
|---------|---------|------|
| Phase 0 - OIDC deployment | Transponder deployed; no IRC linking yet | [06-next/01-federation](spec/06-next/01-federation.md) |
| Phase 1 - Two-server linking | Linked IRC network via Uplink fork | [06-next/01-federation](spec/06-next/01-federation.md) |
| Phase 2 - Cross-org federation | Independent OIDC providers; trust chain across organizations | [06-next/01-federation](spec/06-next/01-federation.md) |

### Mobile Clients

| Feature | Details | Spec |
|---------|---------|------|
| Phase 1 - PWA on mobile | Existing web app already works as PWA on Android and iOS (Safari 16.4+) | [06-next/02-mobile](spec/06-next/02-mobile-clients.md) |
| Phase 2 - Tauri v2 Mobile | Native iOS/Android builds; same Rust backend + Vue frontend | [06-next/02-mobile](spec/06-next/02-mobile-clients.md) |

### Bot & Integration API

| Feature | Details | Spec |
|---------|---------|------|
| Phase 0 - Docs & examples (ships in MVP) | Documentation + example bots (Rust, Python, JS) - zero engineering cost | [06-next/03-bot-api](spec/06-next/03-bot-api.md) |
| Phase 1 - Webhook bridge | Lightweight service connecting to Uplink as IRC client; fires HTTP POST callbacks | [06-next/03-bot-api](spec/06-next/03-bot-api.md) |
| Phase 2 - REST API gateway | JSON HTTP endpoints wrapping common IRC commands | [06-next/03-bot-api](spec/06-next/03-bot-api.md) |
| Phase 3 - Formal API | OpenAPI spec, auth scoping, rate limiting, developer docs portal | [06-next/03-bot-api](spec/06-next/03-bot-api.md) |
| Orbit Extension API | Client-side plugins via `+orbit-ext/<name>/*` tag sub-namespace; pairs with IRC bots | [06-next/03-bot-api](spec/06-next/03-bot-api.md) |

### E2E Encryption

| Feature | Details | Spec |
|---------|---------|------|
| E2E DMs | Double Ratchet (Signal Protocol) via `libsignal-protocol`; `+orbit/e2e` tag | [06-next/05-e2e](spec/06-next/05-e2e-encryption.md) |
| Key transport | X3DH key agreement via identity keys published through `draft/metadata-2` | [06-next/05-e2e](spec/06-next/05-e2e-encryption.md) |
| E2E P2P calls (already in MVP) | Already secure by construction - DTLS-SRTP mandatory, fingerprints verified at handshake | [06-next/05-e2e](spec/06-next/05-e2e-encryption.md) |
| Cross-device key sync | Option A: Depot backup; Option B: P2P transfer; Option C: per-device keys (Signal model) | [06-next/05-e2e](spec/06-next/05-e2e-encryption.md) |

### Push Notifications

| Feature | Details | Spec |
|---------|---------|------|
| Uplink-native push | DMs to offline users, mentions of offline channel members | [06-next/04-push](spec/06-next/04-push-notifications.md) |
| Privacy-first payloads | No message content - sender name and channel only | [06-next/04-push](spec/06-next/04-push-notifications.md) |
| Multi-backend delivery | FCM (Android), APNs (iOS), UnifiedPush (Google-free Android) | [06-next/04-push](spec/06-next/04-push-notifications.md) |
| Web Push (PWA) | Web Push API for mentions/DMs when tab is closed | [04-clients/02-web-app](spec/04-clients/02-web-app.md) |

### Satellite Scaling

| Feature | Details | Spec |
|---------|---------|------|
| Satellite Gateway | Thin routing layer in front of a pool of LiveKit nodes; transparent to clients | [06-next/07-gateway](spec/06-next/07-satellite-gateway.md) |
| K8s autoscaling | HPA autoscaling, preStop drain hooks, STUNner for NAT traversal | [06-next/07-gateway](spec/06-next/07-satellite-gateway.md) |
| AV1 codec support | Post-MVP codec option alongside VP9 | [02-satellite](spec/02-components/02-satellite.md) |

### Server Discovery

| Feature | Details | Spec |
|---------|---------|------|
| Public directory (`orbit.directory`) | Self-service registration; operator submits domain → directory verifies ownership | [06-next/06-discovery](spec/06-next/06-server-discovery.md) |
| Client integration | Built-in "Browse Communities" feature; search by name, tags, region, language | [06-next/06-discovery](spec/06-next/06-server-discovery.md) |

### Other NEXT Items

| Feature | Details | Spec |
|---------|---------|------|
| Custom emoji / stickers | Nice-to-have | [0A-decisions/04-out-of-scope](spec/0A-decisions/04-out-of-scope.md) |
| Custom role / permission system | Extend beyond IRC modes via Extension API + bots | [0A-decisions/04-out-of-scope](spec/0A-decisions/04-out-of-scope.md) |
| E2E encrypted file uploads | Client-side encryption before Depot upload | [03-depot](spec/02-components/03-depot.md) |
| Richer presence | Presence history (last online), structured status states, server-managed avatars | [01-uplink/04-presence](spec/02-components/01-uplink/04-presence.md) |

---

## Research

Active research tracks with no ship commitment. Each requires a standalone prototype and benchmark before any decision is made.

### Media over QUIC / Iroh

| Feature | Details | Spec |
|---------|---------|------|
| QUIC-native media transport | Replace WebRTC with MoQ using the Iroh Rust networking stack for native clients | [0B-research/01-moq-iroh](spec/0B-research/01-moq-iroh.md) |
| WebTransport fallback | Web clients would use WebTransport over HTTP/3 instead of WebRTC | [0B-research/01-moq-iroh](spec/0B-research/01-moq-iroh.md) |

**Blockers:** MoQ is still IETF draft stage; Iroh untested at scale; Safari does not support WebTransport; Iroh↔WebTransport bridging architecture unspecified; maintaining two media transports increases complexity significantly.

### Leptos / WASM Frontend

| Feature | Details | Spec |
|---------|---------|------|
| Rust to WASM frontend | Rewrite desktop frontend from Vue to Leptos; shared Rust types across IPC bridge | [0B-research/02-leptos-wasm](spec/0B-research/02-leptos-wasm.md) |

**Blockers:** Immature ecosystem; WASM debugging harder than JS; high onboarding cost; web widget must remain JS-based regardless (WASM bundles too large for embeddable widget); requires **>30% memory reduction** over Vue to justify the effort. Depends on MVP client being feature-complete for fair comparison.

### Linux Gaming Overlay - Tier 1 (Layer-Shell / X11)

| Feature | Details | Spec |
|---------|---------|------|
| Wayland overlay | `wlr-layer-shell` on wlroots-based compositors (Sway, Hyprland, river) | [0B-research/03-linux-overlay](spec/0B-research/03-linux-overlay.md) |
| X11 overlay | Composite overlay window via Xlib with input passthrough | [0B-research/03-linux-overlay](spec/0B-research/03-linux-overlay.md) |

**Scope:** Deliberately narrow - speaker indicator (avatar + speaking ring) and optional webcam PiP. No chat, no notifications.

**Blockers:** Wayland compositor fragmentation (`wlr-layer-shell` not supported by GNOME Mutter or KDE KWin); dual X11+Wayland maintenance burden; must add <0.5ms per frame.

### Linux Gaming Overlay - Tier 2 (Vulkan Explicit Layer)

| Feature | Details | Spec |
|---------|---------|------|
| Vulkan implicit layer | Custom `.so` intercepting `vkQueuePresentKHR` to composite speaker indicators onto game framebuffer | [0B-research/04-vulkan-overlay](spec/0B-research/04-vulkan-overlay.md) |

**Blockers:** Multi-driver compatibility (AMD RADV/AMDVLK, NVIDIA proprietary, Intel ANV); DXVK/VKD3D-Proton interop; webcam video texture upload; crashing layer = crashing game (zero tolerance). **Depends on Tier 1 shipping first.**

---

## Open Questions

Decisions still pending that affect scope or design. See [03-open-questions](spec/0A-decisions/03-open-questions.md) for full context.

| Question | Context |
|----------|---------|
| ~~Widget voice permissions~~ | Resolved: guests can speak by default. Configurable per Satellite instance by the operator. |
| ~~File upload limits~~ | Resolved: configurable per deployment. Storage-layer enforcement with operator-configurable limits. |
| Message history retention | Default retention period? Per-channel configurability? Storage implications of unlimited retention? |
| Satellite `/info` shape | What should the metadata endpoint return beyond name/region/capacity/version? Codecs? Max participants? TURN hints? |
| BYOS trust UX | Should BYOS invites trigger a confirmation dialog (SSH host key model)? Or is that too much UX friction? |
| Satellite token bootstrapping | Is a public join key sufficient for MVP? (Probably yes - same trust model as public TeamSpeak/Mumble.) |
| Multi-node sessions | Can a voice session span multiple LiveKit nodes? Probably not in MVP. What happens on node failure? |

---

## Architecture Decisions

Key decisions already made. See [0A-decisions](spec/0A-decisions/) for full ADRs.

| Decision | Outcome | Spec |
|----------|---------|------|
| Desktop framework | **Tauri v2** over Electron - ~10-15 MB binary, ~30-50 MB idle RAM | [ADR-01](spec/0A-decisions/01-adr-tauri-vs-electron.md) |
| Frontend framework | **Vue 3 + Vite + VUI** over Leptos, Svelte, Quasar | [ADR-02](spec/0A-decisions/02-adr-vue-alternatives.md) |
| Protocol | **IRC (Ergochat / IRCv3)** - 30 years of tooling, bot ecosystem, component independence | [02-philosophy](spec/01-architecture/02-philosophy.md) |
| Media transport | **WebRTC (LiveKit SFU)** - MoQ/Iroh deferred to research | [02-satellite](spec/02-components/02-satellite.md) |
| Identity provider | **Any OIDC-compliant provider** (Keycloak, Authentik, etc.) - Transponder is a role, not a service | [04-transponder](spec/02-components/04-transponder.md) |
| E2E boundary | **1:1 only** - if a server mediates, no E2E. Channels and group voice explicitly excluded | [02-philosophy](spec/01-architecture/02-philosophy.md) |
| Mobile strategy | **PWA first**, Tauri v2 Mobile second, native Swift/Kotlin left to community | [06-next/02-mobile](spec/06-next/02-mobile-clients.md) |

---

## Explicitly Out of Scope

Items that have been evaluated and deliberately excluded. See [04-out-of-scope](spec/0A-decisions/04-out-of-scope.md).

- Custom application platform / app store
- Discord-style "server" walled gardens
- Group E2E encryption (MLS)
- Native Swift/Kotlin mobile apps (community can build on open protocol)
- Script-tag widget embed (iframe provides proper origin isolation)
