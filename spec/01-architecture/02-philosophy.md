# Design Philosophy

**Orbit is a transport layer and client, not an application platform.**

The core handles four things: text chat, real-time media, identity, and client UX. Everything else - calendars, events, custom moderation workflows, rich permission systems, game integrations - is an **extension**. **Orbit extensions** are client-side plugins for orbit-app that build on top of the Orbit tag namespace. **IRC bots** handle server-side automation as first-class citizens of the IRC ecosystem. Both extend Orbit at the edges without touching the core. This is an opinionated choice. Orbit stays thin so it stays fast and maintainable.

**Orbit is a Discord alternative, but it is not Discord.** The goal is not to clone Discord's feature set and ship it open-source. Orbit is a fundamentally different model for how communities communicate. Discord isolates communities into walled-off servers - you join one, you're in that box, and cross-community interaction is nonexistent. Orbit doesn't work that way. Multiple communities can share a single Uplink instance. Satellite voice sessions can cross community boundaries via BYON. A user's identity can span multiple servers via standard OIDC federation. The architecture assumes communities are porous, not sealed - and that users belong to many of them simultaneously.

This means some Discord features don't translate. Discord's complex role hierarchies exist because each server is a self-contained world that needs its own bureaucracy. Orbit uses IRC channel modes because channels are lightweight, communities overlap, and the overhead of a per-server role system is not justified. Discord's per-channel permission overrides exist because a single server hosts hundreds of channels under one roof. Orbit's answer is: if you need different permissions, make a different channel - or run a bot.

The pitch is honest: Orbit gets you 90% of the way to what you used Discord for, and the remaining 10% is a UX shift, not a missing feature. If you're tired of handing your community's data to a corporation, if you want to self-host, if you want your tools to interoperate with thirty years of IRC ecosystem - Orbit is the alternative. It is not the clone.


**Orbit's MVP runs on Ergo, a stock IRCv3 server. The planned next step is the Uplink fork - a purpose-built IRCd that remains 100% IRC-compatible while adding native support for the features Orbit needs.** Any IRCv3 server that supports message tags can become Orbit-enabled. Two users running Orbit on any compliant IRC network can use [Satellite](../02-components/02-satellite.md) for voice - even if the server operator hasn't configured any official media nodes. Users just bring their own.

**Components are independent.** [Uplink](../02-components/01-uplink/01-overview.md), [Satellite](../02-components/02-satellite.md), [Transponder](../02-components/04-transponder.md), and [Depot](../02-components/03-depot.md) have no runtime dependencies on each other. Uplink is a stock IRC server - it doesn't know Satellite exists. Satellite is a media server - it doesn't know IRC exists. Transponder bridges identity between them but neither requires it to function. Depot stores files and answers to no one. The Orbit client is the only thing that composes these components into a unified experience - but it doesn't require all of them. Connect to just Uplink and you have IRC chat. Connect to just a Satellite with a join key and you have voice. Any other client - a web page, a game, a bot - can compose a different subset of the same components using the same interfaces. The architecture is a set of independent services, not a coupled stack.

**Opinionated simplicity drives every design decision.** Permissions use IRC's built-in channel modes - `+o`, `+v`, `+b` - and nothing more. There is no custom role system, no role colors, no granular permission overrides in the core. Media is handled through independent Satellites that users can self-host (Bring Your Own Satellite). There is no centralized orchestrator bot mediating connections. If a community needs richer functionality, they build an extension. Complexity belongs at the edges, not in the core.
For communities that need role management beyond `+o`/`+v`/`+b` before extensions ship, an IRC bot connected to Uplink can maintain a role database and manage mode grants automatically - monitoring JOINs, checking `account-tag` against its role table, and applying channel modes. This is a standard IRC automation pattern and requires no Orbit-specific API.


## Honest Limitations of IRC Today

IRC is the right foundation, but it is honest to name what it does not natively support:

