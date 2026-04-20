# Permissions and Identity Display

This page covers channel-level permissions and how Orbit clients display and enforce user identity. Permission enforcement is client-side, grounded in the IRC server's `account-tag` assertion. For the full client enforcement rules around `account-tag` - including validation of message edits, deletions, and session participation - see [Tag Trust Model](../02-components/01-uplink/02-tags/02-trust-model.md). For Ergochat channel mode configuration, see [Uplink](../02-components/01-uplink/01-overview.md).

## Permissions

Orbit uses IRC's built-in channel modes. Period.

| IRC Mode | Role         | Capabilities                                                |
|----------|--------------|-------------------------------------------------------------|
| `+o`     | Operator     | Kick, ban, manage channel settings, manage messages         |
| `+v`     | Voiced       | Speak in moderated channels                                 |
| `+b`     | Banned       | Cannot join or speak in the channel                         |
| (none)   | Default user | Read and send messages in unmoderated channels              |

There are no custom roles, no role colors, no granular permission overrides. This is an opinionated decision. IRC has a proven, battle-tested permissions model. It covers the needs of the vast majority of communities.

If you need more - role hierarchies, per-channel upload limits, auto-mod rules, custom role colors - build an extension. An IRC bot connected to Uplink can enforce arbitrarily complex rules by monitoring channel events and acting on them. An Orbit client extension can add UI for complementary features in the desktop client (post-MVP). But the core stays simple.

### Bot-Managed Roles

For communities that need role hierarchies before the extension API ships, an IRC bot connected to Uplink is the standard solution. The pattern is straightforward:

1. The bot maintains a role database mapping accounts to roles (e.g., `moderator`, `trusted`, `newcomer`).
2. On user JOIN, the bot checks the user's `account-tag` against the role table and applies the appropriate channel modes (`+o` for moderators, `+v` for trusted users in moderated channels).
3. Bot operators manage roles via DM commands to the bot (`!role add alice moderator`) or via an external admin interface.

This is how IRC communities have managed roles for decades. Any IRC library in any language can implement it. No Orbit-specific API is needed, and the bot runs on any IRCv3 server - not just Orbit's Uplink.

The Orbit extension API (post-MVP) will enable client-side UI for role management that pairs with a companion bot, giving operators a visual interface on top of the same underlying mechanism.

## Identity Display

Orbit clients MUST clearly distinguish between authenticated and unauthenticated users in all contexts - chat messages, user lists, voice sessions, and DMs. The IRCv3 `account-tag` is the source of truth.

- Users with a verified `account-tag` SHOULD be displayed with their account name and a visual indicator of authentication (e.g., a badge, icon, or distinct styling).
- Users without an `account-tag` (unauthenticated, guest, or connecting from a client that does not support SASL) MUST be visually distinguished as unverified.

This is a gap in traditional IRC clients that Orbit explicitly addresses - identity verification and display is a core client responsibility, not an optional feature.

For the full client enforcement rules governing `account-tag` - including how it is used to validate message edits and deletions - see [Tag Trust Model](../02-components/01-uplink/02-tags/02-trust-model.md).