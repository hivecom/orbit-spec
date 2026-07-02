# Orbit Specification

This directory contains the full set of Orbit design specifications, organized into focused pages. Each page covers a distinct aspect of the system - start with the [Architecture Overview](01-architecture/01-overview.md) for a high-level picture, then dive into whichever section is most relevant to what you're building or reviewing.

## Architecture

- [Overview](01-architecture/01-overview.md) - System diagram, session flows, and component map
- [Philosophy](01-architecture/02-philosophy.md) - Core design principles and opinionated choices
- [Glossary](01-architecture/03-glossary.md) - Definitions for all named components (Uplink, Satellite, Depot, Transponder)
- [IRC Compatibility Profile](01-architecture/04-compatibility-profile.md) - What IRC clients see on an Uplink server
- [Platform Comparison](01-architecture/05-platform-comparison.md) - Honest gap assessment vs Discord, Slack, and Teams

## Components

- [Uplink](02-components/01-uplink/01-overview.md) - IRC layer: IRCv3, required extensions, Ergochat configuration
- [DMs & Group DMs](02-components/01-uplink/03-dms.md) - DM storage model, E2E interaction, always-on delivery
- [Presence](02-components/01-uplink/04-presence.md) - Metadata, avatars, status, read markers
- [Satellite](02-components/02-satellite.md) - Real-time media layer: SFU, voice sessions, BYOS, authentication, codecs
- [Depot](02-components/03-depot.md) - Storage layer: file uploads, the Depot API, download model
- [Transponder](02-components/04-transponder.md) - Identity bridging: signed tokens, verified vs. unverified users
- [IRC Services Abstraction](02-components/05-services.md) - NickServ/ChanServ intent mapping, account claim, always-on, service-notice suppression

## Uplink Tags

- [Namespace](02-components/01-uplink/02-tags/01-namespace.md) - The `+orbit/*` tag table, base64 encoding, extensions sub-namespace
- [Trust Model](02-components/01-uplink/02-tags/02-trust-model.md) - Client enforcement rules for client-asserted tags (edits, deletes, invites)

## Identity & Auth

- [Authentication](03-identity/01-authentication.md) - Registered users (SASL), anonymous guest flow (SASL ANONYMOUS)
- [Permissions](03-identity/02-permissions.md) - IRC channel modes, identity display rules

## Clients

- [Desktop](04-clients/01-desktop.md) - Tauri v2 + Vue desktop client: features, URI scheme, memory discipline, reconnection
- [Web App](04-clients/02-web-app.md) - Web app and PWA: platform adapter, service worker, capability matrix
- [Widget](04-clients/03-widget.md) - Embeddable iframe widget mode
- [Local History Cache & Storage](04-clients/04-local-cache.md) - On-device history cache, progressive loading, IndexedDB/SQLite seam, storage management

## Infrastructure

- [Domain Discovery](05-infrastructure/01-domain-discovery.md) - DNS SRV records, resolution algorithm, per-service discovery
- [Deployment](05-infrastructure/02-deployment.md) - Component resource requirements, TLS, reference Docker Compose
- [Monorepo](05-infrastructure/03-monorepo.md) - Monorepo structure, build commands, CI pipeline
- [Platform](05-infrastructure/04-platform.md) - How core, platform adapters, and entrypoints hang together at runtime

## Next

- [Federation](06-next/01-federation.md) - Federation between Orbit instances and IRC network linking
- [Mobile Clients](06-next/02-mobile-clients.md) - Tauri Mobile for iOS and Android
- [Bot API](06-next/03-bot-api.md) - Bot and Integration API: IRC bots, webhooks, REST gateway
- [Push Notifications](06-next/04-push-notifications.md) - Uplink-native push notification delivery
- [E2E Encryption](06-next/05-e2e-encryption.md) - End-to-end encryption for DMs and group channels
- [Server Discovery](06-next/06-server-discovery.md) - Public server directory and community discovery
- [Satellite Gateway](06-next/07-satellite-gateway.md) - Multi-node Satellite routing and autoscaling

## Decisions & ADRs

- [ADR: Tauri vs. Electron](0A-decisions/01-adr-tauri-vs-electron.md) - ADR: Why Tauri v2 over Electron
- [ADR: Vue Alternatives](0A-decisions/02-adr-vue-alternatives.md) - ADR: Why Vue 3 over Leptos, Svelte, and Quasar
- [Open Questions](0A-decisions/03-open-questions.md) - Unresolved design decisions requiring resolution
- [Out of Scope](0A-decisions/04-out-of-scope.md) - Features explicitly deferred from the MVP

## Research Tracks

- [MoQ / Iroh](0B-research/01-moq-iroh.md) - Media over QUIC using Iroh (R&D)
- [Leptos / WASM](0B-research/02-leptos-wasm.md) - Leptos/WASM frontend rewrite (R&D)
- [Linux Overlay](0B-research/03-linux-overlay.md) - Linux gaming overlay: Wayland layer-shell and X11 (Tier 1)
- [Vulkan Overlay](0B-research/04-vulkan-overlay.md) - Linux gaming overlay: Vulkan explicit layer (Tier 2)
