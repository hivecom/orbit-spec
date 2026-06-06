# Federation

## Overview

> **Status: deferred / exploratory.** Federation is not a planned track for now. The text-layer of federation depends on IRC server-to-server linking, which stock IRCv3 servers (including Ergo) do not provide, and there is no planned Orbit fork to deliver it. The identity layer below (Layer 2) is real, standards-based, and useful on its own today, but cross-server federation as a whole has no current implementation path. This document records the conceptual design so it is ready if an implementation path appears, not because it is scheduled.

The MVP is single-server. All users in a community connect to one Orbit server instance. Users on server A cannot communicate with users on server B. Communities are isolated islands. Federation will connect these islands, enabling cross-server communication while preserving each community's autonomy.

A key requirement for federation is **identity bridging**. IRC handles user authentication within its own protocol boundary (SASL, NickServ, `account-tag`), but those assertions are server-scoped. They don't travel outside the IRC connection. When a user connects to a Satellite (a completely separate service), the Satellite has no native way to verify that this person is the same user who is authenticated on IRC. The MVP uses a public join key model, but that model cannot support federation, cross-server trust, or even basic identity display in voice sessions. Identity bridging solves this.

The identity bridging mechanism will not require modifications to the IRC server. Orbit's design philosophy is explicit: **Orbit is a layer on top of existing IRC.** Any IRCv3 server that supports message tags can become Orbit-enabled. The identity bridging mechanism is a standalone component that works alongside any compliant IRC server, not a patch to one specific implementation. And it is **optional** - if the component isn't deployed, everything else still works. The experience degrades gracefully, not catastrophically.

## Design

Two layers:

1. **IRC network linking** for text-layer federation (the control plane).
2. **Signed identity assertions** for decoupling Satellite from IRC while preserving verifiable identity.

### Layer 1: IRC Network Linking

