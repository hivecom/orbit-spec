# Research: Federation

## Problem

The MVP is single-server. All users in a community connect to one Orbit server instance. Users on server A cannot communicate with users on server B. Communities are isolated islands.

A deeper problem sits underneath federation itself: **identity bridging**. IRC handles user authentication within its own protocol boundary (SASL, NickServ, `account-tag`), but those assertions are server-scoped. They don't travel outside the IRC connection. When a user connects to a Satellite node (a completely separate service), the Satellite has no native way to verify that this person is the same user who is authenticated on IRC. The MVP punts on this with a public join key model, but that model cannot support federation, cross-server trust, or even basic identity display in voice sessions.

Critically, the solution must not require modifications to the IRC server. Orbit's design philosophy is explicit: **Orbit is a layer on top of existing IRC.** Any IRCv3 server that supports message tags can become Orbit-enabled. The identity bridging mechanism must be a standalone component that works alongside any compliant IRC server, not a patch to one specific implementation. And it must be **optional** - if the component isn't deployed, everything else still works. The experience degrades gracefully, not catastrophically.

## Proposal

Two layers:

1. **IRC network linking** for text-layer federation (the control plane).
2. **Signed identity assertions** for decoupling Satellite nodes from IRC while preserving verifiable identity.

### Layer 1: IRC Network Linking

Leverage IRC's native server linking. IRC was designed for multi-server networks from the very beginning - this is one of the reasons it was chosen as the protocol foundation. [Ergo](https://ergo.chat/) (Ground Control's IRC server) supports server-to-server linking out of the box.

However, basic IRC linking (multiple servers forming one logical network) is not the same as true federation (independent organizations running independent servers that interoperate as peers). True federation raises hard questions:

- **Identity portability**: How does a user registered on server A prove their identity when interacting with server B? Federated identity is a deep problem (see: email, Matrix, ActivityPub - all have different trade-offs).
- **Channel namespacing**: How do you distinguish `#general` on server A from `#general` on server B? IRC traditionally uses a flat namespace. Federation requires hierarchical or scoped naming.
- **Media routing**: If users from two federated servers are in the same voice channel, which Satellite node handles the media? Do you relay between nodes? Who pays for the bandwidth?
- **Moderation boundaries**: Who has moderation authority in a federated channel? Can server A's admins moderate users from server B? What about content policies that differ between servers?
- **History synchronization**: Do federated servers share message history? How much? What happens when a server goes offline and comes back - does it backfill?

Start with the simplest model: IRC network linking where multiple Ground Control (Ergo) instances form a single logical network under shared administration. This is well-understood IRC infrastructure with decades of operational experience. True cross-organization federation (more analogous to email or Matrix) is a separate, later problem. Do not attempt to design a general federation protocol until the simpler model is deployed and its limitations are understood in practice.

### Layer 2: Signed Identity Assertions (Identity Bridging)

The identity bridging mechanism is **Transponder** - a standalone, optional identity service with a pluggable auth backend. It signs short-lived identity tokens that Satellite nodes can independently verify, and it serves as the authentication authority that Ground Control delegates to via Ergochat’s `auth-script` mechanism. Transponder is a self-contained HTTP service - it does not connect to IRC. The IRC server is not modified beyond standard configuration.

For the full specification - architecture, API contract, auth backend adapters, Ground Control and Satellite integration, and key publication - see [Transponder](../02-components/04-transponder.md).

## Verified and Unverified Users

A Satellite node is a media server. There is no requirement that every participant be an IRC user. Legitimate use cases exist for non-IRC participants: someone shares a voice room link with a friend who doesn't have an account, a BYON node operator wants open access, a website embeds a voice widget for anonymous visitors.

The Satellite token service can issue tokens in two modes:

| Mode | How they authenticate | LiveKit JWT contains | Orbit UI treatment |
|------|----------------------|---------------------|--------------------|
| **Verified** | Signed identity token from Transponder | `account: "zealsprince"`, `server: "irc.hivecom.net"`, `verified: true` | Display name + verified indicator (e.g., checkmark, badge) |
| **Unverified** | Join key, room password, or other node-level auth | `display_name: "some-name"`, `verified: false` | Display name shown, no verification badge, clear "unverified" indicator |

This is the same pattern as Jitsi (anyone can join, some users are logged in) or Bluesky (verified domain handles vs. unverified accounts). No gatekeeping - just transparency. The Orbit client displays the distinction clearly so users can make informed trust decisions.

Unverified users:
- Can join voice/video sessions if the node's auth policy allows it (join key, password, or open access).
- Have self-asserted display names. These are **not trustworthy** - the UI must never present them as equivalent to verified identities.
- Cannot impersonate a verified user. If a verified user is in the session, an unverified participant claiming the same name should be visually distinguishable.
- Are subject to the same moderation controls as anyone else in the session (mute, kick, etc.).

## Graceful Degradation

Transponder is **optional**. If a server operator doesn't deploy it, nothing breaks:

| Feature | With Transponder | Without Transponder |
|---------|-----------------|-------------------|
| Text chat | Works (pure IRC) | Works (pure IRC) |
| Group voice / video | Works, participants verified | Works, all participants unverified |
| BYON | Works, IRC users verified | Works, everyone unverified |
| Web widget | Works (has its own JWT flow via widget gateway) | Works (has its own JWT flow via widget gateway) |
| P2P calls | Works, caller identity verified | Works, caller identity unverified |

The Orbit client detects whether a Transponder is available for the current domain (via DNS SRV `_transponder._tcp`, `/.well-known/orbit/keys.json`, or DNS TXT `orbit._keys`). If none is found, the client skips the identity token step and joins Satellite sessions as an unverified participant. The UI reflects this - no verification badges for anyone, but everything functions.

