# Extensions and Integrations

Orbit's core handles text chat, real-time media, identity, and client UX.
Everything else - custom moderation workflows, rich permission systems, game
integrations, event calendars - extends Orbit at the edges. Two mechanisms
carry that: IRC bots on the server side and Orbit extensions on the client
side, plus an integration API layered over the bot foundation.

Discord's bot ecosystem is one of its strongest retention mechanisms;
communities build their identity around custom automations. Orbit's
equivalent story starts from a stronger base, because IRC already has one.

## IRC Bots: The Foundation

IRC has a natural bot model - bots are just IRC clients. Any program that
speaks the IRC protocol connects to [Uplink](03-uplink.md), joins channels,
and responds to events, and mature IRC bot libraries exist in every major
language. No Orbit-specific SDK is required, and a bot written against
Uplink runs on any IRCv3 server.

Bots handle server-side automation: moderation and word filters, reminders,
role management beyond raw channel modes
([Identity - Permissions](09-identity.md#permissions)), logging, game stat
tracking. This works today, at zero engineering cost to Orbit beyond
documentation and example bots.

## The Integration API Layers

A formal integration API layers additional capabilities on the IRC
foundation, each layer building on the previous:

- **Webhook bridge.** A lightweight, self-hostable service that connects to
  Uplink as an IRC client, listens for configurable events (message posted,
  user joined, channel created), and fires HTTP callbacks to registered
  endpoints. This lets developers integrate from any HTTP-capable stack
  without implementing IRC protocol handling.
- **REST gateway.** HTTP endpoints wrapping common IRC actions - sending
  messages, managing channels, querying history, updating metadata - in a
  more accessible request/response interface.
- **Formal API specification.** A versioned specification (OpenAPI) over the
  gateway, adding bot-specific authentication (API keys or client
  credentials rather than SASL user accounts), scoped and auditable
  permissions per channel or server, and rate limiting.
- **Registry.** Long-term, a directory where communities discover and
  install bots.

### Risks

- An HTTP API over IRC is a new abstraction over an existing protocol, and
  leaky abstractions are inevitable - IRC's event-driven nature won't map
  cleanly onto request/response semantics everywhere.
- Webhook reliability requires queuing and retry; a naive implementation
  that fires requests and hopes loses events under load or when endpoints
  are down.
- Security scoping needs careful design: IRC's permission model is coarse,
  so the API layer must enforce finer-grained access control on top of it. A
  bot allowed to send messages in one channel must not be able to escalate
  to operator privileges.

## Orbit Extensions

An Orbit extension is a true application-level plugin for the Orbit client.
Extensions are installed into the client and extend its UI and behavior;
they interact with Uplink and Satellite through the standard
[Orbit tag namespace](04-tags-and-trust.md) and may define their own
sub-namespace for custom tags (`+orbit-ext/<name>/*`). Extensions are scoped
to the client - they don't run on the server and don't modify Satellite
infrastructure.

Extensions are the correct place for features that are explicitly out of
scope for the core: custom moderation UI, game integrations, event
calendars, role color schemes. Examples:

- A calendar extension rendering upcoming events inline, sourced from custom
  tags posted by a companion bot.
- A moderation dashboard adding a panel for reviewing flagged messages and
  applying IRC modes.
- A game status extension showing live match state in the sidebar.

The extension API surface itself - how plugins are packaged, installed, and
sandboxed - isn't specified yet; this page records the concept and its
boundaries.

## Bots and Extensions Together

An IRC bot (server-side logic) paired with an Orbit extension (client-side
UI) is the Orbit equivalent of a Discord bot with slash commands and embeds.
The bot enforces and automates over standard IRC; the extension renders rich
UI for it over the tag namespace. This is the path to a rich integration
ecosystem without Orbit itself implementing every feature - the core stays
thin, and complexity lives at the edges.