- **Message editing** is not available in the MVP and is not worked around. No IRCv3 standard for in-place message editing exists yet. Editing is a planned capability of the Uplink fork, where the server maintains canonical current state. Until then, it is absent - not shimmed.
- **Message retractions** use the IRC-standard `REDACT` command via `draft/message-redaction`, which is already in Ergo's git and pending stable release. This is server-enforced: when a message is retracted, the server acts on it. Orbit clients render a tombstone. IRC clients that implement the cap see the message removed. IRC clients without the cap receive a server NOTICE fallback: `*** alice retracted a message ***`. This is not a client-only tag - the gap here is narrow and closing.
- **Reactions** are not available in the MVP. They are a planned capability of the Uplink fork — not a separate service — and are not shimmed or worked around at the IRC level.
- **User metadata** (avatars, display names, presence status) is provided via `draft/metadata-2`, which is already in Ergo's git and pending stable release. Clients subscribe to keys and receive live push updates. Orbit uses the `orbit.avatar` key (a Depot URL), the `display-name` key, and the `orbit.status` key. This narrows the metadata gap significantly before the fork is needed.
- **Presence** beyond basic `AWAY`/`JOIN`/`QUIT` is handled in the MVP by `away-notify`, `extended-monitor`, and `draft/pre-away` - all stable in Ergo. Richer status states build on `draft/metadata-2` via the `orbit.status` key.
- **Threads** are implemented via client-managed sub-channels and a signaling tag. The server has no concept of a thread — it sees a channel like any other. Orbit clients render it differently. IRC clients can join the thread channel directly if they wish. This is an honest trade-off: thread channels create real server-side state, but the client abstraction is good enough and works on any existing IRC server without modification.
- **Federation** between independent Orbit instances is not available with Ergo. Ergo does not support server-to-server linking. This is a planned capability of the Uplink fork.

These limitations are not reasons to abandon IRC. They are the honest current state, and they have a clear resolution path.

## The Next Step: The Uplink Fork

The planned next step after the MVP is to **fork Ergo into a purpose-built Uplink IRCd** - one that remains 100% IRC-compatible at the protocol level while extending the server to natively support the features Orbit needs. This is not contingent on a success threshold. It is the planned continuation of the project.

The fork is not a luxury - it is the most critical technical investment after the MVP ships. Without message editing, full-text search, and federation, Orbit cannot reach the feature parity that mainstream users expect. The MVP on stock Ergo validates the architecture and proves the client. The fork is where Orbit becomes a product that competes. Plan for it from day one.


The compatibility guarantee is non-negotiable: any IRC client - WeeChat, irssi, Textual, ZNC - must continue to work against the Uplink fork. The fork extends the server; it does not break it. Standard clients connecting to an Uplink fork see a normal IRCv3 server. They get text chat, channel modes, SASL, and `chathistory`. What they don't see are the extended capabilities Orbit clients unlock on top.

It is worth noting that the fork's scope is already narrowing before it begins. Message retractions and user metadata are largely addressed by Ergo's upcoming releases (`draft/message-redaction` and `draft/metadata-2`). The fork focuses on what Ergo cannot provide:

- **Atomic message editing**: The server maintains canonical current message state. `CHATHISTORY` returns the *current* version of each message. An IRC client reconnecting sees the edited backlog - it never knows an edit happened underneath. No client-side state reconstruction from an append-only log.
- **Full-text search**: Server-side search over retained channel history. Operator-configured retention governs how much history is searchable. DM search is available for standard (non-E2E) DMs; E2E DMs cannot be searched server-side because the server holds only ciphertext.
- **Server-to-server linking**: Federation between Uplink instances implemented in the fork, on Orbit's timeline, with exactly the semantics the ecosystem needs. No dependency on Ergo upstream's roadmap.
- **Integrated push notifications**: monitoring for mentions targeting offline clients and dispatching push via FCM/APNs/UnifiedPush, handled natively by the Uplink server process.