## Federation Trust Chain

The signed identity model scales naturally to federation:

- **Same-server (MVP)**: Satellite trusts one Transponder's public key. Auto-configured at deployment.
- **Linked network**: Multiple Ground Control instances in a linked Ergo network share a Transponder (or run separate instances with cross-signed keys). All Satellites in the network trust the same key set.
- **True federation**: Satellites maintain a trust store of public keys from federated Transponder instances. Trust establishment can follow one of several models (to be evaluated):
  - **Manual**: server operator explicitly adds keys they trust (like SSH `known_hosts`). Most secure, highest friction.
  - **TOFU (Trust On First Use)**: accept a new server's key the first time it's encountered, warn on key change. Good balance of security and usability for small-scale federation.
  - **Directory-based**: a shared directory (e.g., the discovery service from [Server Discovery](10-server-discovery.md)) vouches for key-to-server bindings. Centralizes trust but simplifies onboarding.
  - **DNS-based**: publish the public key in a DNS TXT record, verified via DNSSEC. Decentralized, leverages existing infrastructure. Analogous to DKIM for email.

This is the same spectrum that email traversed with SPF/DKIM/DMARC and that ActivityPub is navigating now with HTTP Signatures.

## Approach

1. **Phase 0 (pre-federation)**: Implement Transponder as a standalone HTTP identity service with the internal auth backend. Configure Ergochat’s `auth-script` to delegate authentication to Transponder. Implement token verification in the Satellite token service. This replaces the "public join key" model from the MVP for verified users while keeping join key / password access for unverified users. Transponder is a self-contained service - it does not connect to IRC. This is valuable even without federation - it gives Satellite nodes verified identity and centralizes auth in a pluggable service.
2. **Phase 1 (IRC linking)**: Set up a two-server Ergo linked network. Test text federation. Both servers share the same Transponder (or run separate instances with cross-signed keys). Satellites trust both.
3. **Phase 2 (cross-org federation)**: Independent servers with independent Transponder instances and independent keys. Implement trust store management in the Satellite. Evaluate TOFU vs. directory vs. DNS-based trust models via prototype.

Do not jump to Phase 2 until Phase 1 is deployed and its limitations are understood in practice.

## Risks

- IRC server linking is historically fragile under netsplits (network partitions). When the link between servers drops and reconnects, state reconciliation (channel membership, modes, bans) can produce surprising results.
- Media federation (routing voice/video across Satellite node boundaries) is a non-goal by design. Satellite nodes do **not** communicate with each other - there is no inter-node media routing or relay. A server operator advertises one or more Satellite nodes via DNS SRV records (e.g., regional "NA" and "EU" nodes). Users choose which node to join based on region or preference. All participants in a voice session connect to the same Satellite node. Sessions are designed for small pods: **64 concurrent participants maximum** per session. This is a communication tool for communities, not a broadcasting platform.

  > **Resolved**: The session participant maximum is set at **64 concurrent participants per room**. This constraint is now established in [Satellite - Session Limits](../02-components/02-satellite.md#session-limits). Implications for capacity planning and operator guidance are covered there.

- The complexity-to-value ratio may be unfavorable. If most Orbit communities are self-contained (like most Discord servers are), federation may serve a small minority of users at significant engineering cost.
- Key management adds operational burden. Server operators need to protect signing keys, rotate them, and handle key compromise. This is table-stakes for any cryptographic identity system, but it's still work.
- The verified/unverified model could create a two-tier user experience that feels exclusionary if not handled carefully in the UI. The design must frame verification as informational, not as a status hierarchy.

## Evaluation Criteria

**Identity bridging (Phase 0):**

- Implement Transponder: Ed25519 keypair generation, HTTP API (`/auth/verify`, `/token/issue`, `/keys`), internal auth backend, public key publication endpoint.
- Configure Ergochat’s `auth-script` to delegate SASL verification to Transponder’s `/auth/verify` endpoint.
- Implement token verification in the Satellite token service.
- Verify that a user authenticated on IRC receives a verified LiveKit JWT, and that a user with only a join key receives an unverified JWT.
- Verify that the Orbit client displays verified and unverified participants distinctly.
- Measure latency overhead of the token request flow (should be negligible - one additional round-trip).

**IRC linking (Phase 1):**

- Set up a two-server Ergo linked network. Test:
  - Text chat across the link (message delivery, ordering, latency)
  - Media signaling relay (can a user on server A join a voice session hosted on server B's Satellite node, using server A's Transponder token?)
  - History synchronization (what happens to message history when the link drops and reconnects?)
  - Failure modes (what breaks during a netsplit? how does it recover?)

**Federation trust (Phase 2):**

- Test cross-server identity verification: user on server A presents a token to a Satellite that only trusts server B's key. Verify rejection. Add server A's key to the trust store. Verify acceptance.
- Evaluate TOFU, directory, and DNS-based trust establishment. Document trade-offs in operational complexity, security guarantees, and UX friction.

Identify where each phase breaks and document the gaps before proceeding to the next.

## Dependencies

- The MVP must be stable on single-server deployments. Federation on a shaky foundation is a recipe for compounding bugs.
- Phase 0 (Transponder) should be implemented early post-MVP - it improves single-server Satellite auth and is a prerequisite for all federation phases. It requires no IRC server changes. It is also optional - deployments without Transponder degrade gracefully to fully-unverified Satellite sessions.
- Phase 2 depends on [Server Discovery](10-server-discovery.md) if directory-based trust is pursued.
