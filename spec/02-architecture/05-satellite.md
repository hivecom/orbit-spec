# Satellite

Satellite is the real-time media component of an Orbit deployment: voice,
video, screen sharing, and ephemeral in-session chat. It's a bespoke
component Orbit builds. It embeds [LiveKit](https://livekit.io/) as the SFU
for group calls, but it isn't "just LiveKit": Satellite owns its own session
model, 1:1 P2P calling (WebRTC negotiated over IRC tags, no LiveKit),
moderation, discovery, and Bring Your Own Satellite (BYOS). See
[Component Classes](01-overview.md#component-classes).

Satellite is completely decoupled from Uplink - it has no dependency on IRC,
channels, or message history, and can be used standalone without any Uplink
instance. It's also optional: an Orbit deployment without Satellite is a
fully functional IRC-based text chat server.

Session flow diagrams, invite and handshake payloads, knocking mechanics,
codecs, and STUN/TURN configuration live in
[Implementation - Satellite](../03-implementation/03-satellite.md).

## Architecture

A Satellite deployment has two layers:

- **Satellite** - the logical service clients interact with: discovered via
  DNS, queried for metadata, addressed in invite tags. From the client's
  perspective, a Satellite is a single endpoint that hosts rooms.
- **Nodes** - individual LiveKit SFU instances within a Satellite. Each node
  hosts multiple concurrent rooms sharing the same resource pool. Nodes are
  an infrastructure detail; clients never address them directly.

A Satellite is a single node: one LiveKit process and one token service,
deployed as a container pair, analogous to a TeamSpeak or Mumble server - one
process, many rooms. Scaled deployments put multiple nodes behind a gateway
(see [Scaling](#scaling)).

A node has two co-located parts:

- **SFU (LiveKit)**: WebRTC media - audio/video forwarding, bandwidth
  adaptation, STUN/TURN integration - and data channels for ephemeral session
  chat.
- **Token service**: a small HTTP API that issues LiveKit-compatible session
  credentials. In a single-node deployment it's the entry point; in a
  multi-node deployment the gateway absorbs this role.

Satellite sessions include built-in ephemeral text chat via LiveKit data
channels. It isn't persisted - when the session ends, the messages are gone.
It exists for in-session coordination: quick callouts during a call, links
during a screen share. Persistent, searchable chat lives in Uplink; the two
are architecturally distinct.

## Discovery

DNS is the primary discovery mechanism for Satellites. DNS works
independently of any running service, requires no modification to the IRC
server, and lets domains without Uplink still advertise Satellites. The
client resolves the domain's Satellite records, then queries each Satellite's
metadata endpoint for its name, region, capacity, and active rooms. Record
definitions and the resolution algorithm live in
[Infrastructure](12-infrastructure.md) and
[Implementation - Deployment](../03-implementation/09-deployment.md).

The metadata endpoint is intentionally unauthenticated. Unauthenticated
clients - including the embedded client and anonymous users - need to
discover and display available voice sessions before authenticating. This is
consistent with the public nature of IRC channels: anyone connected can see
what exists. Operators who want private session metadata restrict at the
network level (firewall, VPN); a deployment that requires fully private
session metadata shouldn't run a publicly reachable Satellite.

Satellites discovered via the domain's DNS records get a verified badge - the
records are the operator's assertion that these Satellites are official.

If no Satellite records exist for a domain, voice degrades gracefully: P2P
calls still work (they don't need a Satellite) and BYOS Satellites can still
be used, but group voice via server Satellites is unavailable.

An earlier design used a well-known IRC channel with descriptors in the
topic. DNS won because it requires nothing on the IRC server, works for
domains that run Satellites without IRC, and keeps all Orbit service
advertisement in one authoritative place.

## Bring Your Own Satellite (BYOS)

Users can add their own Satellite URL in Orbit's settings and choose it when
starting a session. The invite posted to the channel carries the Satellite
URL, so other participants connect to the user's Satellite.

- BYOS Satellites appear in the UI as "Community" with no verified badge.
- The server operator can't block BYOS - the IRC server just passes the tags.
- This enables voice in communities where the operator hasn't set up any
  Satellite infrastructure: two users on any IRCv3 server with message tags
  can use voice if one of them hosts a Satellite.

## Trust Model

| Type                | Discovery                   | UI Treatment                     | Trust Level      |
|---------------------|-----------------------------|----------------------------------|------------------|
| Server Satellite    | Domain DNS records          | Verified badge, shown by default | Operator-trusted |
| Community Satellite | BYOS, posted via invite     | "Community" label, no badge      | User-discretion  |

The domain's DNS records are the network's authoritative statement of which
Satellites it sanctions. Orbit clients display a clear indicator when joining
a community Satellite, and the user must confirm before connecting to an
unknown Satellite for the first time - the same model as SSH host key
confirmation. Once accepted, the client remembers the decision. The
confirmation exists to stop hijacking: an invite can point anywhere, so a
Satellite the network hasn't sanctioned never gets a connection without the
user's explicit say-so.

## Group Sessions

A group session starts when a user asks a Satellite (server-discovered or
BYOS) to create a room and posts an invite tag to the IRC channel. Other
participants use the invite to request a join from the Satellite's token
service, then connect to the SFU over WebRTC. Uplink's involvement ends at
the invite; media never touches it.

Sessions can be password-protected: the client prompts for the password
before joining, and the password is verified by the token service, never sent
over IRC. Protected sessions are visible in the channel (so people know they
exist) but not freely joinable - useful for private meetings. Protection is
per-session; the same Satellite hosts open and protected sessions
simultaneously.

### Failure Model

- **Unreachable Satellite**: the client shows an error and doesn't join; the
  invite remains visible with an offline indicator.
- **Rejected join** (invalid credential, session full, wrong password): the
  client shows the specific reason returned by the token service.
- **Satellite crash during a session**: all participants are disconnected and
  the client reports the session ended unexpectedly. There is no automatic
  migration - a participant starts a new session and posts a new invite.
- **Competing invites**: if multiple users post invites for the same channel,
  the client displays all active sessions and the user chooses. There's no
  one-session-per-channel constraint; concurrent sessions in one channel are
  valid.

## 1:1 Calls (P2P)

Private connections between two users bypass Satellite entirely. The Orbit
client establishes a direct WebRTC connection using IRC only for the initial
handshake: one offer tag and one answer tag, each carrying compact connection
credentials (ICE credentials, DTLS fingerprint, one candidate) - two IRC
messages total. All further negotiation happens over the WebRTC data channel,
independent of Uplink.

The offer declares an intent - call, video, chat, or file - which sets the
recipient's acceptance prompt and the starting UI state, not a permanent
constraint. A voice call can escalate to video, screen share, side chat, or
file transfer, all negotiated over the data channel. A file transfer is
transactional: it completes and the connection tears down.

After the handshake the connection is fully self-sufficient. Media
negotiation, ICE trickling, and escalation all run over the data channel;
Uplink can go down and the session continues. Only starting a new connection
requires IRC.

Earlier iterations sent full SDP offers over IRC tags. SDPs are large (2-3 KB
with exhaustive codec enumeration), which strained the IRC tag budget and
flood protection. The handshake-first model eliminates this: the IRC payload
carries only connection credentials, and full SDP negotiation happens over
the data channel where size doesn't matter.

### TURN Fallback

If direct connectivity fails (both peers behind symmetric NATs), the
connection falls back to a TURN relay run by the operator. A relayed session
depends on the TURN server staying up, and ICE restart candidates can be
exchanged over the data channel to attempt a new path without re-involving
IRC.

When TURN is in the path, encrypted media transits the operator's relay. P2P
calls use DTLS-SRTP, so the relay sees only ciphertext - it can observe that
a relayed session exists and its duration, but can't decrypt or inspect the
content. What weakens under TURN is the "not on our wires" property, not the
"cannot decrypt" property. This distinction should inform user expectations.

### Privacy

P2P handshake signaling is relayed through Uplink, so the IRC server operator
can observe who is connecting to whom, the intent, and one public IP per peer
from the initial candidate. That's consistent with the trust model for text
chat - the operator can already read message content.

The real privacy shield is cryptographic, not topological. All P2P sessions
are DTLS-SRTP encrypted end to end. Direct routing additionally achieves
non-retention: post-handshake, media and signaling never touch any server.
P2P sessions retain nothing - no recording, no server-side storage. The only
retained trace is the handshake signaling in IRC, which contains connection
metadata, not content. Users who don't want the operator to see connection
metadata should weigh that against group sessions, where signaling metadata
is limited to the invite visible in the channel.

## Authentication

Each Satellite's token service (or gateway) issues session credentials scoped
to a room and identity.

- **With an identity provider** ([Transponder](07-transponder.md)): the token
  service verifies the client's identity token against the provider's
  published keys - standard OIDC consumption, no custom protocol. Verified
  participants carry their account name; participants without a token join
  as unverified.
- **BYOS Satellites**: the operator controls auth entirely.
- **Password-protected sessions**: the token service verifies the session
  password before issuing credentials, independent of identity verification.
- **No identity provider**: credentials are issued to anyone who can reach
  the Satellite; all participants are unverified. Sessions can still be
  password-protected.

Whether unverified participants can publish media - speak, send video, share
a screen - is an operator setting on each Satellite, applied by the token
service when it issues session credentials. The default is publish-enabled: a
guest joining from a link or the embedded client can speak. Operators who
want receive-only guests configure that on their Satellite; verified
participants are unaffected. The token-grant mechanics live in
[Implementation - Satellite](../03-implementation/03-satellite.md#token-service).

The verified/unverified user model is specified in
[Identity](09-identity.md#verified-and-unverified-users).

## Session Permissions

Session permissions are minimal and creator-centric. The user who creates a
session is the session admin; they can delegate moderation to other verified
users, and there are no role hierarchies beyond creator and moderator.
Sessions are ephemeral, and so is their moderation state.

| Role | How you get it | Capabilities |
|------|---------------|--------------|
| **Creator** | Created the session | Mute and kick participants (including moderators), set or change the session password, lock and unlock, manage the allow-list, end the session |
| **Moderator** | Designated by the creator | Mute and kick participants (but not the creator), lock and unlock, admit knockers |
| **Participant** | Joined the session | Publish and subscribe to media, send ephemeral chat |

All session configuration is client-driven: the creator's client sends the
moderator list, allow-list, access mode, and lock state to the token service
at creation (and can update them during the session). The Satellite holds
this state only for the session's duration - when it ends, everything is
gone. No component persists session permissions; if the creator wants the
same setup next time, their client provides it again (the client may store
those preferences locally as a convenience). Creator and moderator
enforcement uses LiveKit's native room-admin grant - mute and kick are
built-in LiveKit operations, not custom Orbit logic. Delegated moderation
requires verifiable identity: unverified users can't receive it.

If you don't like how a room is run, make your own. There's no appeals
process and no server-operator intervention in session moderation.

### Access Control

Three access modes, set at creation and adjustable during the session:

| Mode | Behavior | Use case |
|------|----------|----------|
| **Open** | Anyone with the invite can join (default) | Casual channel voice, open hangouts |
| **Password-protected** | Must present the correct password | Private meetings, restricted briefings |
| **Allow-list** | Only specified verified identities can join; unverified users are always rejected | Trusted-group sessions, team calls |

The creator can also lock a session at any time: a locked session rejects all
new joins regardless of access mode, while current participants stay.

When a session is locked or restricted, a rejected user can knock - a request
delivered to the creator and moderators, who can admit or ignore it. Knocking
is best-effort with a short timeout, holds no SFU resources while the knocker
waits, and keeps no state beyond the knock's TTL - a doorbell, not a waiting
room. The knock delivery and status mechanics live in
[Implementation - Satellite](../03-implementation/03-satellite.md).

## Standalone Usage

Satellites are fully independent services and can be used without Uplink
entirely. The bootstrapping mechanism is a direct `satellite://` link, shared
out-of-band, that the client resolves to the Satellite's token service. The
URI scheme details live in
[Implementation - Clients](../03-implementation/08-clients.md).

This covers quick voice calls between friends who share a link, embedded
voice on websites with no IRC backend, BYOS-only communities, and
bootstrapping a community before setting up an Uplink instance.

In standalone mode there's no IRC identity, but identity verification is
still possible: the client attempts service discovery against the Satellite's
domain, and if an identity provider is found, offers the user the option to
authenticate. Verified and unverified participants can mix in the same
session, exactly as in IRC-signaled sessions. The identity provider is always
the one associated with the Satellite's domain, not the user's home domain -
cross-domain identity is a federation concern
([Federation](14-federation.md)). Ephemeral chat is available; persistent
chat isn't (that requires Uplink).

## Scope Boundary

One media transport stack: WebRTC, via LiveKit for group sessions and the
native browser/Tauri WebRTC stack for P2P. No MoQ, no Iroh, no custom
transport experiments - those are tracked in
[Research: MoQ / Iroh](../0R-research/01-moq-iroh.md).

## Session Limits

Satellite imposes no hardcoded participant cap. Session capacity is bounded
by node hardware - primarily network bandwidth, then CPU. LiveKit has no
built-in per-room participant limit; a single node hosts multiple rooms
drawing from one resource pool. Rough single-node capacity on a 1 Gbps link:

| Session profile | Approximate capacity |
|---|---|
| Audio-only (Opus, ~50 kbps/participant) | ~200-500 participants |
| Mixed audio + video (360p, ~500 kbps/participant) | ~50 participants |
| Mixed audio + video (720p, ~1.5 Mbps/participant) | ~20-30 participants |

These are aggregate-bandwidth estimates; actual capacity depends on NIC
throughput, CPU for SRTP, and simulcast configuration. When a node can't
accept more participants, the token service rejects new joins with a clear
error; in a multi-node deployment the gateway routes new sessions to nodes
with capacity instead.

Communities that need audiences beyond what a Satellite supports should use a
one-to-many streaming setup - Satellite is a communication tool, not a
broadcasting platform. See
[Research: Broadcast Streaming](../0R-research/05-broadcast-streaming.md).

## Scaling

A single node - one LiveKit instance, one token service - covers small and
mid-sized communities. When a community outgrows one node's ceiling, the
operator adds nodes, and the moment there are multiple nodes there's a
routing problem: session creation needs to land on a node with capacity, and
joins need to land on the node that hosts the target room. The **gateway**
solves it.

The gateway is a thin routing layer in front of the node pool. It exposes the
same API surface as a single-node Satellite - clients see no difference and
invites keep pointing at the Satellite, not at nodes. It absorbs the token
service role and adds:

- Routing session creation to the least-loaded node and joins to the room's
  node.
- Aggregating room and participant data into one unified metadata view.
- Health checking: unhealthy nodes stop receiving new rooms; if a node
  becomes unresponsive, its rooms are lost and participants see the session
  end.
- Drain coordination on scale-down: stop routing new sessions to a node, let
  its rooms end naturally, then retire it. At least one node always remains.

Room affinity is inherent: a room lives on one node for its entire lifetime.
Room migration is explicitly out of scope - WebRTC sessions are tied to a
specific server, and moving a room would disconnect every participant, which
is indistinguishable from the room crashing. If a node must be retired while
rooms are active, the operator waits for them to drain.

The preferred gateway design is stateless: it derives its routing table from
the nodes themselves, so a restarted gateway rebuilds state in seconds. That
also means gateway replicas don't need to coordinate, which is the mitigation
for the gateway being a single point of failure for new sessions (active
sessions continue regardless - media flows directly between clients and
nodes). The added hop costs latency only on the credential exchange, not on
media.

Kubernetes is the recommended deployment model for multi-node Satellites
(pod autoscaling, health-based routing, graceful drain), with STUNner as the
NAT traversal layer, but it isn't required - a static set of nodes behind one
gateway is a valid multi-node deployment, and single-node deployments need
neither. Routing internals, load-balancing metrics, and the Kubernetes wiring
live in [Implementation - Satellite](../03-implementation/03-satellite.md).

One risk worth naming: LiveKit Cloud offers hosted multi-node routing.
Operators willing to use it get routing for free; the gateway serves fully
self-hosted deployments where operators keep all infrastructure under their
own control.
