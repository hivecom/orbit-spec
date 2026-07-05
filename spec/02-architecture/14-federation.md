# Federation

Orbit deployments are single-server: all users in a community connect to one
instance, and users on server A can't communicate with users on server B.
Communities are islands. Federation would connect them - cross-server
communication while preserving each community's autonomy. This page records
the design and its one hard dependency.

Federation splits into two layers: identity that travels across service and
server boundaries, and text-layer linking between IRC servers. The first
already exists in Orbit's architecture; the second is blocked upstream.

## Identity Bridging

Federation requires identity assertions that travel outside a single server's
boundary. IRC's own assertions (SASL, NickServ, `account-tag`) are
server-scoped, which is exactly the problem the
[Transponder](07-transponder.md) role solves: a standard OIDC provider issues
signed tokens, and every component - Uplink, Satellite, Depot, or any future
service - verifies them independently against the provider's published keys.
There's no central broker, no token relay, and no coupling between
components, and the mechanism requires no IRC server modifications. It's
optional, and deployments without it degrade gracefully
([Identity](09-identity.md#graceful-degradation)).

That per-component, standards-based verification is the property federation
builds on.

## IRC Network Linking

IRC was designed for multi-server networks from the beginning - one of the
reasons it was chosen as the protocol foundation. But
[Ergo](https://ergo.chat/), the adopted reference server, does not support
server-to-server linking; per the Ergo manual, all clients must connect to
the same instance. Horizontal scalability is on Ergo's radar upstream but not
scheduled.

Text-layer federation therefore has no current implementation path: it isn't
achievable with stock Ergo, and there is no Orbit fork to deliver it
([Protocol Posture](02-protocol-posture.md)). It becomes reachable only if
upstream Ergo, or an alternative stock IRCv3 server, ships linking. The
design below is recorded as a conceptual target.

Basic linking (multiple servers forming one logical network under shared
administration) is well-understood IRC infrastructure with decades of
operational experience. True federation - independent organizations running
independent servers that interoperate as peers - raises hard questions on top
of it:

- **Identity portability**: how does a user registered on server A prove
  their identity to server B? Federated identity is a deep problem; email,
  Matrix, and ActivityPub all landed on different trade-offs.
- **Channel namespacing**: IRC traditionally uses a flat namespace;
  federation needs hierarchical or scoped naming to distinguish `#general`
  on server A from `#general` on server B.
- **Media routing**: if users from two federated servers share a voice
  channel, which Satellite carries the media, and who pays for the
  bandwidth?
- **Moderation boundaries**: who has authority in a federated channel, and
  what happens when content policies differ between servers?
- **History synchronization**: do federated servers share history, how much,
  and what backfills after an outage?

Neither layer should be designed in detail until a linking-capable server
exists. In the meantime, single-instance Ergo scales vertically across cores
and supports high-availability deployment via Kubernetes, which mitigates the
single-instance constraint for reliability - though not for geographic
distribution or organizational federation.

## Federation Trust Chain

The OIDC identity model scales naturally to federation, more cleanly than a
custom identity service would, because OIDC providers are already independent
HTTP services with published keys and standardized discovery:

- **Same-server**: all components point at one issuer URL, auto-configured
  at deployment.
- **Linked network**: multiple Uplink instances in a linked network share an
  OIDC provider, or run separate providers; Satellites trust the key set
  from one or both.
- **True federation**: components maintain a trust store of key endpoints
  from federated identity providers. Because every component verifies
  independently, adding trust for a new issuer is the same operation
  everywhere. Trust establishment could follow one of several models, still
  to be evaluated:
  - **Manual**: operators explicitly add trusted issuers, like SSH
    `known_hosts`. Most secure, highest friction.
  - **TOFU (trust on first use)**: accept a new issuer on first contact,
    warn on key change. A reasonable balance for small-scale federation.
  - **Directory-based**: a shared directory (such as the
    [community directory](12-infrastructure.md#community-directory)) vouches
    for issuer-to-server bindings. Centralizes trust but simplifies
    onboarding.
  - **DNS-based**: publish the issuer in a DNSSEC-verified DNS record.
    Decentralized, analogous to DKIM for email.

Because the identity layer is standard OIDC, federated trust reduces to
"trust additional issuers" - the same pattern enterprise OIDC federation
already uses, and the same spectrum email traversed with SPF/DKIM/DMARC.

## Risks

- IRC server linking is historically fragile under netsplits: when a link
  drops and reconnects, state reconciliation (membership, modes, bans) can
  produce surprising results.
- Media federation is a non-goal by design. Satellites don't communicate
  with each other - there's no inter-Satellite routing or relay. An operator
  advertises one or more Satellites (regional, for example), users choose
  one, and all participants in a session connect to the same Satellite
  ([Satellite - Session Limits](05-satellite.md#session-limits)).
- The complexity-to-value ratio may be unfavorable: if most communities are
  self-contained, federation serves a small minority at significant
  engineering cost.
- Key management adds operational burden - protecting, rotating, and
  recovering signing keys is table stakes for any cryptographic identity
  system, but it's still work.
- The verified/unverified distinction could feel exclusionary if the UI
  handles it badly; verification must read as information, not status
  ([Identity](09-identity.md#verified-and-unverified-users)).
