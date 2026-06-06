# Design Philosophy

## What Orbit Is Solving

The problem with every Discord alternative is that they try to out-Discord Discord. They build custom protocols, custom servers, custom permission systems, custom everything - and then spend years catching up to feature parity while Discord moves further ahead. None of them work everywhere. None of them are easy to self-host. None of them make it trivial to write a bot or integrate with external tooling.

Discord won for one reason above all others: it works everywhere. Browser tab, desktop app, mobile, embedded widget on a website. TeamSpeak is better audio. Mumble is lighter. IRC has more history and more tooling. None of them run in a browser tab without friction. Discord does, and that is why communities migrated.

Orbit's answer is not to rebuild Discord. It is to close the accessibility gap that made Discord the default - while building on a foundation that communities can actually own.

## The Foundation

IRC has been running communities reliably for over thirty years. The protocol is open, well-documented, and owned by no one. The ecosystem is vast: every language has mature IRC libraries, bot frameworks have existed for decades, and server software is battle-tested at scale. A modern IRCv3 server runs on minimal hardware and handles thousands of concurrent connections with ease.

The missing piece has always been the client. Not the protocol - the client.

Orbit is that client. A desktop application, a web app, an embeddable widget that any website can drop in with an iframe. All of them backed by the same IRC server you could have been running for years. The protocol is unchanged. The ecosystem is unchanged. What changes is that a non-technical user can now join a community from their browser, and a website owner can embed a live chat widget without standing up anything new.

## Where Orbit's Value Lives

Orbit is not trying to beat Discord. It is trying to make defection cheap, pleasant, and boringly reliable - so that when people are ready to leave, the door is already open.

The pitch is concrete: everything your friend group needs from Discord, privately, on a $5 VPS. `docker compose up`. For less than $10 a month you get voice, text, files, identity - all self-hosted, all yours. Every alternative that died tried to out-Discord Discord with custom protocols, custom servers, and years spent catching up. They failed because they were not decoupled, they scaled poorly on cost, and they were horrible to maintain. Orbit survives by not owning what it does not have to.

The goal is not a product to sell. It is infrastructure and a cultural push - resistance to surveillance and platform lock-in. If Orbit ever provides hosted instances, that is a small convenience cut for people who do not want to operate their own box, not a SaaS play.

The way Orbit wins on the client side is UX. It has always been UX. People stay on Discord because it is easy. The bar: make leaving take an afternoon instead of a research project. Anonymous access, invite a friend to a call without them making an account, works in the browser, works on a phone. The onboarding is gone. The product demonstrates itself; the viral loop is the absence of a funnel.

IRC already does the "decentralized but nobody cares" thing naturally. An Orbit client can connect to any existing IRC network. Layering a modern client onto existing networks is the slow creep - adoption does not require a migration.

Orbit does not fork the IRC server, and it is not in the business of reimplementing IRC. Orbit's value is the whole experience: a polished client layer together with Satellite and Depot, orchestrated into one cohesive, seamless product. The work is UX and integration - making a thirty-year-old protocol feel modern and effortless - not owning the transport.

The posture toward the protocol is simple: Orbit conforms to IRCv3, and whatever stock Ergo implements, the Orbit client supports. Much of what a private fork was once imagined for is already native in Ergo (push notifications, OIDC/JWT authentication, user metadata, message retraction, an HTTP API, and pluggable history backends), so there is nothing to fork. Where IRC has not standardized something yet, Orbit follows the same draft work the rest of the ecosystem does and handles the remainder at the client and tag layer. If a capability is standardized in IRCv3, great; if Ergo ships it, Orbit gets it for free; if neither has happened, Orbit adapts. We would contribute upstream where it helps, but the leverage is the product, not the protocol.

Federation is not a goal for now. It would need server-to-server linking that stock IRC servers do not provide; Ergo may add it or Orbit may help upstream, but nothing in Orbit depends on it.

## Component Classes

Orbit is made of two classes of parts. The distinction is between roles Orbit adopts and software Orbit builds.

**Abstractions** are adopted roles fulfilled by third-party software. Orbit specifies a contract and adopts an implementation; it does not build the substance.

- **Uplink** is any stock IRCv3 server; Ergo is the reference implementation. There is no Uplink fork. Orbit runs the server stock and adapts at the client layer where IRC has gaps.
- **Transponder** is any OIDC-compliant identity provider (Keycloak, Authentik, Zitadel, Supabase). It is a role, not software Orbit builds.

**Components** are bespoke services Orbit builds and owns.

