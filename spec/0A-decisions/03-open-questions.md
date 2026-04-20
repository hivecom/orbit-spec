# Open Questions

These are genuine unresolved decisions that need resolution before or during MVP implementation. Resolved items are struck through and annotated.

1. ~~**Nickname collision**~~ *Resolved*: The `guest-` prefix is reserved at the NickServ level (see [Uplink - Ergochat Configuration](../02-components/01-uplink/01-overview.md#ergochat-configuration)). Ergochat is configured to reject registration of any nickname starting with `guest-`, preventing collision between registered users and anonymous widget guests.

2. **File upload limits** - Should upload size limits be enforced at the storage endpoint (via pre-signed URL constraints or bucket policies) or at the client level? Storage-layer enforcement is more reliable since there's no centralised upload gateway. See [../02-components/03-depot.md](../02-components/03-depot.md) for Depot architecture.

3. ~~**Presence model**~~ *Resolved*: Presence uses `away-notify` + `extended-monitor` for online/offline, and `draft/metadata-2` for rich status and user profile metadata (avatar, display name, status string). See [Presence](../02-components/01-uplink/04-presence.md).

4. **Widget voice permissions** - Should the web widget ever allow guests to speak in voice channels, or is receive-only the correct permanent default? Allowing guest voice opens abuse vectors but has legitimate use cases (e.g., community Q&A sessions). See [../04-clients/03-widget.md](../04-clients/03-widget.md) for widget constraints.

5. **Message history retention** - What's the default retention period? Should it be configurable per-channel? If a server operator sets unlimited retention, what are the storage implications for Ergochat's internal database?

6. **Satellite metadata endpoint** - What should the Satellite `/info` metadata endpoint return? The spec proposes name, region, capacity, and version. Should it also include supported codecs, maximum participants, or TURN server hints? How much metadata is useful for client-side Satellite selection vs. unnecessary complexity?

7. **Satellite trust** - When a user posts a BYON invite, should Orbit prompt other users with a confirmation dialog ("This voice session is hosted on a community Satellite at `sat.user.com` - connect?")? Or is that too much friction? The spec currently says yes (SSH host key model), but this needs UX validation.

8. **Satellite token bootstrapping** - For server-operated Satellites, what's the identity verification flow between the Orbit client and the token service? A public join key in the Satellite descriptor is simple but means anyone who can read the channel topic can get a token. Is that acceptable for the MVP? (It probably is - the same trust model as a public TeamSpeak or Mumble server.) See [../02-components/02-satellite.md](../02-components/02-satellite.md) for the Satellite token flow.

9. **Multi-node sessions** - Can a voice session span multiple nodes? Probably not for the MVP - one session, one node. But what happens if a node goes down during an active session? Should the client attempt to migrate to another node?

10. ~~**DMs and Group DMs**~~ *Resolved*: DMs are standard IRC `PRIVMSG` to a nickname with operator-configured retention (or ciphertext retention when E2E is active). Group DMs are invite-only private channels (`+s +i`). See [DMs](../02-components/01-uplink/03-dms.md).

11. ~~**Presence model**~~ *Resolved*: Presence uses `away-notify` + `extended-monitor` for online/offline, and `draft/metadata-2` for rich status and user profile metadata (avatar, display name, status string). See [Presence](../02-components/01-uplink/04-presence.md).

12. ~~**Draft extension contingency**~~ *Resolved*: Both `draft/metadata-2` and `draft/message-redaction` have published specs and working implementations in Ergo's development branch. If Ergo's stable release timeline slips or the draft semantics change, the Uplink fork implements the current draft semantics directly - the spec exists, the behavior is well-defined, and the fork is already planned. In the worst case, this work is carried over into the fork on Orbit's timeline rather than waiting on upstream. This is not a blocking risk.

*This document is updated as questions are resolved. Changes are tracked in the repository commit history.*
