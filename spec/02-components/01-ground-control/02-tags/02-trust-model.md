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
| `+orbit/sdp-offer`    | Set by the sending client                                     | Client-asserted    | Yes                                     |
| `+orbit/file-*`       | Set by the sending client                                     | Client-asserted    | Yes                                     |

For how `account-tag` is configured and required on the server side, see [Ground Control](../01-overview.md).

## Client Enforcement Rules

Orbit clients MUST enforce the following rules using the server-asserted `account-tag` as the authoritative source of identity:

1. **Message edits**: Accept a `+orbit/msg-edit` only if the sender's `account-tag` matches the `account-tag` of the original message being edited.
2. **Message deletes**: Accept a `+orbit/msg-delete` only if the sender's `account-tag` matches the original message's `account-tag`, OR the sender is a channel operator (`+o`). See [Permissions](../../../03-identity/02-permissions.md) for channel operator rules.
3. **Satellite invites**: Display the sender's verified identity (via `account-tag`) alongside the invite. Users should know who is inviting them to a node before connecting.
4. **P2P call signaling**: Verify the sender's identity before accepting a call. Do not auto-accept calls from unverified senders.
5. **File metadata**: The `+orbit/file-*` tags (name, size, type) are informational. The client SHOULD verify file metadata independently (e.g., by checking HTTP headers on download) rather than trusting the tags blindly.

## Unverified Senders

Messages from users without an `account-tag` (unauthenticated users) that carry `+orbit/msg-edit` or `+orbit/msg-delete` tags MUST be silently ignored. Unverified users cannot edit or delete messages.

## The Email/DKIM Analogy

This model is analogous to how email works: the transport (SMTP) delivers messages without validating sender claims, but the receiving mail client checks DKIM signatures and SPF records to verify authenticity. In Orbit's case, `account-tag` is the DKIM equivalent - a server-asserted proof of identity that clients use to validate client-asserted claims.

## Third-Party IRC Clients

Orbit clients that do not implement these verification rules are non-compliant. Third-party IRC clients connecting to an Orbit server will not enforce these rules (they don't understand `+orbit/*` tags), but this is expected - they also won't render edits, deletes, or media invites. The security boundary is between Orbit clients, not between arbitrary IRC clients.
