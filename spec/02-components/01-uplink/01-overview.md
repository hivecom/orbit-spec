# Uplink

Uplink is the IRC backbone of an Orbit deployment. It provides persistent text chat, user
authentication, signaling transport for Satellite session coordination, and server-side message
history. Uplink is an abstraction: Orbit specifies a contract that any stock IRCv3 server can
fulfill, with [Ergo](https://ergo.chat/) as the reference implementation. Orbit runs Ergo
unmodified, with no Orbit-specific patches and no Orbit fork. Where stock IRCv3 has gaps, Orbit
handles them in the client and tag layer rather than forking the server, so every IRC client keeps
working and compatibility is preserved. See
[Design Philosophy - Where Orbit's Value Lives](../../01-architecture/02-philosophy.md#where-orbits-value-lives)
and [Component Classes](../../01-architecture/02-philosophy.md#component-classes) for the rationale
and the abstraction-vs-component taxonomy.

Uplink is the only required component in an Orbit deployment. All other components
([Satellite](../02-satellite.md), [Depot](../03-depot.md),
[Transponder](../04-transponder.md)) are optional. An Uplink instance alone is a
fully functional, backward-compatible IRC server that any IRCv3 client can connect to.

The [Orbit tag namespace](02-tags/01-namespace.md) is transported over Uplink's `message-tags`
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
| Built-in federation model    | Server-to-server protocol exists (deferred; not a planned track)    |
| Chathistory extension        | Server-side message history with standardized retrieval              |

IRCv3 gives us a text and signaling transport that works today, scales vertically to the community
sizes targeted in the MVP, and does not lock us into a proprietary protocol.

## Required IRCv3 Extensions

The following extensions MUST be enabled on any Ergochat instance serving as Uplink.

| Extension                  | Purpose                                                                                                                         | Status in Ergo        |
|----------------------------|---------------------------------------------------------------------------------------------------------------------------------|-----------------------|
| `message-tags`             | Transport media signaling and rich metadata                                                                                     | Stable                |
| `batch`                    | Group related messages (history playback, netsplit recovery)                                                                    | Stable                |
| `chathistory`              | Server-side message history retrieval                                                                                           | Stable                |
| `server-time`              | Accurate timestamps on all messages                                                                                             | Stable                |
| `labeled-response`         | Correlate requests with responses for async client operations                                                                   | Stable                |
| `sasl`                     | Authentication (PLAIN and SCRAM over TLS)                                                                                       | Stable                |
| `message-ids`              | Unique message identifiers for retractions and replies                                                                          | Stable                |
| `WebSocket`                | Native WebSocket transport on Ergochat's listener                                                                               | Stable                |
| `account-tag`              | Server-asserted account identity; used by clients for authorship verification and identity display - cannot be forged by clients | Stable                |
| `echo-message`             | Client receives its own messages back from the server with server-applied tags (`server-time`, `msgid`)                         | Stable                |
| `away-notify`              | Clients receive notification when monitored users set or clear AWAY status                                                      | Stable                |
| `extended-monitor`         | Enhanced user presence monitoring beyond basic MONITOR                                                                          | Stable                |
| `draft/pre-away`           | Clients signal they are about to go away, enabling smoother presence transitions                                                | Stable                |
| `draft/read-marker`        | Server-side read position tracking, synced across all client sessions                                                           | Stable                |
| `setname`                  | Users can set a display name separate from their nickname (superseded by `display-name` metadata key when `draft/metadata-2` is available, but kept for compatibility) | Stable |
| `draft/message-redaction`  | Server-enforced message retractions via the IRC-standard `REDACT` command                                                       | Shipped in stable Ergo; requires `history.retention.allow-individual-delete: true` (Ergo does not advertise the cap otherwise) |
| `draft/metadata-2`         | Native user/channel key-value metadata store (avatars, display names, presence status)                                          | Stable in Ergo (2.17.0+); requires the `accounts.metadata` config block |

`account-tag` and `echo-message` are critical to Orbit's client-side trust model. `account-tag` is
the authoritative source of sender identity throughout the Orbit tag system; `echo-message` ensures
that the client's own messages carry server-assigned identifiers required for retraction and reply
references.

For the full set of `+orbit/*` client-only tags transported over `message-tags`, see
[Orbit Tag Namespace](02-tags/01-namespace.md).

## Ergochat Configuration

Key configuration points for an Orbit-compatible Ergochat instance:

- **WebSocket listener**: Ergochat exposes a WebSocket endpoint on a dedicated port. In production,
  TLS for this endpoint is terminated by a reverse proxy (Caddy in the reference deployment) which
  proxies to Ergochat's internal WebSocket port. Ergochat CAN terminate TLS directly for raw IRC
  connections (port 6697), but the reverse proxy is required for any deployment that serves web
  clients or needs to host `/.well-known/orbit/services.json` for service discovery. See
  [Domain Discovery](../../05-infrastructure/01-domain-discovery.md) for the well-known file format.
- **History storage**: Enabled with operator-configured per-channel retention. There is no mandated
  default - operators choose an appropriate retention window for their community (e.g., 7 days or
  10,000 messages). DM query history uses the same operator-configured retention as channels. See
  [Direct Messages](03-dms.md) for the full DM storage model and Ergo configuration snippet.
- **SASL**: Required for registered users. SASL PLAIN and SCRAM-SHA-256 over TLS. By default,
  Ergochat's built-in account database handles credential verification. When a Transponder is
  deployed, Ergochat can delegate SASL verification to the OIDC identity provider via the `auth-script`
  configuration option and a thin auth-script bridge. This enables any OIDC-compliant provider
  (Keycloak, Authentik, etc.) to serve as the identity backend without modifying the IRC server.
  When an identity provider is configured, OIDC is the authoritative identity layer. NickServ does
  **not** have to be disabled: by default it coexists as a recovery and legacy-client layer
  (email-based `SENDPASS`/`RESETPASS` and SASL `IDENTIFY`) while OIDC owns identity. The bridge MUST
  enable account autocreation so the first OIDC login establishes a persistent account and nickname
  reservation. Operators who want a strict single source of truth MAY instead disable NickServ
  registration (`accounts.registration.enabled = false`). Either way, Ergochat's nickname
  enforcement continues to work because it is tied to account login, not to NickServ specifically;
  once `auth-script` confirms an account name, nickname reservation and enforcement behave
  identically to the NickServ path. See [Transponder](../04-transponder.md) for the full coexistence
  vs. strict model.
- **Client-only tag allowlist**: Ergochat relays all `+`-prefixed tags by default per IRCv3 spec.
  No special configuration needed, but the server should enforce maximum tag size limits.
- **Connection limits**: Per-IP connection limits configured to prevent abuse. Browser-based web
  clients connect directly, so limits should accommodate multiple simultaneous guest connections
  from the same IP (e.g., shared NAT or multiple browser tabs).
- **Nickname reservation**: The `guest-` prefix MUST be reserved. In the MVP (NickServ active),
  configure NickServ to reject registration of any nickname starting with `guest-`. When an OIDC
  identity provider is configured, the provider's account management (and/or disabled NickServ
  registration in strict mode) ensures no user can register a `guest-` prefixed username. Either
  way, this prevents collision between registered users and anonymous widget guests.

For DNS SRV record configuration that makes this instance discoverable by Orbit clients, see
[DNS & Service Discovery](../../05-infrastructure/01-domain-discovery.md).

## Channels

IRC channels are flat - there is no hierarchy, no categories, no sub-channels at the protocol
level. The server has no knowledge of any naming convention. Orbit clients use **slash notation** as
a client-side rendering convention: if a channel name contains slashes (e.g., `#dev/frontend`,
`#dev/backend`), the Orbit client interprets the slashes as path separators and renders those channels
in a collapsible tree in the sidebar. Channels without slashes appear at the top level. This is
entirely a client rendering decision - the IRC server sees flat channel names as always.

For the full slash-notation rendering rules and edge cases, see
[Desktop Client - Channel Organization](../../04-clients/01-desktop.md#channel-organization).

For real-time session chat (quick callouts during voice, links shared during a screen share),
[Satellite](../02-satellite.md) provides its own ephemeral chat via LiveKit data channels.
This is intentionally separate from IRC: ephemeral messages are not persisted, not searchable, and
not visible to IRC clients. Persistent, historical chat belongs in Uplink. Throwaway, in-session
chat belongs in Satellite.

### Channel Metadata Keys (`draft/metadata-2`)

Orbit clients use `draft/metadata-2` to store structured display metadata directly on channels. These keys are set by channel operators and fetched on channel join as part of the metadata burst. For unjoined parent channels (e.g. for slash-notation tree verification), clients request metadata explicitly:

```
METADATA #dev GET subchannels
```

The channel metadata keys Orbit defines:

| Key | Content | Set by |
|---|---|---|
| `display-name` | Friendly human-readable name separate from the IRC channel name | Operator |
| `avatar` | URL of the channel's avatar image (Depot URL or external) | Operator |
| `color` | Accent color for the channel as a hex string (without `#`) | Operator |
| `homepage` | URL for the channel's associated project, docs, or community page | Operator |
| `markdown` | Rich channel description, stored as Markdown | Operator |
| `subchannels` | Comma-separated authorized direct child segment names for slash-notation trees | Operator |

These keys are channel-scoped and distinct from the user metadata keys (`avatar`, `display-name`, `orbit.status`) documented in [Presence](04-presence.md). Channels and users share the same `draft/metadata-2` extension but have separate key namespaces in practice - a `METADATA GET` on `#channel` returns channel keys; a `METADATA GET` on a nick returns user keys.

The `subchannels` key is the mechanism for subchannel authorization in slash-notation trees. For the full authorization model and client enforcement rules, see [Desktop Client - Subchannel Authorization](../../04-clients/01-desktop.md#subchannel-authorization).

### Channel Renaming

IRC has a work-in-progress extension, `draft/channel-rename` (the `RENAME` command), that renames a
channel in place while preserving its membership, modes, topic, and ban/invite/exception lists,
instead of forcing everyone to leave one channel and recreate another. Stock Ergo implements it, so
Orbit can support it - but the IRCv3 working group still marks the extension experimental and advises
against relying on it in production. Orbit therefore treats channel renaming as an optional,
capability-gated convenience, not a guaranteed feature, and never depends on it.

The more important point is that native renaming is **not** the tool for the common case. Several
caveats constrain it:

- **Registered channels cannot be renamed.** Ergo refuses to rename a channel that carries persistent
  history, and in practice every channel meant to last is registered. Native `RENAME` is effectively
  limited to ephemeral, unregistered channels - the throwaway kind that nobody usually needs to
  rename. An established community channel cannot be renamed this way without first dropping its
  registration, which sacrifices exactly the persistence that made the channel worth keeping.
- **Slash-notation hierarchy does not follow the rename.** Because the folder tree is a pure
  client-side rendering convention (see [Channels](#channels) above), a rename that changes a path
  segment detaches the channel from its rendered parent and orphans any parent/child grouping the
  client tracks. The server moves a flat name; it has no concept of the hierarchy to migrate, so the
  client must reconcile the tree itself.
- **Clients without the capability see a fallback.** A client that has not negotiated
  `draft/channel-rename` is informed of the change through a synthesized leave-then-rejoin sequence.
  A client relying on that fallback tears down and rebuilds the channel locally and loses in-place
  continuity (read position, scroll, unbroken history under one name); only a client that negotiates
  the capability gets the clean single-event rename. Case-only renames skip the fallback by spec.
- **Operator privilege is required**, and the server rejects unprivileged attempts.

For the routine "this channel should have a different name" intent, the right mechanism is the
`display-name` channel metadata key carried over `draft/metadata-2`, the same key clients already use
to show a friendly label in place of the raw `#slug`. Editing `display-name` changes the name every
Orbit client renders without touching the underlying channel name, its history, its registration, or
its position in the slash-notation tree. It works on registered channels, needs no capability
negotiation, and carries none of the caveats above. Orbit clients SHOULD steer ordinary rename
intent toward `display-name` and reserve native `RENAME` for the narrow ephemeral case where actually
changing the protocol-level channel name is the goal.

A client that does choose to expose native renaming should:

- Negotiate `draft/channel-rename` during the CAP exchange, so the server delivers the single rename
  event rather than the leave/rejoin fallback.
- Treat the server's rename notification as an in-place migration of local state: carry the existing
  channel's messages, member list, modes, topic, and list-mode entries onto the new name, and re-key
  everything else that is keyed by channel name - read markers, cached metadata, the active-channel
  pointer, and any persisted auto-join target - so nothing resets or double-counts.
- Resolve a channel's registration status before offering the action, and present native renaming
  only where the server can actually honour it. Everywhere else, present `display-name` editing
  instead and explain why, rather than letting the user attempt a rename the server will reject.
- Surface the standardized failure replies (name already in use, cannot rename, insufficient
  privilege) to the user, since the operation round-trips through the server and can fail after the
  request is sent.

## Message Retractions and Replies

Each message has a unique ID assigned by Ergochat via the `message-ids` extension. The
`echo-message` extension ensures the sending client receives this server-assigned ID for its own
messages.

### Retractions

Message retractions in the MVP use the IRC-standard `REDACT` command, provided by the
`draft/message-redaction` extension:

```
REDACT #channel <msgid> [reason]
```

This is a server-side operation, not a client tag. The server validates that the sender has
permission to retract the message - either they authored it (matched via `account-tag`) or they
are a channel operator. Clients cannot forge a retraction for another user's message.

**What different clients see:**

- **Orbit clients** supporting `draft/message-redaction` see the message removed from the channel
  and render a tombstone inline: *"This message was retracted."*
- **IRC clients not supporting `draft/message-redaction`** receive a server NOTICE fallback:
  ```
  *** alice retracted a message ***
  ```
- **On reconnect**, the IRCv3 spec lets a server either drop redacted messages from `chathistory`
  entirely or replay them followed by a `REDACT` event. Orbit clients therefore handle `REDACT`
  inside `chathistory` batches the same way they handle a live one (rendering the tombstone) rather
  than assuming the message is simply absent. Either server behaviour converges on the same
  rendered result.

> **Availability**: `draft/message-redaction` is shipped in stable Ergo, so retractions are
> available on the adopted server. No fallback tag-based retraction system is used - the
> `+orbit/msg-retract` tag does not exist.

> **Operator requirement**: Ergo gates `REDACT` behind `history.retention.allow-individual-delete`.
> When this is `false` (Ergo's recommended default), Ergo does **not** advertise the
> `draft/message-redaction` capability ([ergochat/ergo#2215](https://github.com/ergochat/ergo/issues/2215)),
> so Orbit clients see no deletion affordance and `REDACT` commands are rejected. Operators who want
> retractions must set `allow-individual-delete: true`. That single flag enables the full permission
> model described above - authors deleting their own messages **and** channel operators deleting
> others' - and lets redactions persist and replay through `chathistory`. It is independent of
> `enable-account-indexing`, which governs only bulk per-account history purges and is not required
> for per-message redaction. Toggling it on requires an Ergo restart (not just a rehash) before the
> capability is re-advertised.

**Message editing** (`+orbit/msg-amend`) is not standardized in IRC yet. There is active draft work
on in-place editing across the IRC ecosystem; Orbit follows it and handles editing at the client and
tag layer in the meantime. If Ergo or IRCv3 adopts a standard, the Orbit client adopts it too;
`+orbit/msg-amend` is an interim tag, not a long-term design. Deletion already exists via
`draft/message-redaction`.

### Replies

To reply to a message, the client sends a `PRIVMSG` with the standard IRCv3 `+draft/reply=<msgid>`
client tag referencing the original `msgid`, with the reply content as the message body. This is the
same reply mechanism other IRC clients use, so replies interoperate with any client that understands
`+draft/reply`. The Orbit client renders the reply with an inline excerpt of the original message (if
available in the local buffer) and a clickable link to navigate to the original. If the original
message is not in the buffer, the reply is displayed without the excerpt. Clients without reply
support see a normal `PRIVMSG` - the reply context is invisible to them but the reply text is fully
readable.

### Interaction with Chat History

When a client fetches messages via `chathistory`:

- **Retracted messages**: the server either excludes them from the response or replays them
  followed by a `REDACT` event (the IRCv3 spec permits both). Orbit clients handle a `REDACT` event
  in a `chathistory` batch the same way they handle a live one - rendering the tombstone - so both
  server behaviours converge on the same result. Clients never reconstruct retracted *content*; the
  original message body is not recoverable from history.
- **A reply** appears in history as a `PRIVMSG` carrying a `+draft/reply` tag referencing the
  target `msgid`. If the target message is within the loaded history, the client renders the reply
  with an inline excerpt. If the target is outside the loaded window, the reply is displayed without
  the excerpt - the reply text is always self-contained and readable on its own.

For client-side enforcement rules governing the tag-based reply system, see
[Tag Integrity and Trust Model](02-tags/02-trust-model.md).

For file uploads posted as messages in channels, see [Depot](../03-depot.md).

## Threads

Orbit supports threaded conversations on top of standard IRC channels. Threads are implemented
entirely at the client level using standard IRC primitives - no server patches and no bots are
required for the MVP. Any Orbit client can create and participate in threads. Standard IRC clients
can also read and reply to threads by joining the thread channel directly.

### Thread Channels

A thread is a normal IRC channel whose name is derived from the parent channel and the `msgid` of
the message being threaded:

```
<parent-channel>/t-<msgid>
```

For example, a thread on message `abc123` in `#dev` lives at `#dev/t-abc123`. Because this follows
the slash-notation convention, Orbit clients render it as a child of `dev` in the sidebar hierarchy
- but thread channels are **always hidden from the main channel list** in the Orbit UI. They are
surfaced only through the thread panel attached to the parent message.

When a user posts the first reply to a message in thread mode, the Orbit client:

1. JOINs `<parent>.t-<msgid>` - IRC creates the channel automatically on the first JOIN.
2. Sets `+s` (secret) mode, so the thread channel is excluded from `/list` output for classic
   IRC clients.
3. Sets the TOPIC of the thread channel to the original message content, attributed to its author:
   ```
   <original message text> - @alice, 2025-01-15 20:30 UTC
   ```
4. Sends the reply as a `PRIVMSG` in the thread channel.
5. Sends a `PRIVMSG` to the **parent channel** with a human-readable notice pointing IRC clients
   to the thread channel:
   ```
   ↳ Thread started by alice in #dev/t-abc123
   ```
   This message is visible to all IRC clients and allows them to follow the thread by joining the
   named channel directly. Orbit clients suppress this message from the chat view and render the
   thread indicator instead.
6. Sends a `+orbit/msg-thread` `TAGMSG` to the **parent channel**, signaling that a thread now
   exists on the target message.

For each subsequent reply, the client JOINs the thread channel (if not already joined) and sends
the `PRIVMSG` there. That's it - being in the channel is the subscription. No additional TAGMSG
is sent to the parent channel on subsequent replies.

### The `+orbit/msg-thread` Signal Tag

The `+orbit/msg-thread` tag is sent as a `TAGMSG` to the parent channel **only when the thread is
first created** - not on every reply. This is a creation signal, not an activity signal.

Its purpose is **discovery**: Orbit clients loading history via `chathistory` can find existing
threads by scanning for `+orbit/msg-thread` TAGMSGs. The `chathistory` stream for the parent
channel becomes the index of all threads in that channel.

Tag payload (base64 JSON):

```json
{
  "parent_msgid": "abc123",
  "thread_channel": "#dev/t-abc123"
}
```

For the tag's position in the full namespace and trust model rules, see
[Tag Namespace](02-tags/01-namespace.md) and [Tag Trust Model](02-tags/02-trust-model.md).

### Orbit Client Rendering

- **Thread indicator**: Messages that have received a `+orbit/msg-thread` signal display a thread
  indicator inline below the message content.
- **Thread panel**: Clicking the indicator opens a thread panel. The client JOINs the thread
  channel (if not already joined) and fetches history via `chathistory`. The original message is
  shown at the top, followed by all thread replies in chronological order.
- **Thread composition**: Users compose thread replies in the thread panel. The client handles the
  JOIN, `+s` mode, TOPIC, TAGMSG (on creation), and PRIVMSG operations transparently - the user
  sees a reply input, not IRC primitives.
- **Live updates**: Orbit clients that have joined a thread channel receive new replies via normal
  IRC message delivery. Being in the channel is the subscription.
- **Reply counts**: The authoritative reply count comes from `chathistory` on the thread channel,
  fetched when the user opens the thread panel. No client-side approximation from TAGMSGs.
- **Thread channels are hidden** from the Orbit channel list and server browser. They are never
  presented as joinable channels in the Orbit UI.

### IRC Client Experience

Standard IRC clients are not thread-aware but degrade gracefully:

- The `+orbit/msg-thread` TAGMSG is a client-only tag and is invisible to clients that do not
  support message tags - no noise in chat.
- The PRIVMSG fallback (`↳ Thread started by alice in #dev/t-abc123`) is visible to all IRC
  clients, pointing them to the thread channel by name.
- Thread channels are set `+s` and do not appear in `/list` output.
- An IRC client that wants to read or participate in a thread can `/join #dev/t-abc123` directly.
  They see the thread as a normal IRC channel with the original message as the topic and all
  replies as a flat message history.
- IRC clients cannot initiate threads (they have no UI for the TAGMSG signal), but can reply
  freely within an existing thread channel once joined.

### Known Limitations

Thread channels are real IRC channels - they consume server-side state (channel metadata, membership, history). In a busy community with many active threads, this means channel proliferation on the server. Ergochat's configured channel expiry mitigates this: inactive thread channels are cleaned up automatically after the configured TTL.

Other trade-offs:

- **No server-enforced thread-parent relationship.** The server sees thread channels as ordinary channels. The relationship between a thread and its parent message exists only in the client's rendering logic and the `+orbit/msg-thread` tag.
- **Client-managed lifecycle.** Thread creation (JOIN, mode setting, TOPIC, TAGMSG) is orchestrated by the Orbit client. If a client crashes mid-creation, the thread channel may exist in a partially-initialized state. This is recoverable - subsequent clients joining the same thread channel will set the correct state.
- **IRC clients see flat channels.** An IRC user who discovers a thread channel sees it as an ordinary channel with a message as its topic. This is functional but lacks the visual context that Orbit clients provide.

This is an honest trade-off. The sub-channel approach works on any existing IRC server without modification. Orbit clients abstract the mechanics entirely - users see a thread panel, not IRC primitives. The experience is comparable to threading in Slack or Teams, which also took years to mature. Communities that operated without threads (including Discord, which still does not have threaded conversations - its "Threads" are closer to ephemeral sub-channels, and its "Forum" channels are a distinct forum-like structure) demonstrate that threading is a nice-to-have, not a blocker.

## Protocol Posture

Uplink is an abstraction over a stock IRCv3 server, and there is no Orbit fork of Ergo. Orbit
conforms to IRCv3 and supports whatever stock Ergo implements.

Stock Ergo (current stable v2.18.0) already provides nearly everything Orbit needs natively:

- **Push notifications** via `draft/webpush` (v2.15.0).
- **OIDC/JWT authentication** - auth-script bridge (`SASL PLAIN`, any provider/algorithm, JWKS-based)
  or native `accounts.jwt-auth` over `IRCV3BEARER` (RS256/EdDSA/HMAC, static key; no ECDSA/ES256,
  no JWKS fetch). `accounts.oauth2` (`OAUTHBEARER`) covers token-introspection providers. (v2.14.0+)
- **User and channel metadata** (avatars, display names, presence status) via stable
  `draft/metadata-2` (v2.17.0).
- **Message retraction and deletion** via `draft/message-redaction` (the `REDACT` command, shipped).
- **HTTP API** (v2.16.0+) and **Postgres/SQLite history backends** (v2.18.0).

For the rest, Orbit follows the IRC ecosystem and handles what is not yet standardized at the client
layer:

- **Message editing** is not standardized in IRC yet. There is active draft work on it; Orbit
  handles editing at the client and tag layer and will adopt a standard if Ergo or IRCv3 ships one.
- **Reactions** work today, handled client-side via message tags. The only concession is that
  reactions cannot be shown on messages surfaced purely from a search result.

**Federation** needs server-to-server linking that stock Ergo does not provide. It is not a goal for
now; Ergo may add it or Orbit may help upstream, but nothing depends on it.

The compatibility guarantee is non-negotiable: any IRC client - WeeChat, irssi, Textual, ZNC, any
compliant third-party client - works against Uplink without modification, precisely because Uplink
is a stock IRCv3 server.

For the full rationale, see
[Design Philosophy - Where Orbit's Value Lives](../../01-architecture/02-philosophy.md#where-orbits-value-lives).
