# Push Delivery

Push reaches Orbit clients over two paths in very different states. The native path is standard
Web Push: stock Ergo implements the
[`draft/webpush`](https://github.com/ircv3/ircv3-specifications/pull/471) extension (v2.15.0+),
covering event detection, subscription registration, and delivery to any Web Push endpoint. It
works today, and the remaining work is client-side. The relay path covers the proprietary push
services that native mobile apps are required to use (APNs on iOS, FCM on Google-ecosystem
Android). That layer doesn't exist yet, and nothing needs it until native mobile clients exist.
The notification model (which events trigger a push, why detection is server-native) lives in
[Messaging](../02-architecture/10-messaging.md).

## Native Path: Web Push

Ergo detects notification-worthy events itself (it already holds session state, online/offline
status, DM recipients, and channel membership) and POSTs the notification directly to the
subscription endpoint per RFC 8030. No Orbit component sits in this path. Push is disabled by
default; operators enable it with `webpush.enabled: true` in the Ergo config.

The client side:

1. Negotiate the `draft/webpush` capability at connect time.
2. Obtain a push subscription. In a browser this comes from the service worker's Push API; the
   PWA plumbing is in [Clients](08-clients.md#pwa-configuration).
3. Register it with the server via the `WEBPUSH REGISTER` command, passing the endpoint and the
   subscription's key material.
4. Decrypt and render incoming pushes in the service worker, and re-register when the browser
   rotates the subscription.

The payload of each push is exactly one IRC message line (the triggering `PRIVMSG`), encrypted to
the subscription keys per RFC 8291. The push service in the middle relays ciphertext it can't
read; only the subscribed client holds the decryption keys. What the notification displays is the
client's decision at render time - Orbit shows sender and channel, per the notification policy in
[Messaging](../02-architecture/10-messaging.md).

This path serves:

- **The web app / PWA on every platform.** Desktop browsers, Android, and installed PWAs on iOS
  (16.4+), where Safari's push service speaks the same protocol.
- **A future native Android client via UnifiedPush.** Distributors (ntfy, Gotify) expose Web
  Push-compatible endpoints, so Ergo delivers to them directly. Google-free push with no
  server-side additions.

The desktop client doesn't use push at all: it holds a persistent IRC connection, and Ergo's
always-on mode covers delivery while it's closed.

## Relay Path: FCM and APNs

Native mobile apps can't receive Web Push directly. Apple requires APNs for iOS apps, and Android
apps in the Google ecosystem use FCM. Carrying Ergo's push events to those services takes a
relay: a small operator-deployed service holding the platform credentials. This component doesn't
exist yet; this section specifies its surface.

Unlike the native path, FCM and APNs can read what passes through them, so the relay enforces the
minimization policy from [Messaging](../02-architecture/10-messaging.md): sender and channel
only, never message content. Content stays on the server and is retrieved over `CHATHISTORY` on
reconnect.

For iOS, the APNs connection is made by the operator's deployment using the operator's registered
APNs credentials. Token-based auth (JWT, not certificates) avoids certificate expiry and renewal
where possible.

### Push Token Registration API

Called by the Orbit mobile client at login time. Associates a device's push token with the user's
IRC account name.

```
POST /push/register
Authorization: Bearer <JWT>

{
  "platform": "fcm" | "apns",
  "token": "<device push token>"
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

### Token Store

The relay maps account names to registered device tokens:

| Field | Description |
|-------|-------------|
| `account` | IRC account name (from OIDC `preferred_username` or NickServ account) |
| `platform` | `fcm` or `apns` |
| `token` | Platform-issued device push token |
| `registered_at` | Timestamp of registration |

A single user may have multiple registered tokens (phone, tablet, etc.). The relay dispatches to
all registered tokens for the target account and prunes any token the delivery backend reports as
expired or invalid.

Token churn is expected: push tokens are invalidated when users reinstall the app or reset
devices. The relay prunes invalid tokens on delivery failure, and clients re-register on every
login.

## Cross-References

- [Messaging](../02-architecture/10-messaging.md) - the notification model and display policy
- [Uplink](01-uplink.md) - the IRC layer; Web Push subscription and transport are native
- [Clients](08-clients.md) - PWA and service worker configuration
- [Identity](05-identity.md) - JWT verification for the relay registration API
