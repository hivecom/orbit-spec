# Ground Control

Ground Control is the IRC backbone of an Orbit deployment. It provides persistent text chat, user
authentication, signaling transport for Satellite session coordination, and server-side message
history. Ground Control is implemented using [Ergochat](https://ergo.chat/) - a modern,
standards-compliant IRCv3 server.

Ground Control is the only required component in an Orbit deployment. All other components
([Satellite](../02-satellite.md), [Depot](../03-depot.md),
[Transponder](../04-transponder.md)) are optional. A Ground Control instance alone is a
fully functional, backward-compatible IRC server that any IRCv3 client can connect to.

The [Orbit tag namespace](02-tags/01-namespace.md) is transported over Ground Control's `message-tags`
layer as `+`-prefixed client-only tags. For DNS-based service advertisement, see
[DNS & Service Discovery](../../05-infrastructure/01-domain-discovery.md).

## Why IRCv3

| Property                     | Benefit                                                              |
|------------------------------|----------------------------------------------------------------------|
| Ultra-low resource footprint | Ergochat serves thousands of connections on minimal hardware         |
| Native WebSocket support     | Browser clients connect directly via the reverse proxy's WebSocket route - no separate IRC gateway or protocol translation needed. |
| Message tags                 | Rich metadata transport without polluting message content            |
| Battle-tested protocol       | 30+ years of real-world usage; failure modes are well understood     |
| Backward compatibility       | Users can connect with WeeChat, irssi, or any IRCv3 client          |
| Built-in federation model    | Server-to-server protocol exists (deferred to v0.2+)                |
| Chathistory extension        | Server-side message history with standardized retrieval              |

IRCv3 gives us a text and signaling transport that works today, scales vertically to the community
sizes targeted in the MVP, and does not lock us into a proprietary protocol.

## Required IRCv3 Extensions

The following extensions MUST be enabled on any Ergochat instance serving as Ground Control.

| Extension          | Purpose                                                                                                                         |
|--------------------|---------------------------------------------------------------------------------------------------------------------------------|
| `message-tags`     | Transport media signaling and rich metadata                                                                                     |
| `batch`            | Group related messages (history playback, netsplit recovery)                                                                    |
| `chathistory`      | Server-side message history retrieval                                                                                           |
| `server-time`      | Accurate timestamps on all messages                                                                                             |
| `labeled-response` | Correlate requests with responses for async client operations                                                                   |
| `sasl`             | Authentication (PLAIN and SCRAM over TLS)                                                                                       |
| `message-ids`      | Unique message identifiers for amendments, retractions, and replies                                                                      |
| `WebSocket`        | Native WebSocket transport on Ergochat's listener                                                                               |
| `account-tag`      | Server-asserted account identity; used by clients for authorship verification and identity display - cannot be forged by clients |
| `echo-message`     | Client receives its own messages back from the server with server-applied tags (`server-time`, `msgid`)                         |

`account-tag` and `echo-message` are critical to Orbit's client-side trust model. `account-tag` is
the authoritative source of sender identity throughout the Orbit tag system; `echo-message` ensures
that the client's own messages carry server-assigned identifiers required for edit and delete
references.

For the full set of `+orbit/*` client-only tags transported over `message-tags`, see
[Orbit Tag Namespace](02-tags/01-namespace.md).

## Ergochat Configuration

Key configuration points for an Orbit-compatible Ergochat instance:

- **WebSocket listener**: Ergochat exposes a WebSocket endpoint on a dedicated port. In production, TLS for this endpoint is terminated by a reverse proxy (Caddy in the reference deployment) which proxies to Ergochat's internal WebSocket port. Ergochat CAN terminate TLS directly for raw IRC connections (port 6697), but the reverse proxy is required for any deployment that serves web clients or needs to host `/.well-known/orbit/services.json` for service discovery. See [Domain Discovery](../../05-infrastructure/01-domain-discovery.md) for the well-known file format.
- **History storage**: Enabled with configurable per-channel retention (default: 7 days or 10,000
  messages, whichever comes first).
- **SASL**: Required for registered users. SASL PLAIN and SCRAM-SHA-256 over TLS. For the MVP, Ergochat's built-in account database handles credential verification. Post-MVP, Ergochat can delegate SASL verification to an external OIDC identity provider via the `auth-script` configuration option and a thin auth-script bridge. This enables any OIDC-compliant provider (Keycloak, Authentik, etc.) to serve as the identity backend without modifying the IRC server. When an identity provider is configured, **NickServ must be disabled** - the provider is the sole authority for account management (registration, password changes, identity verification). Ergochat's nickname enforcement continues to work because it is tied to account login, not to NickServ specifically; once `auth-script` confirms an account name, nickname reservation and enforcement behave identically to the NickServ path. See [Transponder](../04-transponder.md) for details.
- **Client-only tag allowlist**: Ergochat relays all `+`-prefixed tags by default per IRCv3 spec.
  No special configuration needed, but the server should enforce maximum tag size limits.
- **Connection limits**: Per-IP connection limits configured to prevent abuse. Browser-based web
  clients connect directly, so limits should accommodate multiple simultaneous guest connections
  from the same IP (e.g., shared NAT or multiple browser tabs).
- **Nickname reservation**: The `guest-` prefix MUST be reserved. In the MVP (NickServ active), configure NickServ to reject registration of any nickname starting with `guest-`. When an OIDC identity provider is configured and NickServ is disabled, the provider's account management ensures no user can register a `guest-` prefixed username. Either way, this prevents collision between registered users and anonymous widget guests.

For DNS SRV record configuration that makes this instance discoverable by Orbit clients, see
[DNS & Service Discovery](../../05-infrastructure/01-domain-discovery.md).

## Channels

IRC channels are flat - there is no hierarchy, no categories, no sub-channels at the protocol
level. The server has no knowledge of any naming convention. Orbit clients use **dot notation** as
a client-side rendering convention: if a channel name contains dots (e.g., `#dev.frontend`,
`#dev.backend`), the Orbit client interprets the dots as path separators and renders those channels
in a collapsible tree in the sidebar. Channels without dots appear at the top level. This is
entirely a client rendering decision - the IRC server sees flat channel names as always.

For the full dot-notation rendering rules and edge cases, see
[Desktop Client - Channel Organization](../../04-clients/01-desktop.md#channel-organization).

For real-time session chat (quick callouts during voice, links shared during a screen share),
[Satellite](../02-satellite.md) provides its own ephemeral chat via LiveKit data channels.
This is intentionally separate from IRC: ephemeral messages are not persisted, not searchable, and
not visible to IRC clients. Persistent, historical chat belongs in Ground Control. Throwaway,
in-session chat belongs in Satellite.

## Message Amending, Retracting, and Replies

Each message has a unique ID assigned by Ergochat via the `message-ids` extension. The
`echo-message` extension ensures the sending client receives this server-assigned ID for its own
messages.

**To amend a message:** The client sends a `PRIVMSG` with a `+orbit/msg-amend` tag referencing the
original `msgid`, with the replacement content as the message body.

**To retract a message:** The client sends a `TAGMSG` with a `+orbit/msg-retract` tag referencing
the original `msgid`. No message body is required.

**To reply to a message:** The client sends a `PRIVMSG` with a `+orbit/msg-reply` tag referencing
the original `msgid`, with the reply content as the message body. The Orbit client renders the reply
with an inline excerpt of the original message (if available in the local buffer) and a clickable
link to navigate to the original. If the original message is not in the buffer, the reply is
displayed without the excerpt. Pure IRC clients see reply messages as a normal `PRIVMSG` - the
reply context is invisible to them but the reply text is fully readable.

**Client rendering:**

- Orbit clients render amendments inline with an "amended" indicator and replace retracted messages
  with a tombstone for moderator audit.
- Pure IRC clients amend messages as a new `PRIVMSG` prefixed with `(amended)` and retract
  notifications as `(retracted message <id>)`. Not ideal, but functional - these clients continue to
  work without understanding the tags.

For client-side enforcement rules governing who may send amendments and retractions, see
[Tag Integrity and Trust Model](02-tags/02-trust-model.md).

For file uploads posted as messages in channels, see [Depot](../03-depot.md).

### Interaction with Chat History

When a client fetches messages via `chathistory`, it receives messages in chronological order - including amendment and retraction operations as separate messages. The server does not collapse edits or remove deleted messages from the history stream. This means:

- An **amended message** appears in history as two entries: the original `PRIVMSG` and the subsequent `PRIVMSG` carrying the `+orbit/msg-amend` tag. The client reconstructs the final state by applying amendments to their target messages (matched via `msgid`).
- A **retracted message** appears in history as two entries: the original `PRIVMSG` and the subsequent `TAGMSG` carrying the `+orbit/msg-retract` tag. The client applies the retraction by replacing the original message with a tombstone.
- A **reply** appears in history as a `PRIVMSG` carrying a `+orbit/msg-reply` tag referencing the target `msgid`. If the target message is within the loaded history, the client renders the reply with an inline excerpt. If the target is outside the loaded window, the reply is displayed without the excerpt - the reply text is always self-contained and readable on its own.

Orbit clients MUST process `chathistory` results in order and apply all amendment and retraction operations before rendering. If an amendment or retraction references a `msgid` the client has not yet seen (e.g., the target message was outside the requested history window), the operation is silently discarded.

## Threads

Orbit supports threaded conversations on top of standard IRC channels. Threads are implemented
entirely at the client level using standard IRC primitives - no server patches and no bots are
required for the MVP. Any Orbit client can create and participate in threads. Standard IRC clients
can also read and reply to threads by joining the thread channel directly.

### Thread Channels

A thread is a normal IRC channel whose name is derived from the parent channel and the `msgid` of
the message being threaded:

```
<parent-channel>.t-<msgid>
```

For example, a thread on message `abc123` in `#dev` lives at `#dev.t-abc123`. Because this follows
the dot-notation convention, Orbit clients render it as a child of `dev` in the sidebar hierarchy
- but thread channels are **always hidden from the main channel list** in the Orbit UI. They are
surfaced only through the thread panel attached to the parent message.

When a user posts the first reply to a message in thread mode, the Orbit client:

1. JOINs `<parent>.t-<msgid>` - IRC creates the channel automatically on the first JOIN.
2. Sets `+s` (secret) mode immediately, so the thread channel is excluded from `/list` output for
   IRC clients that are not thread-aware.
3. Sets the TOPIC of the thread channel to the original message content, attributed to its author:
   ```
   <original message text> - @alice, 2025-01-15 20:30 UTC
   ```
4. Sends the reply as a `PRIVMSG` in the thread channel.
5. Sends a `PRIVMSG` to the **parent channel** with a human-readable notice pointing IRC clients
   to the thread channel:
   ```
   ↳ Thread started by alice in #dev.t-abc123
   ```
   This message is visible to all IRC clients and allows them to follow the thread by joining the
   named channel directly. Orbit clients suppress this message from the chat view and render the
   thread indicator instead.
6. Sends a `+orbit/msg-thread` `TAGMSG` to the **parent channel**, signaling that a thread now
   exists on the target message.

For each subsequent reply, the client JOINs the thread channel (if not already joined), sends the
`PRIVMSG` there, and sends another `+orbit/msg-thread` `TAGMSG` to the parent channel as an
activity signal.

### The `+orbit/msg-thread` Signal Tag

The `+orbit/msg-thread` tag is sent as a `TAGMSG` to the parent channel - not to the thread channel
- both when a thread is first created and on every subsequent reply. This serves two purposes:

- **Discovery**: Orbit clients loading history via `chathistory` can find existing threads by
  scanning for `+orbit/msg-thread` TAGMSGs. The `chathistory` stream for the parent channel
  becomes the complete index of all threads in that channel.
- **Live updates**: Clients already connected to the parent channel receive the TAGMSG when any
  reply is posted, allowing them to update thread unread indicators without having joined the
  thread channel themselves.

Tag payload (base64 JSON):

```json
{
  "parent_msgid": "abc123",
  "thread_channel": "#dev.t-abc123"
}
```

For the tag's position in the full namespace and trust model rules, see
[Tag Namespace](02-tags/01-namespace.md) and [Tag Trust Model](02-tags/02-trust-model.md).

### Orbit Client Rendering

- **Thread indicator**: Messages that have received at least one `+orbit/msg-thread` signal display
  a reply count button inline below the message content.
- **Thread panel**: Clicking the reply count opens a thread panel showing the original message at
  the top, followed by all thread replies in chronological order fetched via `chathistory` on the
  thread channel.
- **Thread composition**: Users compose thread replies in the thread panel. The client handles the
  JOIN, `+s` mode, TOPIC, TAGMSG, and PRIVMSG operations transparently - the user sees a reply
  input, not IRC primitives.
- **Reply count tracking**: Orbit clients that have not joined the thread channel increment the
  reply count on each `+orbit/msg-thread` TAGMSG received for a given `parent_msgid`. This is a
  client-side approximation sufficient for unread indicators without requiring every client to
  pre-join every thread channel.
- **Thread channels are hidden** from the Orbit channel list and server browser. They are never
  presented as joinable channels in the Orbit UI.

### IRC Client Experience

Standard IRC clients are not thread-aware but degrade gracefully:

- The `+orbit/msg-thread` TAGMSG is a client-only tag and is invisible to clients that do not
  support message tags - no noise in chat.
- Thread channels are set `+s` and do not appear in `/list` output.
- An IRC client that wants to read or participate in a thread can `/join #dev.t-abc123` directly.
  They see the thread as a normal IRC channel with the original message as the topic and all
  replies as a flat message history.
- IRC clients cannot initiate threads (they have no UI for the TAGMSG signal), but can reply
  freely within an existing thread channel once joined.

### Long-Term: Native Thread Support in the Ground Control Fork

In the MVP, thread lifecycle is entirely client-managed. Thread channels persist on the server
until Ergochat's configured channel expiry removes them. If multiple clients race to create the
same thread channel, the IRC server resolves this naturally - the channel already exists and the
second JOIN simply joins it.

In the Ground Control fork (see [Long-Term: The Ground Control Fork](#long-term-the-ground-control-fork)
below), thread channels become first-class server objects: the server manages creation and expiry,
enforces the naming convention, suppresses thread channels from `LIST` natively without requiring
`+s` mode, and can serve accurate reply counts directly. The client-side tag-based discovery
mechanism remains the same - the fork extends the server without changing the protocol visible to
clients.

## Long-Term: The Ground Control Fork

Today, Ground Control is Ergochat - an excellent production IRC server that the MVP is built on. If
Orbit's adoption reaches the point where IRC's native limitations become genuine product
constraints, the planned path forward is to **fork Ergo into a purpose-built Ground Control IRCd**
that extends the server while maintaining 100% IRC client compatibility.

The compatibility guarantee is non-negotiable: any IRC client - WeeChat, irssi, Textual, ZNC,
any compliant third-party client - must continue to work against a Ground Control fork without
modification. The fork extends the protocol surface available to Orbit clients; it does not break
the protocol surface used by standard clients.

What the fork enables natively, beyond what the tag-based MVP approach provides:

| Feature | MVP (Ergochat + tags) | Ground Control fork |
|---|---|---|
| Message edits | Client reconstructs state from append-only log | Server maintains canonical current state; `CHATHISTORY` returns edited versions |
| Message deletion | Tombstone in client buffer | Server rewrites; IRC clients see clean backlog on reconnect |
| Reactions | Separate Reactor service | Native server-side aggregation; Reactor eliminated |
| Threads | Client-managed sub-channels + `+s` mode | First-class server objects with managed lifecycle and native reply counts |
| Federation | Blocked - Ergo has no server-to-server linking | Implemented in the fork on Orbit's timeline |
| Push notifications | Separate Beacon service (IRC client process) | Integrated into Ground Control; no separate service |

The fork is not a day-one decision. Ergochat is the right server for the MVP and well beyond. The
fork becomes the path when the project is successful enough to justify owning an IRCd. Until then,
everything in this document applies to stock Ergochat.

For the full rationale - including why a custom protocol is not the answer - see
[Design Philosophy](../../01-architecture/02-philosophy.md#the-long-term-path-ground-control-as-a-true-ircd).
