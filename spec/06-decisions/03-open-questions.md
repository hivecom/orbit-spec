# Open Questions

These are genuine unresolved decisions that need resolution before or during MVP implementation. Resolved items are struck through and annotated.

1. ~~**Nickname collision**~~ *Resolved*: The `guest-` prefix is reserved at the NickServ level (see [Ground Control - Ergochat Configuration](../02-components/01-ground-control/01-overview.md#ergochat-configuration)). Ergochat is configured to reject registration of any nickname starting with `guest-`, preventing collision between registered users and anonymous widget guests.

2. **File upload limits** - Should upload size limits be enforced at the storage endpoint (via pre-signed URL constraints or bucket policies) or at the client level? Storage-layer enforcement is more reliable since there's no centralised upload gateway. See [../02-components/03-depot.md](../02-components/03-depot.md) for Depot architecture.

3. **Presence model** - IRC provides `AWAY` and `QUIT`/`JOIN` for basic presence. Is this sufficient for the MVP, or do we need richer status states (online, idle, do-not-disturb, invisible)? Richer presence could be built with client-only tags, but adds complexity. See [../02-components/01-ground-control/01-overview.md](../02-components/01-ground-control/01-overview.md) for IRC primitives.

4. **Widget voice permissions** - Should the web widget ever allow guests to speak in voice channels, or is receive-only the correct permanent default? Allowing guest voice opens abuse vectors but has legitimate use cases (e.g., community Q&A sessions). See [../04-clients/03-widget.md](../04-clients/03-widget.md) for widget constraints.

5. **Message history retention** - What's the default retention period? Should it be configurable per-channel? If a server operator sets unlimited retention, what are the storage implications for Ergochat's internal database?

6. **Satellite metadata endpoint** - What should the Satellite `/info` metadata endpoint return? The spec proposes name, region, capacity, and version. Should it also include supported codecs, maximum participants, or TURN server hints? How much metadata is useful for client-side Satellite selection vs. unnecessary complexity?

7. **Satellite trust** - When a user posts a BYON invite, should Orbit prompt other users with a confirmation dialog ("This voice session is hosted on a community Satellite at `sat.user.com` - connect?")? Or is that too much friction? The spec currently says yes (SSH host key model), but this needs UX validation.

8. **Satellite token bootstrapping** - For server-operated Satellites, what's the identity verification flow between the Orbit client and the token service? A public join key in the Satellite descriptor is simple but means anyone who can read the channel topic can get a token. Is that acceptable for the MVP? (It probably is - the same trust model as a public TeamSpeak or Mumble server.) See [../02-components/02-satellite.md](../02-components/02-satellite.md) for the Satellite token flow.

9. **Multi-node sessions** - Can a voice session span multiple nodes? Probably not for the MVP - one session, one node. But what happens if a node goes down during an active session? Should the client attempt to migrate to another node?

10. ~~**Reaction semantics**~~ *Moved to out-of-scope*: Reactions require aggregated per-message state that IRC's append-only history model cannot provide reliably. Partial history loads produce incorrect reaction counts, and toggle semantics require full history replay. Reactions are deferred until a dedicated message enrichment service can store and serve aggregated reaction state with proper identity verification. See [Out of Scope](04-out-of-scope.md).

*This document is updated as questions are resolved. Changes are tracked in the repository commit history.*
