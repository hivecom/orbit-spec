# Push Notifications

Push notification delivery is a planned capability of the Uplink fork. It is not a separate service — Uplink holds all the state needed to detect notification-worthy events (session state, online/offline status, DM recipients, channel membership) and dispatches notifications internally.

In the **MVP** (Ergo/Ergochat), there are no push notifications. The PWA provides limited background notification via the Web Notifications API on supported desktop browsers. On mobile, PWA push notifications work on Android (Chrome) and iOS 16.4+ (Safari).

Cross-references:
- [Uplink](../02-components/01-uplink/01-overview.md) - the IRC layer; push notifications are implemented in the Uplink fork
- [Transponder](../02-components/04-transponder.md) - OIDC identity model used for JWT-verified push token registration
- [Mobile Clients](02-mobile-clients.md) - the track that push notifications unblock

## Why Uplink-Native

Push notification delivery belongs inside the Uplink server for two reasons:

**It avoids duplicating knowledge.** Online/offline state is core IRC session state — Uplink knows immediately when a user has no active sessions. Nickname highlight matching is something the server can do natively. DM recipients are known at the moment of delivery. A separate service would need to re-derive all of this from the outside, which is redundant and fragile.

**It can observe DMs.** In standard IRC, a `PRIVMSG` to a nickname is delivered only to that nickname — no other connected client sees it. A separate bot-based approach could monitor channel mentions but could never see DMs directed at other users without IRC operator-level privileges, carrying serious privacy implications. Native integration has no such constraint.

## What Push Notifications Cover

Uplink detects the following events and dispatches push notifications for offline users:

- **DM**: a `PRIVMSG` to a nickname where the target user has zero active sessions
- **Mention**: a channel `PRIVMSG` that contains the nick of an offline channel member

No message content is included in the push payload. The notification says only *"New message from alice"* or *"alice mentioned you in #gaming"*. Message content stays on the Uplink server and is retrieved by the client via `CHATHISTORY` on reconnect. Sensitive content never passes through FCM, APNs, or any third-party relay infrastructure.

## Delivery Backends

Uplink dispatches push notifications via the appropriate platform delivery channel:

- **FCM (Firebase Cloud Messaging)** — Android (Google ecosystem)
- **APNs (Apple Push Notification service)** — iOS (required by the platform; unavoidable)
- **UnifiedPush** — Android FOSS alternative to FCM; enables fully Google-free push delivery on Android

All three are configured by the server operator. Hivecom is not in the notification delivery path.

For Android, UnifiedPush support means operators can route notifications through any compatible distributor (e.g., ntfy, Gotify), removing the FCM dependency entirely. This makes a fully FOSS, Google-free push stack possible.

For iOS, APNs is unavoidable — Apple's platform does not permit alternative push delivery mechanisms. However, the APNs connection is made by the operator's Uplink instance using the operator's registered APNs credentials.

## Push Token Registration API

Called by the Orbit mobile client at login time. Associates a device's push token with the user's IRC account name. This endpoint is served by Uplink directly.

```
POST /push/register
Authorization: Bearer <JWT>

{
  "platform": "fcm" | "apns" | "unifiedpush",
  "token": "<device push token>",
  "endpoint": "<UnifiedPush distributor URL, if platform=unifiedpush>"
}
```

The JWT is verified against the domain's OIDC provider (the [Transponder](../02-components/04-transponder.md) role) if one is configured. Without an OIDC provider, account identity falls back to the NickServ account name from the IRC connection.

```
DELETE /push/register
Authorization: Bearer <JWT>
```

Called at logout. Removes the token for the authenticated account on this device.

## Token Store

Uplink maintains a small table mapping account names to registered device tokens, stored within its own database:

| Field | Description |
|-------|-------------|
| `account` | IRC account name (from OIDC `preferred_username` or NickServ account) |
| `platform` | `fcm`, `apns`, or `unifiedpush` |
| `token` | Platform-issued device push token |
| `endpoint` | UnifiedPush distributor URL (UnifiedPush only) |
| `registered_at` | Timestamp of registration |

A single user may have multiple registered tokens (phone, tablet, etc.). Uplink dispatches to all registered tokens for the target account and prunes any token that the delivery backend reports as expired or invalid.

## Risks

- **APNs certificate management**: APNs certificates expire and must be renewed. Operators need tooling or documentation for this. Mitigation: use APNs token-based auth (JWT, not certificates) where possible.
- **Token churn**: push tokens are invalidated when users reinstall the app or reset devices. Mitigation: prune invalid tokens on delivery failure; re-register on every login.
- **Operator burden**: configuring FCM and APNs credentials is non-trivial for small operators. Mitigation: clear documentation; UnifiedPush as a simpler alternative for Android-only communities.

## Dependencies

- The Uplink fork must be underway — push notifications are native to Uplink and cannot be backported to unmodified Ergochat.
- [Mobile Clients](02-mobile-clients.md) — push notifications are only meaningful when native mobile apps exist. The mobile client track and push notification implementation should be planned together.
- [Transponder](../02-components/04-transponder.md) — OIDC identity provider; without it, push token registration falls back to NickServ account identity.