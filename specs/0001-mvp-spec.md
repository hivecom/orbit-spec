# Orbit - MVP Specification

## 1. Overview

Orbit is a decentralized, open-source communication platform built by Hivecom. It targets communities, gaming groups, and privacy-conscious users who want an alternative to Discord without surrendering control of their data or infrastructure. Orbit is designed from the ground up to be self-hostable, lightweight, and built on open standards rather than proprietary protocols.

The system is split into multiple named layers. **Ground Control** is the IRC layer - an Ergochat instance running IRCv3, handling text messaging, presence, channel state, and signaling. **Satellite** is the real-time media layer - independent media nodes (SFU instances) that handle voice, video, and streaming. **Depot** is the storage layer - S3-compatible object storage (MinIO, S3, or equivalent) for file uploads and avatars. Orbit itself is the client application (desktop and web widget) that ties these layers together into a cohesive experience. A fourth named component, **Transponder**, is an optional standalone identity service that bridges IRC authentication to Satellite nodes via signed tokens - it is not part of the MVP but is designed as the first post-MVP addition (see [Spec 0002 §5](./0002-research-roadmap.md#5-track-federation)).

The goal of the MVP is to ship a working product - not a prototype, not a demo. That means text chat with history, group voice via Satellite nodes, an anonymous web widget for embedding on external sites, and a lightweight desktop client that doesn't eat 500 MB of RAM at idle. Every component must be functional enough for a small community to use daily. This document is scoped strictly to the MVP. Advanced features - gaming overlays, Media over QUIC transport, Leptos/WASM rewrites, federation, mobile clients, and end-to-end encryption - are explicitly deferred to the research roadmap ([Spec 0002](./0002-research-roadmap.md)). If a feature isn't in this document, it's not in the MVP.

## 2. Design Philosophy

**Orbit is a transport layer and client, not an application platform.**

The core handles four things: text chat, real-time media, identity, and client UX. Everything else - calendars, events, custom moderation workflows, rich permission systems, game integrations - is an **extension**. **Orbit extensions** are client-side plugins for the orbit-app that build on top of the Orbit tag namespace. **IRC bots** handle server-side automation as first-class citizens of the IRC ecosystem. Both extend Orbit at the edges without touching the core. This is an opinionated choice. Orbit stays thin so it stays fast and maintainable.

**Orbit is a layer on top of existing IRC.** Any IRCv3 server that supports message tags can become Orbit-enabled. Two users running Orbit on any compliant IRC network can use Satellite for voice - even if the server operator hasn't configured any official media nodes. Users just bring their own.

**Components are independent.** Ground Control, Satellite, Transponder, and Depot have no runtime dependencies on each other. Ground Control is a stock IRC server - it doesn't know Satellite exists. Satellite is a media server - it doesn't know IRC exists. Transponder bridges identity between them but neither requires it to function. Depot stores files and answers to no one. The Orbit client is the only thing that composes these components into a unified experience - but it doesn't require all of them. Connect to just Ground Control and you have IRC chat. Connect to just a Satellite with a join key and you have voice. Any other client - a web page, a game, a bot - can compose a different subset of the same components using the same interfaces. The architecture is a set of independent services, not a coupled stack.

**Opinionated simplicity drives every design decision.** Permissions use IRC's built-in channel modes - `+o`, `+v`, `+b` - and nothing more. There is no custom role system, no role colors, no granular permission overrides in the core. Media is handled through independent Satellite nodes that users can self-host (Bring Your Own Node). There is no centralized orchestrator bot mediating connections. If a community needs richer functionality, they build an extension. Complexity belongs at the edges, not in the core.

## 3. Architecture Overview

```
┌────────────────────────────────────────────────────────────────────┐
│                          CLIENT LAYER                              │
│                                                                    │
│   ┌────────────────┐  ┌────────────────┐  ┌─────────────────────┐  │
│   │ Orbit Desktop  │  │ Orbit Web      │  │ Third-party IRC     │  │
│   │ Tauri + Svelte │  │ Client (Svelte)│  │ WeeChat, irssi, ... │  │
│   └───────┬────────┘  └───────┬────────┘  └──────────┬──────────┘  │
│           │                   │                      │             │
└───────────┼───────────────────┼──────────────────────┼─────────────┘
            │ IRC/WS            │ IRC/WS               │ IRC/TLS
            │                   │                      │
┌───────────┼───────────────────┼──────────────────────┼─────────────┐
│           │                   │                      │             │
│   ┌────────────────────────────────────────────────────────┐       │
│   │           GROUND CONTROL - Ergochat (IRCv3)            │       │
│   │     Text, presence, signaling, channel state           │       │
│   └────────────────────────────────────────────────────────┘       │
│                                                                    │
│   ┌────────────────────────────────────────────────────────┐       │
│   │         SATELLITE NODES - Independent Media Services   │       │
│   │     SFU (LiveKit) + token service per node             │       │
│   │     Server-operated (official) or user-operated (BYON) │       │
│   └────────────────────────────────────────────────────────┘       │
│                                                                    │
│   ┌────────────────────────────────────────────────────────┐       │
│   │         DEPOT - S3-compatible Storage (MinIO / S3)     │       │
│   │     File uploads, avatars                              │       │
│   └────────────────────────────────────────────────────────┘       │
│                                                                    │
│                         SERVER LAYER                               │
└────────────────────────────────────────────────────────────────────┘
```

**Discovery flow:** When an Orbit client connects to a domain, it resolves DNS SRV records to discover available services - Ground Control (IRC), Satellite nodes, Depot, and (post-MVP) Transponder. DNS is the primary discovery mechanism because it works independently of any running service: a domain can advertise Satellite nodes without running IRC, or IRC without Satellite. The client resolves `_satellite._tcp.example.com` and queries each discovered node's metadata endpoint for name, region, and capacity. Users can also configure their own Satellite node URL in Orbit's settings (BYON).

**Session flow:** When a user starts a voice session, the client picks a Satellite node (server-advertised or BYON), requests a session token directly from that node's HTTP API, then posts a `+orbit/sat-invite` TAGMSG to the channel. Other Orbit users see "Voice session active" and can join by connecting to the same node. Pure IRC clients see nothing - client-only tags are silently ignored.

**There is no orchestrator bot.** Satellite nodes are independent services. Clients talk to them directly. Ground Control handles text transport and signaling only.

## 4. Ground Control - IRC (IRCv3)

### 4.1 Why IRCv3

| Property                     | Benefit                                                              |
|------------------------------|----------------------------------------------------------------------|
| Ultra-low resource footprint | Ergochat serves thousands of connections on minimal hardware         |
| Native WebSocket support     | Browser clients connect without a separate gateway                   |
| Message tags                 | Rich metadata transport without polluting message content            |
| Battle-tested protocol       | 30+ years of real-world usage; failure modes are well understood     |
| Backward compatibility       | Users can connect with WeeChat, irssi, or any IRCv3 client          |
| Built-in federation model    | Server-to-server protocol exists (deferred to v0.2+)                |
| Chathistory extension        | Server-side message history with standardized retrieval              |

IRCv3 gives us a text and signaling transport that works today, scales vertically to the community sizes we're targeting in the MVP, and doesn't lock us into a proprietary protocol.

### 4.2 Required IRCv3 Extensions

| Extension          | Purpose                                                        |
|--------------------|----------------------------------------------------------------|
| `message-tags`     | Transport media signaling and rich metadata                    |
| `batch`            | Group related messages (history playback, netsplit recovery)   |
| `chathistory`      | Server-side message history retrieval                          |
| `server-time`      | Accurate timestamps on all messages                            |
| `labeled-response` | Correlate requests with responses for async client operations  |
| `sasl`             | Authentication (PLAIN and SCRAM over TLS)                      |
| `message-ids`      | Unique message identifiers for edits, deletes, and reactions   |
| `WebSocket`        | Native WebSocket transport on Ergochat's listener              |

### 4.3 Orbit Tag Namespace

WebRTC session descriptions, Satellite invites, and metadata are transported as IRCv3 `TAGMSG` messages with client-only tags (prefixed with `+`). Orbit clients parse these tags. Pure IRC clients ignore them - no noise, no garbage in chat.

| Tag                        | Payload                                  | Direction          |
|----------------------------|------------------------------------------|--------------------|
| `+orbit/sat-invite`        | Node URL + room ID + metadata (base64 JSON) | Client → Channel |
| `+orbit/sat-leave`         | Room ID                                  | Client → Channel   |
| `+orbit/sat-status`        | Node health/capacity (base64 JSON)       | Client → Channel   |
| `+orbit/sdp-offer`         | SDP (base64, for P2P calls)              | Client → Client    |
| `+orbit/sdp-answer`        | SDP (base64, for P2P calls)              | Client → Client    |
| `+orbit/ice-candidate`     | ICE candidate (JSON, base64)             | Client → Client    |
| `+orbit/msg-edit`          | `target-msgid` + new content             | Client → Channel   |
| `+orbit/msg-delete`        | `target-msgid`                           | Client → Channel   |
| `+orbit/file-name`         | Original filename                        | Client → Channel   |
| `+orbit/file-size`         | File size in bytes                       | Client → Channel   |
| `+orbit/file-type`         | MIME type                                | Client → Channel   |

Base64 encoding is used because IRC message tags have restricted character sets. Payloads that exceed a single IRC message (512/4096 byte limit depending on config) are split across a `batch`.

### 4.4 Ergochat Configuration

Key configuration points for an Orbit-compatible Ergochat instance:

- **WebSocket listener**: Enabled on a dedicated port (e.g., `wss://irc.example.com:6698/ws`).
- **History storage**: Enabled with configurable per-channel retention (default: 7 days or 10,000 messages, whichever comes first).
- **SASL**: Required for registered users. SASL PLAIN and SCRAM-SHA-256 over TLS.
- **Client-only tag allowlist**: Ergochat relays all `+`-prefixed tags by default per IRCv3 spec. No special configuration needed, but the server should enforce maximum tag size limits.
- **Connection limits**: Per-IP connection limits configured to prevent abuse. Browser-based web clients connect directly, so limits should accommodate multiple simultaneous guest connections from the same IP (e.g., shared NAT or multiple browser tabs).
- **Nickname reservation**: The `guest-` prefix MUST be reserved at the NickServ level. Ergochat supports nickname reservation patterns - configure it to reject registration of any nickname starting with `guest-`. This prevents collision between registered users and anonymous widget guests.

### 4.5 Mapping IRC Primitives to Orbit Concepts

#### Channels

IRC channels are flat - there is no hierarchy, no categories, no sub-channels. Orbit treats them as-is. Channels are presented to the user as a list, with no client-side interpretation of naming conventions. If a community wants to organize channels by prefix (e.g., `#gaming-strategy`, `#gaming-lfg`), they are free to do so, but Orbit does not parse, enforce, or render any convention as a hierarchy. This keeps the client honest to the protocol and avoids ambiguity with existing IRC channels that contain dots, dashes, or other separators.

For real-time session chat (quick callouts during voice, links shared during a screen share), Satellite provides its own ephemeral chat via LiveKit data channels (see [Section 5.1](#51-what-is-a-satellite-node)). This is intentionally separate from IRC - ephemeral messages are not persisted, not searchable, and not visible to IRC clients. Persistent, historical chat belongs in Ground Control. Throwaway, in-session chat belongs in Satellite.

#### Message Editing and Deletion

- Each message has a unique ID assigned by Ergochat (via the `message-ids` extension).
- To edit: the client sends a PRIVMSG with a `+orbit/msg-edit` tag referencing the original message ID, with the new content as the message body.
- To delete: the client sends a TAGMSG with a `+orbit/msg-delete` tag referencing the original message ID.
- The Orbit client renders edits inline (with an "edited" indicator) and removes deleted messages (with a tombstone for moderator audit).
- Pure IRC clients see edit messages as a new message prefixed with `[edit]` and delete notifications as `[deleted message <id>]`. Not beautiful, but functional.

**Identity Verification for Edits and Deletes**

The `+orbit/*` message tags are client-only tags - any IRC client can send any value. There is no server-side enforcement of who can edit or delete which messages, because Ergochat does not interpret Orbit-specific tags. The mitigation is **client-side enforcement using the server-asserted `account-tag`**.

The IRCv3 `account-tag` is set by the IRC server based on the sender's authenticated SASL session. It cannot be forged by clients. Orbit clients MUST use `account-tag` as the authoritative source of message authorship and MUST enforce the following rules:

- **Edits**: An `+orbit/msg-edit` MUST be accepted only if the sender's `account-tag` matches the `account-tag` of the original message being edited.
- **Deletes**: An `+orbit/msg-delete` MUST be accepted only if the sender's `account-tag` matches the original message's `account-tag`, OR the sender has operator privileges (`+o`) in the channel.
- **Unverified senders**: If a message carrying `+orbit/msg-edit` or `+orbit/msg-delete` has no `account-tag` (i.e., the sender is not authenticated), the edit or delete MUST be silently ignored.

This is a known architectural trade-off of building on client-only IRC tags. The security boundary is at the client, not the server. Orbit clients that do not implement these checks are non-compliant and may expose users to spoofed edits or deletions.

#### Permissions

Orbit uses IRC's built-in channel modes. Period.

| IRC Mode | Role            | Capabilities                                      |
|----------|-----------------|---------------------------------------------------|
| `+o`     | Operator        | Kick, ban, manage channel settings, manage messages |
| `+v`     | Voiced          | Speak in moderated channels, upload files          |
| `+b`     | Banned          | Cannot join or speak in the channel                |
| (none)   | Default user    | Read and send messages in unmoderated channels     |

There are no custom roles, no role colors, no granular permission overrides. This is an opinionated decision. IRC has a proven, battle-tested permissions model. It covers the needs of the vast majority of communities.

If you need more - role hierarchies, per-channel upload limits, auto-mod rules, custom role colors - build an extension. An IRC bot connected to Ground Control can enforce arbitrarily complex rules by monitoring channel events and acting on them. An Orbit client extension can add UI for complementary features in the desktop client (post-MVP). But the core stays simple.

**Identity Display**

Orbit clients MUST clearly distinguish between authenticated and unauthenticated users in all contexts - chat messages, user lists, voice sessions, and DMs. The IRCv3 `account-tag` is the source of truth. Users with a verified `account-tag` SHOULD be displayed with their account name and a visual indicator of authentication (e.g., a badge, icon, or distinct styling). Users without an `account-tag` (unauthenticated, guest, or connecting from a client that doesn't support SASL) MUST be visually distinguished as unverified. This is a gap in traditional IRC clients that Orbit explicitly addresses - identity verification and display is a core client responsibility, not an optional feature.

#### File Sharing

File sharing uses Depot (S3-compatible object storage) as a public content store. The flow:

1. User selects a file in the Orbit client.
2. Client requests a pre-signed upload URL from Depot's upload API endpoint (a thin HTTP service in front of the S3 backend). The upload API enforces rate limits per IP and maximum file size.
3. Client uploads the file directly to S3 via the pre-signed URL.
4. On success, the client posts a PRIVMSG containing the public file URL and metadata as message tags (`+orbit/file-name`, `+orbit/file-size`, `+orbit/file-type`).
5. The Orbit client renders an inline preview (images, audio, video) or a download card.
6. Pure IRC clients see a plain URL.

**Downloads are public.** Anyone with the URL can fetch the file directly from Depot. The URL is the access control - if you don't want someone to access a file, don't share the URL. This is the same model as Imgur, public S3 buckets, or paste services.

**Uploads are rate-limited, not identity-gated (for the MVP).** The upload API enforces per-IP rate limits and a maximum file size (configurable by the server operator). Per-user quotas, upload authentication tied to IRC accounts, and file deletion by uploaders are deferred to post-MVP. Files are immutable once uploaded - server operators can purge files via S3 admin tools.

Anonymous web users (guests) cannot upload files; the Depot upload API rejects requests from unauthenticated users.

## 5. Satellite - Real-Time Media

### 5.1 What Is a Satellite Node

A Satellite node is an independent real-time service that handles voice, video, streaming, and **ephemeral chat**. Under the hood, it's an SFU (LiveKit for the MVP) with a lightweight token service for authentication. Satellite nodes are completely decoupled from Ground Control - they don't need to know about IRC, channels, or message history. They handle real-time sessions only.

Satellite sessions support a built-in ephemeral text chat via LiveKit's data channels. This chat is **not persisted** - when the session ends, the messages are gone. It exists for in-session coordination: quick callouts during a voice call, links shared during a screen share, reactions during a stream. Persistent, searchable, historical chat lives in Ground Control (IRC). Ephemeral, throwaway chat lives in Satellite. The two are architecturally distinct and intentionally so.

A Satellite node consists of two components:

- **SFU (LiveKit)**: Handles WebRTC media - audio/video forwarding, bandwidth adaptation, STUN/TURN integration - and data channels for ephemeral session chat.
- **Token service**: A small HTTP API co-located with the SFU that issues LiveKit-compatible JWTs for session authentication.

### 5.2 Node Discovery

DNS is the primary discovery mechanism for Satellite nodes. This is an intentional architectural choice: DNS works independently of any running service, requires no modification to the IRC server, and allows domains without Ground Control to still advertise Satellite nodes.

**DNS SRV discovery.** The client resolves `_satellite._tcp.example.com` SRV records. Each record points to a Satellite node's host and port. The client then queries each discovered node's metadata endpoint (`GET /info`) to retrieve:

```
{
  "name": "US East",
  "region": "us-east",
  "capacity": { "current": 12, "max": 100 },
  "version": "0.1.0"
}
```

SRV record priority and weight are respected for load balancing and failover. Multiple SRV records can advertise multiple nodes under the same domain.

Nodes discovered via DNS are shown as "Server Nodes" with a verified badge - the domain's DNS records are the operator's assertion that these nodes are official.

**Fallback: no DNS records.** If no `_satellite._tcp` SRV records exist for the domain, no server-operated Satellite nodes are available. Voice features degrade gracefully - P2P calls still work (they don't need a Satellite node), and BYON nodes can still be used, but group voice via server nodes is unavailable.

**Why not an IRC channel?** Earlier designs used a well-known IRC channel (`#orbit.satellites`) with node descriptors in the topic. DNS is preferred because: (1) it doesn't require creating or configuring anything on the IRC server, (2) it works for domains that run Satellite nodes without IRC, and (3) DNS changes propagate without touching the IRC server, keeping all Orbit service advertisement in one authoritative place.

### 5.3 Bring Your Own Node (BYON)

Users can add their own Satellite node URL in Orbit's settings. When starting a session, they choose their own node instead of a server-advertised one. The `+orbit/sat-invite` posted to the channel includes the node URL, so other participants connect to the user's node.

- BYON nodes appear in the UI as "Community Node" or "User Node" (no verified badge).
- The server operator cannot block BYON usage - Orbit clients can always choose their own node. The IRC server just passes the tags.
- This enables voice in communities where the server operator hasn't set up any Satellite infrastructure. Two users on any IRCv3 server with message tags can use voice if one of them hosts a Satellite node.

### 5.4 Node Trust Model

| Node Type            | Discovery                       | UI Treatment                    | Trust Level      |
|----------------------|---------------------------------|---------------------------------|------------------|
| Server Node          | DNS SRV (`_satellite._tcp`)     | Verified badge, shown by default | Operator-trusted |
| User/Community Node  | BYON, posted via invite         | "Community" label, no badge     | User-discretion  |

Orbit clients display a clear indicator when joining a non-server node. The user must confirm before connecting to an unknown node for the first time - similar to SSH host key confirmation. Once a user has accepted a BYON node, the client remembers that decision.

### 5.5 Voice Session Flow (Group)

```
User A               Ground Control (IRC)         Satellite Node         User B
  │                        │                           │                    │
  │ Pick Satellite node    │                           │                    │
  │ (from discovery or     │                           │                    │
  │  BYON settings)        │                           │                    │
  │                        │                           │                    │
  │ POST /session/create   │                           │                    │
  │ (username, channel)    │                           │                    │
  │────────────────────────────────────────────────────►│                    │
  │                        │                           │                    │
  │◄────────────────────────────────────────────────────│                    │
  │ {token, room_id}       │                           │                    │
  │                        │                           │                    │
  │ TAGMSG #gaming.strategy│                           │                    │
  │ +orbit/sat-invite=...  │                           │                    │
  │───────────────────────►│                           │                    │
  │                        │ Relay to channel          │                    │
  │                        │──────────────────────────────────────────────►│
  │                        │                           │                    │
  │                        │                           │  POST /session/join │
  │                        │                           │  (username, room_id)│
  │                        │                           │◄───────────────────│
  │                        │                           │───────────────────►│
  │                        │                           │  {token}           │
  │                        │                           │                    │
  │ Connect to SFU         │                           │     Connect to SFU │
  │════════════════════════════════════════════════════►│◄═══════════════════│
  │                  Bidirectional media (WebRTC)       │                    │
```

The `+orbit/sat-invite` payload is a base64-encoded JSON object:

```
{
  "node": "https://sat1.example.com",
  "room": "gaming-strategy-a7f3e2",
  "initiator": "alice",
  "started": "2025-01-15T20:30:00Z",
  "protected": false
}
```

When `"protected": true`, the session is password-protected. The Orbit client displays a password prompt before attempting to join. The password is sent to the Satellite node's token service in the `/session/join` request - if it matches, a token is issued; if not, the join is rejected. The password is never sent over IRC.

Password-protected sessions are useful for private meetings, restricted briefings, or any case where the session should be visible in the channel (so people know it exists) but not freely joinable. The session creator sets the password when creating the session; it can be shared out-of-band (DM, external chat, etc.).

##### Error Handling and Edge Cases

- **Unreachable node**: If the Satellite node in a `+orbit/sat-invite` is unreachable, the client displays an error ("Voice node unavailable") and does not join. The invite remains visible in the channel with a "node offline" indicator.
- **Token rejection**: If the token service rejects a join request (invalid key, session full, password wrong), the client shows the specific error reason returned by the token service.
- **Node crash during session**: If a Satellite node goes down during an active session, all participants are disconnected. The client shows "Voice session ended unexpectedly." There is no automatic migration to another node in the MVP - the session initiator (or any participant) must start a new session on a different node and post a new `+orbit/sat-invite`.
- **Competing invites**: If multiple users post `+orbit/sat-invite` for the same channel simultaneously (different nodes or different rooms), the Orbit client displays all active sessions. Users choose which to join. There is no "one active session per channel" constraint - multiple concurrent voice sessions in the same channel are valid (e.g., different sub-groups).

### 5.6 1-on-1 Calls (P2P)

Private calls between two users bypass Satellite entirely:

1. Caller sends `+orbit/sdp-offer` via TAGMSG to the callee's nickname.
2. Callee responds with `+orbit/sdp-answer`.
3. ICE candidates are exchanged via `+orbit/ice-candidate` tags.
4. A direct P2P WebRTC connection is established.
5. If P2P fails (symmetric NAT), fall back to a TURN relay.

No server processes media for 1-on-1 calls. The only server involvement is signaling relay through Ground Control.

**Privacy Note:** P2P call signaling is relayed through Ground Control (IRC). This means the IRC server operator can observe who is calling whom, ICE candidates (which may reveal IP addresses including local/private IPs), and SDP content (codec preferences, media capabilities). This is consistent with the trust model for text chat - the server operator can already read message content (E2E encryption is deferred to [Spec 0002](./0002-research-roadmap.md)). Users who do not trust the server operator with call metadata should use a Satellite node for group calls instead, where signaling metadata is limited to the `+orbit/sat-invite` tag visible in the channel.

### 5.7 Satellite Authentication

Each Satellite node runs a token service (a small HTTP API). For the MVP:

- **Server-operated nodes**: The token service can be configured to verify identity via one of:
  - A shared secret between Ground Control and the node (the client presents a proof obtained from NickServ or SASL).
  - A simple API key that the server operator distributes (e.g., returned by the node's metadata endpoint or configured out-of-band).
  - For the MVP, the simplest viable model: the node descriptor includes a public join key, and the token service issues guest-level tokens to anyone who presents the key. Full identity verification is deferred to post-MVP.
- **BYON nodes**: The node operator controls auth entirely. They issue tokens however they see fit.
- **Password-protected sessions**: When a session is created with a password, the token service stores the password hash for that room. Clients joining a protected session must include the password in their `/session/join` request. The token service verifies it before issuing a JWT. This is per-session, not per-node - the same node can host both open and protected sessions simultaneously.
- LiveKit's built-in JWT auth is used. The token service issues LiveKit-compatible JWTs scoped to a room and identity.

### 5.8 STUN/TURN

- Self-hosted `coturn` (or STUNner for Kubernetes deployments) for NAT traversal.
- LiveKit is configured to use the TURN server for candidates.
- For P2P calls, the client is configured with the same STUN/TURN servers.
- Public STUN servers (e.g., Google's) may be used as a fallback, but self-hosted is preferred to avoid leaking metadata.

### 5.9 Codec Defaults

| Media | Codec | Bitrate (default)                      | Notes                                                     |
|-------|-------|----------------------------------------|-----------------------------------------------------------|
| Audio | Opus  | 64 kbps (voice), 128 kbps (music mode) | Mandatory. No alternative in the MVP.                     |
| Video | VP9   | Adaptive (300–2500 kbps)               | SVC profile for bandwidth adaptation. AV1 is a post-MVP option. |

### 5.10 Scope Boundary

One media transport stack for the MVP: WebRTC via LiveKit (group) and native browser/Tauri WebRTC (P2P), supporting voice and video. No MoQ, no Iroh, no custom transport experiments. Those belong in [Spec 0002](./0002-research-roadmap.md).

### 5.11 Standalone Satellite Usage

Satellite nodes are fully independent services. They can be used without Ground Control (IRC) entirely. Two users can connect to a Satellite node for voice, video, and ephemeral chat without any IRC server involvement.

The bootstrapping mechanism is a direct link:

```
satellite://sat1.example.com/room-id?name=Hangout
```

The `satellite://` URI scheme is dedicated exclusively to Satellite standalone links and is registered separately from `orbit://` (see §7.3). The host and path encode the Satellite node URL and room identifier. User A creates a session on a Satellite node, generates a shareable link, and sends it out-of-band (text message, email, another chat platform). User B opens the link, the Orbit client connects directly to the Satellite's token service, obtains a JWT, and joins the session.

This enables several use cases that do not require IRC infrastructure:

- **Quick voice calls** between friends who share a Satellite link
- **Embedded voice** on websites using only a Satellite node (no IRC backend)
- **BYON-only communities** where users host their own Satellite and share room links
- **Bootstrapping new communities** before setting up a full Ground Control instance

In standalone mode, all participants are unverified (there is no Transponder or IRC identity to verify against). Ephemeral chat via LiveKit data channels is available; persistent chat is not (that requires Ground Control). This is an intentional, honest trade-off - the experience is reduced but functional.

## 6. Identity and Authentication

### 6.1 Registered Users

- Authenticate via SASL (PLAIN or SCRAM-SHA-256) over TLS to Ground Control (Ergochat).
- NickServ (Ergochat's built-in service) handles account registration, password changes, email verification (optional), and nickname enforcement.
- Registered nicknames are reserved. Unregistered users attempting to use a registered nickname are forced to rename.
- Account data is stored in Ergochat's internal database.

### 6.2 Anonymous Web Widget Users

The Orbit web client (whether embedded as a widget or deployed as a full web app) connects directly to Ergochat's WebSocket endpoint — the same path as the desktop client. There is no middleware proxy.

```
Browser (web client / widget)               Ground Control
     │                                           │
     │  Connect WSS                              │
     │  SASL ANONYMOUS (auto guest-* nick)       │
     │──────────────────────────────────────────►│
     │                                           │
     │◄══════════════════════════════════════════│
     │           IRC session (direct)            │
```

- Guest users connect via SASL ANONYMOUS. Ergochat assigns a `guest-*` nickname automatically.
- No account creation, no backend, no JWT, no session tokens required.
- Guest nicknames are prefixed with `guest-` and cannot be registered via NickServ.
- Any IRC client — including third-party web UIs — can connect the same way. This is intentional: Orbit does not gatekeep access to a standard IRC server.

## 7. Orbit Desktop Client - Tauri v2 + Svelte

### 7.1 Technology Choice

**Why Tauri v2:**

| Metric          | Tauri v2        | Electron             |
|-----------------|-----------------|----------------------|
| Binary size     | ~10–15 MB       | ~150+ MB             |
| Idle RAM        | ~30–50 MB       | ~200–500 MB          |
| Rendering       | OS WebView      | Bundled Chromium     |
| Backend         | Rust            | Node.js              |
| Auto-update     | Built-in        | Built-in             |

Tauri uses the operating system's native WebView (WebKitGTK on Linux, WebView2 on Windows, WKWebView on macOS) instead of shipping an entire browser. The Rust backend handles performance-critical work: IRC protocol parsing, file I/O, audio device management, and IPC.

**Why Svelte over Leptos:**

Svelte compiles to minimal vanilla JavaScript with no Virtual DOM. It achieves memory efficiency comparable to WASM frameworks in practice, while offering a mature ecosystem, vastly better developer tooling, a larger talent pool, and faster iteration cycles. The estimated ~10% memory premium over a Leptos/WASM approach is an acceptable tradeoff for development velocity. Leptos remains a research track in [Spec 0002](./0002-research-roadmap.md).

### 7.2 Key Features

**Text Chat:**
- IRC connection management: connect, auto-reconnect with exponential backoff, TLS required.
- Channel list presented as a flat list. No hierarchy parsing - channels are displayed as-is.
- Message history fetched via IRCv3 `chathistory` on channel join.
- Rich rendering: inline link previews, image thumbnails, emoji (Unicode + custom per-server), basic Markdown (bold, italic, code, strikethrough).
- Message editing and deletion (rendered from `+orbit/msg-edit` and `+orbit/msg-delete` tags).
- Unread indicators and mention highlights.

**Voice & Video:**
- Satellite node selector: server nodes shown with verified badge, BYON nodes shown with community label.
- Join/leave voice sessions - "Join voice" means picking a Satellite node and joining or creating a session in the current channel.
- Visual participant list showing who is in the active session.
- Mute/deafen controls.
- Per-user volume adjustment.
- Voice activity indicators.
- Push-to-talk support (configurable keybind).
- Webcam video: toggle on/off per user.
- Video layout: grid view for small groups, speaker-focused view for larger sessions.
- Bandwidth adaptation: LiveKit's simulcast/SVC handles varying connection quality automatically.

**System Integration:**
- System tray icon with unread/mention badge counts.
- Desktop notifications (OS-native) for mentions and DMs.
- Audio device selection (input/output) in settings.
- Notification preferences (per-channel mute, DM-only mode).

### 7.3 Custom URI Scheme - `orbit://`

**Ground Control links:**

| URI                                                     | Behavior                                              |
|---------------------------------------------------------|-------------------------------------------------------|
| `orbit://server.example.com/`                           | Connect to server, show channel list                  |
| `orbit://server.example.com/channel-name`               | Connect and navigate to `#channel-name`               |
| `orbit://server.example.com/channel-name?voice=true`    | Connect, navigate to `#channel-name`, auto-join voice |

Channel names omit the `#` prefix in the URI path. The client prepends `#` when joining - this avoids URL-encoding issues with the `#` character (which is a fragment delimiter in URIs).

**Satellite standalone links (`satellite://`):**

| URI                                                     | Behavior                                              |
|---------------------------------------------------------|-------------------------------------------------------|
| `satellite://sat1.example.com/room-id`                  | Connect directly to a Satellite node and join room    |
| `satellite://node-url/room-id?name=Display+Name`        | Connect with a display name hint                      |

The `satellite://` scheme is registered separately from `orbit://` and is dedicated exclusively to direct Satellite node connections. The host is the Satellite node's hostname (e.g., `sat1.example.com`).

**Invite model:** Since Orbit is decentralized, there is no central invite service. An `orbit://` link *is* the invite - sharing the link is sharing the invite. The server operator controls access via IRC channel modes (`+i` for invite-only, `+k` for key-protected channels). The `orbit://` URI is essentially a deep link, not a magic token.

**Platform registration:**

- **Linux**: `.desktop` file in `~/.local/share/applications/` with `MimeType=x-scheme-handler/orbit;x-scheme-handler/satellite`. Registered via `xdg-mime`.
- **Windows**: Registry key under `HKEY_CLASSES_ROOT\orbit` pointing to the Orbit executable with `%1` argument. Add a second registry key under `HKEY_CLASSES_ROOT\satellite` pointing to the same executable.
- **macOS**: `CFBundleURLTypes` entry in `Info.plist` with scheme `orbit`. Add a second `CFBundleURLTypes` entry with scheme `satellite`.

On invocation, the app launches (or focuses if already running) and routes to the specified server/channel or Satellite session. If the client is not installed, `orbit://` links are inert - there is no web fallback in the MVP (the full web client could serve as a fallback in a post-MVP update).

### 7.4 DNS SRV Resolution

DNS is the universal discovery mechanism for all Orbit services. Users enter only a domain (e.g., `example.com`); the client resolves SRV records to find each service. This means a domain can advertise any combination of services - IRC only, Satellite only, or the full stack - and the client adapts automatically.

| Record                              | Purpose                              |
|-------------------------------------|--------------------------------------|
| `_satellite._tcp.example.com`      | Satellite node discovery             |
| `_depot._tcp.example.com`          | Depot (file storage) discovery       |
| `_transponder._tcp.example.com`    | Transponder (identity) discovery *(post-MVP)* |

**Ground Control resolution:**
1. Attempt connection to `example.com` on default port `6697` (IRC over TLS).
2. Failure → prompt user for manual host:port entry.

**Satellite resolution:**
1. SRV records `_satellite._tcp.example.com` → query each node's `/info` endpoint for metadata (name, region, capacity).
2. No SRV records → no server-operated Satellite nodes. BYON and P2P calls still work.

**Depot resolution:**
1. SRV record `_depot._tcp.example.com` → use returned host and port for file uploads/downloads.
2. No SRV record → file sharing unavailable (client hides upload UI).

**Transponder resolution (post-MVP):**
1. SRV record `_transponder._tcp.example.com` → use for identity token requests.
2. Fallback: `/.well-known/orbit/keys.json` on the domain.
3. Fallback: DNS TXT record `orbit._keys.example.com` containing the public key.
4. No discovery → all Satellite participants are unverified (graceful degradation).

This DNS-first model means that an operator can point a domain at Orbit services without modifying any IRC server configuration. Add SRV records, and Orbit clients discover everything automatically. It also means domains without IRC (Ground Control) can still host Satellite nodes, Depot, or Transponder independently.

### 7.5 Memory Discipline

| Concern              | Strategy                                                                               |
|----------------------|----------------------------------------------------------------------------------------|
| Chat history         | Paginated loading. Keep at most 200 messages per channel in the DOM. Older messages are evicted and re-fetched on scroll-up via `chathistory`. |
| Image rendering      | Images are proxied and resized by the Rust backend (or server-side) before display. No raw multi-MB images loaded into the WebView. |
| Large IPC payloads   | File downloads and bulk history loads are served via Tauri's custom protocol handler (`tauri://`), not JSON-serialized IPC. |
| Layout thrashing     | Resize events are debounced aggressively (200ms minimum). This works around a known WebKitGTK memory leak on Linux triggered by rapid resize cycles. |
| Audio buffers        | Managed entirely in the Rust backend via `cpal`. No Web Audio API overhead in the WebView. |

### 7.6 Offline Messages and Reconnection

Ergochat includes built-in bouncer/always-on functionality that handles offline message delivery without additional infrastructure.

**Always-on mode**: When enabled, Ergochat keeps the user's session alive on the server even when no client is connected. The user remains in all joined channels and accumulates messages. This is configured server-side (`accounts.multiclient.always-on`) and can be user-toggled.

**Reconnection flow**: When the Orbit client reconnects after a disconnection:

1. The client authenticates via SASL.
2. The client issues `CHATHISTORY TARGETS` to discover which channels and DMs have new messages since the last known timestamp.
3. For each target with new messages, the client issues `CHATHISTORY LATEST` to fetch missed messages.
4. Messages are deduplicated against any already-displayed messages (using `msgid`).
5. Unread indicators and mention highlights are updated.

**Partial disconnection**: Because Ground Control and Satellite are independent, one can drop while the other stays live:

- **IRC drops, Satellite stays up**: Voice/video continues uninterrupted. Ephemeral Satellite chat continues. The client displays a warning ("Text chat reconnecting...") and attempts to re-establish the IRC connection with exponential backoff. No voice session interruption occurs.
- **Satellite drops, IRC stays up**: Text chat continues. The client displays "Voice session disconnected" and optionally attempts to rejoin the same Satellite room. Other participants see the user leave the voice session.
- **Both drop**: The client reconnects to IRC first (primary), then rejoins the Satellite session if one was active.

**Message outbox**: Messages composed during an IRC disconnection are held in a client-side outbox. They are sent when the connection is re-established and only cleared from the outbox after the server acknowledges delivery (confirmed by the server returning a `msgid` for the message). If delivery fails after reconnection, the user is notified.

Orbit deployments SHOULD enable Ergochat's always-on mode for registered users. This ensures that users receive all messages sent while they were offline, and that their channel memberships are preserved across disconnections.

## 8. Orbit Web Widget

### 8.1 Overview

The Orbit web client is a lightweight Svelte application that can be embedded in third-party sites as a widget or deployed as a full standalone web app. It connects directly to Ground Control via WebSocket and to Satellite nodes via WebRTC — the same paths as the desktop client, with no middleware required. Guest access uses SASL ANONYMOUS; no backend or proxy needed.

### 8.2 Integration

Two embedding options:

```
<!-- Option A: iframe (recommended) -->
<iframe
  src="https://widget.hivecom.net/embed?server=irc.hivecom.net&channel=general"
  width="400" height="600"
  allow="microphone"
></iframe>

<!-- Option B: script tag -->
<script
  src="https://widget.hivecom.net/orbit-widget.js"
  data-server="irc.hivecom.net"
  data-channel="general"
></script>
```

### 8.3 Technical Details

- Built as a standalone Svelte app, compiled to a single JS bundle. **Target: <50 KB gzipped.**
- Connects directly to Ground Control via secure WebSocket (same path as the desktop client). No backend or proxy required.
- No account creation required for guest access. No cookies stored. No local storage.

### 8.4 Capabilities

| Feature                    | Supported     | Notes                                                             |
|----------------------------|---------------|-------------------------------------------------------------------|
| Read text chat             | Yes           | Real-time message stream                                          |
| Send text messages         | Configurable  | Server operator can set read-only or read-write                   |
| Voice listen-in            | Yes           | Connects to the Satellite node advertised in the voice session invite (receive-only) |
| Voice speak                | No            | Guests cannot transmit audio in the MVP                           |
| File uploads               | No            | Guests cannot upload files                                        |
| Message history            | Limited       | Last 50 messages on load, no scroll-back                          |
| User list                  | Yes           | Shows participants in the channel                                 |

Media connections go directly from the web client to the Satellite node.

### 8.5 Rate Limiting

Rate limiting for guest connections is handled by Ergochat's built-in flood protection and per-IP connection limits, configured by the server operator in Ergochat's settings. No separate service or configuration is needed. The same limits apply to all connecting clients — desktop, web, or third-party IRC clients.

### 8.6 Full Web Client

Separate from the embeddable widget, a **full-featured Svelte web app** can provide nearly the same experience as the desktop client. Since the Orbit desktop client's frontend is built in Svelte (running inside Tauri's webview), the same codebase can be deployed as a standalone web application with minimal adaptation.

**What works natively in the browser:**

- Ground Control connection via WebSocket - browsers support WebSocket natively, identical to the Tauri webview path.
- Satellite voice and video via WebRTC - browsers have native WebRTC support. Group sessions (LiveKit) and P2P calls both work without any Tauri backend involvement.
- All text chat features: message history, editing, deletion, rich rendering, unread indicators.
- File uploads and downloads via S3 pre-signed URLs.

**What's NOT available in the web client:**

| Capability                    | Desktop (Tauri)                              | Web Client                                              |
|-------------------------------|----------------------------------------------|---------------------------------------------------------|
| Custom URI schemes (`orbit://`, `satellite://`) | Registered with OS; opens/focuses the app | Not available - browsers cannot register custom URI handlers |
| System tray with badges       | Native OS system tray with unread counts     | Not available - uses browser tab title and favicon badges instead |
| Audio device management       | Rust-side audio via `cpal` with fine control | Web Audio API (functional but less control over device selection) |
| OS-native notifications       | Full OS notification integration             | Web Notifications API (requires permission grant; less reliable) |

Everything else - text chat, channel management, voice & video sessions, file sharing, message history, user presence - works identically.

**Deployment:** The same Svelte application serves both roles — embedded as a constrained widget on third-party sites, or deployed in full as a web app for users who prefer not to install the desktop client.

**MVP priority:** For the MVP, the desktop client is the primary target. The full web client is a natural fast-follow since it shares the same Svelte codebase - the main work is abstracting the Tauri-specific APIs (URI handling, tray, audio device selection) behind a platform adapter layer.

## 9. Infrastructure and Deployment

### 9.1 Component Overview

| Component                         | Technology         | Resource Target (~100 users) | Required |
|-----------------------------------|--------------------|------------------------------|----------|
| Ground Control (Ergochat)         | Go, single binary  | 1 vCPU, 256 MB RAM          | Yes      |
| Satellite (LiveKit + token service) | Go + thin HTTP API | 1 vCPU, 512 MB RAM (scales with users) | No (optional) |
| Depot (Object Storage)            | MinIO or S3        | Storage-dependent            | No (only for file uploads) |
| coturn (STUN/TURN)                | C, single binary   | 1 vCPU, 128 MB RAM          | No (only for NAT traversal) |
| **Total minimum (text-only)**     |                    | **~$5/month VPS**            |          |
| **Total with voice**              |                    | **~$10/month VPS**           |          |

A community can run Orbit text-only with just Ground Control (Ergochat). Satellite nodes and Depot are optional components added as needed.

### 9.2 TLS

TLS is required on every connection. No plaintext IRC, no plaintext HTTP, no exceptions.

- Let's Encrypt for public deployments (automated via certbot or Caddy).
- Self-signed certificates acceptable for local/development setups only.
- Ergochat terminates TLS for IRC connections directly.
- A reverse proxy (Caddy or nginx) terminates TLS for WebSocket, Satellite token service endpoints, and Depot endpoints.

### 9.3 DNS Configuration

DNS is the primary discovery mechanism for all Orbit services. These records are **advertisement pointers** published under the community's domain — they tell Orbit clients which services are associated with this community, not that those services run on `example.com` directly. Each SRV record resolves to the actual host and port of the service, which may be on entirely separate infrastructure. A record's absence simply means the client won't attempt to discover that service.

| Record                              | Type    | Purpose                            | Add when |
|-------------------------------------|---------|------------------------------------|----------|
| `_satellite._tcp.example.com`      | SRV     | Satellite node discovery           | If running Satellite |
| `_depot._tcp.example.com`          | SRV     | Depot (file storage) discovery     | If running Depot |
| `_transponder._tcp.example.com`    | SRV     | Transponder (identity) discovery   | Post-MVP |
| `irc.example.com`                  | A/AAAA  | Ground Control (IRC + WebSocket)   | If running IRC |
| `sat.example.com`                  | A/AAAA  | Satellite node endpoint            | If running Satellite |
| `depot.example.com`                | A/AAAA  | Depot (object storage) endpoint    | If running Depot |
| `turn.example.com`                 | A/AAAA  | TURN server                        | If running TURN |

A minimal text-only deployment needs only the A/AAAA record for the IRC server; the Orbit client connects to port 6697 (IRC over TLS) by default. A Satellite-only deployment (no IRC) needs only `_satellite._tcp` and the A/AAAA record for the Satellite node. The client adapts based on which records exist.

Multiple `_satellite._tcp` SRV records can be configured to advertise multiple Satellite nodes. SRV priority and weight control load distribution and failover.

### 9.4 Reference Docker Compose

A reference `docker-compose.yml` will be provided for self-hosters that includes:

- Ergochat (Ground Control - with WebSocket enabled, SASL configured, history enabled)
- LiveKit + token service (Satellite node - **optional**, can be removed for text-only deployment)
- MinIO (Depot - **optional**, only needed for file uploads)
- coturn (STUN/TURN)
- Caddy (reverse proxy, automatic TLS)

One `docker compose up` should yield a fully functional Orbit instance. Configuration is done via a single `.env` file and an `orbit.toml` for server-specific settings (domain, channel list). Satellite node discovery is handled via DNS SRV records configured at the domain level - no IRC channel configuration is needed.

Note: Satellite is optional. You can run `docker compose up ergochat caddy` for a minimal text-only Orbit server.

## 10. Extensions

Orbit intentionally does not solve custom roles, calendars, events, game integrations, advanced moderation workflows, or any other domain-specific feature. These are solved by two complementary mechanisms: **Orbit extensions** and **IRC bots**.

**Orbit Extensions**

An Orbit extension is a true application-level plugin for the Orbit client (orbit-app). Extensions are installed into the client and extend its UI and behavior. They interact with Ground Control and Satellite through the standard Orbit tag namespace and may define their own sub-namespace for custom tags (e.g., `+orbit-ext/<name>/*`). Extensions are scoped to the Orbit client — they do not run on the server, and they do not modify Satellite infrastructure.

The orbit-app extension API is **deferred to post-MVP**. For the MVP, extensions are a design target, not a shipping feature.

**IRC Bots**

IRC bots are first-class citizens of the Orbit ecosystem by virtue of Orbit being built on IRC. Any IRC bot — written in any language, using any framework (Limnoria, Sopel, a custom script) — connects to Ground Control and works out of the box. Bots handle server-side automation: posting reminders, enforcing moderation rules, managing channel state, and responding to commands. They are not Orbit extensions; they are IRC bots.

**Examples of future Orbit extensions:**
- A **calendar extension** that renders upcoming events inline in the Orbit client, sourced from custom `+orbit-ext/calendar/*` tags posted by a companion bot.
- A **moderation dashboard extension** that adds a UI panel for reviewing flagged messages and applying IRC modes.
- A **game status extension** that displays live match state in the sidebar using custom tags.

**Examples of IRC bots (not extensions):**
- A **permissions bot** that implements role hierarchies beyond IRC modes.
- A **reminder bot** that posts scheduled messages to channels.
- A **moderation bot** with auto-mod rules, word filters, and spam detection.

## 11. Tag Integrity and Client Trust Model

Orbit's rich features (message editing, deletion, media signaling, file metadata) are transported as IRCv3 client-only tags (`+orbit/*`). These tags are relayed by the IRC server without interpretation or validation - the server passes them through as opaque metadata. This is by design (it means Orbit works on any IRCv3-compliant server), but it creates a trust boundary that clients must enforce.

**Server-asserted vs. client-asserted data:**

| Data | Source | Trust Level | Forgeable? |
|------|--------|-------------|------------|
| `account-tag` | Set by the IRC server based on SASL authentication | Server-asserted | No - the server is authoritative |
| `msgid` | Assigned by the IRC server | Server-asserted | No |
| `server-time` | Set by the IRC server | Server-asserted | No |
| `+orbit/msg-edit` | Set by the sending client | Client-asserted | Yes - any client can send this tag |
| `+orbit/msg-delete` | Set by the sending client | Client-asserted | Yes |
| `+orbit/sat-invite` | Set by the sending client | Client-asserted | Yes |
| `+orbit/sdp-offer` | Set by the sending client | Client-asserted | Yes |
| `+orbit/file-*` | Set by the sending client | Client-asserted | Yes |

**Client enforcement rules:**

1. **Message edits**: Accept only if the sender's `account-tag` matches the original message's `account-tag`.
2. **Message deletes**: Accept only if the sender's `account-tag` matches the original message's `account-tag`, OR the sender is a channel operator (`+o`).
3. **Satellite invites**: Display the sender's verified identity (via `account-tag`) alongside the invite. Users should know who is inviting them to a node before connecting.
4. **P2P call signaling**: Verify the sender's identity before accepting a call. Do not auto-accept calls from unverified senders.
5. **File metadata**: The `+orbit/file-*` tags (name, size, type) are informational. The client SHOULD verify file metadata independently (e.g., by checking HTTP headers on download) rather than trusting the tags blindly.

**Unverified senders**: Messages from users without an `account-tag` (unauthenticated users) that carry `+orbit/msg-edit` or `+orbit/msg-delete` tags MUST be silently ignored. Unverified users cannot edit or delete messages.

This model is analogous to how email works - the transport (SMTP) delivers messages without validating sender claims, but the receiving mail client checks DKIM signatures and SPF records to verify authenticity. In Orbit's case, `account-tag` is the DKIM equivalent: a server-asserted proof of identity that clients use to validate client-asserted claims.

Orbit clients that do not implement these verification rules are non-compliant. Third-party IRC clients connecting to an Orbit server will not enforce these rules (they don't understand `+orbit/*` tags), but this is expected - they also won't render edits, deletes, or media invites. The security boundary is between Orbit clients, not between arbitrary IRC clients.

## 12. Explicitly Out of Scope for the MVP

The following are **not** part of the MVP release. Each item either requires significant research, has unresolved design questions, or is a non-essential enhancement. They are tracked in [Spec 0002](./0002-research-roadmap.md).

| Feature                                  | Reason for deferral                                                  |
|------------------------------------------|----------------------------------------------------------------------|
| Media over QUIC / Iroh transport         | Experimental; WebRTC is proven and sufficient for the MVP            |
| Leptos / WASM frontend rewrite           | Research track; Svelte delivers faster for now                       |
| Vulkan gaming overlay (Linux)            | Requires Vulkan layer injection - complex and fragile                |
| Wayland layer-shell / X11 overlays       | Platform-specific complexity; needs prototyping                      |
| End-to-end encryption                    | Key management UX is unsolved for group chat at scale                |
| Federation between Orbit instances       | Requires protocol extensions beyond IRCv3 server linking             |
| Mobile clients (iOS / Android)           | Tauri v2 mobile support exists but is immature                       |
| Server directory / discovery service     | Centralization concern; needs careful design                         |
| Screen sharing                           | Voice and video are in the MVP; screen sharing is a post-MVP fast-follow |
| Multi-server support in client           | Connect to one server at a time in the MVP                           |
| Custom emoji / sticker packs             | Nice-to-have; not essential for launch                               |
| Message threads / forums                 | IRC doesn't have a native threading model; needs design work         |
| Custom role / permission system          | Use IRC modes; extend via extensions if needed                       |
| Orbit extension API (orbit-app plugins)  | Extension architecture is defined; the client-side plugin host and API surface are post-MVP |

## 13. Open Questions

These are genuine unresolved decisions that need resolution before or during MVP implementation.

1. ~~**Nickname collision**~~ *Resolved*: The `guest-` prefix is reserved at the NickServ level (see §4.4). Ergochat is configured to reject registration of any nickname starting with `guest-`, preventing collision between registered users and anonymous widget guests.

2. **File upload limits**: Should upload size limits be enforced at the storage endpoint (via pre-signed URL constraints or bucket policies) or at the client level? Storage-layer enforcement is more reliable since there's no centralized upload gateway.

3. **Presence model**: IRC provides `AWAY` and `QUIT`/`JOIN` for basic presence. Is this sufficient for the MVP, or do we need richer status states (online, idle, do-not-disturb, invisible)? Richer presence could be built with client-only tags, but adds complexity.

4. **Widget voice permissions**: Should the web widget ever allow guests to *speak* in voice channels, or is receive-only the correct permanent default? Allowing guest voice opens abuse vectors but has legitimate use cases (e.g., community Q&A sessions).

5. **Message history retention**: What's the default retention period? Should it be configurable per-channel? If a server operator sets unlimited retention, what are the storage implications for Ergochat's internal database?

6. **Satellite metadata endpoint**: What should the Satellite node's `/info` metadata endpoint return? The spec proposes name, region, capacity, and version. Should it also include supported codecs, maximum participants, or TURN server hints? How much metadata is useful for client-side node selection vs. unnecessary complexity?

7. **Satellite node trust**: When a user posts a BYON invite, should Orbit prompt other users with a confirmation dialog ("This voice session is hosted on a community node at `sat.user.com` - connect?")? Or is that too much friction? The spec currently says yes (SSH host key model), but this needs UX validation.

8. **Satellite token bootstrapping**: For server-operated nodes, what's the identity verification flow between the Orbit client and the token service? A public join key in the node descriptor is simple but means anyone who can read the channel topic can get a token. Is that acceptable for the MVP? (It probably is - the same trust model as a public TeamSpeak or Mumble server.)

9. **Multi-node sessions**: Can a voice session span multiple Satellite nodes? Probably not for the MVP - one session, one node. But what happens if a node goes down during an active session? Should the client attempt to migrate to another node?


*This is a living document. It will be updated as implementation progresses and open questions are resolved. Changes are tracked in the repository's commit history.*
