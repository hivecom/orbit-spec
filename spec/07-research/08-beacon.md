# Research: Beacon (Push Notification Relay)

Beacon is a self-hostable push notification relay for Orbit. Like [Transponder](../02-components/04-transponder.md), it is an optional, standalone component that extends Orbit's capabilities without modifying the IRC server. Unlike Transponder (which is a pure HTTP service), Beacon connects to [Ground Control](../02-components/01-ground-control/01-overview.md) as an IRC bot because it needs real-time access to the message stream - IRC is the natural event source for detecting mentions and DMs directed at offline users.

## What is Beacon?

Beacon is a small, self-hostable service that monitors Ground Control for events directed at offline users and fires push notifications via the appropriate platform delivery channel. It is deployed by the server operator and runs entirely under their control.

Beacon connects to [Ground Control](../02-components/01-ground-control/01-overview.md) as an IRC bot. It monitors events - mentions in channels, direct messages - and when it detects an event directed at a user who is not actively connected, it dispatches a push notification through the configured delivery backend:

- **FCM (Firebase Cloud Messaging)** - Android (Google ecosystem)
- **APNs (Apple Push Notification service)** - iOS (required by the platform; unavoidable)
- **UnifiedPush** - Android FOSS alternative to FCM; enables fully Google-free push delivery on Android

Beacon is **optional**. If a server operator does not deploy it, no push notifications are delivered. Everything else in Orbit continues to work. The [PWA](../04-clients/02-web-app.md) provides limited background notification capability via the Web Notifications API on supported browsers, which covers the non-native case for MVP.

## Why It's Needed

Mobile operating systems - iOS and Android - aggressively kill background processes. An app that is not in the foreground cannot maintain a persistent connection to Ground Control or poll for new messages without triggering OS-level termination. This is a hard platform constraint, not an Orbit limitation.

The only reliable mechanism for reaching an offline mobile user is a **push notification delivered via the OS push relay infrastructure** (FCM for Android, APNs for iOS). This requires a server-side component that:

1. Remains connected to Ground Control even when the user's device app is not running
2. Knows which users have registered which push tokens with which delivery endpoints
3. Can dispatch a push payload in response to IRC events (mentions, DMs)

Beacon is that component.

## Architecture

```
Ground Control (IRCv3)
        |
  [Beacon IRC bot]
        |
   event filter
   (mention? DM? target offline?)
        |
   dispatch
   /    |    \
FCM  APNs  UnifiedPush
```

1. **Beacon connects** to Ground Control as a standard IRC bot (SASL-authenticated, `+orbit/beacon` capability negotiated if defined).
2. **Beacon registers push tokens** - the mobile client, at install and login time, registers its FCM/APNs/UnifiedPush token with the Beacon HTTP endpoint. The token is associated with the user's IRC account name.
3. **Beacon monitors events** - on receiving a channel message with a mention or a private message directed at a registered user, Beacon checks whether that user is currently connected to Ground Control. If not (no active session), it dispatches a push notification.
4. **The mobile OS delivers** the notification, which wakes the app if needed to display the alert.

## Self-Hosting and iOS Constraint

Beacon runs under the **server operator's control**, not Hivecom's. This preserves the decentralized model - operators decide whether to enable push notifications for their community and configure their own FCM project credentials and APNs certificates.

For Android, UnifiedPush support means operators can route push notifications through any UnifiedPush-compatible distributor (e.g., ntfy, Gotify), removing the FCM dependency entirely. This makes a fully FOSS, Google-free push stack possible.

For iOS, APNs is unavoidable - Apple's platform does not permit alternative push delivery mechanisms. However, Beacon itself remains self-hosted. The APNs connection is made by the operator's Beacon instance using the operator's registered APNs credentials. Hivecom is not in the notification delivery path.

## MVP Status

Beacon is **not part of the MVP**. The PWA (see [Web App & PWA](../04-clients/02-web-app.md)) provides limited background notification via the Web Notifications API on supported desktop browsers. On mobile, PWA push notifications work on Android (Chrome) and iOS 16.4+ (Safari). This covers the majority of the mobile use case without a native app or Beacon.

Beacon becomes necessary when native mobile apps (see [Mobile Clients](07-mobile-clients.md)) ship and users expect reliable background notification delivery regardless of browser state.

## Cross-References

- [Mobile Clients](07-mobile-clients.md) - the track that Beacon unblocks
- [Transponder](../02-components/04-transponder.md) - architectural comparison (both are optional standalone services; Beacon uses an IRC bot, Transponder is a pure HTTP service)
- [Ground Control](../02-components/01-ground-control/01-overview.md) - the IRC server Beacon connects to as a bot
