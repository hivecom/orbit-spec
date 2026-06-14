# Orbit

Orbit is a modern communication platform built on IRC. It is not an IRC client. It is a polished client layer, a voice/video service, and a file storage gateway, orchestrated into one cohesive product over infrastructure communities have trusted for thirty years.

Everything your friend group needs from Discord, privately, on a $5 VPS. `docker compose up`.

## Why

Every Discord alternative that died tried to out-Discord Discord - custom protocols, custom servers, years catching up to feature parity. They failed because they were not decoupled, scaled poorly on cost, and were horrible to maintain.

Orbit survives by not owning what it does not have to. The IRC server is stock. The identity provider is stock. Orbit builds only the parts where product value lives: the clients, Satellite (voice and video), and Depot (file storage). The rest is adopted, battle-tested, and someone else's maintenance burden.

## What It Is

- **Uplink** - any stock IRCv3 server (Ergo is the reference). Text, history, presence, signaling. No fork, no patches.
- **Satellite** - real-time voice, video, and screen sharing. Embeds LiveKit, owns the session model, supports 1:1 P2P calls that work without any server infrastructure at all.
- **Depot** - thin storage gateway over S3-compatible backends or local disk. Handles auth, quotas, and policy so a raw bucket doesn't have to.
- **Transponder** - any OIDC provider (Keycloak, Authentik, Supabase). One login, verified everywhere.
- **Clients** - desktop (Tauri), web app, embeddable widget. Where the UX lives.

Entry is frictionless. Anonymous guest access, no-account voice calls, browser-first on any device. A friend joins a call from a link without signing up for anything. The product demonstrates itself.

## What It Is Not

Orbit is not a Discord clone. Discord isolates communities into walled-off servers. Orbit works the way IRC always has: channels are lightweight, communities are porous, users belong to many at once. Permissions use IRC channel modes and the bot ecosystem that has existed for decades.

Orbit is not a product to sell. It is infrastructure and a cultural push - resistance to surveillance and platform lock-in. If Orbit ever provides hosted instances, that is a convenience service, not a SaaS play.

## Specification

The `spec/` directory contains the full design spec:

| Section | Contents |
|---------|----------|
| [Architecture](spec/01-architecture/) | System overview, design philosophy, component glossary, platform comparison |
| [Components](spec/02-components/) | Uplink (incl. tag namespace and trust model), Satellite, Depot, Transponder |
| [Identity and Auth](spec/03-identity/) | Authentication, permissions |
| [Clients](spec/04-clients/) | Desktop, web app, widget, local history cache |
| [Infrastructure](spec/05-infrastructure/) | DNS discovery, deployment, monorepo |
| [Next](spec/06-next/) | Mobile, bot API, push delivery, E2E encryption, server discovery, Satellite gateway |
| [Decisions](spec/0A-decisions/) | ADRs, open questions, out-of-scope |
| [Research](spec/0B-research/) | MoQ/Iroh, Leptos/WASM, Linux overlay, Vulkan overlay |

Start with the [Platform Comparison](spec/01-architecture/05-platform-comparison.md) if your first question is "how does this compare to Discord." Then read the [Design Philosophy](spec/01-architecture/02-philosophy.md) for the reasoning behind every major decision.

## Status

Living documents. Designs change based on prototyping, community feedback, and what ships in stock Ergo and the IRCv3 ecosystem.

## LLM Policy

Code, documentation, architecture, and project structure must be designed and created by humans.

Developers may consult an LLM or chat agent when researching a problem. However, LLM-generated output should be treated as guidance only. Developers are expected to understand, evaluate, and implement solutions themselves rather than allowing the LLM to be in the driver's seat.

This project should be built with intention and judgement. Human finess is the key.