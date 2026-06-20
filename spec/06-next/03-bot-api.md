# Bot and Integration API

Cross-references:
- [Uplink](../02-components/01-uplink/01-overview.md) - the IRC server bots connect to
- [Tag Namespace](../02-components/01-uplink/02-tags/01-namespace.md) - the `+orbit-ext/<name>/*` sub-namespace used by extension plugins

## Motivation

Discord's bot ecosystem is one of its strongest retention mechanisms. Communities build custom moderation tools, music playback, game stat tracking, welcome flows, and hundreds of other automations via bots. Orbit needs an equivalent story.

## Current State

IRC already has a natural bot model - bots are just IRC clients. Any program that can speak the IRC protocol can connect to [Uplink](../02-components/01-uplink/01-overview.md) (Ergo), join channels, and respond to messages. There are mature IRC bot libraries in every major language. For the MVP, this works.

## Design

A formal Orbit Bot API will layer additional capabilities on top of the IRC foundation:

- **Standardized event webhooks**: HTTP callbacks fired when specific events occur (message posted, user joined, channel created). This lets bot developers use any HTTP-capable language/framework without implementing IRC protocol handling.
- **REST API for actions**: HTTP endpoints for sending messages, managing channels, querying message history, updating user metadata. Wraps IRC commands in a more accessible interface.
- **Bot-specific authentication**: API keys or OAuth2 client credentials rather than SASL user accounts. Scoped permissions per channel or server.
- **Rate limiting and permission scoping**: Bots get specific, auditable permissions. Prevents a misbehaving bot from spamming or performing unauthorized actions.
- **Registry / marketplace**: Long-term, a directory where communities can discover and install bots. Very long-term.

## Approach

This will be phased incrementally to avoid over-engineering:

1. **Phase 0 (day-one, ships with MVP)**: Write documentation on how to build IRC bots against Ergo. Provide example bots in Rust, Python, and JavaScript. This is **zero engineering cost** and enables the community immediately.
2. **Phase 1**: Build an HTTP webhook bridge - a lightweight service that connects to Uplink as an IRC client, listens for configurable events, and fires HTTP POST requests to registered webhook endpoints. This is a small, self-contained service.
3. **Phase 2**: Build a REST API gateway that wraps common IRC commands (send message, join channel, set topic, kick user, query WHOIS) in HTTP endpoints with JSON request/response bodies.
4. **Phase 3**: Formalize as a versioned API specification (OpenAPI). Add authentication scoping, rate limiting, and developer documentation portal.

### Orbit Extension API

The **Orbit Extension API (Orbit application plugins)** - client-side plugins for the Orbit desktop and web clients that add UI and behavior, interact with Uplink via the tag namespace, and may define their own `+orbit-ext/<name>/*` sub-namespace (see [Tag Namespace](../02-components/01-uplink/02-tags/01-namespace.md)) - is considered high-value and will be designed early in the post-MVP phase.

The combination of an IRC bot (server-side logic) and an Orbit extension (UI integration) is the Orbit equivalent of a Discord bot with slash commands and embeds. This is the path to a rich integration ecosystem without requiring Orbit itself to implement every feature.

## Risks

- Building an HTTP API layer on top of IRC is building a new abstraction over an existing protocol. Leaky abstractions are inevitable - some IRC behaviors won't map cleanly to REST semantics (e.g., IRC's event-driven nature vs. REST's request-response model).
- Webhook reliability requires queuing and retry infrastructure. A naive implementation that fires HTTP requests and hopes they arrive will lose events under load or when endpoints are temporarily down.
- Security scoping for bots needs careful design. A bot with `send_message` permission in `#general` should not be able to escalate to operator privileges. IRC's permission model is coarse - the API layer must enforce finer-grained access control on top of it.

## Validation Criteria

Ship the webhook bridge (Phase 1) as an optional, self-hostable service. Announce it to the community. If bot developers actually adopt it and build useful integrations, that validates further investment in Phase 2+. Adoption data will drive the roadmap.

## Dependencies

MVP [Uplink](../02-components/01-uplink/01-overview.md) deployment must be stable. The webhook bridge is a client of Ergo - if Uplink is unstable, the bridge inherits that instability.
