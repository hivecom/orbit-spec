# Orbit Tag Namespace

## Overview

WebRTC session descriptions, Satellite invites, and metadata are transported as IRCv3 `TAGMSG` messages with client-only tags (prefixed with `+`). Orbit clients parse these tags. Pure IRC clients ignore them - no noise, no garbage in chat.

All `+orbit/*` tags are client-only and are relayed by the IRC server without interpretation or validation. Enforcement of edit, delete, and signaling rules is the responsibility of Orbit clients. See [Tag Trust Model](02-trust-model.md) for the full enforcement rules.

For Ergochat's tag relay configuration, see [Ground Control](../01-overview.md).

## The `+orbit/*` Tag Table

| Tag                        | Payload                                          | Direction          |
|----------------------------|--------------------------------------------------|--------------------|
| `+orbit/sat-invite`        | Node URL + room ID + metadata (base64 JSON)      | Client -> Channel   |
| `+orbit/p2p-offer`         | P2P handshake: intent + ICE credentials + DTLS fingerprint + candidate (base64 JSON) | Client -> Client    |
| `+orbit/p2p-answer`        | P2P handshake response: ICE credentials + DTLS fingerprint + candidate (base64 JSON) | Client -> Client    |
| `+orbit/msg-thread`        | `parent_msgid` + thread channel name (base64 JSON) | Client -> Channel   |
| `+orbit/msg-amend`          | `target-msgid` + new content                     | Client -> Channel   |
| `+orbit/msg-retract`        | `target-msgid`                                   | Client -> Channel   |
| `+orbit/msg-react`         | `target-msgid`                                   | Client -> Channel   |
| `+orbit/msg-reply`         | `target-msgid`                                   | Client -> Channel   |
| `+orbit/file`                | File metadata: name, size, type (base64 JSON)    | Client -> Channel   |

> **Note on `+orbit/msg-react`**: This tag is a cache invalidation signal only. When a client receives it, the correct response is to refetch reaction state for the referenced `target-msgid` from Reactor. The tag carries no reaction content - no key, no account, no count. Clients MUST NOT attempt to reconstruct reaction state from IRC history. The [Reaction Service](../../../07-research/12-reaction-service.md) (Reactor, post-MVP) is the sole authoritative source of reaction state.

> **Note on `+orbit/msg-thread`**: This tag serves a dual role as both a **creation signal** and an **activity signal**. It is sent to the parent channel - not the thread channel - when a thread is first created AND on every subsequent reply. The payload identifies the thread channel by name so receiving clients can discover and join it. Clients MUST NOT attempt to reconstruct thread reply counts solely from the number of TAGMSGs received - a client that was offline may have missed signals. The authoritative reply list is always the `chathistory` of the thread channel itself.
>
> On thread creation only, the client also sends a plain `PRIVMSG` to the parent channel alongside this TAGMSG:
>
> ```
> ↳ Thread started by alice in #dev.t-abc123
> ```
>
> This PRIVMSG is visible to all IRC clients and allows them to follow the thread by joining the named channel directly. Orbit clients suppress it from the chat view and render the thread indicator instead. Subsequent replies do NOT send a PRIVMSG to the parent channel - only the TAGMSG activity signal is sent on each reply.
>
> See [Ground Control - Threads](../01-overview.md#threads) for the full thread design.

## Encoding

Base64 encoding is used because IRC message tags have restricted character sets. IRC message tags have a budget of up to 8191 bytes per the IRCv3 message-tags spec. The message body has a separate limit (512 bytes classic, or server-configured). Payloads exceeding the tag budget are split across a `batch`.

## Versioning

The `+orbit/*` tag namespace has no version field. Orbit clients MUST ignore unknown tags and unknown fields within known tag payloads. This is the forward-compatibility rule: a client that does not understand a tag silently skips it, and a client that receives a known tag with extra fields ignores the fields it does not recognize. This allows the tag namespace to evolve without breaking older clients.

If a future change alters the semantics of an existing tag in a backward-incompatible way, a new tag name MUST be introduced rather than redefining the existing tag. This ensures that older clients continue to behave correctly (by ignoring the new tag) rather than misinterpreting a changed payload format.

## Extensions

### Orbit Extensions

An Orbit extension is a true application-level plugin for the Orbit client (orbit-app). Extensions are installed into the client and extend its UI and behavior. They interact with Ground Control and Satellite through the standard Orbit tag namespace and may define their own sub-namespace for custom tags (e.g., `+orbit-ext/<name>/*`). Extensions are scoped to the Orbit client - they do not run on the server, and they do not modify Satellite infrastructure.

The orbit-app extension API is **deferred to post-MVP**. For the MVP, extensions are a design target, not a shipping feature.

**Examples of future Orbit extensions:**

- A **calendar extension** that renders upcoming events inline in the Orbit client, sourced from custom `+orbit-ext/calendar/*` tags posted by a companion bot.
- A **moderation dashboard extension** that adds a UI panel for reviewing flagged messages and applying IRC modes.
- A **game status extension** that displays live match state in the sidebar using custom tags.

### IRC Bots

IRC bots are first-class citizens of the Orbit ecosystem by virtue of Orbit being built on IRC. Any IRC bot - written in any language, using any framework (Limnoria, Sopel, a custom script) - connects to Ground Control and works out of the box. Bots handle server-side automation: posting reminders, enforcing moderation rules, managing channel state, and responding to commands. They are not Orbit extensions; they are IRC bots.

**Examples of IRC bots (not extensions):**

- A **permissions bot** that implements role hierarchies beyond IRC modes.
- A **reminder bot** that posts scheduled messages to channels.
- A **moderation bot** with auto-mod rules, word filters, and spam detection.
