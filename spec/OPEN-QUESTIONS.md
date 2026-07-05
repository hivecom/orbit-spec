# Open Questions

Unresolved decisions that need answering before or during implementation. Resolved items are removed once their resolution lands in the spec; the commit history is the record.

1. **Satellite metadata endpoint** - What should the Satellite `/info` metadata endpoint return? The spec proposes name, region, capacity, and version. Should it also include supported codecs, maximum participants, or TURN server hints? How much metadata is useful for client-side Satellite selection vs. unnecessary complexity?

2. **Satellite token bootstrapping** - For server-operated Satellites, what's the identity verification flow between the Orbit client and the token service? A public join key in the Satellite descriptor is simple but means anyone who can read the channel topic can get a token. Is that acceptable? (It probably is; a public TeamSpeak or Mumble server has the same trust model.) See [Satellite implementation](03-implementation/03-satellite.md) for the Satellite token flow.

3. **E2E cross-device key sync** - Three options are on the table: A, key backup to Depot; B, direct P2P transfer between devices; C, per-device keys (the Signal model). Genuinely undecided. See [E2E Encryption](02-architecture/13-e2e.md).

*This document is updated as questions are resolved. Changes are tracked in the repository commit history.*
