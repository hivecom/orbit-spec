# End-to-End Encryption

For the identity model that E2E encryption depends on, see
[Authentication](../03-identity/01-authentication.md).

## Scope

E2E encryption in Orbit applies to exactly two contexts:

- **1:1 DMs** (`PRIVMSG` between two registered users)
- **P2P calls** (WebRTC direct connections via `+orbit/p2p-offer` / `+orbit/p2p-answer`)

It intentionally does not apply to channels or group voice sessions.

E2E-encrypted DMs coexist with the server's standard DM storage model. The server stores and
delivers encrypted messages using the same retention and delivery mechanisms as plaintext DMs -
it just can't read the content. Offline delivery works (the server holds ciphertext until the
recipient reconnects). History works (`chathistory` returns encrypted messages; the client decrypts
locally). Server-side search does not work (the server cannot index ciphertext); client-side search
over decrypted local history is the answer. See [DMs](../02-components/01-uplink/03-dms.md) for the
two-tier storage model.

**Channels** are server-mediated by design. History, search, and threads all require the
server to read the content. A channel is a room the server runs. If you join it, you trust the
server operator. E2E encryption on a channel would silently break every server-side feature users
depend on. It is not a trade-off worth making.

**Group voice via Satellite** routes media through an SFU. The SFU must be able to forward media
streams - it cannot do that with E2E encrypted content. If you need a private call with no server
in the path, use P2P. That is exactly what P2P is for.

The rule is consistent across the entire stack: **if a server is mediating the content, there is
no E2E. If it is point-to-point, there is.**

## Transport Security vs. E2E Security

These are distinct and both matter:

**Transport security (TLS)** is already in place for every connection in the stack:
- Uplink: IRC over TLS (port 6697), WSS for web clients
- Satellite: HTTPS for the token service, DTLS-SRTP for WebRTC media (mandatory by the WebRTC spec)
- Depot: HTTPS only
- Auth: OIDC over HTTPS, SASL over TLS

TLS protects against network eavesdroppers. Nobody on the wire between the client and server can
read the content.

**E2E encryption** protects against the server operator. The server terminates TLS, so the operator
can see plaintext content at the server level. For channels, this is accepted. For 1:1
DMs, E2E means the operator sees ciphertext it cannot read. The server stores and delivers
encrypted messages using operator-configured retention - the same mechanics as plaintext DMs - but
cannot read the content.

The combination for 1:1 DMs: TLS in transit + E2E encryption means the server operator has neither
the plaintext nor the key. Ciphertext is stored for offline delivery and history, but is purged
when the operator's retention window expires.

## Protocol

**Double Ratchet (Signal Protocol)** via `libsignal-protocol` (Rust) is the protocol for 1:1 DMs:

- Well-proven in production at scale
- Forward secrecy: past messages are safe even if the current key is compromised
- Post-compromise security: the ratchet heals after a key compromise
- Mature Rust implementation exists and is maintained by Signal

MLS (RFC 9420) is out of scope. It is designed for group E2E encryption. Since group E2E is not in
scope, MLS adds complexity with no benefit.

## Key Transport

The Double Ratchet requires an initial key exchange. For Orbit, this uses the existing IRC
connection as the key exchange channel:

1. Both users publish their identity public keys via `draft/metadata-2` (`orbit.identity-key`
   metadata key). Keys are server-visible but that is fine - public keys are public.
2. On the first DM, the initiating client performs an X3DH key agreement using the recipient's
   published identity key and a one-time prekey (also published via metadata).
3. The initial encrypted message is sent as a `PRIVMSG` with a `+orbit/dm-init` tag carrying the
   ratchet header. Subsequent messages carry `+orbit/dm` tags with ratchet state.
4. The recipient's client completes the X3DH agreement and derives the same root key. The ratchet
   is established.

### The `+orbit/e2e` Tag

E2E-encrypted messages are transported as normal `PRIVMSG` where the body is ciphertext. The
`+orbit/e2e` tag is attached to each encrypted message and contains the sender's key ID
(fingerprint), which the recipient uses to select the correct decryption key and ratchet state.

```
@+orbit/e2e=<base64-key-id> PRIVMSG bob :<base64-ciphertext>
```

Orbit clients inspect incoming DMs for this tag:

- Present: decrypt the body using the corresponding ratchet state, render the plaintext.
- Absent: render the message body as-is (plaintext DM).

A conversation's E2E status is **client-derived**. If recent messages carry `+orbit/e2e` tags, the
client treats the conversation as encrypted and continues encrypting outbound messages. When E2E is
first established, the client inserts a visible marker: *"🔒 Messages from this point are
end-to-end encrypted."*

