# Orbit Specification

The full Orbit design spec, split into three layers by altitude plus a research track. Product says what, why, and for whom. Architecture says what the parts are and where the lines between them run. Implementation is real and runnable: configs, payloads, sequences. Start with the [Architecture Overview](02-architecture/01-overview.md) for the system picture, or [Vision](01-product/01-vision.md) for the reasoning behind it, then dive into whichever page covers what you're building or reviewing.

## Product

- [Vision](01-product/01-vision.md) - The problem, who Orbit is for, and where the value lives
- [Experience](01-product/02-experience.md) - What using Orbit feels like per surface: desktop, web/PWA, mobile, embedded
- [Scope](01-product/03-scope.md) - What Orbit is not, and the exclusions that keep it honest
- [Platform Comparison](01-product/04-platform-comparison.md) - Honest gap assessment vs Discord, Slack, and Teams

## Architecture

- [Overview](02-architecture/01-overview.md) - System diagram, component map, component independence
- [Protocol Posture](02-architecture/02-protocol-posture.md) - IRCv3 conformance, the no-fork rule, compatibility profile
- [Uplink](02-architecture/03-uplink.md) - Channel model, metadata, renaming, retractions, threads
- [Tags and Trust](02-architecture/04-tags-and-trust.md) - The tag layer, versioning, trust boundary, identity display rules
- [Satellite](02-architecture/05-satellite.md) - Session model, discovery, BYOS, trust, permissions, scaling
- [Depot](02-architecture/06-depot.md) - Storage boundary, driver contract, credential model, moderation posture
- [Transponder](02-architecture/07-transponder.md) - Identity provider role, verification model, multi-server identity
- [Services](02-architecture/08-services.md) - NickServ/ChanServ/HistServ abstraction principles
- [Identity](02-architecture/09-identity.md) - Auth model, permissions, verified vs unverified, graceful degradation
- [Messaging](02-architecture/10-messaging.md) - DMs, presence, retention and erasure, moderation, notifications
- [Clients](02-architecture/11-clients.md) - Client family, platform adapters, cache design, reconnection, mobile
- [Infrastructure](02-architecture/12-infrastructure.md) - Topology, discovery, community directory, TLS, backups
- [E2E Encryption](02-architecture/13-e2e.md) - Double Ratchet design, key transport, key sync options, risks
- [Federation](02-architecture/14-federation.md) - Identity bridging, trust chain, the IRC linking dependency
- [Extensions](02-architecture/15-extensions.md) - IRC bots, webhook bridge, REST gateway, the Orbit extension API
- [ADR: Tauri vs. Electron](02-architecture/decisions/01-adr-tauri-vs-electron.md) - Why Tauri v2 over Electron
- [ADR: Vue Alternatives](02-architecture/decisions/02-adr-vue-alternatives.md) - Why Vue 3 over Leptos, Svelte, and Quasar

## Implementation

- [Uplink](03-implementation/01-uplink.md) - Ergo configuration, IRC command mechanics, thread creation sequence
- [Tags](03-implementation/02-tags.md) - The `+orbit/*` table, payloads, encoding, standard IRCv3 tags
- [Satellite](03-implementation/03-satellite.md) - Session flows, handshake payloads, knocking, codecs, STUN/TURN, multi-node mechanics
- [Depot](03-implementation/04-depot.md) - API table, presign flow, keys, quotas, schema, configuration
- [Identity](03-implementation/05-identity.md) - OIDC flows, token refresh, auth bridge, account claim, notice routing
- [Platform](03-implementation/06-platform.md) - How core, platform adapters, and entrypoints hang together at runtime
- [Local Cache](03-implementation/07-local-cache.md) - Schema, seeding and paging, lifecycle, eviction, environment limits
- [Clients](03-implementation/08-clients.md) - URI schemes, reconnection sequence, memory values, PWA config, embedding
- [Deployment](03-implementation/09-deployment.md) - DNS records, services.json, Docker Compose, CORS, monitoring, backups
- [Monorepo](03-implementation/10-monorepo.md) - Monorepo structure, build commands, CI pipeline
- [Push](03-implementation/11-push.md) - The native Web Push path, the FCM/APNs relay, token store

## Shared

- [Glossary](GLOSSARY.md) - Definitions for every named component and concept
- [Open Questions](OPEN-QUESTIONS.md) - The working decision log: unresolved items and the record of resolved ones

## Research

Deliberate explorations with no commitment; each needs a standalone prototype and benchmark before any decision.

- [MoQ / Iroh](0R-research/01-moq-iroh.md) - Media over QUIC using the Iroh networking stack
- [Leptos / WASM](0R-research/02-leptos-wasm.md) - Leptos/WASM frontend rewrite
- [Linux Overlay](0R-research/03-linux-overlay.md) - Linux gaming overlay: Wayland layer-shell and X11 (Tier 1)
- [Vulkan Overlay](0R-research/04-vulkan-overlay.md) - Linux gaming overlay: Vulkan implicit layer (Tier 2)
- [Broadcast Streaming](0R-research/05-broadcast-streaming.md) - One-to-many livestreaming beside the Satellite call model
