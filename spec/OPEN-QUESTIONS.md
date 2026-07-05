# Open Questions

Unresolved decisions that need answering before or during implementation. Resolved items are struck through and annotated.

1. ~~**Nickname collision**~~ *Resolved*: The `anon-` prefix is reserved at the NickServ level (see [Uplink implementation](03-implementation/01-uplink.md)). SASL ANONYMOUS assigns `anon-*` nicknames automatically, and Ergochat is configured to reject registration of any nickname starting with `anon-`, preventing collision between registered users and anonymous embedded-client guests.

2. ~~**File upload limits**~~ *Resolved*: Upload size limits are configurable per deployment and enforced at the storage layer (pre-signed URL constraints and bucket policies). Operators can configure limits via the Depot configuration. Client-level enforcement is advisory only. See [Depot](02-architecture/06-depot.md) for Depot architecture.

3. ~~**Presence model**~~ *Resolved*: Presence uses `away-notify` + `extended-monitor` for online/offline, and `draft/metadata-2` for rich status and user profile metadata (avatar, display name, status string). See [Messaging](02-architecture/10-messaging.md).

4. ~~**Embedded client voice permissions**~~ *Resolved*: Guests can speak by default. Voice permissions are configurable per Satellite instance by the operator. Operators who want receive-only guests can configure that on their Satellite. See [Experience](01-product/02-experience.md) for embedded client constraints.

5. **Message history retention** - What's the default retention period? Should it be configurable per-channel? If a server operator sets unlimited retention, what are the storage implications for Ergochat's internal database?

6. **Satellite metadata endpoint** - What should the Satellite `/info` metadata endpoint return? The spec proposes name, region, capacity, and version. Should it also include supported codecs, maximum participants, or TURN server hints? How much metadata is useful for client-side Satellite selection vs. unnecessary complexity?

7. ~~**Satellite trust**~~ *Resolved*: Yes, the confirmation stays. The domain's DNS records are the network's authoritative statement of which Satellites it sanctions; those connect without friction. Anything else (a BYOS invite posted in chat) is unsanctioned, and the client requires first-connect confirmation (SSH host key model) so connections can't be hijacked by invites pointing at Satellites the network never sanctioned. See [Satellite](02-architecture/05-satellite.md#trust-model).

8. **Satellite token bootstrapping** - For server-operated Satellites, what's the identity verification flow between the Orbit client and the token service? A public join key in the Satellite descriptor is simple but means anyone who can read the channel topic can get a token. Is that acceptable? (It probably is; a public TeamSpeak or Mumble server has the same trust model.) See [Satellite implementation](03-implementation/03-satellite.md) for the Satellite token flow.

9. **Multi-node sessions** - Can a voice session span multiple nodes? Probably not at first: one session, one node. But what happens if a node goes down during an active session? Should the client attempt to migrate to another node?

10. ~~**DMs and Group DMs**~~ *Resolved*: DMs are standard IRC `PRIVMSG` to a nickname with operator-configured retention (or ciphertext retention when E2E is active). Group DMs are invite-only private channels (`+s +i`). See [Messaging](02-architecture/10-messaging.md).

11. ~~**Draft extension contingency**~~ *Resolved*: Both `draft/metadata-2` and `draft/message-redaction` are native in stock Ergo (`draft/metadata-2` stable in v2.17.0, `REDACT`/`draft/message-redaction` shipped). Orbit does not fork the IRC server; it conforms to IRCv3 and supports whatever stock Ergo ships. If draft semantics change, Orbit follows the current spec rather than maintaining a divergent fork. This is not a blocking risk.

12. ~~**Message editing**~~ *Resolved*: Deferred until IRC standardizes in-place editing; Orbit adopts whatever Ergo or IRCv3 ships. If nothing has landed by the time editing matters, the workaround is client-side edit message tags. That covers display only: it does not remove the original from stored history the way `REDACT` does for retractions.

*This document is updated as questions are resolved. Changes are tracked in the repository commit history.*
