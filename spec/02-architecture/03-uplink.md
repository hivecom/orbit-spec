# Uplink

Uplink is the IRC backbone of an Orbit deployment. It provides persistent
text chat, user authentication, signaling transport for Satellite session
coordination, and server-side message history. Uplink is an abstraction:
Orbit specifies a contract that any stock IRCv3 server can fulfill, with
[Ergo](https://ergo.chat/) as the reference implementation, run unmodified.
Where stock IRCv3 has gaps, Orbit handles them in the client and tag layer
rather than forking the server - see
[Protocol Posture](02-protocol-posture.md).

Uplink is the only required component in an Orbit deployment. All other
components ([Satellite](05-satellite.md), [Depot](06-depot.md),
[Transponder](07-transponder.md)) are optional. An Uplink instance alone is a
fully functional, backward-compatible IRC server that any IRCv3 client can
connect to.

IRCv3 earns the role: it's a battle-tested protocol with an ultra-low
resource footprint (Ergo serves thousands of connections on minimal
hardware), native WebSocket support so browser clients connect without a
separate gateway, message tags for rich metadata transport, server-side
history via `chathistory`, and full backward compatibility with every
existing IRC client and bot.

The [Orbit tag namespace](04-tags-and-trust.md) is transported over Uplink's
`message-tags` layer as `+`-prefixed client-only tags. Ergo configuration and
IRC command mechanics live in
[Implementation - Uplink](../03-implementation/01-uplink.md).

## Required IRCv3 Extensions

The following extensions MUST be enabled on any Ergo instance serving as
Uplink. This table is the contract any Uplink must satisfy.

| Extension                  | Purpose                                                                                                                         | Status in Ergo        |
|----------------------------|---------------------------------------------------------------------------------------------------------------------------------|-----------------------|
| `message-tags`             | Transport media signaling and rich metadata                                                                                     | Stable                |
| `batch`                    | Group related messages (history playback, netsplit recovery)                                                                    | Stable                |
| `chathistory`              | Server-side message history retrieval                                                                                           | Stable                |
| `server-time`              | Accurate timestamps on all messages                                                                                             | Stable                |
| `labeled-response`         | Correlate requests with responses for async client operations                                                                   | Stable                |
| `sasl`                     | Authentication (PLAIN and SCRAM over TLS)                                                                                       | Stable                |
| `message-ids`              | Unique message identifiers for retractions and replies                                                                          | Stable                |
| `WebSocket`                | Native WebSocket transport on Ergo's listener                                                                                   | Stable                |
| `account-tag`              | Server-asserted account identity; used by clients for authorship verification and identity display - cannot be forged by clients | Stable                |
| `echo-message`             | Client receives its own messages back from the server with server-applied tags (`server-time`, `msgid`)                         | Stable                |
| `away-notify`              | Clients receive notification when monitored users set or clear AWAY status                                                      | Stable                |
| `extended-monitor`         | Enhanced user presence monitoring beyond basic MONITOR                                                                          | Stable                |
| `draft/pre-away`           | Clients signal they are about to go away, enabling smoother presence transitions                                                | Stable                |
| `draft/read-marker`        | Server-side read position tracking, synced across all client sessions                                                           | Stable                |
| `setname`                  | Users can set a display name separate from their nickname (superseded by the `display-name` metadata key when `draft/metadata-2` is available, but kept for compatibility) | Stable |
| `draft/message-redaction`  | Server-enforced message retractions via the IRC-standard `REDACT` command                                                       | Shipped in stable Ergo; gated behind an operator retention setting - Ergo does not advertise the capability otherwise (see [Implementation - Uplink](../03-implementation/01-uplink.md)) |
| `draft/metadata-2`         | Native user/channel key-value metadata store (avatars, display names, presence status)                                          | Stable in Ergo (2.17.0+); enabled per deployment by the operator |

`account-tag` and `echo-message` are critical to Orbit's client-side trust
model. `account-tag` is the authoritative source of sender identity
throughout the Orbit tag system; `echo-message` ensures the client's own
messages carry the server-assigned identifiers required for retraction and
reply references. See [Tags and Trust](04-tags-and-trust.md).

## Channels

IRC channels are flat: no hierarchy, no categories, no sub-channels at the
protocol level. Orbit clients use slash notation as a client-side rendering
convention - a channel name like `#dev/frontend` renders as a child of `dev`
in a collapsible tree. This is entirely a client rendering decision; the
server sees flat channel names as always. The rendering semantics and
authorization model live in [Clients](11-clients.md#channel-organization).

For real-time session chat (quick callouts during voice, links shared during
a screen share), [Satellite](05-satellite.md) provides its own ephemeral chat
via LiveKit data channels. Ephemeral messages aren't persisted, aren't
searchable, and aren't visible to IRC clients. Persistent, historical chat
belongs in Uplink; throwaway, in-session chat belongs in Satellite.

### Channel Metadata

Orbit clients use `draft/metadata-2` to store structured display metadata
directly on channels: a friendly display name, an avatar, an accent color, a
homepage URL, a rich description, and the authorized-subchannel list that
backs slash-notation trees. These keys are set by channel operators and
arrive as part of the metadata burst on join; clients can request metadata
explicitly for channels they haven't joined.

Channel keys are scoped separately from user metadata keys - channels and
users share the same extension but have distinct key namespaces in practice.
The canonical key table and command mechanics live in
[Implementation - Uplink](../03-implementation/01-uplink.md); the subchannel
authorization model built on these keys is specified in
[Clients](11-clients.md#subchannel-authorization).

### Channel Renaming

IRC has a work-in-progress extension, `draft/channel-rename`, that renames a
channel in place while preserving membership, modes, topic, and list modes.
Stock Ergo implements it, but the IRCv3 working group still marks it
experimental and advises against relying on it in production. Orbit treats
native renaming as an optional, capability-gated convenience and never
depends on it.

Native renaming is also not the tool for the common case:

- **Registered channels can't be renamed.** Ergo refuses to rename a channel
  that carries persistent history, and in practice every channel meant to
  last is registered. Native renaming is effectively limited to ephemeral,
  unregistered channels.
- **Slash-notation hierarchy doesn't follow the rename.** The folder tree is
  a client-side rendering convention, so a rename that changes a path segment
  detaches the channel from its rendered parent. The server moves a flat
  name; the client must reconcile the tree itself.
- **Clients without the capability see a fallback.** A client that hasn't
  negotiated the extension is informed of the change through a synthesized
  leave-then-rejoin sequence and loses in-place continuity (read position,
  scroll, unbroken history under one name).
- **Operator privilege is required**, and the server rejects unprivileged
  attempts.

For the routine "this channel should have a different name" intent, the right
mechanism is the `display-name` channel metadata key. Editing it changes the
name every Orbit client renders without touching the underlying channel name,
its history, its registration, or its position in the tree. It works on
registered channels and needs no capability negotiation. Orbit clients SHOULD
steer ordinary rename intent toward `display-name` and reserve native
renaming for the narrow ephemeral case where changing the protocol-level name
is the goal. Client-side handling rules for the rename event live in
[Implementation - Clients](../03-implementation/08-clients.md).

## Message Retractions and Replies

Each message has a unique ID assigned by the server via `message-ids`;
`echo-message` ensures the sending client receives the server-assigned ID for
its own messages.

### Retractions

Retractions use the IRC-standard `REDACT` command, provided by
`draft/message-redaction`. This is a server-side operation, not a client tag:
the server validates that the sender has permission to retract the message -
either they authored it (matched via `account-tag`) or they're a channel
operator. Clients can't forge a retraction for another user's message.

What different clients see:

- Orbit clients see the message removed and render a tombstone inline:
  *"This message was retracted."*
- IRC clients without the capability receive a server NOTICE fallback.
- In history replay, the IRCv3 spec lets a server either drop redacted
  messages entirely or replay them followed by a redaction event. Orbit
  clients handle a redaction inside a history batch the same way as a live
  one, so both server behaviors converge on the same rendered result.
  Retracted content is never reconstructable from history.

Ergo gates `REDACT` behind an operator retention setting and doesn't
advertise the capability when it's off, so clients see no deletion affordance
on deployments that haven't enabled it. That one setting enables the full
permission model - authors deleting their own messages and channel operators
deleting others'. Configuration specifics live in
[Implementation - Uplink](../03-implementation/01-uplink.md).

There is no fallback tag-based retraction system - retraction is
server-enforced or not available.

Message editing isn't standardized in IRC yet; Orbit defers it until a
standard lands. See
[Protocol Posture](02-protocol-posture.md#current-gaps-and-how-each-resolves).

### Replies

Replies use the standard IRCv3 reply client tag referencing the original
message ID, so they interoperate with any client that understands the
extension - there is no Orbit-specific reply tag. The Orbit client renders
the reply with an inline excerpt of the original message when it's available
locally and a link to navigate to it; when it isn't, the reply is displayed
without the excerpt, and the reply text is always self-contained. Clients
without reply support see a normal message.

Client-side enforcement rules for the reply tag live in
[Tags and Trust](04-tags-and-trust.md).

## Threads

Orbit supports threaded conversations on top of standard IRC channels,
implemented entirely at the client level using standard IRC primitives - no
server patches, no bots. Standard IRC clients can read and reply to threads
by joining the thread channel directly.

### Design

A thread is a normal IRC channel whose name is derived from the parent
channel and the message ID being threaded:

```
<parent-channel>/t-<msgid>
```

A thread on message `abc123` in `#dev` lives at `#dev/t-abc123`. Because this
follows the slash-notation convention, the thread is a child of its parent in
the client's tree logic, but thread channels are always hidden from the main
channel list in the Orbit UI - they surface only through the thread panel
attached to the parent message.

On first reply, the Orbit client creates the thread channel (IRC creates
channels on first join), marks it secret so it's excluded from channel
listings, sets its topic to the original message attributed to its author,
posts a human-readable notice to the parent channel pointing IRC clients at
the thread channel by name, and sends a thread-creation signal tag to the
parent channel. Subsequent replies are ordinary messages in the thread
channel - being in the channel is the subscription. The exact creation
sequence lives in [Implementation - Uplink](../03-implementation/01-uplink.md);
the signal tag payload lives in
[Implementation - Tags](../03-implementation/02-tags.md).

The signal tag exists for discovery: clients loading history can find
existing threads by scanning the parent channel's history for creation
signals. The authoritative reply list is always the thread channel's own
history. Thread rendering behavior is specified in
[Product - Experience](../01-product/02-experience.md), with client mechanics
in [Implementation - Clients](../03-implementation/08-clients.md).

### IRC Client Experience

Standard IRC clients aren't thread-aware but degrade gracefully:

- The signal tag is client-only and invisible to clients without tag support.
- The plain-message notice in the parent channel points them to the thread
  channel by name.
- Thread channels don't appear in channel listings, but any IRC client can
  join one directly and sees a normal channel with the original message as
  the topic and the replies as flat history.
- IRC clients can't initiate threads (they have no UI for the signal), but
  can reply freely within an existing thread channel.

### Known Limitations

Thread channels are real IRC channels: they consume server-side state, and a
busy community with many active threads means channel proliferation on the
server. Ergo's configured channel expiry mitigates this by cleaning up
inactive thread channels automatically.

Other trade-offs:

- **No server-enforced thread-parent relationship.** The server sees thread
  channels as ordinary channels; the relationship exists only in client
  rendering logic and the signal tag.
- **Client-managed lifecycle.** Thread creation is orchestrated by the
  client. If a client crashes mid-creation, the thread channel may exist
  partially initialized. This is recoverable - subsequent clients joining the
  same thread channel set the correct state.
- **IRC clients see flat channels.** Functional, but without the visual
  context Orbit clients provide.

The sub-channel approach works on any existing IRC server without
modification, and Orbit clients abstract the mechanics entirely - users see a
thread panel, not IRC primitives.
