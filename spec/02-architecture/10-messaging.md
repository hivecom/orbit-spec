# Messaging

The messaging model: DMs and their storage, presence, retention and erasure,
moderation, the transport-vs-E2E boundary, and notifications. Server
configuration and command mechanics live in
[Implementation - Uplink](../03-implementation/01-uplink.md); notification
delivery mechanics live in [Implementation - Push](../03-implementation/11-push.md).

## Direct Messages

Direct messages are standard IRC `PRIVMSG` to a nickname. No special channel
convention, no synthetic construct, no new primitive: if you know someone's
nickname, you message them, and Orbit clients render the exchange in a
conversation UI over plain IRC transport.

If a group wants a private conversation, they make an invite-only private
channel and invite the participants. That's already IRC - there's no separate
"group DM" concept in the spec because the tool already exists.

### Storage

DM history uses a two-tier model based on whether end-to-end encryption is
active.

**Standard DMs** use operator-configured retention, the same model as channel
history. Offline delivery is guaranteed for registered users via always-on
mode, and history is available on reconnect via `chathistory`. This is the
default: DMs behave like any other conversation, with history, offline
delivery, and continuity across sessions.

**E2E-encrypted DMs** store ciphertext with the same retention and delivery
model. The server holds, delivers, and eventually expires encrypted messages
exactly as it would plaintext - it just can't read them. Offline delivery
works, history works (the client decrypts locally), and server-side search
doesn't (the server can't index ciphertext; client-side search over decrypted
local history is the answer). When retention expires the ciphertext is
purged; the operator has neither the key nor the content.

A conversation's E2E status is client-derived, not server-flagged: clients
detect encrypted messages by their tag and continue encrypting outbound
messages in kind, inserting a visible marker when E2E is first established so
the user always knows which messages are protected. The encryption design
lives in [E2E Encryption](13-e2e.md); the tag itself in
[Implementation - Tags](../03-implementation/02-tags.md).

### Always-On Mode

Orbit deployments MUST enable Ergo's always-on mode for registered users.
Without it, a DM sent while the recipient is disconnected is lost, which
isn't acceptable for a platform targeting mainstream users. In always-on
mode, the user's session stays alive on the server while they're
disconnected, and DMs are held until the client reconnects. For standard DMs
that means plaintext is stored during the delivery window; for E2E DMs the
server holds ciphertext it can't read.

### Eavesdropping

Operator-configured retention doesn't mean zero risk. Standard DMs travel
through the Uplink server in plaintext, and the operator can in principle
read content at the server level - the same trust model as channels: you're
trusting the server you joined.

The real answer to eavesdropping is end-to-end encryption. With E2E the
server sees only ciphertext, and the operator can't read content even by
inspecting the database. E2E for DMs is designed but not built; until it
ships, DM content is stored with operator-configured retention and isn't
encrypted, and users who need confidentiality should know that.

This doesn't apply to channels. Channels are intentionally server-readable -
encrypting one would break the features that make channels useful: history,
search, threads, moderation. Choose your server accordingly.

## Presence

Presence is layered on Ergo's existing IRCv3 extensions: standard `AWAY` for
basic online/offline state, and the metadata extension for richer status and
profile data.

- **Online/offline** comes from `away-notify` (immediate away-state
  notifications for shared-channel users), `extended-monitor` (presence
  tracking for DM contacts without a shared channel), and `draft/pre-away`
  (smoother transitions when a client is about to disconnect). Three states
  derive from these: online, away, offline.
- **Rich status and profile** build on `draft/metadata-2`: an avatar URL, a
  display name, and a vendor-prefixed status string. The avatar and display
  name keys are the IRCv3 quasi-standard ones so other metadata-aware clients
  interoperate. Clients subscribe to the keys they understand and receive
  live push updates - no polling. On join, current metadata for visible users
  arrives as part of the burst.
