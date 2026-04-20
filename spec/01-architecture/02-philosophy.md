# Design Philosophy

## What Orbit Is Solving

The problem with every Discord alternative is that they try to out-Discord Discord. They build custom protocols, custom servers, custom permission systems, custom everything - and then spend years catching up to feature parity while Discord moves further ahead. None of them work everywhere. None of them are easy to self-host. None of them make it trivial to write a bot or integrate with external tooling.

Discord won for one reason above all others: it works everywhere. Browser tab, desktop app, mobile, embedded widget on a website. TeamSpeak is better audio. Mumble is lighter. IRC has more history and more tooling. None of them run in a browser tab without friction. Discord does, and that is why communities migrated.

Orbit's answer is not to rebuild Discord. It is to close the accessibility gap that made Discord the default - while building on a foundation that communities can actually own.

## The Foundation

IRC has been running communities reliably for over thirty years. The protocol is open, well-documented, and owned by no one. The ecosystem is vast: every language has mature IRC libraries, bot frameworks have existed for decades, and server software is battle-tested at scale. A modern IRCv3 server runs on minimal hardware and handles thousands of concurrent connections with ease.

The missing piece has always been the client. Not the protocol - the client.

Orbit is that client. A desktop application, a web app, an embeddable widget that any website can drop in with an iframe. All of them backed by the same IRC server you could have been running for years. The protocol is unchanged. The ecosystem is unchanged. What changes is that a non-technical user can now join a community from their browser, and a website owner can embed a live chat widget without standing up anything new.

## The Uplink Fork Is the Product

The MVP ships on stock Ergochat. This is not the destination - it is the starting point. The MVP validates the architecture and proves the client layer works. It ships a usable product for communities that want to move today.

**The Uplink fork is the product.** A purpose-built IRCd, forked from Ergo, that remains 100% IRC-compatible at the protocol level while adding the capabilities that stock Ergo cannot provide: atomic message editing, message reactions, full-text search, integrated push notifications, and server-to-server linking for federation. Every architectural decision in this spec is made with the fork in mind. The MVP is step one. The fork is the platform.

The fork does not change the compatibility guarantee. Any IRC client - WeeChat, irssi, Textual, ZNC - connects to an Uplink fork and sees a normal IRCv3 server. Standard clients get text chat, channel modes, SASL, and `chathistory`. Orbit clients get the full experience on top. The fork extends the server. It does not break it.

The fork's scope is already narrowing before it begins. `draft/message-redaction` and `draft/metadata-2` - both already in Ergo's git and pending stable release - address message retractions and user metadata without requiring a fork at all. What the fork owns is what Ergo structurally cannot provide on its current architecture: atomic in-place editing, server-side search, server-to-server linking, and native push delivery.

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

[Uplink](../02-components/01-uplink/01-overview.md), [Satellite](../02-components/02-satellite.md), [Transponder](../02-components/04-transponder.md), and [Depot](../02-components/03-depot.md) have no runtime dependencies on each other. Uplink is a stock IRC server - it does not know Satellite exists. Satellite is a media server - it does not know IRC exists. Transponder bridges identity between them but neither requires it to function. Depot stores files and answers to no one.

The Orbit client is the only thing that composes these components into a unified experience - but it does not require all of them. Connect to just Uplink and you have IRC chat. Connect to just a Satellite with a join key and you have voice. Any other client - a web page, a game, a bot - can compose a different subset of the same components using the same interfaces.

This is why a custom protocol is the wrong answer. A custom protocol does not reduce the number of services. Satellite, Depot, Transponder, coturn, Caddy - all of them still exist regardless of what the chat transport is. What a custom protocol does is replace a battle-tested IRC server with something you must maintain forever, eliminate the bot ecosystem, and re-couple every service to every other service. The Uplink path keeps thirty years of tooling and adds what is missing on top. That is strictly better, at the cost of eventually owning an IRCd - a cost paid on a planned schedule, not deferred indefinitely.

## Honest Limitations of IRC Today

IRC is the right foundation, but it is honest to name what it does not natively support and how each gap is resolved:

- **Message editing** is not available in the MVP and is not worked around. No IRCv3 standard for in-place message editing exists today. Editing is a fork milestone: the server maintains canonical current message state, and `CHATHISTORY` returns the current version of each message. Until the fork ships it, editing is absent - not shimmed.
- **Message retractions** use the IRC-standard `REDACT` command via `draft/message-redaction`, already in Ergo's git and pending stable release. Server-enforced. Not a client tag. Orbit clients render a tombstone. IRC clients without the cap receive a `NOTICE` fallback. The gap is narrow and closing.
- **Reactions** are not in the MVP and are not shimmed. They are a fork milestone.
- **User metadata** (avatars, display names, presence status) is handled by `draft/metadata-2`, already in Ergo's git and pending stable release. Clients subscribe to keys and receive live push updates.
- **Presence** beyond `AWAY`/`JOIN`/`QUIT` is handled by `away-notify`, `extended-monitor`, and `draft/pre-away` - all stable in Ergo today. Richer status strings build on `draft/metadata-2`.
- **Threads** are implemented via client-managed sub-channels and a signaling tag. The server sees a normal channel. Orbit clients render the thread UI. IRC clients can `/join` the thread channel directly. This works on any IRCv3 server without modification and is an honest trade-off for the MVP.
- **Federation** requires server-to-server linking, which Ergo does not currently support. This is a fork milestone. Single-instance Ergo scales vertically and supports high-availability deployment via Kubernetes in the interim.

None of these are reasons to abandon IRC. They are the current state, each with a defined resolution path.

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