## Message Storage

Channel message history uses **operator-configured retention**. Ergo supports this today: operators set how long history is kept per channel. There is no mandate to store everything forever. DMs use the same operator-configured retention model. When E2E encryption is active, the server stores ciphertext with the same retention and delivery mechanics - it just can't read the content. See [DMs](../02-components/01-uplink/03-dms.md) for the full two-tier storage model.

## Security Model: Transport vs. End-to-End

Orbit has two distinct layers of security. They are not interchangeable and should not be confused.

**Transport security (TLS)** protects content in transit against network-level eavesdroppers. Every connection in the Orbit stack is TLS-encrypted:

- Uplink: IRC over TLS (port 6697), WSS for web clients
- Satellite: HTTPS for the token service, DTLS-SRTP for WebRTC media (encrypted by the WebRTC spec itself, unconditionally)
- Depot: HTTPS only
- Authentication: OIDC over HTTPS, SASL over TLS

A passive observer on the network sees nothing. This is already mandated throughout the spec - no plaintext connections, no exceptions.

**End-to-end encryption** protects content from the server operator. The server terminates TLS, so the operator can see plaintext content at the server level. The rule for when E2E applies is the same across the entire Orbit stack:

| Context | E2E | Reason |
|---|---|---|
| Channels | No | Server-mediated; history, search, and moderation require the server to read content. You are trusting the server you joined. |
| Voice/video (group) | No | The Satellite SFU routes and forwards media. It must be able to read the stream. |
| 1:1 DMs (`PRIVMSG`) | Yes (post-MVP) | Two parties only. The server carries the envelope but cannot open it. Ciphertext is stored with operator-configured retention for offline delivery and history, but the server cannot read it. |
| 1:1 calls/video (P2P) | Yes | Direct WebRTC connection. The server is not in the media path after the initial handshake. |

The rule is simple: **if a server mediates the content, there is no E2E. If it is point-to-point, there is.** No exceptions, no partial E2E, no per-channel encryption toggles.

This means the server trust model is explicit and honest: when you join a channel or a Satellite room, you are trusting the operator of that server with the content of your communication. If you do not trust the operator, use a DM or a P2P call. Orbit does not obscure this trade-off.

This is a stronger and more coherent privacy story than platforms that offer partial or inconsistent E2E - where users cannot easily reason about what is and is not protected. On Orbit, the boundary is clear.

## Why Not a Custom Protocol?

The service count argument is the strongest case against a custom protocol. A purpose-built chat server does not reduce the number of services needed - it replaces IRC with something that must be maintained indefinitely, while losing the ecosystem entirely. Satellite, Depot, Transponder, coturn, Caddy - all still exist regardless of what the chat transport is. The comparison is direct:

| | Uplink path | Custom protocol path |
|---|---|---|
| Chat server | Ergo fork - extending something battle-tested | Custom - you own every bug forever |
| Push notifications | Built into the Uplink fork | Still need a separate service |
| Bot ecosystem | Every IRC library, every language, zero friction | Custom SDK, docs, language bindings |
| Minimal deployment | Single Ergo binary + Caddy | Everything coupled; nothing runs alone |
| Component independence | Any subset of services runs standalone | All services depend on all other services |

A custom protocol also collapses the component independence model. When the chat server is proprietary, every other service must know about it. Satellite, Depot, and Transponder currently have zero knowledge of each other - the client is the only compositor. A custom stack re-couples them. You cannot run "just the chat server" and have it work with existing tools, and writing a bot goes from a 50-line Python script using any IRC library to a custom integration against a proprietary API.

The Uplink path keeps thirty years of IRC tooling, keeps the ecosystem, keeps the bot story, and adds the missing features on top - strictly better, at the cost of eventually owning an IRCd. That cost is real, but it is paid on a planned schedule, not deferred indefinitely.
