# Direct Messages

Direct messages in Orbit are standard IRC `PRIVMSG` to a nickname. There is no special channel
convention, no synthetic `&dm-` construct, no new primitive. If you know someone's nickname, you
send them a `PRIVMSG`. Orbit clients render incoming and outgoing `PRIVMSG` queries in a
conversation UI - but the transport is plain IRC and always has been.

If a group of people want a private conversation, they make a private channel, set it `+s +i`, and
invite the participants. That is already IRC. There is no "group DM" concept in the spec - the
tool already exists.

## Storage

DM history uses a two-tier model based on whether end-to-end encryption is active.

### Standard DMs (No E2E)

Standard DMs use **operator-configured retention** - the same model as channel history. Ergo stores
query history according to the operator's configured retention window. Offline delivery is
guaranteed for registered users via always-on mode. History is available on reconnect via
`chathistory`. Server-side search over DM history is available post-fork.

This is the default mode. DMs behave like any other conversation with history, offline delivery,
and continuity across sessions.

### E2E-Encrypted DMs

When end-to-end encryption is active between two Orbit clients, the server stores **ciphertext**
with the same retention and delivery model as standard DMs. The server holds, delivers, and
eventually expires encrypted messages exactly as it would plaintext - it just can't read them.

- Offline delivery works: the server holds ciphertext until the recipient reconnects.
- History works: `chathistory` returns encrypted messages. The client decrypts locally.
- Server-side search does **not** work: the server cannot index ciphertext. Client-side search
  over decrypted local history is the answer here.

The combination of E2E plus operator-configured retention means: the server holds ciphertext for
delivery and history, but cannot read it. When retention expires, the ciphertext is purged. The
operator has neither the key nor the content.

### Detecting E2E Status

Orbit clients determine whether a conversation is E2E-encrypted by inspecting incoming messages:

- Messages carrying a `+orbit/e2e` tag are encrypted. The client decrypts and renders them.
- Messages without the tag are plaintext. The client renders them normally.
- A conversation's E2E status is **client-derived**, not server-flagged. If recent messages carry
  `+orbit/e2e` tags, the client treats the conversation as encrypted and continues encrypting
  outbound messages.

When E2E is first established, the client inserts a visible marker in the conversation:
*"🔒 Messages from this point are end-to-end encrypted."* Previous messages remain in whatever
state they were in. This is explicit - the user always knows which messages are protected.

See [Research: E2E Encryption](../../../06-next/05-e2e-encryption.md) for the key exchange
protocol, cross-device key synchronization, and the `+orbit/e2e` tag design.

## Always-On Mode

Orbit deployments MUST enable Ergo's always-on mode (`accounts.multiclient.always-on`) for
registered users. Without it, a `PRIVMSG` sent while the recipient is disconnected is lost. This
is not acceptable for a platform targeting mainstream users.

In always-on mode, the user's session stays alive on the server while they are disconnected. DMs
are delivered into that session and held until the client reconnects. For standard DMs, this means
the message is stored in plaintext during the delivery window. For E2E DMs, the server holds
ciphertext it cannot read.

## Eavesdropping

Operator-configured retention does not mean zero risk. Standard DMs travel through the Uplink
server in plaintext. The server operator can, in principle, read content at the server level. This
is the same trust model as channels - you are trusting the server you joined.

**The real answer to eavesdropping is end-to-end encryption.** When two Orbit clients exchange
E2E-encrypted DMs, the server sees only ciphertext. The operator cannot read the content even if
they inspect the database. Users who need confidentiality should enable E2E. Users who don't
need confidentiality get the convenience of server-side history and search.

This does not apply to channels. Channels are intentionally server-readable. Encrypting a channel
would break the features that make channels useful: history, search, threads, moderation.
The trust model for a channel is simple - you are trusting the server you joined. Choose your
server accordingly.

E2E encryption for DMs is tracked in [Research: E2E Encryption](../../../06-next/05-e2e-encryption.md)
and is not part of the MVP. For the MVP, the honest position is: DM content is stored with
operator-configured retention, and it is not encrypted. Users who need confidentiality before E2E
ships should be aware of this.

## Ergo Configuration

DM query history uses Ergo's standard retention configuration:

```yaml
history:
  enabled: true
  channel-length: 2048        # channel history
  query-length: 2048          # DM query history: enabled, operator-configured
```

Always-on mode MUST be enabled:

```yaml
accounts:
  multiclient:
    always-on: opt-out        # registered users are always-on by default
```

This is standard Ergo configuration. No patches, no plugins.

## Cross-References

- [Uplink Overview](01-overview.md) - always-on mode configuration
- [Research: E2E Encryption](../../../06-next/05-e2e-encryption.md) - end-to-end encryption
  for DMs, key exchange, cross-device sync
