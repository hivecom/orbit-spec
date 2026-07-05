# Scope

What Orbit deliberately is not. Out of scope is a product decision, not an oversight, and this document is where those decisions live. For how the design stacks up feature by feature against the platforms people actually use, see the [Platform Comparison](04-platform-comparison.md).

## Not a Discord Clone

Discord isolates communities into walled-off servers. Every server is its own world, with its own roles, its own permission matrix, its own channels. Cross-community interaction is nonexistent by design.

Orbit works the way IRC always has. A single server hosts many communities simultaneously. Channels are lightweight. Communities are porous: users belong to many at once, and the server doesn't enforce artificial boundaries between them. A community on Orbit is a set of channels, not a walled namespace.

Some Discord features don't translate to this model:

- **Complex role hierarchies** exist in Discord because each server is a self-contained organization that needs its own bureaucracy. On Orbit, IRC channel modes cover the permission model, and communities that need richer automation run an IRC bot. That bot story is one of Orbit's strengths: any IRC bot library in any language, zero Orbit-specific API required.
- **Per-channel permission overrides** exist in Discord because one server hosts hundreds of channels under one administrative roof. On Orbit, a community that needs its own administrative boundary runs its own instance, a single-binary deployment. This is how IRC networks have always been structured.
- **Server-level settings dashboards** exist in Discord because operators have no other way to manage their walled garden. Orbit operators have thirty years of IRC management tooling.

The transition from one platform to another is always different from what you had before. People moved from TeamSpeak and Skype to Discord even though the model changed significantly. The bet here is the same: what Orbit gets right matters more than what it does differently. Users won't miss what they didn't need.

## A Transport Layer and Client, Not an Application Platform

The core handles four things: text chat, real-time media, identity, and client UX. Everything else, calendars, events, custom moderation workflows, rich permission systems, game integrations, is an extension. Client-side Orbit extensions and server-side IRC bots extend the platform at the edges without touching the core; the extension and bot design lives in [Extensions](../02-architecture/15-extensions.md).

Orbit stays thin so it stays fast and maintainable. Complexity belongs at the edges.

## Exclusions

Evaluated and excluded, with the reasoning:

- **Enterprise compliance products.** DLP, eDiscovery, and audit exports are not planned. Operators get the technical levers for retention and erasure (see [Messaging](../02-architecture/10-messaging.md)), but Orbit ships no compliance product and is not a replacement for Slack or Teams in regulated industries.
- **An application platform.** Discord's Activities, Slack's app marketplace, Teams' tab ecosystem: Orbit doesn't embed third-party applications. Bots and extensions cover automation.
- **Group end-to-end encryption.** The E2E boundary is a product rule: if a server mediates the content (channels, group voice), there is no E2E; if the conversation is point-to-point, there is. No per-channel toggles, no partial E2E. The rationale and threat model live in [Messaging](../02-architecture/10-messaging.md).
- **A custom role and permission system in the core.** IRC channel modes are the permission model. Communities that want more layer it on with bots and extensions.
- **Script-tag embedding.** The embedded client is an iframe only, because the iframe provides proper origin isolation.
- **Pure native clients (Swift, Kotlin, GTK/Qt).** The ceiling of platform integration, but each one is a separate frontend codebase, and that maintenance burden would fragment a small team away from the platform every client depends on. The open protocol is explicitly designed so the community can build them: any third-party client can target the same servers with no changes to the stack.
- **Forking the IRC server.** Orbit runs a stock server and builds its value in the client layer, Satellite, and Depot. The full posture, including how protocol gaps are handled, is [Protocol Posture](../02-architecture/02-protocol-posture.md).
- **Federation, for now.** Linking independent Orbit instances needs server-to-server support that stock IRC servers don't provide. Nothing in Orbit depends on it. The design, and what it's blocked on, lives in [Federation](../02-architecture/14-federation.md).

Build status for everything above and beyond lives in the [Feature Map](../../FEATURES.md), and undecided design questions in [Open Questions](../OPEN-QUESTIONS.md).
