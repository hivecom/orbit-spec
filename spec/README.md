# Orbit Specification

This directory contains the full set of Orbit design specifications, organized into focused pages. Each page covers a distinct aspect of the system - start with the [Architecture Overview](01-architecture/01-overview.md) for a high-level picture, then dive into whichever section is most relevant to what you're building or reviewing.

## Architecture

- [Overview](01-architecture/01-overview.md) - System diagram, session flows, and component map
- [Philosophy](01-architecture/02-philosophy.md) - Core design principles and opinionated choices
- [Glossary](01-architecture/03-glossary.md) - Definitions for all named components (Ground Control, Satellite, Depot, Transponder, Beacon)

## Components

- [Ground Control](02-components/01-ground-control/01-overview.md) - IRC layer: IRCv3, required extensions, Ergochat configuration, message editing
- [Satellite](02-components/02-satellite.md) - Real-time media layer: SFU, voice sessions, BYON, authentication, codecs
- [Depot](02-components/03-depot.md) - Storage layer: file uploads, the Depot API, download model
- [Transponder](02-components/04-transponder.md) - Identity bridging: signed tokens, verified vs. unverified users (post-MVP)

## Ground Control Tags

- [Namespace](02-components/01-ground-control/02-tags/01-namespace.md) - The `+orbit/*` tag table, base64 encoding, extensions sub-namespace
- [Trust Model](02-components/01-ground-control/02-tags/02-trust-model.md) - Client enforcement rules for client-asserted tags (edits, deletes, invites)

## Identity & Auth

- [Authentication](03-identity/01-authentication.md) - Registered users (SASL), anonymous guest flow (SASL ANONYMOUS)
- [Permissions](03-identity/02-permissions.md) - IRC channel modes, identity display rules

## Clients

- [Desktop](04-clients/01-desktop.md) - Tauri v2 + Vue desktop client: features, URI scheme, memory discipline, reconnection
- [Web App](04-clients/02-web-app.md) - Web app and PWA: platform adapter, service worker, capability matrix
- [Widget](04-clients/03-widget.md) - Embeddable iframe widget mode

## Infrastructure

- [Domain Discovery](05-infrastructure/01-domain-discovery.md) - DNS SRV records, resolution algorithm, per-service discovery
- [Deployment](05-infrastructure/02-deployment.md) - Component resource requirements, TLS, reference Docker Compose
- [Monorepo](05-infrastructure/03-monorepo.md) - Monorepo structure, build commands, CI pipeline

## Decisions & ADRs

- [ADR: Tauri vs. Electron](06-decisions/01-adr-tauri-vs-electron.md) - ADR: Why Tauri v2 over Electron
- [ADR: Vue Alternatives](06-decisions/02-adr-vue-alternatives.md) - ADR: Why Vue 3 over Leptos, Svelte, and Quasar
- [Open Questions](06-decisions/03-open-questions.md) - Unresolved design decisions requiring resolution
- [Out of Scope](06-decisions/04-out-of-scope.md) - Features explicitly deferred from the MVP

## Research Tracks

- [MoQ / Iroh](07-research/01-moq-iroh.md) - Media over QUIC using Iroh (R&D)
- [Leptos / WASM](07-research/02-leptos-wasm.md) - Leptos/WASM frontend rewrite (R&D)
- [Linux Overlay](07-research/03-linux-overlay.md) - Linux gaming overlay: Wayland layer-shell and X11 (Tier 1)
- [Vulkan Overlay](07-research/04-vulkan-overlay.md) - Linux gaming overlay: Vulkan explicit layer (Tier 2)
- [Federation](07-research/05-federation.md) - Federation between Orbit instances and IRC network linking
- [E2E Encryption](07-research/06-e2e-encryption.md) - End-to-end encryption for DMs and group channels
- [Mobile Clients](07-research/07-mobile-clients.md) - Tauri Mobile for iOS and Android
- [Beacon](07-research/08-beacon.md) - Push notification relay component (post-MVP)
- [Bot API](07-research/09-bot-api.md) - Bot and Integration API: IRC bots, webhooks, REST gateway
- [Server Discovery](07-research/10-server-discovery.md) - Public server directory and community discovery
- [Prioritization](07-research/11-prioritization.md) - Prioritization matrix, track lifecycle process, revision history
