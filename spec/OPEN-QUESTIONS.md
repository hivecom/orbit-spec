# Open Questions

Unresolved decisions that need answering before or during implementation. Resolved items are removed once their resolution lands in the spec; the commit history is the record.

1. **Satellite metadata endpoint** - What should the Satellite `/info` metadata endpoint return? The spec proposes name, region, capacity, and version. Should it also include supported codecs, maximum participants, or TURN server hints? How much metadata is useful for client-side Satellite selection vs. unnecessary complexity?

2. **Satellite token bootstrapping** - For server-operated Satellites, what's the identity verification flow between the Orbit client and the token service? A public join key in the Satellite descriptor is simple but means anyone who can read the channel topic can get a token. Is that acceptable? (It probably is; a public TeamSpeak or Mumble server has the same trust model.) See [Satellite implementation](03-implementation/03-satellite.md) for the Satellite token flow.

3. **E2E cross-device key sync** - Three options are on the table: A, key backup to Depot; B, direct P2P transfer between devices; C, per-device keys (the Signal model). Genuinely undecided. See [E2E Encryption](02-architecture/13-e2e.md).

4. **Image proxy path** - The memory discipline rule says images are proxied and resized before display ([Clients implementation](03-implementation/08-clients.md#memory-discipline)), and the desktop client's Rust backend can do that locally. What serves the web client? Options: a Depot-side resize endpoint, a generic proxy in the reverse proxy layer, or accepting direct loads on the web with strict size caps. Loading a remote image also reveals the reader's IP to the host, so the answer doubles as a privacy decision.

5. **Link unfurl path** - Link preview cards need metadata from the target URL. A client-direct fetch hits CORS on the web and reveals the reader's IP to arbitrary hosts; a proxy avoids both but adds infrastructure that anonymous and BYOS deployments may not have. Related to the image proxy question above and likely shares its answer.

*This document is updated as questions are resolved. Changes are tracked in the repository commit history.*
