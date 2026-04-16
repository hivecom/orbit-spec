# Permissions and Identity Display

This page covers channel-level permissions and how Orbit clients display and enforce user identity. Permission enforcement is client-side, grounded in the IRC server's `account-tag` assertion. For the full client enforcement rules around `account-tag` - including validation of message edits, deletions, and session participation - see [Tag Trust Model](../02-components/01-ground-control/02-tags/02-trust-model.md). For Ergochat channel mode configuration, see [Ground Control](../02-components/01-ground-control/01-overview.md).

## Permissions

Orbit uses IRC's built-in channel modes. Period.

| IRC Mode | Role         | Capabilities                                                |
|----------|--------------|-------------------------------------------------------------|
| `+o`     | Operator     | Kick, ban, manage channel settings, manage messages         |
| `+v`     | Voiced       | Speak in moderated channels, upload files                   |
| `+b`     | Banned       | Cannot join or speak in the channel                         |
| (none)   | Default user | Read and send messages in unmoderated channels              |

There are no custom roles, no role colors, no granular permission overrides. This is an opinionated decision. IRC has a proven, battle-tested permissions model. It covers the needs of the vast majority of communities.

If you need more - role hierarchies, per-channel upload limits, auto-mod rules, custom role colors - build an extension. An IRC bot connected to Ground Control can enforce arbitrarily complex rules by monitoring channel events and acting on them. An Orbit client extension can add UI for complementary features in the desktop client (post-MVP). But the core stays simple.

## Identity Display

Orbit clients MUST clearly distinguish between authenticated and unauthenticated users in all contexts - chat messages, user lists, voice sessions, and DMs. The IRCv3 `account-tag` is the source of truth.

- Users with a verified `account-tag` SHOULD be displayed with their account name and a visual indicator of authentication (e.g., a badge, icon, or distinct styling).
- Users without an `account-tag` (unauthenticated, guest, or connecting from a client that does not support SASL) MUST be visually distinguished as unverified.

This is a gap in traditional IRC clients that Orbit explicitly addresses - identity verification and display is a core client responsibility, not an optional feature.

For the full client enforcement rules governing `account-tag` - including how it is used to validate message edits and deletions - see [Tag Trust Model](../02-components/01-ground-control/02-tags/02-trust-model.md).
