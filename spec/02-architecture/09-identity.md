# Identity and Permissions

Authentication in Orbit is built on standard OIDC. The Orbit client
authenticates against the domain's identity provider (the
[Transponder](07-transponder.md) role) using the Authorization Code flow with
PKCE, obtains a signed identity token, and uses it across all Orbit
components - Uplink, Satellite, and Depot. One login, verified everywhere on
that domain: each component verifies the token independently against the
provider's published keys, per the
[verification model](07-transponder.md#the-verification-model).

For deployments without an identity provider, Ergo's built-in NickServ/SASL
provides the IRC authentication path; identity verification on Satellite and
Depot is unavailable. See [Graceful Degradation](#graceful-degradation).

The OIDC flow diagrams, token refresh behavior, and auth wiring live in
[Implementation - Identity](../03-implementation/05-identity.md).

## NickServ and the Identity Provider

When an identity provider is configured, OIDC is the authoritative source of
truth for accounts: a valid token wins, and the user's provider account maps
directly to their IRC account via the configured username claim.

That doesn't require disabling NickServ. The two layers coexist cleanly
because they own different things: OIDC owns identity and login; NickServ
provides a compatibility and recovery surface OIDC structurally can't -
legacy SASL identify, self-serve renames, and email-based password recovery.
Both configurations are supported:

| Configuration | Account management | Credential verification | Nickname enforcement |
|---------------|-------------------|------------------------|---------------------|
| **No identity provider** | NickServ handles registration, password changes, email verification | Ergo's built-in SASL against NickServ's database | NickServ enforces registered nicknames |
| **Provider configured (coexistence, recommended)** | OIDC owns identity; accounts are autocreated on first token login. NickServ remains for recovery and legacy identify | Token verified by the auth bridge or Ergo's native verification; NickServ verifies legacy identify | Ergo enforces registered nicknames - enforcement is tied to account login, not to NickServ specifically |
| **Provider configured (strict single-source)** | OIDC owns everything; NickServ registration disabled | Auth bridge or native verification | Ergo enforces registered nicknames |

**Autocreation** is a requirement of the coexistence model: Ergo MUST be
configured to autocreate accounts, so a persistent account and nickname
reservation are established on a user's first OIDC login. Without it, OIDC
users may get neither.

**Account claim.** An OIDC-autocreated account starts without a recovery
email; Orbit clients surface a non-blocking claim flow to establish one - see
[Services - Account Claim](08-services.md#account-claim-recovery-readiness).

### Namespace Conflicts

Conflicts are possible but self-inflicted, and they don't resolve themselves
the way one might expect. If a NickServ account registers a nick before an
OIDC user first claims it, the OIDC login doesn't transparently take over the
name: autocreation resolves the username to the existing account and logs the
OIDC user into it, while the squatter's stored password remains valid. Both
can then authenticate, and both sessions can attach to the same account
simultaneously. This is effective co-ownership, not an automatic win.

The detectable signal is email divergence: the OIDC user's verified provider
email won't match the email on the squatted NickServ record. Orbit clients
surface the mismatch as an account-integrity warning and prompt a re-claim,
which syncs the email; fully evicting the prior credential additionally
requires overwriting the stored password through the recovery flow or an
operator clearing it. There is no automatic eviction - token verification
only resolves the account name and can't mutate IRC state.

Operators who want to eliminate this risk entirely choose the strict
single-source configuration, where no one can pre-register a nick.

### Migration from NickServ to a Provider

1. Import existing NickServ accounts into the provider, or connect the
   provider to the same backing store.
2. Enable autocreation and configure the verification path
   ([Transponder - Uplink](07-transponder.md#uplink)).
3. Decide on coexistence (keep NickServ for recovery and legacy clients) or
   strict mode (disable NickServ registration).
4. Existing nickname reservations continue to work - Ergo's enforcement
   mechanism is unchanged; only the credential verification path is swapped.

## Legacy IRC Clients

Traditional IRC clients can't perform the OIDC browser flow, and there's no
expectation that they do: tokens are long, short-lived, and
awkward to paste as a password, so making a user fetch one out-of-band is
unacceptable UX. Legacy clients authenticate via NickServ identify, which
bypasses token verification entirely and works regardless of provider
configuration. That is the contract, and a primary reason to keep NickServ in
coexistence mode.

## Anonymous Users

The Orbit web client (embedded or full web app) connects directly to Ergo's
WebSocket endpoint, the same path as the desktop client - no middleware
proxy. Anonymous guests connect via SASL ANONYMOUS and receive an
automatically assigned nickname under a reserved guest prefix. No account
creation, no backend, no token, no session state. The guest prefix is
reserved so no registered user can collide with it, and any IRC client -
including third-party web UIs - can connect the same way: Orbit doesn't
gatekeep access to a standard IRC server.

Anonymous users have no persistent identity, which is why they can't
participate in E2E-encrypted DMs ([E2E Encryption](13-e2e.md#risks)) and
can't upload files ([Depot](06-depot.md#guest-users)).

## Permissions

Orbit uses IRC's built-in channel modes. Period.

| IRC Mode | Role         | Capabilities                                                |
|----------|--------------|-------------------------------------------------------------|
| `+o`     | Operator     | Kick, ban, manage channel settings, manage messages         |
| `+v`     | Voiced       | Speak in moderated channels                                 |
| `+b`     | Banned       | Cannot join or speak in the channel                         |
| (none)   | Default user | Read and send messages in unmoderated channels              |

There are no custom roles, no role colors, no granular permission overrides.
This is an opinionated decision: IRC has a proven, battle-tested permissions
model that covers the needs of the vast majority of communities.

Channel modes are ephemeral - they live only while the channel is in memory.
Persistent channel state (founder ownership, standing grants, standing bans,
survival across restarts) is owned by ChanServ; Orbit keeps live moderation
on raw channel modes and maps persistent administration to ChanServ behind a
channel-settings UI. See [Services - ChanServ](08-services.md#chanserv).

Communities that need more - role hierarchies, per-channel upload limits,
auto-mod rules - run an IRC bot: a bot connected to Uplink can enforce
arbitrarily complex rules by mapping accounts to roles and applying channel
modes, the way IRC communities have managed roles for decades, on any IRCv3
server. The concrete pattern lives in
[Implementation - Identity](../03-implementation/05-identity.md). The Orbit
extension API adds client-side UI that pairs with such bots - see
[Extensions](15-extensions.md).

How clients display and enforce identity (the `account-tag` rules) lives in
[Tags and Trust](04-tags-and-trust.md#identity-display).

## Verified and Unverified Users

There's no requirement that every participant be an authenticated user.
Legitimate unverified use exists: a friend joins a voice room from a shared
link without an account, a BYOS operator wants open access, a website embeds
voice for anonymous visitors. The Satellite issues session credentials in two
modes:

| Mode | How they authenticate | Session identity | Orbit UI treatment |
|------|----------------------|------------------|--------------------|
| **Verified** | Signed token from the OIDC provider | Account name and home domain, marked verified | Display name plus a verified indicator |
| **Unverified** | Join key, room password, or open access | Self-asserted display name, marked unverified | Display name shown, no badge, clear "unverified" indicator |

Unverified users:

- Can join voice and video sessions if the Satellite's access policy allows
  it, and publish media subject to the Satellite's guest publish policy
  (publish-enabled by default - see
  [Satellite](05-satellite.md#authentication)).
- Have self-asserted display names, which are not trustworthy - the UI must
  never present them as equivalent to verified identities.
- Can't impersonate a verified user: an unverified participant claiming the
  same name as a verified one must be visually distinguishable.
- Are subject to the same session moderation controls as anyone else.

The verification must be framed as informational, not as a status hierarchy -
a two-tier feel in the UI is a real product risk.

## Graceful Degradation

An identity provider is optional. If an operator doesn't deploy one, nothing
breaks:

| Feature | With identity provider | Without identity provider |
|---------|----------------------|--------------------------|
| Text chat | Works - Ergo verifies provider tokens | Works - Ergo uses built-in NickServ/SASL |
| Group voice / video | Works, participants verified | Works, everyone unverified |
| BYOS | Works, participants verified | Works, everyone unverified |
| Embedded client | Works (guests use SASL ANONYMOUS regardless) | Works (guests use SASL ANONYMOUS regardless) |
| P2P calls | Works, caller identity verified | Works, caller identity unverified |

The client detects whether a provider is available for the current domain via
service discovery ([Infrastructure](12-infrastructure.md)). If none is found,
it skips the identity token step, joins Satellite sessions unverified, and
hides verification UI rather than presenting a broken state. Without a
provider there's no single sign-on across components and Depot can't
attribute uploads to an identity - an acceptable configuration for
communities that don't need verified identity.
