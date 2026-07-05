# Tags and Trust

## The Tag Layer

Orbit's rich features - media signaling, file metadata, thread discovery -
are transported as IRCv3 client-only tags in the `+orbit/*` namespace. Orbit
clients parse these tags; pure IRC clients ignore them, so there's no noise
and no garbage in chat. The IRC server relays them without interpretation or
validation, which is what lets Orbit work on any IRCv3-compliant server, and
which creates the trust boundary this page defines.

Message retractions are explicitly not in this category: they use the
IRC-standard `REDACT` command, a server-enforced operation the server
validates and authorizes. Clients don't need to verify retraction legitimacy.
See [Uplink](03-uplink.md#message-retractions-and-replies). Replies are also
not an Orbit tag - they use the standard IRCv3 reply tag, interoperable with
any client that implements it.

The full tag table, payload formats, encoding rules, and the standard IRCv3
client tags Orbit handles (replies, reactions, typing) live in
[Implementation - Tags](../03-implementation/02-tags.md).

## Versioning

The `+orbit/*` namespace has no version field. Orbit clients MUST ignore
unknown tags and unknown fields within known tag payloads. That's the
forward-compatibility rule: the namespace can evolve without breaking older
clients. If a future change alters the semantics of an existing tag in a
backward-incompatible way, a new tag name MUST be introduced rather than
redefining the existing one, so older clients ignore the new tag instead of
misinterpreting a changed payload.

Extensions may define their own tag sub-namespace on the same transport - see
[Extensions](15-extensions.md).

## Trust Boundary

Because the server relays `+orbit/*` tags as opaque metadata, every claim a
tag makes is client-asserted and forgeable. Clients enforce trust using the
data the server itself asserts.

### Server-Asserted vs. Client-Asserted Data

| Data                | Source                                              | Trust Level     | Forgeable?                       |
|---------------------|-----------------------------------------------------|-----------------|----------------------------------|
| `account-tag`       | Set by the IRC server based on SASL authentication  | Server-asserted | No - the server is authoritative |
| `msgid`             | Assigned by the IRC server                          | Server-asserted | No                               |
| `server-time`       | Set by the IRC server                               | Server-asserted | No                               |
| `+draft/reply`      | Set by the sending client                           | Client-asserted | Yes                              |
| `+orbit/msg-thread` | Set by the sending client                           | Client-asserted | Yes                              |
| `+orbit/sat-invite` | Set by the sending client                           | Client-asserted | Yes                              |
| `+orbit/p2p-offer`  | Set by the sending client                           | Client-asserted | Yes                              |
| `+orbit/p2p-answer` | Set by the sending client                           | Client-asserted | Yes                              |
| `+orbit/file`       | Set by the sending client                           | Client-asserted | Yes                              |

This model is analogous to email: the transport (SMTP) delivers messages
without validating sender claims, and the receiving client checks DKIM and
SPF to verify authenticity. In Orbit's case, `account-tag` is the DKIM
equivalent - a server-asserted proof of identity that clients use to validate
client-asserted claims.

## Client Enforcement Rules

Orbit clients MUST enforce the following rules using the server-asserted
`account-tag` as the authoritative source of identity:

1. **Satellite invites**: display the sender's verified identity alongside
   the invite. Users should know who is inviting them to a Satellite before
   connecting.
2. **P2P connections**: verify the sender's identity before accepting a P2P
   offer. Display the verified identity and the connection intent (call,
   video, chat, file) in the acceptance prompt. Never auto-accept offers from
   unverified senders.
3. **File metadata**: the file tag (name, size, type) is informational. The
   client SHOULD verify file metadata independently, for example by checking
   HTTP headers on download, rather than trusting the tag blindly.
4. **Message replies**: the reply tag references a target message ID. No
   identity verification is required beyond what the server already provides
   via `account-tag` on the reply message itself.
5. **Thread signals**: accept a thread signal as a creation notice for the
   referenced parent message. Thread signals are informational; no identity
   verification is required beyond `account-tag`.

### Unverified Senders

Replies and thread signals from unauthenticated users are permitted - a reply
is a new message, not a modification of an existing one, and starting a
thread doesn't require a registered account.

Satellite invites and P2P offers from users without an `account-tag` MUST be
visually flagged as unverified, so the user knows the sender's identity can't
be confirmed before they accept.

## Abuse Vectors and Mitigations

### Satellite Invite Spoofing

A malicious user can post an invite tag pointing to a Satellite they control,
potentially to intercept media streams. Orbit clients mitigate this by
displaying the sender's verified identity alongside the invite and by
visually distinguishing server-operated Satellites (verified badge via DNS
discovery) from community/BYOS Satellites (no badge). Users are prompted with
a confirmation dialog before connecting to any BYOS Satellite for the first
time (see [Satellite - Trust Model](05-satellite.md#trust-model)). A spoofed
invite from an unauthenticated sender is additionally flagged as unverified.

### Tag Flooding

A user flooding a channel with junk tags (mass fake invites, spurious thread
signals) is subject to Ergo's standard fakelag flood protection, which
throttles any client exceeding the burst limit. Tag-only messages count
against this limit identically to ordinary messages. Beyond protocol-level
throttling, channel operators can ban abusive users with standard IRC
moderation. No tag-specific rate limiting is needed.

## Identity Display

Orbit clients MUST clearly distinguish between authenticated and
unauthenticated users in all contexts - chat messages, user lists, voice
sessions, and DMs. The IRCv3 `account-tag` is the source of truth.

- Users with a verified `account-tag` SHOULD be displayed with their account
  name and a visual indicator of authentication (a badge, icon, or distinct
  styling).
- Users without an `account-tag` (unauthenticated, guest, or connecting from
  a client without SASL support) MUST be visually distinguished as
  unverified.

This is a gap in traditional IRC clients that Orbit explicitly addresses:
identity verification and display is a core client responsibility, not an
optional feature. User metadata (avatars, display names, status) is never an
identity signal - see
[Services](08-services.md#metadata-is-not-an-identity-signal).

## Third-Party IRC Clients

Orbit clients that don't implement these verification rules are
non-compliant. Third-party IRC clients connecting to an Uplink server won't
enforce these rules (they don't understand `+orbit/*` tags), but that's
expected - they also won't render media invites, thread indicators, or file
previews. The security boundary is between Orbit clients, not between
arbitrary IRC clients.