- **Fallback**: when metadata is unavailable (older server, or the operator
  hasn't enabled it), clients fall back to generated avatars, nickname as
  display name, and the basic away/online model. Clients SHOULD negotiate the
  metadata extension when advertised and degrade gracefully when not.

Metadata is user-set and unverified: avatar URLs are attacker-controlled for
arbitrary users, so clients MUST sanitize them, SHOULD proxy them, and MUST
never treat any metadata value as an identity claim - see
[Services](08-services.md#metadata-is-not-an-identity-signal).

Read position is tracked server-side per channel via `draft/read-marker` and
synced across all of a user's sessions; Orbit clients MUST use it for unread
tracking rather than device-local state.

The state-detection details, metadata commands, and avatar upload flow live
in [Implementation - Uplink](../03-implementation/01-uplink.md).

## Retention and Erasure

Channel history uses operator-configured retention: operators set how long
history is kept per channel, with no mandate to store everything forever and
no mandated default. DMs use the same model, and E2E content follows the same
mechanics as ciphertext.

Orbit ships no compliance product - DLP, eDiscovery, and audit exports are
out of scope ([Product - Scope](../01-product/03-scope.md)). But because
Orbit is self-hosted infrastructure, the operator is the data controller and
carries any legal obligations (GDPR, CCPA, and similar) for their instance.
Orbit's posture is to give operators the technical levers, not to make the
decisions for them:

- **Time-bounded retention.** Per-channel and per-DM retention windows mean
  personal data isn't kept indefinitely by default - data minimisation is a
  setting, not a custom build.
- **Per-message erasure.** Server-enforced retraction lets users delete their
  own messages and operators delete others', satisfying targeted takedown and
  correction requests. Gated behind an operator setting - see
  [Uplink](03-uplink.md#retractions).
- **Account-wide erasure ("right to be forgotten").** An account can be
  unregistered and its stored messages purged. Ergo can index history by
  account, which makes account-scoped erasure complete and efficient;
  operators expecting to honour erasure requests should enable it, weighing
  the storage and privacy trade-off of the index itself. Configuration lives
  in [Implementation - Uplink](../03-implementation/01-uplink.md).
- **Architectural non-retention.** For P2P calls and E2E DMs, the operator
  never holds plaintext or media at all - the strongest erasure guarantee is
  never having the data.

Whether to enable any of these is a deliberate operator choice. A hobby
instance among friends and an EU-facing public deployment will reasonably
configure retention and erasure very differently; Orbit doesn't impose a
default that presumes either.

## Moderation

IRC moderation at scale isn't hypothetical: channel ops, services, bots, and
network bans ran networks of ninety thousand concurrent users for decades,
and the primitives work. Orbit adds client-side UX on top - operator tooling,
moderation panels, visible trust indicators - and doesn't reinvent the
permission model ([Identity - Permissions](09-identity.md#permissions)).

Orbit isn't text-only like traditional IRC. Depot adds file uploads and
Satellite adds voice and video - a content surface IRC never had, and scale
requires additional care there. The posture is layered:

- **Shared content (files via Depot):** every upload is attributed to a user
  identity, content is removable by operators, and Depot provides a gateway
  chokepoint where automated scanning can be added at scale. Small instances
  don't need it; large public ones can add it. The safe default: anonymous
  users can join calls and chat but can't upload files. The scanning
  postures and driver asymmetry live in
  [Depot - Content Moderation](06-depot.md#content-moderation).
- **Private content (P2P calls, E2E DMs):** the shield is cryptographic, not
  topological. End-to-end encryption plus non-retention means Orbit can't
  see, decrypt, or produce 1:1 content. "We cannot decrypt and did not
  store" is the posture - the same model as Signal.

Orbit is infrastructure: for private content it architecturally can't see it;
for shared content it attributes it and can remove it.

## Security Model: Transport vs. End-to-End

Two distinct, non-interchangeable layers of security.

**Transport security (TLS)** protects content in transit. Every connection in
the Orbit stack is TLS-encrypted; a passive observer on the network sees
nothing.

**End-to-end encryption** protects content from the server operator. The
server terminates TLS, so the operator can read plaintext at the server
level. The rule for when E2E applies:

| Context | E2E | Reason |
|---|---|---|
| Channels | No | Server-mediated. History, search, and moderation require the server to read content. You are trusting the operator. |
| Voice / video (group) | No | The Satellite SFU routes and forwards media. It must be able to read the stream. |
| 1:1 DMs | Yes (designed, not built) | Two parties only. The server carries the envelope but cannot open it. |
| 1:1 calls / video (P2P) | Yes | Direct WebRTC connection. The server is not in the media path. |

If a server mediates the content, there is no E2E. If it's point-to-point,
there is. No exceptions, no per-channel toggles, no partial E2E.

This is a stronger privacy story than platforms that offer inconsistent or
partial E2E, because the boundary is explicit: when you join a channel or a
Satellite room, you're trusting the operator of that server. If you don't
trust the operator, use a DM or a P2P call. The E2E protocol design lives in
[E2E Encryption](13-e2e.md).

## Notifications

The IRC server side of push notifications is native in stock Ergo, which
implements the `draft/webpush` specification (v2.15.0+): subscription
registration and transport are built into the adopted server, and Ergo
dispatches push events internally. What Orbit builds is the delivery layer -
the relays that carry those events to each platform - specified in
[Implementation - Push](../03-implementation/11-push.md).

Detection belongs inside the IRC server for two reasons:

- **It avoids duplicating knowledge.** Online/offline state is core session
  state, highlight matching is native, and DM recipients are known at
  delivery time. A separate service would have to re-derive all of this from
  the outside - redundant and fragile.
- **It can observe DMs.** A DM is delivered only to its target; a bot-based
  approach could never see DMs directed at other users without operator-level
  privileges and the privacy implications those carry. Native integration has
  no such constraint.

Push covers two events for offline users: a DM to a user with no active
sessions, and a channel message mentioning an offline member.

The payload policy is privacy-first: no message content, ever. The
notification says only "New message from alice" or "alice mentioned you in
#gaming". Content stays on the server and is retrieved by the client over
`chathistory` on reconnect, so sensitive content never passes through FCM,
APNs, or any third-party relay infrastructure. The delivery layer enforces
this policy.

The delivery-side risks are operational: platform push credentials expire and
need renewal, device tokens churn on reinstall, and configuring platform
credentials is real work for small operators. Mitigations live with the
delivery mechanics in [Implementation - Push](../03-implementation/11-push.md).