The `+orbit/e2e` tag is defined in the [Tag Namespace](../02-components/01-uplink/02-tags/01-namespace.md)
as a post-MVP reserved tag.

All subsequent messages are encrypted. The server sees tag payloads it cannot decrypt.

## P2P Calls

P2P WebRTC connections already provide strong security guarantees at the media layer:

- DTLS-SRTP is mandatory for all WebRTC media - the browser and Tauri WebRTC stack enforce this.
- The IRC handshake (`+orbit/p2p-offer` / `+orbit/p2p-answer`) carries DTLS fingerprints, which
  are verified by both peers at connection time. A man-in-the-middle cannot substitute their own
  DTLS certificate without the fingerprint check failing.
- Post-handshake, the server is not in the media path at all. It cannot intercept calls even if it
  wanted to.

P2P calls are therefore E2E-secure by construction via the WebRTC stack. No additional application-
layer encryption is needed.

## Cross-Device Key Synchronization

A user with multiple devices (desktop + mobile, multiple desktops) needs their E2E keys
synchronized. Three approaches, in order of complexity:

### Option A: Key Backup to Depot (Initial Approach)

The user's conversation keys are encrypted with a recovery passphrase (or derived from the OIDC
identity plus a passphrase) and stored as an object in Depot. A new device authenticates via OIDC,
fetches the encrypted key bundle from Depot, prompts for the passphrase, decrypts, and has access
to all E2E conversation keys and history.

- Works offline (keys are in Depot, not on another device)
- Uses existing infrastructure (Depot + OIDC)
- Passphrase is a UX burden
- If the user loses the passphrase, they lose access to E2E history

### Option B: Device-to-Device Transfer via P2P

When adding a new device, the user initiates a P2P WebRTC connection from their existing device to
the new one (using the same `+orbit/p2p-offer` mechanism). Keys are transferred over the encrypted
data channel. Verification via a short code displayed on both screens.

- No passphrase needed
- Uses existing P2P infrastructure
- Requires both devices online simultaneously
- First device is the single point of failure

### Option C: Per-Device Keys (Signal Model)

Each device has its own key pair. The user publishes a device list via `draft/metadata-2` (e.g.,
an `orbit.devices` metadata key listing active device key fingerprints). When sending an E2E
message, the client encrypts to *all* of the recipient's known devices. Each device can
independently decrypt.

- No passphrase, no device-to-device transfer required
- Devices are independent
- Message size scales with recipient device count
- Device management UX complexity (revoking old devices, trusting new ones)
- Most complex to implement

**Plan**: Ship with Option A (single-device primary, key backup for recovery). Option B is an
alternative path. Option C is the long-term target but the complexity is not justified in the
initial implementation.

## Risks

**Key verification**: How does a user confirm they are talking to the right person and not an
impersonator with a different key? Options include safety number comparison (Signal's model) or QR
code scanning. This requires UX prototyping - it must be simple enough that users actually do it.

**Key recovery after device loss**: If a user loses their device and has no key backup (Option A)
or another device (Option B/C), their ratchet state is gone. Messages sent before key recovery
cannot be decrypted. E2E encryption means the server cannot help you recover messages. Key backup
to Depot is the primary mitigation.

**Anonymous users**: E2E requires stable identity (persistent key pairs). Guest users (`SASL
ANONYMOUS`) have no persistent identity and cannot participate in E2E-encrypted DMs. The Orbit
client must make this clear - E2E DMs require a registered account.

**Transition between E2E and non-E2E**: A conversation that starts unencrypted can transition to
E2E when both clients negotiate keys. The reverse (E2E to non-E2E) should be discouraged by the
client UX but is technically possible. The client should warn clearly if encryption is being
disabled.

## Implementation Milestones

1. Implement X3DH key exchange and Double Ratchet for 1:1 DMs in the Tauri desktop client
2. Validate key exchange over `draft/metadata-2` (prekey publication and consumption)
3. Test single-device send/receive round-trip with correct decryption
4. Prototype key verification UX (safety number display)
5. Confirm graceful fallback: if the recipient has no published keys, the DM is sent unencrypted
   with a clear UI indicator ("This conversation is not encrypted")
6. Measure latency impact on first-message key exchange vs. subsequent ratchet steps

## Dependencies

- `draft/metadata-2` must be stable in Ergo - key publication relies on it
- Stable account registration and SASL authentication
- See [Authentication](../03-identity/01-authentication.md) for the identity model
- See [DMs](../02-components/01-uplink/03-dms.md) for the two-tier DM storage model (standard
  retention for plaintext, ciphertext retention for E2E)
