# Orbit Tag Namespace

The `+orbit/*` tags are IRCv3 client-only tags transported over `message-tags`. The server relays
them without interpretation or validation; enforcement is the Orbit client's job. The trust
boundary, enforcement rules, and versioning policy live in
[Tags and Trust](../02-architecture/04-tags-and-trust.md). This page is the canonical home for
every tag payload.

Message retractions are **not** handled by a client tag. Retractions use the IRC-standard
`REDACT` command provided by the `draft/message-redaction` extension - a server-enforced
operation. See [Uplink](01-uplink.md#message-retractions-redact).

## The `+orbit/*` Tag Table

| Tag                  | Payload                                                                                           | Direction         |
|----------------------|---------------------------------------------------------------------------------------------------|-------------------|
| `+orbit/sat-invite`  | Node URL + room ID + metadata (base64 JSON)                                                       | Client → Channel  |
| `+orbit/p2p-offer`   | P2P handshake: intent + ICE credentials + DTLS fingerprint + candidate (base64 JSON)              | Client → Client   |
| `+orbit/p2p-answer`  | P2P handshake response: ICE credentials + DTLS fingerprint + candidate (base64 JSON)              | Client → Client   |
| `+orbit/msg-thread`  | `parent_msgid` + thread channel name (base64 JSON)                                                | Client → Channel  |
| `+orbit/file`        | File metadata: name, size, type (base64 JSON)                                                     | Client → Channel  |
| `+orbit/msg-amend`  | Edited message body + original `msgid` (base64 JSON)                                             | Client → Channel  |
| `+orbit/e2e`         | Sender key ID / fingerprint (base64)                                                              | Client → Client   |

> **Replies are not an Orbit tag.** Message replies use the standard IRCv3 `+draft/reply=<msgid>`
> client tag, the same mechanism other IRC clients use. There is no `+orbit/` reply tag, so replies
> interoperate with any client that understands `+draft/reply`.

## Payloads

### `+orbit/sat-invite`

Posted to a channel to announce a Satellite session. Base64-encoded JSON:

```json
{
  "node": "https://sat1.example.com",
  "room": "gaming-strategy-a7f3e2",
  "initiator": "alice",
  "started": "2025-01-15T20:30:00Z",
  "protected": false
}
```

When `"protected": true`, the session is password-protected and the client prompts before joining.
The password goes to the Satellite's token service in the `/session/join` request, never over IRC.
The session flows are in [Satellite](03-satellite.md).

### `+orbit/p2p-offer` and `+orbit/p2p-answer`

Sent as a `TAGMSG` to a nickname to open a direct WebRTC connection. The offer payload
(base64 JSON):

```json
{
  "intent": "call",
  "ice_ufrag": "abcd",
  "ice_pwd": "longRandomString",
  "dtls_fingerprint": "sha-256 AA:BB:CC:...",
  "dtls_role": "actpass",
  "candidate": "candidate:1 1 udp 2122260223 203.0.113.5 54321 typ host"
}
```

The `+orbit/p2p-answer` carries the same fields - the recipient's own ICE credentials, DTLS
fingerprint, and a candidate. Two IRC messages total, ~300-400 bytes each; all further
negotiation happens over the WebRTC data channel. The `intent` values (`call`, `video`, `chat`,
`file`) and the full handshake flow are in [Satellite](03-satellite.md#p2p-handshake).

### `+orbit/msg-thread`

Sent as a `TAGMSG` to the parent channel **only when a thread is first created**. It is NOT sent
on subsequent replies - subsequent replies are normal PRIVMSGs in the thread channel; being in
the channel is the subscription.

Tag payload (base64 JSON):

```json
{
  "parent_msgid": "abc123",
  "thread_channel": "#dev/t-abc123"
}
```

Its purpose is **discovery**: Orbit clients loading history via `chathistory` can find existing
threads by scanning for `+orbit/msg-thread` TAGMSGs. The `chathistory` stream for the parent
channel becomes the index of all threads in that channel. The payload identifies the thread
channel by name so receiving clients can discover and join it; the authoritative reply list is
always the `chathistory` of the thread channel itself.

On thread creation, the client also sends a plain `PRIVMSG` to the parent channel alongside this
TAGMSG:

```
↳ Thread started by alice in #dev/t-abc123
```

This PRIVMSG is visible to all IRC clients and allows them to follow the thread by joining the
named channel directly. Orbit clients suppress it from the chat view and render the thread
indicator instead. The full creation sequence is in
[Uplink](01-uplink.md#thread-creation-sequence).

### `+orbit/file`

Attached to the `PRIVMSG` that carries an uploaded file's URL. Base64-encoded JSON:

```json
{
  "name": "screenshot.png",
  "size": 245760,
  "type": "image/png"
}
```

| Field  | Content                                      |
|--------|----------------------------------------------|
| `name` | Original filename (e.g., `screenshot.png`)   |
| `size` | File size in bytes                           |
| `type` | MIME type (e.g., `image/png`)                |

This metadata is client-asserted and informational. Orbit clients SHOULD verify file metadata
independently by checking HTTP response headers (`Content-Type`, `Content-Length`) on download.
See [Tags and Trust](../02-architecture/04-tags-and-trust.md) for enforcement rules and
[Depot](04-depot.md) for the upload flow.

### `+orbit/msg-amend`

An **interim mechanism** for in-place message editing, used until the IRC ecosystem standardizes
message editing (active draft work is ongoing). The payload is base64 JSON:
`{ "msgid": "<original-msgid>", "body": "<new-message-body>" }`. Receiving clients that
understand this tag SHOULD update the displayed message body and render an "edited" indicator.
Clients that do not understand the tag see nothing - the edit is invisible to them. If Ergo or
IRCv3 adopts a standard editing mechanism, Orbit clients will adopt it and retire this tag.

### `+orbit/e2e`

Reserved for end-to-end encrypted DMs; not yet active. When present on a `PRIVMSG`, it indicates
the message body is ciphertext encrypted with the Double Ratchet protocol. The tag value contains
the sender's key ID, which the recipient uses to select the correct decryption key. See
[E2E Encryption](../02-architecture/13-e2e.md) for the protocol design and
[Messaging](../02-architecture/10-messaging.md) for how clients derive a conversation's E2E
status from this tag.

## Encoding

Base64 encoding is used because IRC message tags have restricted character sets. IRC message tags
have a budget of 8191 bytes total per the IRCv3 message-tags spec, split evenly: at most 4094
bytes of client-sent tag data and 4094 bytes of server-added tag data per message. Orbit
client-published payloads share the client half with all other tags on the message. The message
body has a separate
limit (512 bytes classic, or server-configured). Payloads exceeding the tag budget are split across
a `batch`.

## Standard IRCv3 Client Tags

Orbit clients handle the following standard IRCv3 client-only tags in addition to the `+orbit/*` namespace. These are defined by the IRCv3 working group and interoperate with any compliant client.

| Tag | Spec | Purpose |
|---|---|---|
| `+draft/reply` / `+reply` | [IRCv3 message-replies](https://ircv3.net/specs/extensions/message-replies) | Reference the `msgid` of the message being replied to. Always sent as a client tag on the `PRIVMSG`. Orbit renders an inline excerpt of the original message and a navigation link. |
| `+draft/react` / `+react` | [IRCv3 react](https://ircv3.net/specs/client-tags/react) | Emoji reaction on a message. Always paired with `+reply` pointing at the parent `msgid`. Orbit renders the reaction inline on the original message. |
| `+draft/unreact` / `+unreact` | [IRCv3 react](https://ircv3.net/specs/client-tags/react) | Remove a previously sent emoji reaction. Always paired with `+reply`. Orbit removes the reaction from the inline display. |
| `+typing` | [IRCv3 typing](https://ircv3.net/specs/client-tags/typing) | Typing notification with value `active`, `paused`, or `done`. Sent as a `TAGMSG` to the channel or user. Orbit renders a typing indicator in the composer area. |

These tags are transported as `TAGMSG` messages (reactions, typing) or as client tags on `PRIVMSG` (replies). They are subject to the same `account-tag`-based identity display rules as `PRIVMSG` content - see [Tags and Trust](../02-architecture/04-tags-and-trust.md). They are also subject to Ergochat's standard fakelag flood protection identically to `PRIVMSG`.

> **Reactions and replies interop:** Because `+draft/reply` and the react tags are IRCv3 standards, other clients that implement these specs will interoperate with Orbit clients automatically. A third-party IRC client that understands `+draft/reply` will render replies; one that understands `+draft/react` will render reactions. Orbit does not define custom alternatives for these features.

## Cross-References

- [Tags and Trust](../02-architecture/04-tags-and-trust.md) - trust boundary, enforcement rules, versioning, the extension tag concept
- [Uplink](01-uplink.md) - tag relay configuration, REDACT, thread creation
- [Satellite](03-satellite.md) - the session flows that carry `sat-invite` and the P2P handshake
- [Depot](04-depot.md) - the upload flow that produces `+orbit/file`
