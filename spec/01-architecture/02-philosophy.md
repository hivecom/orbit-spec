# Design Philosophy

**Orbit is a transport layer and client, not an application platform.**

The core handles four things: text chat, real-time media, identity, and client UX. Everything else - calendars, events, custom moderation workflows, rich permission systems, game integrations - is an **extension**. **Orbit extensions** are client-side plugins for the orbit-app that build on top of the Orbit tag namespace. **IRC bots** handle server-side automation as first-class citizens of the IRC ecosystem. Both extend Orbit at the edges without touching the core. This is an opinionated choice. Orbit stays thin so it stays fast and maintainable.

**Orbit is a layer on top of existing IRC.** Any IRCv3 server that supports message tags can become Orbit-enabled. Two users running Orbit on any compliant IRC network can use [Satellite](../02-components/02-satellite.md) for voice - even if the server operator hasn't configured any official media nodes. Users just bring their own.

**Components are independent.** [Ground Control](../02-components/01-ground-control/01-overview.md), [Satellite](../02-components/02-satellite.md), [Transponder](../02-components/04-transponder.md), and [Depot](../02-components/03-depot.md) have no runtime dependencies on each other. Ground Control is a stock IRC server - it doesn't know Satellite exists. Satellite is a media server - it doesn't know IRC exists. Transponder bridges identity between them but neither requires it to function. Depot stores files and answers to no one. The Orbit client is the only thing that composes these components into a unified experience - but it doesn't require all of them. Connect to just Ground Control and you have IRC chat. Connect to just a Satellite with a join key and you have voice. Any other client - a web page, a game, a bot - can compose a different subset of the same components using the same interfaces. The architecture is a set of independent services, not a coupled stack.

**Opinionated simplicity drives every design decision.** Permissions use IRC's built-in channel modes - `+o`, `+v`, `+b` - and nothing more. There is no custom role system, no role colors, no granular permission overrides in the core. Media is handled through independent Satellites that users can self-host (Bring Your Own Satellite). There is no centralized orchestrator bot mediating connections. If a community needs richer functionality, they build an extension. Complexity belongs at the edges, not in the core.

## Honest Limitations of IRC Today

IRC is the right foundation, but it is honest to name what it does not natively support:

- **Message editing and deletion** are implemented as new messages with client-only tags referencing the original `msgid`. The server stores both the original and the operation as separate entries. Clients reconstruct canonical state from an append-only log. This works, but a partial history load can surface unedited content - enforcement is client-side, not server-enforced.
- **Reactions** require aggregated per-message state that IRC's append-only history cannot serve reliably. They are deferred to a dedicated service (Reactor) rather than being a native server feature.
- **Threads** are implemented via client-managed sub-channels and a signaling tag. The server has no concept of a thread - it sees a channel like any other. Orbit clients render it differently. IRC clients can join the thread channel directly if they wish.
- **Presence** is limited to `AWAY` and `JOIN`/`QUIT`. Richer status states (idle, do-not-disturb, invisible) are not natively available.
- **Federation** between independent Orbit instances is not available with Ergo today. Ergo does not support server-to-server linking.

These limitations are not reasons to abandon IRC. They are the honest current state, and they have a clear resolution path.

## The Long-Term Path: Ground Control as a True IRCd

If Orbit reaches the point where these limitations become genuine product constraints, the path forward is to **fork Ergo into a purpose-built Ground Control IRCd** - one that remains 100% IRC-compatible at the protocol level while extending the server to natively support the features Orbit needs.

The compatibility guarantee is non-negotiable: any IRC client - WeeChat, irssi, Textual, ZNC - must continue to work against a Ground Control fork. The fork extends the server; it does not break it. Standard clients connecting to a Ground Control fork see a normal IRCv3 server. They get text chat, channel modes, SASL, and `chathistory`. What they don't see are the extended capabilities Orbit clients unlock on top.

What the fork enables natively:

- **Atomic edits and deletions**: The server maintains canonical current message state. `CHATHISTORY` returns the *current* version of each message. An IRC client reconnecting sees the edited backlog - it never knows an edit happened underneath. No client-side state reconstruction from an append-only log.
- **Native reaction aggregation**: A server-handled reaction command that the server aggregates and serves. Reactor disappears as a separate service. The server becomes the source of truth for reaction counts.
- **Native thread management**: Thread sub-channels become first-class server objects. The server manages lifecycle - archiving idle threads, suppressing them from `LIST` for IRC clients that don't understand threads, serving reply counts natively.
- **Server-to-server linking**: Federation between Ground Control instances implemented in the fork, on Orbit's timeline, with exactly the semantics the ecosystem needs. No dependency on Ergo upstream's roadmap.
- **Integrated push notifications**: Beacon's functionality - monitoring for mentions targeting offline clients, dispatching push via FCM/APNs/UnifiedPush - moves into the Ground Control process. No separate service, no IRC bot pretending to be a user.

The fork is not a day-one decision. It is the natural growth path if Orbit's adoption justifies it. Until then, Ergochat is an excellent production IRC server and the right foundation for the MVP and well beyond.

## Why Not a Custom Protocol?

The service count argument is the strongest case against a custom protocol. A purpose-built chat server does not reduce the number of services needed - it replaces IRC with something that must be maintained indefinitely, while losing the ecosystem entirely. Satellite, Depot, Transponder, coturn, Caddy - all still exist regardless of what the chat transport is. The comparison is direct:

| | IRC fork path | Custom protocol path |
|---|---|---|
| Chat server | Ergo fork - extending something battle-tested | Custom - you own every bug forever |
| Reactions | Built into the fork | Still need a separate aggregation layer |
| Push notifications | Built into the fork | Still need a separate service |
| Bot ecosystem | Every IRC library, every language, zero friction | Custom SDK, docs, language bindings |
| Minimal deployment | Single Ergo binary + Caddy | Everything coupled; nothing runs alone |
| Component independence | Any subset of services runs standalone | All services depend on all other services |

A custom protocol also collapses the component independence model. When the chat server is proprietary, every other service must know about it. Satellite, Depot, and Transponder currently have zero knowledge of each other - the client is the only compositor. A custom stack re-couples them. You cannot run "just the chat server" and have it work with existing tools, and writing a bot goes from a 50-line Python script using any IRC library to a custom integration against a proprietary API.

The fork path keeps thirty years of IRC tooling, keeps the ecosystem, keeps the bot story, and adds the missing features on top - strictly better, at the cost of eventually owning an IRCd. That cost is real, but it is paid when the project is successful enough to warrant it, not before.
