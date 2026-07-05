# Transponder

Transponder is an adopted role, not software Orbit builds: it refers to
whatever OIDC-compliant identity provider the server operator deploys. This
can be [Keycloak](https://www.keycloak.org/),
[Authentik](https://goauthentik.io/), [Authelia](https://www.authelia.com/),
[Zitadel](https://zitadel.com/), or any other provider that implements
[OpenID Connect Discovery](https://openid.net/specs/openid-connect-discovery-1_0.html).
Orbit doesn't ship its own identity service. See
[Component Classes](01-overview.md#component-classes).

The operator deploys a provider, points Orbit components at its issuer URL,
and everything else - credential verification, token issuance, key
publication - is handled by the provider. Uplink, Satellite, Depot, and any
future service consume the same identity layer without custom adapters.

Transponder is optional and deployments without one degrade gracefully:
Ergo's built-in NickServ/SASL handles IRC authentication, voice and video
still function, and all Satellite participants appear unverified. The full
degradation model lives in
[Identity](09-identity.md#graceful-degradation).

OIDC flows, token refresh, provider examples, and the operational notes on
key caching and rotation live in
[Implementation - Identity](../03-implementation/05-identity.md).

## Why a Shared Identity Layer

Without an identity provider, authentication lives inside Uplink's own
boundary: SASL to Ergo, NickServ for account management, `account-tag` for
identity assertion on the IRC wire. That works for text chat, but the
assertions are server-scoped - they don't travel outside the IRC connection.
When a user connects to a Satellite, a completely separate service, the
Satellite has no way to verify that this is the same authenticated user.

The deeper problem is architectural: identity embedded inside Uplink can't be
consumed by anything else. An external OIDC provider extracts identity into a
standalone authority that all components plug into equally.

## The Verification Model

The entire system hinges on one configuration value: the OIDC issuer URL.
Every OIDC-compliant provider serves a standard discovery document that tells
any consumer where its endpoints and signing keys live. No component needs to
know which provider is behind the URL.

The user authenticates once against the provider (standard Authorization
Code flow with PKCE - the provider controls the login experience entirely:
passwords, SSO, passkeys, MFA are its configuration, not Orbit's). The
resulting signed identity token is then used across all Orbit components for
that domain. Each component verifies the token independently against the
provider's published keys (JWKS): a local cryptographic operation, no
component contacts any other component to check identity, and the provider
stays out of the hot path.

### Uplink

Stock Ergo verifies provider tokens with no fork. Two integration paths
exist; which applies depends mainly on how the provider signs its tokens.

- **Auth bridge (general-purpose, recommended).** Ergo's standard
  authentication hook calls a small, stateless service that verifies the
  token against the provider's published keys and returns the account name.
  This path works with any provider: it performs full JWKS-based
  verification, follows key rotation, supports every common signing
  algorithm (RS256, ES256, EdDSA, HS256), and validates issuer, audience,
  and expiration. Because it sits in the authentication critical path for
  every token-authenticated connection, the bridge is a production
  dependency despite its small scope, with health checking, structured
  logging, and fail-loud startup requirements specified in
  [Implementation - Identity](../03-implementation/05-identity.md).
- **Ergo's native JWT verification (constrained).** Ergo can verify tokens
  internally with no helper process, but its built-in verifier is minimal:
  HMAC, RSA, or Ed25519 signatures only (no ECDSA/ES256, which rules out
  providers like Supabase), a statically pinned key with no JWKS fetch
  (rotation is a manual config change), and validation of signature and
  expiry only - not issuer or audience.

Either way the IRC server itself is unmodified. The account name comes from a
configured claim, conventionally the OIDC preferred username; that claim MUST
be present and MUST be a valid IRC nickname, and the provider is responsible
for keeping those values unique and nickname-conformant. If the claim is
absent, authentication MUST be rejected.

### Satellite and Depot

Both follow the same pattern with no Orbit-specific adapter: fetch the
provider's published keys once, cache them, and verify incoming tokens
locally. Satellite issues verified session credentials on success and falls
back to the unverified join flow otherwise; Depot authenticates uploads as
standard Bearer token consumption.

### Failure Model

If the provider is unreachable and cached keys have expired: Uplink rejects
new token-based logins with a clear error (already-connected IRC sessions are
unaffected - authentication is checked only at connect time, and the native
verification path doesn't depend on the provider at all); Satellite falls
back to the unverified join flow; Depot rejects authenticated uploads while
public downloads continue. Nothing hangs silently, and existing sessions
survive provider downtime.

## Multi-Server Identity

An Orbit user may be connected to multiple servers simultaneously, each with
its own identity provider: one community runs Keycloak, another runs
Authentik, a third runs no provider at all (NickServ only). Each domain is
its own identity domain - one community's token means nothing to another, and
that's correct; they're separate communities with separate user databases.
The Orbit client maintains a token per domain, the same way a browser keeps
separate cookies per domain.

From the user's perspective: join a community, its login page opens, sign in
once, and you're verified everywhere on that domain - IRC, voice, uploads.
Join a second community and you sign in again, because it's a second
identity.

## Relationship to Accounts and Federation

When a provider is configured, OIDC is the authoritative source of truth for
accounts; NickServ coexists as a recovery and legacy-client layer. The full
account model - coexistence configurations, autocreation, namespace
conflicts, migration - has one home in [Identity](09-identity.md).

Because the identity layer is standard OIDC rather than a custom protocol, it
extends naturally to federation: trusting another community's users means
trusting additional issuers. See [Federation](14-federation.md).

How clients discover a domain's identity provider is part of the general
service discovery model in [Infrastructure](12-infrastructure.md).
