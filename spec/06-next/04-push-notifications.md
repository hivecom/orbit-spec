# Push Notifications

The IRC server side of push notifications is **native in stock Ergo**. Since v2.15.0, Ergo implements the `draft/webpush` specification, so push subscription registration and transport are built into the adopted server - this is no longer a fork capability. Ergo already holds the state needed to detect notification-worthy events (session state, online/offline status, DM recipients, channel membership) and dispatches push events internally.

What remains as NEXT work is the **delivery side**: the relays and integration that carry Ergo's push events to each platform - FCM (Android), APNs (iOS), UnifiedPush (Google-free Android), and Web Push for the PWA - plus the privacy-first payload policy (sender and channel only, never content). The server support exists today; the platform delivery wiring is what Orbit builds.

Cross-references:
- [Uplink](../02-components/01-uplink/01-overview.md) - the IRC layer; push subscription/transport is native via `draft/webpush`
- [Transponder](../02-components/04-transponder.md) - OIDC identity model used for JWT-verified push token registration
- [Mobile Clients](02-mobile-clients.md) - the track that push notification delivery unblocks

## Why Server-Native Detection

Notification detection belongs inside the IRC server, and stock Ergo provides it for two reasons:

**It avoids duplicating knowledge.** Online/offline state is core IRC session state - Ergo knows immediately when a user has no active sessions. Nickname highlight matching is something the server does natively. DM recipients are known at the moment of delivery. A separate service would need to re-derive all of this from the outside, which is redundant and fragile.

**It can observe DMs.** In standard IRC, a `PRIVMSG` to a nickname is delivered only to that nickname - no other connected client sees it. A separate bot-based approach could monitor channel mentions but could never see DMs directed at other users without IRC operator-level privileges, carrying serious privacy implications. Native `draft/webpush` integration has no such constraint.

## What Push Notifications Cover

Ergo detects the following events and dispatches push notifications for offline users:

- **DM**: a `PRIVMSG` to a nickname where the target user has zero active sessions
- **Mention**: a channel `PRIVMSG` that contains the nick of an offline channel member

No message content is included in the push payload. The notification says only *"New message from alice"* or *"alice mentioned you in #gaming"*. Message content stays on the server and is retrieved by the client via `CHATHISTORY` on reconnect. Sensitive content never passes through FCM, APNs, UnifiedPush, Web Push, or any third-party relay infrastructure. This privacy-first payload policy is enforced by the delivery layer Orbit builds.

## Delivery Backends

The delivery layer carries Ergo's `draft/webpush` events to the appropriate platform delivery channel:

- **FCM (Firebase Cloud Messaging)** - Android (Google ecosystem)
- **APNs (Apple Push Notification service)** - iOS (required by the platform; unavoidable)
- **UnifiedPush** - Android FOSS alternative to FCM; enables fully Google-free push delivery on Android
- **Web Push** - the PWA subscribes via the standard browser Push API and receives notifications through the same `draft/webpush` flow

All of these are configured by the server operator.

For Android, UnifiedPush support means operators can route notifications through any compatible distributor (e.g., ntfy, Gotify), removing the FCM dependency entirely. This makes a fully FOSS, Google-free push stack possible.

For iOS, APNs is unavoidable - Apple's platform does not permit alternative push delivery mechanisms. However, the APNs connection is made by the operator's deployment using the operator's registered APNs credentials.

## Push Token Registration API

Called by the Orbit mobile client at login time. Associates a device's push token with the user's IRC account name. Ergo's `draft/webpush` support handles subscription registration natively; the endpoint shape below illustrates the delivery layer's view of registration.

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

The push subscription store maps account names to registered device tokens. Ergo persists `draft/webpush` subscriptions natively; the delivery layer reads them to dispatch:

| Field | Description |
|-------|-------------|
| `account` | IRC account name (from OIDC `preferred_username` or NickServ account) |
| `platform` | `fcm`, `apns`, or `unifiedpush` |
| `token` | Platform-issued device push token |
| `endpoint` | UnifiedPush distributor URL (UnifiedPush only) |
| `registered_at` | Timestamp of registration |

A single user may have multiple registered tokens (phone, tablet, etc.). The delivery layer dispatches to all registered tokens for the target account and prunes any token that the delivery backend reports as expired or invalid.

## Risks

- **APNs certificate management**: APNs certificates expire and must be renewed. Operators need tooling or documentation for this. Mitigation: use APNs token-based auth (JWT, not certificates) where possible.
- **Token churn**: push tokens are invalidated when users reinstall the app or reset devices. Mitigation: prune invalid tokens on delivery failure; re-register on every login.
- **Operator burden**: configuring FCM and APNs credentials is non-trivial for small operators. Mitigation: clear documentation; UnifiedPush as a simpler alternative for Android-only communities.

## Dependencies

- Stock Ergo (v2.15.0+) provides `draft/webpush` server support - no fork or server changes are required. The NEXT work is the platform delivery wiring (FCM/APNs/UnifiedPush relays and the PWA Web Push integration).
- [Mobile Clients](02-mobile-clients.md) - push notifications are only meaningful when native mobile apps exist. The mobile client track and the push delivery layer should be planned together.
- [Transponder](../02-components/04-transponder.md) - OIDC identity provider; without it, push token registration falls back to NickServ account identity.
