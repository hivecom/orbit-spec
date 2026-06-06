# Orbit Design Specification

> [!NOTE]
> **Matrix Evaluation**
>
> Matrix is still being evaluated as an alternative chat backend before committing to IRC/Ergochat. It offers built-in federation, persistent message history, and a broad ecosystem of clients and bridges. On paper it maps well to what Orbit needs.
>
> In practice, several caveats are pushing the decision toward IRC:
>
> - **Integration complexity and surface.** The Matrix client-server API is large, stateful, and opinionated. Building on top of it means conforming to its event model, room state resolution, and sync protocol - all of which add significant integration surface and constrain how the client layer behaves.
> - **Infrastructure overhead.** Running a production-grade Matrix homeserver (Synapse or Dendrite) is substantially heavier than Ergochat. Storage requirements grow aggressively with room history and federation traffic. The operational burden is meaningfully higher.
> - **No meaningful decoupling.** The Matrix protocol is tightly coupled to the homeserver implementation. There is no clean separation between transport and logic that would allow Orbit to extend or replace layers independently. Orbit would effectively become a UI skin over Matrix rather than a platform with its own architecture.
>
> IRC's simplicity is the point. It gives Orbit a minimal, well-understood transport that Orbit builds a modern client layer over, without inheriting the complexity of a second opinionated platform.

IRC is a solid protocol for community communication. Thirty years of reliability, a vast ecosystem of bots and tooling, a server you can run yourself on minimal hardware, and an open standard that nobody owns. What it has never had is a client layer that meets modern expectations - one that runs in a browser tab, embeds on a website, works on any device without installation, and doesn't require users to understand IRC at all.

Orbit takes this to the next level. It is a client layer built on IRC, but it is not an IRC client. It is a modern, full-featured communication platform that happens to use IRC as its transport and message store. Orbit adopts a stock IRCv3 server and an OIDC identity provider, and builds two bespoke services on top: Satellite for real-time media and Depot for storage. Orbit's value is the whole experience, a great client layer plus Satellite and Depot orchestrated into one cohesive product, not a fork of the server.

## What Orbit Is

Orbit is a modern client layer built on IRC. The text and signaling transport is IRCv3. The voice and video layer is WebRTC via LiveKit. File storage is S3-compatible or local disk. Identity is standard OIDC. The Orbit client - desktop, web app, and embeddable widget - composes these into a single coherent experience that works everywhere, on any device, without installation friction.

**Orbit is a client layer plus two bespoke services over adopted infrastructure.** The text transport is a stock IRCv3 server (Uplink, with [Ergo](https://ergo.chat/) as the reference implementation) and identity is any OIDC-compliant provider (Transponder). Orbit does not fork the IRC server. On top of these adopted parts, Orbit builds and owns the components where product value lives: the clients, Satellite (real-time media), and Depot (storage). Capabilities once imagined for a server fork - push, OIDC/JWT auth, user metadata, message retraction, history backends, an HTTP API - are now native in stock Ergo. Orbit conforms to IRCv3 and supports whatever Ergo ships; the one text feature IRC has not standardized yet (message editing) is handled at the client layer until it does. Every architectural decision in this spec is made with that model in mind.

## What Orbit Is Not

**Orbit is not a Discord clone.** The goal is not to replicate Discord's feature set and ship it open-source. Discord isolates communities into walled-off servers - you join one, you're in that box. Orbit works the way IRC always has: a single Uplink instance can host many communities simultaneously. Channels are lightweight. Communities are porous. Users belong to many at once. There is no per-server bureaucracy, no layered role hierarchy, no matrix of per-channel permission overrides built into the core.

Permissions use IRC channel modes - `+o`, `+v`, `+b` - and a bot ecosystem that has existed for decades. Operators who need richer tooling have the full suite of IRC management utilities available to them. If a community's needs outgrow what a shared instance provides, they run their own. That has always been how IRC works, and it is by design.

The transition to a new platform is always different from the old one. People moved from TeamSpeak and Skype to Discord even though the model changed significantly. The bet here is the same: what Orbit gets right matters more than what it does differently. What Orbit gets right is that it works everywhere, it is open, you own your data, and the infrastructure underneath it is something communities have trusted for thirty years.

## About This Repository

This repository contains the design specifications that guide the development of Orbit. Each document captures architectural decisions, technical choices, and product scope for a specific aspect of the system. The spec is organized into focused, single-topic pages rather than monolithic documents - individual decisions can be updated without touching unrelated sections.

## Specification

The `spec/` directory contains the full set of Orbit design specifications:

| Section | Contents |
|---------|----------|
| [Architecture](spec/01-architecture/) | System overview, design philosophy, component glossary, platform comparison |
| [Components](spec/02-components/) | Uplink (incl. tag namespace & trust model), Satellite, Depot, Transponder |
| [Identity & Auth](spec/03-identity/) | Authentication, permissions |
| [Clients](spec/04-clients/) | Desktop, web app, widget |
| [Infrastructure](spec/05-infrastructure/) | DNS discovery, deployment, monorepo |
| [Next](spec/06-next/) | Federation, mobile, bot API, push, E2E encryption, server discovery, Satellite gateway |
| [Decisions](spec/0A-decisions/) | ADRs, open questions, out-of-scope |
| [Research](spec/0B-research/) | R&D tracks: MoQ/Iroh, Leptos/WASM, Linux overlay, Vulkan overlay |

Start with the [Platform Comparison](spec/01-architecture/05-platform-comparison.md) if your first question is "how does this compare to Discord, Slack, or Teams." Then read the [Architecture Overview](spec/01-architecture/01-overview.md) for the system diagram and session flows, and the [Design Philosophy](spec/01-architecture/02-philosophy.md) for the reasoning behind every major decision.

## Status

These are **living documents** subject to revision as the project evolves. Designs may change based on prototyping results, community feedback, and shifting priorities. They will be updated as stock Ergo and the IRCv3 ecosystem evolve.