IRC was designed for multi-server networks from the very beginning - this is one of the reasons it was chosen as the protocol foundation. However, [Ergo](https://ergo.chat/) (Uplink's adopted IRC server) **does not currently support server-to-server linking**. Per the Ergo manual: *"Ergo does not currently support server-to-server linking (federation), meaning that all clients must connect to the same instance."* Horizontal scalability is on the Ergo roadmap but is not scheduled for development in the near term.

Layer 1 is therefore not achievable with stock Ergo today, and there is **no planned Orbit fork** to deliver it. Orbit adopts a stock IRCv3 server rather than maintaining a fork (see [Design Philosophy - Where Orbit's Value Lives](../01-architecture/02-philosophy.md#where-orbits-value-lives)), so server-to-server linking has no current implementation path on Orbit's side. It would arrive only if upstream Ergo (or an alternative stock IRCv3 server) implements linking. Until then, Layer 1 - and federation as a whole - is deferred. The design below is recorded as a conceptual target, not as scheduled work.

Basic IRC linking (multiple servers forming one logical network) is not the same as true federation (independent organizations running independent servers that interoperate as peers). True federation raises hard questions:

- **Identity portability**: How does a user registered on server A prove their identity when interacting with server B? Federated identity is a deep problem (see: email, Matrix, ActivityPub - all have different trade-offs).
- **Channel namespacing**: How do you distinguish `#general` on server A from `#general` on server B? IRC traditionally uses a flat namespace. Federation requires hierarchical or scoped naming.
- **Media routing**: If users from two federated servers are in the same voice channel, which Satellite handles the media? Do you relay between Satellites? Who pays for the bandwidth?
- **Moderation boundaries**: Who has moderation authority in a federated channel? Can server A's admins moderate users from server B? What about content policies that differ between servers?
- **History synchronization**: Do federated servers share message history? How much? What happens when a server goes offline and comes back - does it backfill?

The simplest conceivable model would be IRC network linking where multiple Uplink instances form a single logical network under shared administration. This is well-understood IRC infrastructure with decades of operational experience, but it still requires a server that supports linking, which the adopted stock server does not. True cross-organization federation (more analogous to email or Matrix) is a separate, even harder problem. Neither should be designed in detail until a linking-capable server exists.

For the MVP, Uplink is single-instance Ergo. Ergo scales vertically across multiple CPU cores and supports high-availability deployment via Kubernetes (shared volume, load balancer), which mitigates the single-instance constraint for reliability - but not for geographic distribution or organizational federation. Server-to-server linking is not on Orbit's roadmap and is not an MVP requirement.

### Layer 2: Signed Identity Assertions (Identity Bridging)

The identity bridging mechanism is **Transponder** - a role filled by any OIDC-compliant identity provider (e.g., Keycloak, Authentik, Zitadel). The provider issues signed JWTs via standard OpenID Connect flows. Each Orbit component that needs to verify identity - Uplink, Satellite, Depot, or anything added in the future - independently verifies those JWTs against the provider's published keys (JWKS). No component contacts any other component to check identity; each one fetches the provider's JWKS endpoint and performs local cryptographic verification.

This means identity assertions are **per-component and independent**. If an Orbit-compatible service points at an OIDC issuer URL, it can verify user identity - regardless of whether it's an Uplink instance, a Satellite, a Depot server, or a third-party service built on the Orbit ecosystem. There is no central broker, no token relay, and no coupling between components. The OIDC provider is the single source of truth; every consumer verifies against it directly.

- **Uplink** verifies identity natively. Stock Ergo supports OIDC/JWT authentication directly via OAUTHBEARER and IRCV3BEARER SASL with `accounts.jwt-auth`/`oauth2`, validating tokens against the provider's JWKS with no extra component. This is the primary integration. The legacy `auth-script` bridge remains available as an optional compatibility fallback for setups that need it. Either way, the IRC server is not modified beyond standard configuration.
- **Satellite** verifies identity tokens directly against the JWKS endpoint. No Orbit-specific code - standard JWT verification.
- **Depot** verifies Bearer tokens against the same JWKS endpoint. Same pattern, same keys.
- **Any future component** follows the same model: point at the OIDC issuer URL, fetch the JWKS, verify JWTs locally. That's it.


For the full specification - OIDC discovery, component integration flows, native Ergo JWT verification, the optional auth-script bridge, and the Keycloak deployment example - see [Transponder](../02-components/04-transponder.md).

## Verified and Unverified Users

A Satellite is a media service. There is no requirement that every participant be an IRC user. Legitimate use cases exist for non-IRC participants: someone shares a voice room link with a friend who doesn't have an account, a BYOS Satellite operator wants open access, a website embeds a voice widget for anonymous visitors.

The Satellite token service can issue tokens in two modes:

| Mode | How they authenticate | LiveKit JWT contains | Orbit UI treatment |
|------|----------------------|---------------------|--------------------|
| **Verified** | Signed JWT from the OIDC identity provider, verified against JWKS | `account: "zealsprince"`, `server: "irc.hivecom.net"`, `verified: true` | Display name + verified indicator (e.g., checkmark, badge) |
| **Unverified** | Join key, room password, or other node-level auth | `display_name: "some-name"`, `verified: false` | Display name shown, no verification badge, clear "unverified" indicator |

This is the same pattern as Jitsi (anyone can join, some users are logged in) or Bluesky (verified domain handles vs. unverified accounts). No gatekeeping - just transparency. The Orbit client displays the distinction clearly so users can make informed trust decisions.

Unverified users:
- Can join voice/video sessions if the node's auth policy allows it (join key, password, or open access).
- Have self-asserted display names. These are **not trustworthy** - the UI must never present them as equivalent to verified identities.
- Cannot impersonate a verified user. If a verified user is in the session, an unverified participant claiming the same name should be visually distinguishable.
- Are subject to the same moderation controls as anyone else in the session (mute, kick, etc.).

## Graceful Degradation

An identity provider is **optional**. If a server operator doesn't deploy one, nothing breaks:

| Feature | With Identity Provider | Without Identity Provider |
|---------|----------------------|--------------------------|
| Text chat | Works - Ergo verifies OIDC/JWT natively (auth-script bridge optional fallback) | Works - Ergo uses built-in NickServ/SASL |
| Group voice / video | Works, participants verified | Works, all participants unverified |
| BYOS | Works, IRC users verified | Works, everyone unverified |
| Web widget | Works (guests use SASL ANONYMOUS regardless) | Works (guests use SASL ANONYMOUS regardless) |
| P2P calls | Works, caller identity verified | Works, caller identity unverified |

The Orbit client detects whether an identity provider is available for the current domain (via `/.well-known/orbit/oidc` or DNS SRV `_transponder._tcp`). If none is found, the client skips the identity token step and joins Satellite sessions as an unverified participant. The UI reflects this - no verification badges for anyone, but everything functions.

## Federation Trust Chain

The signed identity model scales naturally to federation:

- **Same-server**: All components point at one OIDC issuer URL. Each component independently fetches the JWKS and verifies tokens locally. Auto-configured at deployment.
- **Linked Uplink network**: Multiple Uplink instances in a linked Ergo network share an OIDC provider, or run separate providers. All Satellites in the network trust the JWKS from one or both issuers.
- **True federation**: Components maintain a trust store of JWKS endpoints from federated identity providers. Because every component verifies independently, adding trust for a new issuer is the same operation everywhere - add the issuer URL to the trust list, fetch its JWKS. Trust establishment can follow one of several models (to be evaluated):
  - **Manual**: server operator explicitly adds keys they trust (like SSH `known_hosts`). Most secure, highest friction.
  - **TOFU (Trust On First Use)**: accept a new server's key the first time it's encountered, warn on key change. Good balance of security and usability for small-scale federation.
  - **Directory-based**: a shared directory (e.g., the discovery service from [Server Discovery](10-server-discovery.md)) vouches for key-to-server bindings. Centralizes trust but simplifies onboarding.
  - **DNS-based**: publish the public key in a DNS TXT record, verified via DNSSEC. Decentralized, leverages existing infrastructure. Analogous to DKIM for email.

Because the identity layer is standard OIDC - not a custom protocol - federated trust is just "trust additional issuers." This is the same pattern used by OIDC federation in enterprise environments (multi-tenant Keycloak, Azure AD B2B, etc.) and the same spectrum that email traversed with SPF/DKIM/DMARC.

## Rollout Plan

1. **Phase 0 (identity bridging, achievable today)**: Deploy an OIDC-compliant identity provider (e.g., Keycloak). Configure Ergo to verify OIDC/JWT natively via OAUTHBEARER/`jwt-auth` (the optional auth-script bridge is available as a fallback). Point Satellite and Depot at the same OIDC issuer URL for JWT verification against the provider's JWKS. This replaces the "public join key" model from the MVP for verified users while keeping join key / password access for unverified users. This is valuable on its own, independent of federation - it gives every component verified identity via standard OIDC, with each component verifying independently. Phase 0 is native and unblocked.
2. **Phase 1 (IRC linking) - blocked**: Set up a two-server linked network. This phase cannot start because no stock IRCv3 server Orbit can adopt supports server-to-server linking, and there is no planned Orbit fork to add it. It would become reachable only if upstream Ergo (or an alternative stock server) ships linking. Treat as deferred.
3. **Phase 2 (cross-org federation) - blocked**: Independent servers with independent OIDC providers and independent keys, with trust store management in each component that verifies identity (Satellite, Depot, etc.) and an evaluation of TOFU vs. directory vs. DNS-based trust models. Depends on Phase 1, which is itself blocked.

Phases 1 and 2 are not on the roadmap. Phase 0 is the only part that proceeds today.

## Risks

- IRC server linking is historically fragile under netsplits (network partitions). When the link between servers drops and reconnects, state reconciliation (channel membership, modes, bans) can produce surprising results.
- Media federation (routing voice/video across Satellite boundaries) is a non-goal by design. Satellites do **not** communicate with each other - there is no inter-Satellite media routing or relay. A server operator advertises one or more Satellites via DNS SRV records (e.g., regional "NA" and "EU" Satellites). Users choose which Satellite to join based on region or preference. All participants in a voice session connect to the same Satellite. Sessions are designed for small-to-medium groups, bounded by the node's hardware capacity. This is a communication tool for communities, not a broadcasting platform.

  > **Resolved**: Session capacity is hardware-bounded with no hardcoded cap. See [Satellite - Session Limits](../02-components/02-satellite.md#session-limits) for capacity guidance.

- The complexity-to-value ratio may be unfavorable. If most Orbit communities are self-contained (like most Discord servers are), federation may serve a small minority of users at significant engineering cost.
- Key management adds operational burden. Server operators need to protect signing keys, rotate them, and handle key compromise. This is table-stakes for any cryptographic identity system, but it's still work.
- The verified/unverified model could create a two-tier user experience that feels exclusionary if not handled carefully in the UI. The design must frame verification as informational, not as a status hierarchy.

## Evaluation Criteria

**Identity bridging (Phase 0):**

- Deploy an OIDC provider (e.g., Keycloak) with an `orbit` realm/tenant, create an `orbit-client` application with Authorization Code + PKCE flow.
- Configure Ergo's native OIDC/JWT verification (OAUTHBEARER/`jwt-auth`) against the provider's JWKS; optionally deploy the auth-script bridge as a compatibility fallback.
- Configure Satellite and Depot to verify JWTs against the provider's JWKS endpoint.
- Verify that a user authenticated on IRC receives a verified LiveKit JWT, and that a user with only a join key receives an unverified JWT.
- Verify that the Orbit client displays verified and unverified participants distinctly.
- Measure latency overhead of the token request flow (should be negligible - one additional round-trip).

**IRC linking (Phase 1):**

> **Blocked: no server-to-server linking available.** Stock IRCv3 servers (including Ergo) do not provide server linking, and there is no planned Orbit fork to add it. Phase 1 evaluation cannot proceed until a linking-capable stock server exists. See [Design Philosophy - Where Orbit's Value Lives](../01-architecture/02-philosophy.md#where-orbits-value-lives) for the no-fork rationale.

- Set up a two-server linked network. Test:
  - Text chat across the link (message delivery, ordering, latency)
  - Media signaling relay (can a user on server A join a voice session hosted on server B's Satellite, using server A's OIDC-issued JWT?)
  - History synchronization (what happens to message history when the link drops and reconnects?)
  - Failure modes (what breaks during a netsplit? how does it recover?)

**Federation trust (Phase 2):**

- Test cross-server identity verification: user on server A presents a JWT to a Satellite that only trusts server B's OIDC issuer. Verify rejection. Add server A's issuer to the trust store. Verify acceptance.
- Evaluate TOFU, directory, and DNS-based trust establishment. Document trade-offs in operational complexity, security guarantees, and UX friction.

Identify where each phase breaks and document the gaps before proceeding to the next.

## Dependencies

- The MVP must be stable on single-server deployments. Federation on a shaky foundation is a recipe for compounding bugs.
- Phase 0 (OIDC identity provider) ships with the MVP as an optional component. It improves single-server Satellite auth and is the only federation-adjacent phase that is achievable today. It requires no IRC server changes - Ergo verifies OIDC/JWT natively. It is also optional - deployments without an identity provider degrade gracefully to fully-unverified Satellite sessions.
- Phases 1 and 2 depend on a stock IRCv3 server that supports server-to-server linking. No such server is available to Orbit, and no Orbit fork is planned, so these phases have no current implementation path.
- Phase 2 would additionally depend on [Server Discovery](06-server-discovery.md) if directory-based trust is pursued.
