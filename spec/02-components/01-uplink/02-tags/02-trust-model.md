# Tag Integrity and Client Trust Model

## Trust Boundary

Orbit's rich features - media signaling, file metadata, thread discovery, replies - are transported
as IRCv3 client-only tags (`+orbit/*`). These tags are relayed by the IRC server without
interpretation or validation: the server passes them through as opaque metadata. This is by design
(it means Orbit works on any IRCv3-compliant server), but it creates a trust boundary that clients
must enforce.

Message retractions are explicitly **not** in this category. In the MVP, retractions use the
IRC-standard `REDACT` command via `draft/message-redaction` - a server-enforced operation that the
server validates and authorises. Clients do not need to verify retraction legitimacy; the server
already has. See [Uplink Overview - Message Retractions](../01-overview.md#message-retractions-and-replies)
for the full retraction flow.

For the full list of `+orbit/*` tags and their payloads, see [Tag Namespace](01-namespace.md).

## Server-Asserted vs. Client-Asserted Data

| Data                | Source                                                         | Trust Level     | Forgeable?                       |
|---------------------|----------------------------------------------------------------|-----------------|----------------------------------|
| `account-tag`       | Set by the IRC server based on SASL authentication             | Server-asserted | No - the server is authoritative |
| `msgid`             | Assigned by the IRC server                                     | Server-asserted | No                               |
| `server-time`       | Set by the IRC server                                          | Server-asserted | No                               |
| `+orbit/msg-reply`  | Set by the sending client                                      | Client-asserted | Yes                              |
| `+orbit/msg-thread` | Set by the sending client                                      | Client-asserted | Yes                              |
| `+orbit/sat-invite` | Set by the sending client                                      | Client-asserted | Yes                              |
| `+orbit/p2p-offer`  | Set by the sending client                                      | Client-asserted | Yes                              |
| `+orbit/p2p-answer` | Set by the sending client                                      | Client-asserted | Yes                              |
| `+orbit/file`       | Set by the sending client                                      | Client-asserted | Yes                              |

For how `account-tag` is configured and required on the server side, see
[Uplink Overview](../01-overview.md).

## Client Enforcement Rules

Orbit clients MUST enforce the following rules using the server-asserted `account-tag` as the
authoritative source of identity:

1. **Satellite invites**: Display the sender's verified identity (via `account-tag`) alongside the
   invite. Users should know who is inviting them to a node before connecting.

2. **P2P connections**: Verify the sender's identity (`account-tag`) before accepting a
   `+orbit/p2p-offer`. Display the sender's verified identity and the connection intent (call,
   video, chat, file) in the acceptance prompt. Do not auto-accept offers from unverified senders.

3. **File metadata**: The `+orbit/file` tag (name, size, type) is informational. The client SHOULD
   verify file metadata independently (e.g., by checking HTTP headers on download) rather than
   trusting the tag blindly.

4. **Message replies**: A `+orbit/msg-reply` tag references a `target-msgid`. The client SHOULD
   display the reply with an excerpt of the original message (if available in the local buffer) and
   a link to navigate to it. No identity verification is required beyond what the server already
   provides via `account-tag` on the reply message itself. If the target message is not in the
   client's buffer, the reply is displayed without the excerpt.

5. **Thread signals**: Accept a `+orbit/msg-thread` TAGMSG as a thread creation signal for the
   referenced `parent_msgid`. This tag is sent only when a thread is first created, not on
   subsequent replies. No identity verification is required beyond what the server already
   provides via `account-tag`. Thread signals are informational - they notify other clients that a
   thread channel exists. Unverified senders (no `account-tag`) MAY post thread signals - starting
   a thread does not require a registered account.

## Unverified Senders

Replies (`+orbit/msg-reply`) from unauthenticated users are permitted - a reply is a new message,
not a modification of an existing one. Thread signals (`+orbit/msg-thread`) from unauthenticated
users are also permitted for the same reason.

Satellite invite tags (`+orbit/sat-invite`) and P2P offer tags (`+orbit/p2p-offer`,
`+orbit/p2p-answer`) from users without an `account-tag` MUST be visually flagged as unverified in
the Orbit client UI. The user is informed that the sender's identity cannot be confirmed before
they accept the invite or connection.

## Abuse Vectors and Mitigations

This section addresses known attack surfaces in the client-asserted tag model and how they are
mitigated.

### Satellite Invite Spoofing

A malicious user can post a `+orbit/sat-invite` tag pointing to a Satellite they control,
potentially to intercept media streams. Orbit clients mitigate this by displaying the sender's
verified identity (`account-tag`) alongside the invite and by visually distinguishing
server-operated Satellites (verified badge via DNS SRV) from community/BYON Satellites (no badge).
Users are prompted with a confirmation dialog before connecting to any BYON Satellite (see
[Satellite - Trust Model](../../02-satellite.md#trust-model)). A spoofed invite from an
unauthenticated sender is visually flagged as unverified.

### Tag Flooding

A user flooding a channel with junk `+orbit/*` tags (mass fake invites, spurious thread signals,
etc.) is subject to Ergochat's standard `fakelag` flood protection, which throttles any client
exceeding the burst limit (~5 commands, then ~2 messages/second sustained). `TAGMSG` counts against
this limit identically to `PRIVMSG`. Beyond protocol-level throttling, channel operators can ban
(`+b`) abusive users using standard IRC moderation. No additional tag-specific rate limiting is
needed.

## The Email/DKIM Analogy

This model is analogous to how email works: the transport (SMTP) delivers messages without
validating sender claims, but the receiving mail client checks DKIM signatures and SPF records to
verify authenticity. In Orbit's case, `account-tag` is the DKIM equivalent - a server-asserted
proof of identity that clients use to validate client-asserted claims.

## Third-Party IRC Clients

Orbit clients that do not implement these verification rules are non-compliant. Third-party IRC
clients connecting to an Uplink server will not enforce these rules (they don't understand
`+orbit/*` tags), but this is expected - they also won't render media invites, thread indicators,
or file previews. The security boundary is between Orbit clients, not between arbitrary IRC
clients.
