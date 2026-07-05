# Push Delivery

The IRC server side of push notifications is native in stock Ergo: since v2.15.0, Ergo implements
the `draft/webpush` specification, so push subscription registration and transport are built into
the adopted server. Ergo already holds the state needed to detect notification-worthy events
(session state, online/offline status, DM recipients, channel membership) and dispatches push
events internally.

The delivery side - the relays and integration that carry Ergo's push events to each platform -
does not exist yet; this page specifies it. The notification model (why detection is
server-native, which events trigger a push, and the payload privacy policy) lives in
[Messaging](../02-architecture/10-messaging.md). The short version of the payload policy: a push
carries sender and channel only, never message content - content stays on the server and is
retrieved via `CHATHISTORY` on reconnect.

## Delivery Backends

The delivery layer carries Ergo's `draft/webpush` events to the appropriate platform delivery
channel:

- **FCM (Firebase Cloud Messaging)** - Android (Google ecosystem)
- **APNs (Apple Push Notification service)** - iOS (required by the platform; unavoidable)
- **UnifiedPush** - Android FOSS alternative to FCM; enables fully Google-free push delivery on Android
- **Web Push** - the PWA subscribes via the standard browser Push API and receives notifications through the same `draft/webpush` flow

All of these are configured by the server operator.

For Android, UnifiedPush support means operators can route notifications through any compatible
distributor (e.g., ntfy, Gotify), removing the FCM dependency entirely. This makes a fully FOSS,
Google-free push stack possible.

For iOS, APNs is unavoidable - Apple's platform does not permit alternative push delivery
mechanisms. However, the APNs connection is made by the operator's deployment using the
operator's registered APNs credentials.

## Push Token Registration API

Called by the Orbit mobile client at login time. Associates a device's push token with the user's
IRC account name. Ergo's `draft/webpush` support handles subscription registration natively; the
endpoint shape below illustrates the delivery layer's view of registration.

```
POST /push/register
Authorization: Bearer <JWT>

{
  "platform": "fcm" | "apns" | "unifiedpush",
  "token": "<device push token>",
  "endpoint": "<UnifiedPush distributor URL, if platform=unifiedpush>"
}
```

The JWT is verified against the domain's OIDC provider (the
[Transponder](../02-architecture/07-transponder.md) role) if one is configured. Without an OIDC
provider, account identity falls back to the NickServ account name from the IRC connection.

```
DELETE /push/register
Authorization: Bearer <JWT>
```

Called at logout. Removes the token for the authenticated account on this device.

## Token Store

The push subscription store maps account names to registered device tokens. Ergo persists
`draft/webpush` subscriptions natively; the delivery layer reads them to dispatch:

| Field | Description |
|-------|-------------|
| `account` | IRC account name (from OIDC `preferred_username` or NickServ account) |
| `platform` | `fcm`, `apns`, or `unifiedpush` |
| `token` | Platform-issued device push token |
| `endpoint` | UnifiedPush distributor URL (UnifiedPush only) |
| `registered_at` | Timestamp of registration |

A single user may have multiple registered tokens (phone, tablet, etc.). The delivery layer
dispatches to all registered tokens for the target account and prunes any token that the delivery
backend reports as expired or invalid.

Token churn is expected: push tokens are invalidated when users reinstall the app or reset
devices. The delivery layer prunes invalid tokens on delivery failure, and clients re-register on
every login. For APNs, token-based auth (JWT, not certificates) avoids certificate expiry and
renewal where possible.

## Cross-References

- [Messaging](../02-architecture/10-messaging.md) - the notification model and payload privacy policy
- [Uplink](01-uplink.md) - the IRC layer; push subscription/transport is native via `draft/webpush`
- [Identity](05-identity.md) - JWT verification for token registration
