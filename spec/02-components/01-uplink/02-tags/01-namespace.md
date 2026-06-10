# Orbit Tag Namespace

## Overview

WebRTC session descriptions, Satellite invites, and metadata are transported as IRCv3 `TAGMSG`
messages with client-only tags (prefixed with `+`). Orbit clients parse these tags. Pure IRC
clients ignore them - no noise, no garbage in chat.

All `+orbit/*` tags are client-only and are relayed by the IRC server without interpretation or
validation. Enforcement of signaling rules is the responsibility of Orbit clients. See
[Tag Trust Model](02-trust-model.md) for the full enforcement rules.

Message retractions are **not** handled by a client tag. In the MVP, retractions use the
IRC-standard `REDACT` command provided by the `draft/message-redaction` extension - a
server-enforced operation. See [Uplink Overview - Message Retractions](../01-overview.md#message-retractions-and-replies)
for the full retraction flow.

Message editing is **not standardized in IRC yet** and does not exist as a stable feature today.
Orbit handles editing at the client and tag layer and will adopt a standard if Ergo or IRCv3 ships
one. `+orbit/msg-amend` is an interim tag, not a long-term design.

For Ergochat's tag relay configuration, see [Uplink Overview](../01-overview.md).

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

> **Note on `+orbit/msg-amend`**: This tag is an **interim mechanism** for in-place message editing, used until the IRC ecosystem standardizes message editing (active draft work is ongoing). The payload is base64 JSON: `{ "msgid": "<original-msgid>", "body": "<new-message-body>" }`. Receiving clients that understand this tag SHOULD update the displayed message body and render an "edited" indicator. Clients that do not understand the tag see nothing - the edit is invisible to them. If Ergo or IRCv3 adopts a standard editing mechanism, Orbit clients will adopt it and retire this tag.

> **Note on `+orbit/msg-thread`**: This tag is a **creation signal only**. It is sent to the parent
> channel - not the thread channel - when a thread is first created. It is NOT sent on subsequent
> replies. Subsequent replies are normal PRIVMSGs in the thread channel - being in the channel is
> the subscription.
>
> The payload identifies the thread channel by name so receiving clients can discover and join it.
> The authoritative reply list is always the `chathistory` of the thread channel itself.
>
> On thread creation, the client also sends a plain `PRIVMSG` to the parent channel alongside
> this TAGMSG:
>
> ```
> ↳ Thread started by alice in #dev/t-abc123
> ```
>
> This PRIVMSG is visible to all IRC clients and allows them to follow the thread by joining the
> named channel directly. Orbit clients suppress it from the chat view and render the thread
> indicator instead.
>
> See [Uplink Overview - Threads](../01-overview.md#threads) for the full thread design.

> **Note on `+orbit/e2e`**: This tag is **post-MVP** and is reserved for end-to-end encrypted DMs.
> When present on a `PRIVMSG`, it indicates the message body is ciphertext encrypted with the
> Double Ratchet protocol. The tag value contains the sender's key ID, which the recipient uses to
> select the correct decryption key. See [Research: E2E Encryption](../../../06-next/05-e2e-encryption.md)
> for the full design.

## Encoding

Base64 encoding is used because IRC message tags have restricted character sets. IRC message tags
have a budget of up to 8191 bytes per the IRCv3 message-tags spec. The message body has a separate
limit (512 bytes classic, or server-configured). Payloads exceeding the tag budget are split across
a `batch`.

## Versioning

The `+orbit/*` tag namespace has no version field. Orbit clients MUST ignore unknown tags and
unknown fields within known tag payloads. This is the forward-compatibility rule: a client that
does not understand a tag silently skips it, and a client that receives a known tag with extra
fields ignores the fields it does not recognize. This allows the tag namespace to evolve without
breaking older clients.

If a future change alters the semantics of an existing tag in a backward-incompatible way, a new
tag name MUST be introduced rather than redefining the existing tag. This ensures that older clients
continue to behave correctly (by ignoring the new tag) rather than misinterpreting a changed
payload format.

## Extensions

### Orbit Extensions

An Orbit extension is a true application-level plugin for the Orbit client (orbit-app). Extensions
are installed into the client and extend its UI and behavior. They interact with Uplink and
Satellite through the standard Orbit tag namespace and may define their own sub-namespace for
custom tags (e.g., `+orbit-ext/<name>/*`). Extensions are scoped to the Orbit client - they do
not run on the server, and they do not modify Satellite infrastructure.

The orbit-app extension API is **deferred to post-MVP**. For the MVP, extensions are a design
target, not a shipping feature.

**Examples of future Orbit extensions:**

- A **calendar extension** that renders upcoming events inline in the Orbit client, sourced from
  custom `+orbit-ext/calendar/*` tags posted by a companion bot.
- A **moderation dashboard extension** that adds a UI panel for reviewing flagged messages and
  applying IRC modes.
- A **game status extension** that displays live match state in the sidebar using custom tags.

### IRC Bots

IRC bots are first-class citizens of the Orbit ecosystem by virtue of Orbit being built on IRC.
Any IRC bot - written in any language, using any framework (Limnoria, Sopel, a custom script) -
connects to Uplink and works out of the box. Bots handle server-side automation: posting reminders,
enforcing moderation rules, managing channel state, and responding to commands. They are not Orbit
extensions; they are IRC bots.

**Examples of IRC bots (not extensions):**

- A **permissions bot** that implements role hierarchies beyond IRC modes.
- A **reminder bot** that posts scheduled messages to channels.
- A **moderation bot** with auto-mod rules, word filters, and spam detection.

## Standard IRCv3 Client Tags

Orbit clients handle the following standard IRCv3 client-only tags in addition to the `+orbit/*` namespace. These are defined by the IRCv3 working group and interoperate with any compliant client.

| Tag | Spec | Purpose |
|---|---|---|
| `+draft/reply` / `+reply` | [IRCv3 message-replies](https://ircv3.net/specs/extensions/message-replies) | Reference the `msgid` of the message being replied to. Always sent as a client tag on the `PRIVMSG`. Orbit renders an inline excerpt of the original message and a navigation link. |
| `+draft/react` / `+react` | [IRCv3 react](https://ircv3.net/specs/client-tags/react) | Emoji reaction on a message. Always paired with `+reply` pointing at the parent `msgid`. Orbit renders the reaction inline on the original message. |
| `+draft/unreact` / `+unreact` | [IRCv3 react](https://ircv3.net/specs/client-tags/react) | Remove a previously sent emoji reaction. Always paired with `+reply`. Orbit removes the reaction from the inline display. |
| `+typing` | [IRCv3 typing](https://ircv3.net/specs/client-tags/typing) | Typing notification with value `active`, `paused`, or `done`. Sent as a `TAGMSG` to the channel or user. Orbit renders a typing indicator in the composer area. |

These tags are transported as `TAGMSG` messages (reactions, typing) or as client tags on `PRIVMSG` (replies). They are subject to the same `account-tag`-based identity display rules as `PRIVMSG` content - see [Identity Display](../../../03-identity/02-permissions.md#identity-display). They are also subject to Ergochat's standard fakelag flood protection identically to `PRIVMSG`.

> **Reactions and replies interop:** Because `+draft/reply` and the react tags are IRCv3 standards, other clients that implement these specs will interoperate with Orbit clients automatically. A third-party IRC client that understands `+draft/reply` will render replies; one that understands `+draft/react` will render reactions. Orbit does not define custom alternatives for these features.