- **Satellite** is the real-time media product. It embeds LiveKit as the SFU but owns substantial bespoke logic: the session model, 1:1 P2P over IRC tags, moderation, discovery, and Bring Your Own Satellite. LiveKit is a substrate it embeds, not what Satellite is.
- **Depot** is a thin storage gateway that abstracts an S3-compatible backend or a local filesystem behind one contract: verify a credential, apply policy, and issue a pre-signed S3 URL or a proxied filesystem upload. It is not a UI app.
- **Clients** are the desktop, web, and widget surfaces where product value lives.

The rule: an abstraction is Uplink or Transponder only. Depot and Satellite are not abstractions; they are bespoke components Orbit builds. Satellite embeds LiveKit and Depot abstracts S3 or disk, but both are built and owned by Orbit.

## What Orbit Is Not

**Orbit is not a Discord clone.** This is not a statement about ambition - it is a statement about model.

Discord isolates communities into walled-off servers. Every server is its own world, with its own roles, its own permission matrix, its own channels. Cross-community interaction is nonexistent by design. You are in the box or you are not.

Orbit works the way IRC always has. A single Uplink instance hosts many communities simultaneously. Channels are lightweight. Communities are porous - users belong to many at once, and the server does not enforce artificial boundaries between them. A community on Orbit is a set of channels, not a walled namespace.

This means some Discord features do not translate, and that is intentional:

- **Complex role hierarchies** exist in Discord because each server is a self-contained organization that needs its own bureaucracy. On Orbit, communities share infrastructure. IRC channel modes - `+o`, `+v`, `+b` - cover the permission model. Communities that need richer automation run an IRC bot. That bot story is one of Orbit's genuine strengths: any IRC bot library in any language, zero Orbit-specific API required.
- **Per-channel permission overrides** exist in Discord because one server hosts hundreds of channels under one administrative roof. On Orbit, if a community's needs require a different administrative boundary, they run their own Uplink instance. That is a single-binary deployment. It has always been how IRC networks are structured.
- **Server-level settings and dashboards** exist in Discord because operators have no other way to manage their walled garden. On Orbit, operators have thirty years of IRC management tooling: `SAMODE`, channel modes, ban lists, `SAJOIN`, services bots, monitoring, everything. The tools are already there.

The transition from one platform to another is always different from what you had before. People moved from TeamSpeak and Skype to Discord even though the model changed significantly. The bet here is the same: what Orbit gets right matters more than what it does differently. Users will not miss what they did not need.

## Orbit Is a Transport Layer and Client, Not an Application Platform

The core handles four things: text chat, real-time media, identity, and client UX. Everything else - calendars, events, custom moderation workflows, rich permission systems, game integrations - is an extension. **Orbit extensions** are client-side plugins for orbit-app that build on top of the Orbit tag namespace. **IRC bots** handle server-side automation as first-class citizens of the IRC ecosystem. Both extend Orbit at the edges without touching the core.

Orbit stays thin so it stays fast and maintainable. Complexity belongs at the edges.

## Components Are Independent

[Uplink](../02-components/01-uplink/01-overview.md), [Satellite](../02-components/02-satellite.md), [Transponder](../02-components/04-transponder.md), and [Depot](../02-components/03-depot.md) have no runtime dependencies on each other. Uplink is a stock IRC server - it does not know Satellite exists. Satellite is a media server - it does not know IRC exists. Transponder is the adopted OIDC provider; components verify its JWTs natively (Ergo via its own JWT/OAUTHBEARER support), so identity is native-first, with any bridge being optional. Depot stores files and answers to no one.

The Orbit client is the only thing that composes these components into a unified experience - but it does not require all of them. Connect to just Uplink and you have IRC chat. Connect to just a Satellite with a join key and you have voice. Any other client - a web page, a game, a bot - can compose a different subset of the same components using the same interfaces.

This is why a custom protocol is the wrong answer. A custom protocol does not reduce the number of services. Satellite, Depot, Transponder, coturn, Caddy - all of them still exist regardless of what the chat transport is. What a custom protocol does is replace a battle-tested IRC server with something you must maintain forever, eliminate the bot ecosystem, and re-couple every service to every other service. Adopting a stock IRCv3 server keeps thirty years of tooling and lets Orbit build what is missing on top. That is strictly better: Orbit owns the client layer and two bespoke services, not an IRCd it must maintain forever.

## Honest Limitations of IRC Today

IRC is the right foundation, but it is honest to name what it does not natively support and how each gap is resolved:

