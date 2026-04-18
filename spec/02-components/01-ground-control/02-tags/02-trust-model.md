# Tag Integrity and Client Trust Model

## Trust Boundary

Orbit's rich features - message editing, deletion, media signaling, file metadata - are transported as IRCv3 client-only tags (`+orbit/*`). These tags are relayed by the IRC server without interpretation or validation: the server passes them through as opaque metadata. This is by design (it means Orbit works on any IRCv3-compliant server), but it creates a trust boundary that clients must enforce.

For the full list of `+orbit/*` tags and their payloads, see [Tag Namespace](01-namespace.md).

## Server-Asserted vs. Client-Asserted Data

| Data                  | Source                                                        | Trust Level        | Forgeable?                              |
|-----------------------|---------------------------------------------------------------|--------------------|-----------------------------------------|
| `account-tag`         | Set by the IRC server based on SASL authentication            | Server-asserted    | No - the server is authoritative        |
| `msgid`               | Assigned by the IRC server                                    | Server-asserted    | No                                      |
| `server-time`         | Set by the IRC server                                         | Server-asserted    | No                                      |
| `+orbit/msg-edit`     | Set by the sending client                                     | Client-asserted    | Yes - any client can send this tag      |
| `+orbit/msg-delete`   | Set by the sending client                                     | Client-asserted    | Yes                                     |
| `+orbit/sat-invite`   | Set by the sending client                                     | Client-asserted    | Yes                                     |
| `+orbit/p2p-offer`   | Set by the sending client                                     | Client-asserted    | Yes                                     |
| `+orbit/file`         | Set by the sending client                                     | Client-asserted    | Yes                                     |

For how `account-tag` is configured and required on the server side, see [Ground Control](../01-overview.md).

## Client Enforcement Rules

Orbit clients MUST enforce the following rules using the server-asserted `account-tag` as the authoritative source of identity:

1. **Message edits**: Accept a `+orbit/msg-edit` only if the sender's `account-tag` matches the `account-tag` of the original message being edited.
2. **Message deletes**: Accept a `+orbit/msg-delete` only if the sender's `account-tag` matches the original message's `account-tag`, OR the sender is a channel operator (`+o`). See [Permissions](../../../03-identity/02-permissions.md) for channel operator rules.
3. **Satellite invites**: Display the sender's verified identity (via `account-tag`) alongside the invite. Users should know who is inviting them to a node before connecting.
4. **P2P connections**: Verify the sender's identity (`account-tag`) before accepting a `+orbit/p2p-offer`. Display the sender's verified identity and the connection intent (call, video, chat, file) in the acceptance prompt. Do not auto-accept offers from unverified senders.
5. **File metadata**: The `+orbit/file` tag (name, size, type) is informational. The client SHOULD verify file metadata independently (e.g., by checking HTTP headers on download) rather than trusting the tag blindly.

## Unverified Senders

Messages from users without an `account-tag` (unauthenticated users) that carry `+orbit/msg-edit` or `+orbit/msg-delete` tags MUST be silently ignored. Unverified users cannot edit or delete messages.

## Abuse Vectors and Mitigations

This section addresses known attack surfaces in the client-asserted tag model and how they are mitigated.

### Edit/Delete Spoofing

A malicious client (or IRC script) can send `+orbit/msg-edit` or `+orbit/msg-delete` tags targeting another user's messages. This is a non-issue: compliant Orbit clients verify the sender's `account-tag` against the original message's `account-tag` before accepting the operation. If they don't match, the tag is silently dropped. Unauthenticated senders (no `account-tag`) carrying edit or delete tags are also silently ignored. The spoofed tag is relayed by the server but has no effect on any compliant client.

### Satellite Invite Spoofing

A malicious user can post a `+orbit/sat-invite` tag pointing to a Satellite they control, potentially to intercept media streams. Orbit clients mitigate this by displaying the sender's verified identity (`account-tag`) alongside the invite and by visually distinguishing server-operated Satellites (verified badge via DNS SRV) from community/BYON Satellites (no badge). Users are prompted with a confirmation dialog before connecting to any BYON Satellite (see [Satellite - Trust Model](../../02-components/02-satellite.md#trust-model)). A spoofed invite from an unauthenticated sender is visually flagged as unverified.

### Tag Flooding

A user flooding a channel with junk `+orbit/*` tags (mass edit attempts, fake invites, etc.) is subject to Ergochat's standard `fakelag` flood protection, which throttles any client exceeding the burst limit (~5 commands, then ~2 messages/second sustained). `TAGMSG` counts against this limit identically to `PRIVMSG`. Beyond protocol-level throttling, channel operators can ban (`+b`) abusive users using standard IRC moderation. No additional tag-specific rate limiting is needed.

### Bait-and-Switch Edits

A registered user can post a message, wait for reactions or trust, then edit it to something entirely different. This is a social/moderation problem common to any platform that supports message editing (including Discord, Slack, and Matrix). Orbit clients display an "edited" indicator on modified messages. Channel operators can delete any message regardless of authorship (see [Client Enforcement Rules](#client-enforcement-rules)). Communities that need stricter controls can deploy moderation bots that log original message content or restrict edit windows.

## The Email/DKIM Analogy

This model is analogous to how email works: the transport (SMTP) delivers messages without validating sender claims, but the receiving mail client checks DKIM signatures and SPF records to verify authenticity. In Orbit's case, `account-tag` is the DKIM equivalent - a server-asserted proof of identity that clients use to validate client-asserted claims.

## Third-Party IRC Clients

Orbit clients that do not implement these verification rules are non-compliant. Third-party IRC clients connecting to an Orbit server will not enforce these rules (they don't understand `+orbit/*` tags), but this is expected - they also won't render edits, deletes, or media invites. The security boundary is between Orbit clients, not between arbitrary IRC clients.
