# Glossary

Shared vocabulary for the Orbit specification. Each entry says what the term
means and links to the page that specifies it.

## Named Services

### Uplink

Uplink is the IRC layer of an Orbit deployment: any stock IRCv3 server, with
[Ergo](https://ergo.chat/) as the reference implementation, run with no
Orbit-specific patches. All persistent communication - messages, channel
membership, history, user identity - flows through Uplink.

See [Uplink](02-architecture/03-uplink.md).

### Satellite

Satellite is the real-time media layer: a logical service that clients reach
through a single endpoint, hosting concurrent rooms for voice, video,
streaming, and ephemeral in-session chat. It's fully decoupled from Uplink and
has no knowledge of IRC, channels, or message history.

Under the hood a Satellite is one or more nodes, each a LiveKit SFU instance
paired with a token service that issues session credentials. A Satellite is a
single node; scaled deployments put multiple nodes behind a gateway that
routes requests transparently, and the gateway absorbs the token service
role. Satellites can be server-operated (discovered via DNS) or user-operated
(Bring Your Own Satellite), and can be used entirely without Uplink via a
`satellite://` link.

See [Satellite](02-architecture/05-satellite.md).

### Depot

Depot is the storage layer: a thin policy-and-signing gateway in front of an
S3-compatible backend or a local filesystem. It verifies credentials, applies
quotas and policy, and issues upload URLs; the bytes live in the backing
store. Downloads are URL-gated.

See [Depot](02-architecture/06-depot.md).

### Transponder

Transponder refers to whatever OIDC-compliant identity provider the operator
deploys (Keycloak, Authentik, Authelia, Zitadel, or any other). Components
discover it via standard OpenID Connect Discovery and verify its identity
tokens independently against the provider's published keys. Transponder is
optional: deployments without one use Ergo's built-in NickServ/SASL for IRC
authentication, and Satellite participants appear unverified.

See [Transponder](02-architecture/07-transponder.md).

## Client and Extension Concepts

### Orbit Client

The application that composes the named services into a unified experience.
It ships as the desktop and mobile client (Tauri v2 + Vue) and the web client
(Vue, deployable as a full web app, PWA, or embedded client). The Orbit
client is the only component with knowledge of all services: it speaks IRC to
Uplink, WebRTC to Satellite, and HTTP to Depot. Third-party IRCv3 clients
interoperate with Uplink without it.

See [Clients](02-architecture/11-clients.md).

### Embedded Client (formerly "widget")

The Orbit web client running in a constrained, single-channel presentation
inside an iframe on an external site. Earlier drafts of this specification
called it the widget.

See [Clients](02-architecture/11-clients.md).

### Orbit Extensions

Client-side plugins for the Orbit application, built on the Orbit tag
namespace. They add UI and behavior to the clients without modifying the
core, may define their own tag sub-namespace, and may pair with IRC bots for
server-side logic.

See [Extensions](02-architecture/15-extensions.md).

### IRC Bots

Standard IRC clients that connect to Uplink and respond to events in
channels. Any program that speaks IRCv3 can act as a bot - moderation,
logging, notifications, game integrations - without touching the Orbit core.
Paired with an Orbit extension for client-side UI, a bot is the Orbit
equivalent of a Discord bot with slash commands and embeds.

See [Extensions](02-architecture/15-extensions.md).

## Core Concepts

### DMs and Group DMs

Direct messages are standard IRC `PRIVMSG` to a nickname. The server stores
DM history with operator-configured retention, the same model as channels.
When end-to-end encryption is active, the server stores ciphertext with the
same retention and delivery mechanics - it just can't read the content.
Offline delivery for registered users is guaranteed via always-on mode. Group
DMs are invite-only private channels.

See [Messaging](02-architecture/10-messaging.md).

### Message Retractions

Retractions use the IRC-standard `REDACT` command, a server-enforced
operation. Orbit clients render a tombstone in place of the retracted
message; IRC clients that implement the capability see the message removed,
and clients without it receive a notice fallback.

See [Uplink](02-architecture/03-uplink.md).

### Message Storage

Channel and DM history use operator-configured retention. Operators choose
how long history is kept; there is no mandate to store everything forever.
E2E-encrypted content is stored as ciphertext under the same retention.

See [Messaging](02-architecture/10-messaging.md).

### Threads

Threads are client-managed IRC sub-channels with a naming convention and a
signaling tag in the parent channel. The server has no concept of a thread -
it sees an ordinary channel. Orbit clients render a thread panel; IRC clients
can join the thread channel directly and participate normally.

See [Uplink](02-architecture/03-uplink.md).

### User Metadata

Avatars, display names, and presence status, stored and distributed via the
IRCv3 metadata extension. Metadata is user-set and cosmetic - never an
identity signal.

See [Messaging](02-architecture/10-messaging.md).

## IRC Capabilities

The IRCv3 capabilities Orbit relies on, and their status in stock Ergo, are
listed in [Uplink - Required IRCv3 Extensions](02-architecture/03-uplink.md#required-ircv3-extensions).
