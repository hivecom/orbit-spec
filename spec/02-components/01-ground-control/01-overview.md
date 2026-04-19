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

IRC channels are flat - there is no hierarchy, no categories, no sub-channels. Orbit treats them
as-is. Channels are presented to the user as a list, with no client-side interpretation of naming
conventions. If a community wants to organize channels by prefix (e.g., `#gaming-strategy`,
`#gaming-lfg`), they are free to do so, but Orbit does not parse, enforce, or render any convention
as a hierarchy. This keeps the client honest to the protocol and avoids ambiguity with existing IRC
channels that contain dots, dashes, or other separators.

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
