# IRC Services Abstraction

Ergo ships built-in services: NickServ (account management), ChanServ
(channel management), and HistServ (history access). They expose the standard
IRC services interface - a user sends commands to a service bot and reads its
replies as notices - and decades of IRC clients drive them by typing raw
commands.

Orbit treats services as implementation details of the Uplink deployment.
Orbit clients express user intent (claim my account, stay reachable offline,
ban a user, register a channel) and translate it into the appropriate service
commands behind the scenes. Users never type service commands and never see
raw service notice traffic unless they explicitly opt into a power-user/raw
mode.

This page defines which service interactions Orbit abstracts and how. It
complements [Transponder](07-transponder.md) (the OIDC identity role) and
[Identity](09-identity.md) (the account and permission model). The
account-claim sequence, service-notice routing table, and conformance
mechanics live in
[Implementation - Identity](../03-implementation/05-identity.md).

## Design Principles

1. **Intent over commands.** UI surfaces actions ("Claim account", "Stay
   online", "Promote to operator"); the client maps each to the underlying
   service commands. Command syntax is never the contract the user sees.
2. **Silence the service chatter.** Service replies triggered by
   client-initiated background operations MUST NOT open a query buffer or
   pollute the log - they're parsed silently and reflected in UI state.
   Unsolicited service notices MAY be surfaced, but SHOULD be rendered as
   structured UI, not raw text.
3. **Services are not the source of truth for identity.** When an identity
   provider is configured, OIDC is authoritative (see
   [Transponder](07-transponder.md)); NickServ is a coexisting compatibility
   and recovery layer.
4. **Degrade honestly.** On a deployment with no identity provider, the same
   UI actions map directly onto NickServ/ChanServ. The abstraction is
   identical; only the backing authority changes.
5. **Never gatekeep raw IRC.** A power user or a third-party IRC client can
   always drive the services directly. Orbit's abstraction is additive,
   never a lock-in.

## NickServ

NickServ owns the IRC account: registration, credentials, email, and
per-account settings such as always-on. In an OIDC deployment the account is
autocreated on first provider login (see
[Identity](09-identity.md#nickserv-and-the-identity-provider)), and NickServ
remains the surface for what OIDC doesn't own: offline delivery configuration
and email-based account recovery for legacy clients.

### Account Claim (Recovery Readiness)

An OIDC-autocreated account starts with no email on its NickServ record. The
presence of a verified email is the signal that the account is "claimed" -
recoverable through NickServ's password-recovery flow from a legacy IRC
client, independent of the identity provider.

The client probes account state silently on connect. If the account is
unclaimed, it surfaces a non-blocking prompt to claim it by setting and
verifying an email. The email is the user's choice (the client SHOULD prefill
it from the provider's email claim), and it MUST NOT be treated as an
identity assertion - it's a recovery channel (see
[Metadata Is Not an Identity Signal](#metadata-is-not-an-identity-signal)).

Email match is the account-integrity signal. Comparing the NickServ email
against the provider's email claim is how the client detects an account that
wasn't cleanly claimed - most importantly the co-ownership case where someone
registered the nick before the OIDC user first logged in (see
[Identity](09-identity.md#namespace-conflicts)). The client SHOULD surface
three states:

- **Claimed (in sync):** NickServ email present and equal to the provider
  email. No action.
- **Mismatch:** NickServ email present but different - warn prominently and
  offer a re-claim. This is the squatting/co-ownership tell.
- **Unclaimed:** no NickServ email - offer the claim flow.

The exact probe-and-claim sequence, and why the claim path is the only
self-service route to a password on an OIDC-origin account, live in
[Implementation - Identity](../03-implementation/05-identity.md).

### Always-On (Offline Delivery)

[Messaging](10-messaging.md#always-on-mode) requires always-on mode so DMs
aren't lost while a registered user is disconnected; deployments MUST default
registered users to always-on. The client therefore should never need to
enable it on a correctly configured deployment, but it SHOULD:

- Surface the current state as plain language ("You stay reachable while
  offline"), not a raw setting value.
- Warn when always-on is off for a registered user, since that contradicts
  the deployment requirement and means missed DMs. A disable toggle is an
  acceptable user choice, but "off" is a warning state, not a neutral one.

### What Stays Raw

OIDC users have no reason to ever type a NickServ command - registration,
password, MFA, and renames live in the identity provider. Legacy IRC clients
continue to use NickServ identify and recovery commands directly; that path
is unchanged and is the reason the email claim matters.

## ChanServ

ChanServ owns persistent channel state. This is the gap raw channel modes
don't cover: [Identity](09-identity.md#permissions) scopes Orbit to IRC's
built-in channel modes, which are ephemeral - they live only while
the channel is in memory. ChanServ is what makes a channel survive a restart,
remembers the founder, and re-applies standing grants and bans when the
channel is recreated.

| Concern | Layer |
|---|---|
| Live moderation (op, voice, kick, ban) | Channel modes, raw IRC (see [Identity](09-identity.md#permissions)) |
| Channel survives restart / zero members | ChanServ registration |
| Founder and standing operator grants | ChanServ access list |
| Standing bans (re-applied on rejoin) | ChanServ auto-kick list |
| Topic, entry message, channel settings | ChanServ settings |

The Orbit abstraction: live moderation is intent-mapped to channel modes,
which works on any IRCv3 server; persistent channel administration ("make
this channel permanent", "permanent operator", "permanent ban", channel
settings) is intent-mapped to ChanServ, reserved for founders/operators and
surfaced in a channel-settings panel, not a command line. Reply notices are
suppressed the same way as NickServ's - the client confirms success or
failure through UI state.

## HistServ

HistServ exposes stored history through a service command interface. Orbit
doesn't use it as a user-facing surface: history retrieval is handled by the
`chathistory` capability (see [Uplink](03-uplink.md)), which the client
drives natively. HistServ messages are treated as service chatter - filtered
out of unread/mention/badge state and never rendered as ordinary chat.

## Metadata Is Not an Identity Signal

Ergo's metadata store (avatars, display names, status - see
[Messaging](10-messaging.md#presence)) is a per-account key/value store that
users set themselves. It's the right surface for cosmetic, cross-client
profile signals, and it is explicitly not a claim or identity-verification
signal - anyone can set their own metadata. The authoritative identity signal
is the server-asserted `account-tag` (see
[Tags and Trust](04-tags-and-trust.md#identity-display)); the recovery signal
is the verified NickServ email. Clients MUST NOT conflate metadata with
either.
