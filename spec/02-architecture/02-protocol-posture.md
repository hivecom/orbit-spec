# Protocol Posture

Orbit conforms to IRCv3 and supports whatever stock Ergo implements. There is
no Orbit fork of the IRC server, and Orbit is not in the business of
reimplementing IRC. Where IRC hasn't standardized something yet, Orbit
follows the same draft work the rest of the ecosystem does and handles the
remainder at the client and tag layer. If a capability is standardized in
IRCv3, great; if Ergo ships it, Orbit gets it for free; if neither has
happened, Orbit adapts. Orbit contributes upstream where it helps, but the
leverage is the product, not the protocol.

Much of what a private fork was once imagined for is already native in stock
Ergo (current stable v2.18.0):

- Push notifications via `draft/webpush` (v2.15.0).
- OIDC/JWT authentication - a small auth bridge for any provider, or Ergo's
  built-in JWT verification for a constrained set of providers (see
  [Transponder](07-transponder.md)).
- User and channel metadata (avatars, display names, presence status) via
  stable `draft/metadata-2` (v2.17.0).
- Message retraction and deletion via `draft/message-redaction` (the `REDACT`
  command).
- An HTTP API (v2.16.0+) and Postgres/SQLite history backends (v2.18.0).

The compatibility guarantee is non-negotiable: any IRC client - WeeChat,
irssi, Textual, ZNC, any compliant third-party client - works against Uplink
without modification, precisely because Uplink is a stock IRCv3 server.

## Current Gaps and How Each Resolves

IRC is the right foundation, and it's worth naming what it doesn't natively
support and how each gap is resolved:

- **Message editing** isn't standardized in IRC yet. There is active draft
  work on in-place editing across the IRC ecosystem; Orbit follows it and
  adopts a standard when Ergo or IRCv3 ships one. Until then, Orbit edits
  through a client-side convention over message tags (`+orbit/msg-amend`) -
  the interim direction until a real solution exists at the protocol level,
  retired the moment one ships. It covers display only: stored history keeps
  the original, unlike retraction, where `REDACT` deletes server-side. The
  tag lives in [Implementation - Tags](../03-implementation/02-tags.md).
- **Message retractions** use the IRC-standard `REDACT` command via
  `draft/message-redaction`, shipped and stable in Ergo. Server-enforced, not
  a client tag. See [Uplink](03-uplink.md#message-retractions-and-replies).
- **Reactions** work today, handled client-side via message tags. The one
  concession is historic search: reactions can't be shown on messages
  surfaced purely from a search result.
- **User metadata** (avatars, display names, presence status) is handled by
  `draft/metadata-2`, stable in Ergo (v2.17.0+). Clients subscribe to keys
  and receive live push updates.
- **Presence** beyond `AWAY`/`JOIN`/`QUIT` is handled by `away-notify`,
  `extended-monitor`, and `draft/pre-away`, all stable in Ergo. Richer status
  strings build on `draft/metadata-2`. See
  [Messaging - Presence](10-messaging.md#presence).
- **Threads** are implemented via client-managed sub-channels and a signaling
  tag. The server sees a normal channel; this works on any IRCv3 server
  without modification. See [Uplink - Threads](03-uplink.md#threads).
- **Federation** needs server-to-server linking that stock IRC servers don't
  provide. Nothing in Orbit depends on it. See
  [Federation](14-federation.md).

None of these are reasons to abandon IRC. They're the current state, each
with a defined resolution path.

## Compatibility Profile

What standard IRC clients experience when connecting to an Uplink server.
Standard IRC clients are fully supported and get a great experience, but some
Orbit-specific features are invisible to them.

### Connection and Authentication

Standard IRC clients connect identically to how they connect to any IRCv3
server. SASL PLAIN and SCRAM-SHA-256 are supported. SASL ANONYMOUS is
supported for guest access.

### Feature Compatibility Table

| Feature | Orbit Client | IRC Client (modern, IRCv3) | IRC Client (basic) |
|---|---|---|---|
| Text chat | ✓ Full | ✓ Full | ✓ Full |
| Message history (`chathistory`) | ✓ Full | ✓ Full | Limited (no `chathistory` support) |
| Message retractions | ✓ Tombstone rendered | ✓ Message removed (via `draft/message-redaction`) | Sees `*** alice retracted a message ***` NOTICE |
| Message editing | ✓ Interim edit tag (`+orbit/msg-amend`) | Sees original text (tag invisible) | Sees original text |
| Threads | ✓ Thread panel UI | Can `/join` thread channel manually | Can `/join` thread channel manually |
| Voice / video | ✓ Full | Not available | Not available |
| File uploads | ✓ Inline preview | Sees plain URL | Sees plain URL |
| User avatars | ✓ Rendered (via Metadata) | ✓ Accessible (via `METADATA GET`) | Not available |
| Display names | ✓ Rendered (via Metadata) | ✓ Accessible (via `METADATA GET`) | Not available |
| Presence (online/away) | ✓ Rich (via Metadata + away-notify) | ✓ Basic (AWAY status) | ✓ Basic (AWAY status) |
| Read markers | ✓ Synced across devices | ✓ If client supports `draft/read-marker` | Not available |
| DMs | ✓ Full history; E2E optional (designed, not built) | ✓ Standard IRC PRIVMSG with history | ✓ Standard IRC PRIVMSG |
| Group DMs | ✓ Invite-only private channel | ✓ Standard IRC private channel | ✓ Standard IRC private channel |
| Satellite invites | ✓ Rendered as join button | Sees TAGMSG (invisible if no tag support) | Not visible |
| P2P call invites | ✓ Rendered as call prompt | Sees TAGMSG (invisible if no tag support) | Not visible |

### Bots

Any IRC bot - written in any language, using any IRC library - connects to
Uplink and works immediately. Bots can moderate channels, post reminders,
respond to commands, and trigger Satellite session announcements. No
Orbit-specific SDK is needed. See [Extensions](15-extensions.md).

## Upstream Directions

Richer presence capabilities are tracked as potential IRCv3 upstream
contributions rather than as an Orbit-owned server, so every IRC client
benefits and compatibility is preserved. Candidates include:

- Richer status states with structured semantics, not just a string.
- Presence history (when a user was last online).
