# Orbit Tag Namespace

## Overview

WebRTC session descriptions, Satellite invites, and metadata are transported as IRCv3 `TAGMSG` messages with client-only tags (prefixed with `+`). Orbit clients parse these tags. Pure IRC clients ignore them - no noise, no garbage in chat.

All `+orbit/*` tags are client-only and are relayed by the IRC server without interpretation or validation. Enforcement of edit, delete, and signaling rules is the responsibility of Orbit clients. See [Tag Trust Model](02-trust-model.md) for the full enforcement rules.

For Ergochat's tag relay configuration, see [Ground Control](../01-overview.md).

## The `+orbit/*` Tag Table

| Tag                        | Payload                                          | Direction          |
|----------------------------|--------------------------------------------------|--------------------|
| `+orbit/sat-invite`        | Node URL + room ID + metadata (base64 JSON)      | Client -> Channel   |
| `+orbit/sat-leave`         | Room ID                                          | Client -> Channel   |
| `+orbit/sat-status`        | Node health/capacity (base64 JSON)               | Client -> Channel   |
| `+orbit/sdp-offer`         | SDP (base64, for P2P calls)                      | Client -> Client    |
| `+orbit/sdp-answer`        | SDP (base64, for P2P calls)                      | Client -> Client    |
| `+orbit/ice-candidate`     | ICE candidate (JSON, base64)                     | Client -> Client    |
| `+orbit/msg-edit`          | `target-msgid` + new content                     | Client -> Channel   |
| `+orbit/msg-delete`        | `target-msgid`                                   | Client -> Channel   |
| `+orbit/file-name`         | Original filename                                | Client -> Channel   |
| `+orbit/file-size`         | File size in bytes                               | Client -> Channel   |
| `+orbit/file-type`         | MIME type                                        | Client -> Channel   |

## Encoding

Base64 encoding is used because IRC message tags have restricted character sets. IRC message tags have a budget of up to 8191 bytes per the IRCv3 message-tags spec. The message body has a separate limit (512 bytes classic, or server-configured). Payloads exceeding the tag budget are split across a `batch`.

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
