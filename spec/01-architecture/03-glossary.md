# Glossary

This page defines the named components and core concepts used throughout the Orbit specification. Each entry covers what the component is, what it does, its MVP status, and a link to its detailed specification page.

## Named Services

### Uplink

Uplink is the IRC layer of an Orbit deployment - an adopted role, not software Orbit builds. Uplink is any stock IRCv3 server; [Ergo](https://ergo.chat/) is the reference implementation, run with no Orbit-specific patches. All persistent communication - messages, channel membership, message history, user identity - flows through Uplink. It is a standard IRCv3 server: no Orbit-specific plugins, no knowledge of Satellite or Depot at the protocol level. Orbit clients and plain IRC clients connect identically. There is no Uplink fork; Orbit runs the server stock and conforms to IRCv3, supporting whatever stock Ergo implements.

-> See [Uplink](../02-components/01-uplink/01-overview.md) for the full specification.

### Satellite

Satellite is the real-time media layer - a logical service that clients interact with via a single endpoint. It hosts multiple concurrent rooms for voice, video, streaming, and ephemeral in-session chat. Satellite is completely decoupled from Uplink: it does not need to know about IRC, channels, or message history.

Under the hood, a Satellite consists of one or more **nodes** (LiveKit SFU instances), each hosting multiple rooms. For the MVP, a Satellite is a single node. For scaled deployments, multiple nodes sit behind a gateway that routes requests transparently. Each node has two co-located components: the LiveKit SFU (WebRTC media routing) and a lightweight token service (HTTP API) that issues LiveKit-compatible JWTs for session authentication. In multi-node deployments, the token service role is absorbed by the gateway.

Satellites can be server-operated (discovered via DNS SRV) or user-operated (Bring Your Own Satellite). Satellite can also be used entirely without Uplink via a `satellite://` direct link.

-> See [Satellite](../02-components/02-satellite.md) for the full specification.

### Depot

Depot is the storage layer - an S3-compatible object store (MinIO, AWS S3, or equivalent) fronted by a thin HTTP upload API. It handles file uploads (images, video, audio, documents) and user avatars. Uploads flow through a pre-signed URL mechanism: the Orbit client requests a pre-signed URL from the upload API, uploads the file directly to S3, then posts the resulting public URL to the IRC channel as a PRIVMSG with metadata tags. Downloads are public and URL-gated - anyone with the URL can fetch the file.

-> See [Depot](../02-components/03-depot.md) for the full specification.

### Transponder

Transponder is a role, not a service. It refers to whatever OIDC-compliant identity provider the server operator deploys (e.g., Keycloak, Authentik, Authelia, Zitadel). Orbit components consume the provider via standard OpenID Connect Discovery - the operator configures a single OIDC issuer URL, and each component discovers endpoints, fetches signing keys (JWKS), and verifies identity tokens independently. Uplink verifies provider JWTs via the auth-script bridge (any provider/algorithm) or, for RS256/EdDSA/HMAC providers, Ergo's native `accounts.jwt-auth` (`IRCV3BEARER`); Satellite and Depot verify JWTs directly against the provider's published keys. Neither Uplink nor Satellite needs to know which provider is in use.

Transponder is optional: Orbit deployments without an identity provider use Ergochat's built-in NickServ/SASL for IRC authentication and degrade gracefully - voice and video still function, but all Satellite participants appear unverified.

-> See [Transponder](../02-components/04-transponder.md) for the full specification.



## Client and Extension Concepts

### Orbit Client

The Orbit client is the application that composes the named services into a unified user experience. It comes in two forms: the [desktop client](../04-clients/01-desktop.md) (Tauri v2 + Vue, targeting Windows, macOS, and Linux) and the [web client / widget](../04-clients/02-web-app.md) (Vue, deployable as a full web app, PWA, or embeddable iframe widget). The Orbit client is the only component that has knowledge of all services - it speaks IRC to Uplink, WebRTC to Satellite, and HTTP to Depot. Third-party clients that speak IRCv3 can interoperate with Uplink without using the Orbit client at all.

### Orbit Extensions

Orbit extensions are client-side plugins for the Orbit application that build on top of the Orbit tag namespace. They add UI and behavior to the desktop and web clients without modifying the core. Extensions may define their own `+orbit-ext/<name>/*` tag sub-namespace and may pair with IRC bots for server-side logic. Extensions are the correct place for features that are explicitly out of scope for the core - custom moderation UI, game integrations, event calendars, role color schemes.

### IRC Bots

IRC bots are standard IRC clients that connect to Uplink and respond to events in channels. They are first-class citizens of the IRC ecosystem: any program that speaks IRCv3 can act as a bot. Bots handle server-side automation - moderation, logging, game stat tracking, notifications - without touching the Orbit core. When paired with an Orbit extension (client-side UI), an IRC bot is the Orbit equivalent of a Discord bot with slash commands and embeds.

## IRC Capabilities in Use

This section lists the IRCv3 capabilities Orbit relies on and their availability in stock Ergo, along with status for capabilities IRC has not standardized yet.

| Capability | Purpose | Availability / Status |
|---|---|---|
| `sasl` | Authentication (PLAIN, SCRAM-SHA-256, ANONYMOUS) | Stable in Ergo |
| `message-tags` | Carry Orbit metadata on IRC messages | Stable in Ergo |
| `batch` | Group related messages (history, etc.) | Stable in Ergo |
| `chathistory` | Server-side message history replay | Stable in Ergo |
| `away-notify` | Real-time away status updates | Stable in Ergo |
| `extended-monitor` | Track online/offline state for a list of nicks | Stable in Ergo |
| `draft/pre-away` | Signal impending disconnect before it happens | Stable in Ergo |
| `draft/read-marker` | Server-side read state synced across devices | Stable in Ergo |
| `draft/message-redaction` | Server-enforced message retractions (`REDACT` command) | Shipped/stable in Ergo; only advertised when `history.retention.allow-individual-delete` is enabled |
| `draft/metadata-2` | Native key/value store per user and channel (avatars, display names, status) | Shipped/stable in Ergo (2.17.0+); requires the `accounts.metadata` config block |
| `draft/webpush` | Native push notification delivery | Stable in Ergo (2.15.0+) |
| In-place message editing | Edited message state | Not yet standardized in IRC; handled client-side |
| Full-text search | Search over retained channel history | Via Ergo history backends + external indexer |

## Core Concepts

### DMs and Group DMs

Direct messages are standard IRC `PRIVMSG` to a nickname. The server stores DM history using operator-configured retention - the same model as channel history. When end-to-end encryption is active, the server stores ciphertext with the same retention and delivery model - it just can't read the content. Offline delivery for registered users is guaranteed via always-on mode (MUST be enabled). Group DMs are invite-only private channels (`+s +i`) with standard retention. See [DMs](../02-components/01-uplink/03-dms.md) for the full storage model and E2E interaction.

### Message Retractions

Message retractions in the MVP use the IRC-standard `REDACT` command via `draft/message-redaction` (shipped and stable in Ergo). This is server-enforced, not a client-only tag. Orbit clients render a tombstone in place of the retracted message. IRC clients that implement the cap see the message removed. IRC clients without the cap receive a server NOTICE fallback: `*** alice retracted a message ***`. Note that Ergo only advertises the capability when `history.retention.allow-individual-delete: true`; see [Uplink Overview - Message Retractions](../02-components/01-uplink/01-overview.md#message-retractions-and-replies) for the operator requirement.

### Message Storage

Channel message history uses operator-configured retention. Ergo supports this today: operators configure how long history is kept per channel. There is no mandate to store everything forever. DM history uses the same operator-configured retention model. When E2E encryption is active, the server stores ciphertext with the same retention and delivery mechanics - it just can't read the content. See [DMs](../02-components/01-uplink/03-dms.md) for the full two-tier storage model.

### Threads

Threads are implemented as client-managed IRC sub-channels with a naming convention (e.g., `#channel/t-<msgid-prefix>`) and a signaling tag in the parent channel. The Uplink server has no concept of a thread - it sees a channel like any other. Orbit clients render sub-channels as a thread panel. IRC clients can join thread channels directly and participate normally.

### User Metadata

User metadata - avatars, display names, and presence status - is stored and distributed via `draft/metadata-2` (stable in Ergo 2.17.0+; enabled per deployment via the `accounts.metadata` config block). Orbit uses the following keys:

- `avatar` - a URL pointing to the user's avatar image (Orbit uses a Depot URL); IRCv3 quasi-standard key, unprefixed for interop
- `display-name` - the user's preferred display name
- `orbit.status` - the user's presence/status string (vendor-prefixed; no standard equivalent)

Clients subscribe to these keys and receive live push updates when they change. IRC clients that implement `draft/metadata-2` can read and set these keys via `METADATA GET` / `METADATA SET`.
