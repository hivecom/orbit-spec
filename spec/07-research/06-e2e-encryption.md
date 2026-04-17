# Research: End-to-End Encryption

For the identity model that E2E encryption depends on, see [Authentication](../03-identity/01-authentication.md).

## Problem

The MVP provides transport encryption (TLS for IRC, DTLS-SRTP for WebRTC media). The server can read all text messages and could theoretically inspect media streams. For many communities this is fine - they trust their own server. But some users and organizations require true end-to-end encryption where the server is unable to read message content even in principle.

## Options

Two established approaches:

- **Double Ratchet (Signal Protocol)**: Well-proven for 1-on-1 messaging with forward secrecy and post-compromise security. Mature Rust implementations exist (`libsignal-protocol`). The challenge is extending to group chats - Sender Keys (as used by Signal's group messaging) work but have weaker security properties than pairwise ratchets. Scales to ~1000 members before performance degrades.
- **MLS (Messaging Layer Security)**: [IETF RFC 9420](https://www.rfc-editor.org/rfc/rfc9420.html). Purpose-built for group E2E encryption with efficient key management via ratchet trees. Designed to scale to large groups. [OpenMLS](https://openmls.tech/) is a Rust implementation. More complex than Signal but architecturally better suited for group channels.

## Risks

E2E encryption fundamentally conflicts with several server-side features:

- **Search**: The server can't index messages it can't read. Search must happen client-side, which means downloading and decrypting potentially large amounts of history.
- **History for new members**: When a new user joins an E2E-encrypted channel, they can't read messages sent before they joined (by design). This is a UX surprise for users coming from Discord.
- **Link previews**: Server-side link unfurling doesn't work. Previews must be generated client-side.
- **Moderation**: Server administrators cannot review reported messages in E2E channels. Content scanning for abuse prevention is impossible. This creates real trust & safety challenges.
- **Key management UX**: Key verification (ensuring you're talking to the right person), multi-device key synchronization, and key recovery after device loss are notoriously hard UX problems. Signal has invested years in making this seamless. We would be starting from scratch.
- **Anonymous users**: The MVP anonymous/guest access model and E2E encryption are architecturally at odds. E2E encryption requires stable identity (persistent key pairs). Anonymous users are, by definition, transient.

  > **Proposed resolution**: E2E-encrypted channels would require authenticated users only. Anonymous/guest users would be excluded from these channels. The UI must make this gating explicit - joining an E2E channel should require account authentication, with a clear explanation of why.

## Evaluation Criteria

Prototype E2E encrypted direct messages using the Signal Protocol (via `libsignal-protocol` Rust crate):

- Implement key exchange and ratcheting for 1-on-1 DMs
- Evaluate UX for key verification (QR code scanning, safety number comparison)
- Test multi-device scenarios (user has desktop + mobile - how do keys sync?)
- Measure performance impact on message send/receive latency

Only extend to group encryption (via MLS) if DM encryption ships successfully and there is demonstrated user demand.

## Dependencies

Stable identity system from the MVP (account registration, authentication, device management). See [Authentication](../03-identity/01-authentication.md) for the identity model this track depends on.