- **Message editing** is not standardized in IRC yet. Orbit follows the draft work happening across the IRC ecosystem and handles editing at the client and tag layer; it does not fork the server to add it, and will adopt the standard if and when it lands.
- **Message retractions** use the IRC-standard `REDACT` command via `draft/message-redaction`, shipped and stable in Ergo. Server-enforced. Not a client tag. Orbit clients render a tombstone. IRC clients without the cap receive a `NOTICE` fallback.
- **Reactions** work today, handled client-side via message tags. The one concession is historic search: reactions cannot be shown on messages surfaced purely from a search result.
- **User metadata** (avatars, display names, presence status) is handled by `draft/metadata-2`, stable in Ergo (v2.17.0+). Clients subscribe to keys and receive live push updates.
- **Presence** beyond `AWAY`/`JOIN`/`QUIT` is handled by `away-notify`, `extended-monitor`, and `draft/pre-away` - all stable in Ergo today. Richer status strings build on `draft/metadata-2`.
- **Threads** are implemented via client-managed sub-channels and a signaling tag. The server sees a normal channel. Orbit clients render the thread UI. IRC clients can `/join` the thread channel directly. This works on any IRCv3 server without modification and is an honest trade-off for the MVP.
- **Federation** needs server-to-server linking that stock IRC servers do not provide. It is not a goal for now; Ergo may add it or Orbit may help upstream, but nothing depends on it. Single-instance Ergo scales vertically and supports high-availability deployment via Kubernetes.

None of these are reasons to abandon IRC. They are the current state, each with a defined resolution path.

## Moderation and Trust

IRC moderation at scale is not hypothetical. Freenode ran roughly 90,000 concurrent users for twenty years with channel ops, services, bots, and network bans. Libera inherited that model and continues it. The primitives work. Orbit adds good client-side UX on top of them - operator tooling, moderation panels, visible trust indicators - but does not reinvent the permission model.

The thing that actually killed Freenode was not a moderation failure. It was a governance and ownership failure - the 2021 hostile takeover. Everyone migrated to Libera in a week. That cuts in Orbit's favor: when your platform is decoupled and built on a standard protocol with no lock-in, what would be a death for a centralized service becomes a one-week migration. Portability over permanence is a stronger defense than moderation features alone.

Orbit is not text-only like traditional IRC. Depot handles file uploads and Satellite handles voice and video. These add a content surface IRC never had, and scale requires additional care here. The posture is layered:

- **Shared content (files via Depot):** Every upload is attributed to a user identity via OIDC. Content is removable by operators. Depot provides a gateway chokepoint where automated scanning can be bolted on at scale. Small instances do not need it; large public ones can add it. The safe default: anonymous users can join calls and chat, but cannot upload files to public channels (unless configured and desired).
- **Private content (P2P calls, E2E DMs):** The shield is cryptographic, not topological. End-to-end encryption plus non-retention means Orbit cannot see, decrypt, or produce 1:1 content. "We cannot decrypt and did not store" is the posture. This is the same model as Signal.

The principled summary: Orbit is infrastructure. For private content, it architecturally cannot see it. For shared content, it attributes it and can remove it.

## Message Storage

Channel history uses operator-configured retention. Ergo supports this today - operators set how long history is kept per channel. There is no mandate to store everything forever, and no mandated default. DMs use the same model. When E2E encryption is active, the server stores ciphertext with the same retention and delivery mechanics; it just cannot read the content.

## Security Model: Transport vs. End-to-End

Two distinct layers of security. They are not interchangeable.

**Transport security (TLS)** protects content in transit. Every connection in the Orbit stack is TLS-encrypted - no plaintext, no exceptions. A passive observer on the network sees nothing.

**End-to-end encryption** protects content from the server operator. The server terminates TLS, so the operator can read plaintext at the server level. The rule for when E2E applies:

| Context | E2E | Reason |
|---|---|---|
| Channels | No | Server-mediated. History, search, and moderation require the server to read content. You are trusting the operator. |
| Voice / video (group) | No | The Satellite SFU routes and forwards media. It must be able to read the stream. |
| 1:1 DMs | Yes (post-MVP) | Two parties only. The server carries the envelope but cannot open it. |
| 1:1 calls / video (P2P) | Yes | Direct WebRTC connection. The server is not in the media path. |

The rule is simple: **if a server mediates the content, there is no E2E. If it is point-to-point, there is.** No exceptions, no per-channel toggles, no partial E2E.

This is a stronger privacy story than platforms that offer inconsistent or partial E2E. On Orbit the boundary is explicit: when you join a channel or a Satellite room, you are trusting the operator of that server. If you do not trust the operator, use a DM or a P2P call. Orbit does not obscure this trade-off.
