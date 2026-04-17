# Glossary

This page defines the named components and core concepts used throughout the Orbit specification. Each entry covers what the component is, what it does, its MVP status, and a link to its detailed specification page.

## Named Services

### Ground Control

Ground Control is the IRC layer of an Orbit deployment - an [Ergochat](https://ergo.chat/) instance running IRCv3. It handles text messaging, presence, channel state, and media signaling. All persistent communication - messages, channel membership, message history, user identity - flows through Ground Control. It is a standard IRCv3 server: no Orbit-specific patches, no plugins, no knowledge of Satellite or Depot at the protocol level. Orbit clients and plain IRC clients connect identically.

-> See [Ground Control](../02-components/01-ground-control/01-overview.md) for the full specification.

### Satellite

Satellite is the real-time media layer - independent nodes running a LiveKit SFU that handle voice, video, streaming, and ephemeral in-session chat. Satellite nodes are completely decoupled from Ground Control: they do not need to know about IRC, channels, or message history. A Satellite node consists of two co-located components: the LiveKit SFU (WebRTC media routing) and a lightweight token service (HTTP API) that issues LiveKit-compatible JWTs for session authentication.

Satellite nodes can be server-operated (discovered via DNS SRV) or user-operated (Bring Your Own Node). Satellite can also be used entirely without Ground Control via a `satellite://` direct link.

-> See [Satellite](../02-components/02-satellite.md) for the full specification.

### Depot

Depot is the storage layer - an S3-compatible object store (MinIO, AWS S3, or equivalent) fronted by a thin HTTP upload API. It handles file uploads (images, video, audio, documents) and user avatars. Uploads flow through a pre-signed URL mechanism: the Orbit client requests a pre-signed URL from the upload API, uploads the file directly to S3, then posts the resulting public URL to the IRC channel as a PRIVMSG with metadata tags. Downloads are public and URL-gated - anyone with the URL can fetch the file.

-> See [Depot](../02-components/03-depot.md) for the full specification.

### Transponder

Transponder is a role, not a service. It refers to whatever OIDC-compliant identity provider the server operator deploys (e.g., Keycloak, Authentik, Authelia, Zitadel). Orbit components consume the provider via standard OpenID Connect Discovery - the operator configures a single OIDC issuer URL, and each component discovers endpoints, fetches signing keys (JWKS), and verifies identity tokens independently. Ground Control integrates via Ergochat’s `auth-script` mechanism through a thin auth-script bridge; Satellite and Depot verify JWTs directly against the provider’s published keys. Neither Ground Control nor Satellite needs to know which provider is in use.

Transponder is optional: Orbit deployments without an identity provider use Ergochat’s built-in NickServ/SASL for IRC authentication and degrade gracefully - voice and video still function, but all Satellite participants appear unverified.

-> See [Transponder](../02-components/04-transponder.md) for the full specification.

### Beacon

Beacon is a proposed self-hostable push notification relay for mobile Orbit clients. It connects to Ground Control as an IRC client, monitors for mentions and direct messages directed at offline users, and dispatches push notifications via FCM (Android), APNs (iOS), or UnifiedPush (FOSS alternative on Android). Beacon runs under the server operator's control - not Hivecom's. Without Beacon, mobile clients receive no push notifications, but all other functionality remains intact. Beacon is optional and has no effect on any non-mobile surface.

-> See [07-research/07-mobile-clients.md](../07-research/07-mobile-clients.md) for current research status.

## Client and Extension Concepts

### Orbit Client

The Orbit client is the application that composes the named services into a unified user experience. It comes in two forms: the [desktop client](../04-clients/01-desktop.md) (Tauri v2 + Vue, targeting Windows, macOS, and Linux) and the [web client / widget](../04-clients/02-web-app.md) (Vue, deployable as a full web app, PWA, or embeddable iframe widget). The Orbit client is the only component that has knowledge of all services - it speaks IRC to Ground Control, WebRTC to Satellite, and HTTP to Depot. Third-party clients that speak IRCv3 can interoperate with Ground Control without using the Orbit client at all.

### Orbit Extensions

Orbit extensions are client-side plugins for orbit-app that build on top of the Orbit tag namespace. They add UI and behavior to the desktop and web clients without modifying the core. Extensions may define their own `+orbit-ext/<name>/*` tag sub-namespace and may pair with IRC bots for server-side logic. Extensions are the correct place for features that are explicitly out of scope for the core - custom moderation UI, game integrations, event calendars, role color schemes.

### IRC Bots

IRC bots are standard IRC clients that connect to Ground Control and respond to events in channels. They are first-class citizens of the IRC ecosystem: any program that speaks IRCv3 can act as a bot. Bots handle server-side automation - moderation, logging, game stat tracking, notifications - without touching the Orbit core. When paired with an Orbit extension (client-side UI), an IRC bot is the Orbit equivalent of a Discord bot with slash commands and embeds.
