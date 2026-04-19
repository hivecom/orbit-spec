# Research: Reaction Service

Cross-references:
- [Ground Control](../02-components/01-ground-control/01-overview.md) - message IDs (`msgid`) used as the reaction target key
- [Transponder](../02-components/04-transponder.md) - OIDC identity model used for JWT-verified writes
- [Depot](../02-components/03-depot.md) - parallel optional service pattern this track follows
- [Tag Namespace](../02-components/01-ground-control/02-tags/01-namespace.md) - `+orbit/*` tag namespace for live reaction delivery

## Problem

Reactions require aggregated per-message state: counts, per-user tracking, and toggle semantics (adding the same reaction again removes it). IRC's append-only history model cannot serve this reliably. A client that fetches a `chathistory` window has no way to know the current reaction state of messages - it would need to replay the entire channel history from the beginning to reconstruct accurate counts, and even then, partial loads produce stale or incorrect data.

This is a fundamentally different storage problem than text messages. Messages are immutable facts - they happened, they are recorded, and the record is append-only by design. Reactions are mutable aggregates - the "current state" of a reaction on a message is a function of every add and remove event since the message was sent. IRC cannot express this efficiently.

The solution is a dedicated service that owns reaction state: accepts writes, verifies identity, and serves aggregated reads. This service is a natural fit for the Orbit architecture - optional, independent, discovered via DNS, degrades gracefully when absent.

## Proposal

**Reactor** is a self-hostable HTTP service that stores reactions on addressable objects. For the MVP use case, those objects are IRC messages identified by their server-assigned `msgid`. The design is intentionally generic: the service stores reactions on IDs, and what those IDs refer to is the client's concern.

### Core Concepts

**Reactor** stores reactions as a set of `(target_id, account, reaction_key)` tuples. Each tuple is unique - a given account can only have one instance of a given reaction key on a given target. Adding the same reaction again removes it (toggle). This gives correct, consistent state regardless of when the client queries.

A **reaction key** is a string identifier for a reaction. For standard emoji, this is the Unicode codepoint sequence (e.g., `U+1F44D` for 👍). For custom reactions, this is a namespaced string defined by the operator (e.g., `custom:aware`, `sticker:wave`). The client resolves reaction keys to displayable assets using the server's reaction manifest.

The **reaction manifest** is a JSON document served by Reactor that describes what reactions are available on this server: the permitted emoji set, custom reaction definitions (name, image URL, category), and sticker packs. Clients fetch this manifest on connect and use it to render the reaction picker UI and resolve incoming reaction keys to images.

### API Surface

```
GET  /manifest                         Reaction manifest (emoji set, custom reactions, stickers)
GET  /reactions?ids=msgid1,msgid2,...  Batch fetch aggregated reaction state for a list of target IDs
POST /reactions/:target_id             Add or toggle a reaction (JWT required)
DELETE /reactions/:target_id/:key      Explicitly remove a reaction (JWT required; alternative to toggle)
GET  /reactions/:target_id             Full reaction state for a single target ID
```

**Batch fetch** is the primary read path. When a client loads a history window, it extracts all `msgid` values and sends a single `GET /reactions?ids=...` request. Reactor returns a map of target ID to aggregated reaction state:

```json
{
  "abc123": {
    "U+1F44D": { "count": 5, "self": true },
    "U+2764":  { "count": 2, "self": false },
    "custom:pepe": { "count": 1, "self": false }
  },
  "def456": {
    "U+1F602": { "count": 3, "self": false }
  }
}
```

`self: true` means the authenticated user has this reaction active. Unauthenticated clients receive the same response without `self` fields.

**Write path**: `POST /reactions/:target_id` with a JSON body containing the reaction key and a JWT for identity verification. Reactor verifies the JWT against the server's OIDC provider JWKS (same pattern as Satellite and Depot), records the reaction under the verified account name, and toggles if already present.

```json
POST /reactions/abc123
Authorization: Bearer <jwt>
{ "key": "U+1F44D" }
```

### Live Delivery

Reactor does not push reaction updates to connected clients. Live reaction delivery - seeing someone else's reaction appear in real time - is handled by the existing IRC tag layer. When a client posts a reaction, it simultaneously:

1. Sends `POST /reactions/:target_id` to Reactor (durable write, verified identity)
2. Sends a `TAGMSG` with a `+orbit/msg-react` tag to the channel (live delivery to connected clients)

Connected clients update their local reaction state from the IRC tag immediately. Disconnected clients get accurate state from Reactor on reconnect via the batch fetch. This is the same split used for file uploads: Depot owns the durable record, IRC carries the live announcement.

The `+orbit/msg-react` tag is therefore reinstated in the tag namespace - but its role is live notification only, not the source of truth. Clients MUST NOT use IRC history to reconstruct reaction state. Reactor is the authoritative source.

### Reaction Manifest

The manifest is the operator's control surface for what reactions are available. It is served at `GET /manifest` and fetched by clients on connect (and cached with a reasonable TTL).

```json
{
  "version": "1",
  "emoji": {
    "mode": "subset",
    "allowed": ["U+1F44D", "U+1F44E", "U+2764", "U+1F602", "U+1F614"]
  },
  "custom": [
    {
      "key": "custom:pepe",
      "label": "Pepe",
      "url": "https://cdn.example.com/reactions/pepe.webp",
      "category": "Community"
    },
    {
      "key": "custom:gg",
      "label": "GG",
      "url": "https://cdn.example.com/reactions/gg.webp",
      "category": "Gaming"
    }
  ],
  "stickers": [
    {
      "pack": "wave-pack",
      "label": "Wave Pack",
      "stickers": [
        { "key": "sticker:wave:hi", "label": "Hi", "url": "https://cdn.example.com/stickers/hi.webp" },
        { "key": "sticker:wave:bye", "label": "Bye", "url": "https://cdn.example.com/stickers/bye.webp" }
      ]
    }
  ]
}
```

**Emoji mode** controls the available standard emoji:
- `"mode": "all"` - full Unicode emoji set available (client renders standard emoji picker)
- `"mode": "subset"` - only the listed codepoints are permitted; client shows a constrained picker
- `"mode": "none"` - standard emoji reactions disabled; only custom reactions and stickers available

**Custom reactions** are operator-defined images. The `url` field points to a WebP or GIF asset served from any HTTPS host (not necessarily Depot, but Depot is a natural choice). The `category` field is used by the client to group reactions in the picker UI.

**Stickers** are grouped into packs. A sticker is a larger image intended to be posted as a standalone message reaction rather than a small inline emoji. Sticker keys follow the `sticker:<pack>:<name>` convention. The client renders stickers at a larger size than custom reactions.

Clients that receive a reaction key they cannot resolve from the manifest display a fallback (e.g., a question mark or the raw key string). This handles the case where a reaction was posted under a previous manifest version and the key has since been removed.

### Identity and Authorization

Reactor follows the same JWT verification pattern as Satellite and Depot: it fetches the OIDC provider's JWKS from the discovery document, caches it, and verifies incoming JWTs locally. No contact with any other Orbit component at request time.

- **Authenticated users**: reactions recorded under the verified `preferred_username`. Toggle semantics enforced per account.
- **Unauthenticated users**: reaction writes rejected (401). Read endpoints are public - anyone can fetch reaction counts.
- **Operators**: the manifest is operator-controlled (static config or admin API). No user can add reaction keys outside the manifest. If a client attempts to post a key not in the manifest, Reactor rejects the write (400).

### Storage

SQLite for small deployments, Postgres for larger ones. The core schema is minimal:

```sql
CREATE TABLE reactions (
  target_id   TEXT    NOT NULL,
  account     TEXT    NOT NULL,
  reaction_key TEXT   NOT NULL,
  created_at  INTEGER NOT NULL,
  PRIMARY KEY (target_id, account, reaction_key)
);

CREATE INDEX idx_reactions_target ON reactions (target_id);
```

The primary key on `(target_id, account, reaction_key)` enforces the uniqueness constraint - one reaction per user per key per target - at the database level. Toggle is an upsert: insert, and if it already exists, delete.

### Graceful Degradation

Reactor is optional. If a server operator does not deploy it:

- The reaction picker UI is hidden in the Orbit client
- `+orbit/msg-react` tags are still relayed over IRC for any client that sends them, but Orbit clients do not display reaction counts (no authoritative source)
- No reaction data is persisted

This is consistent with how Depot degrades: file upload UI is hidden when Depot is absent, not broken.

### Discovery

Reactor is discovered via the standard Orbit service discovery mechanisms:

- **DNS SRV**: `_reactor._tcp.example.com`
- **Well-known URL**: `reactor` key in `/.well-known/orbit/services.json`

```json
{
  "irc":      { "host": "irc.example.com", "port": 6697, "ws_port": 6698 },
  "satellite": [...],
  "depot":    { "host": "depot.example.com", "port": 3000 },
  "reactor":  { "host": "reactor.example.com", "port": 4000 }
}
```

## Risks

- **Reactor as a new dependency**: adding another optional service increases the operational surface for self-hosters. Mitigation: keep the binary small and the config minimal - a single env var for the OIDC issuer and a static manifest file should be sufficient for most deployments.
- **Manifest drift**: if an operator removes a custom reaction key, clients holding cached manifests will still display the old key for existing reactions. Mitigation: manifest versioning and a `Cache-Control` TTL. Clients that receive an unknown key fall back gracefully.
- **Spam/abuse on reaction writes**: a user could flood Reactor with reaction toggles. Mitigation: standard rate limiting per account per target (e.g., max 1 write/second per account), enforced at the Reactor HTTP layer.
- **`+orbit/msg-react` as a non-authoritative signal**: clients that reconstruct reaction state from IRC history (ignoring Reactor) will show incorrect counts. The spec must be explicit that IRC tags are delivery only. Third-party clients that implement reactions without Reactor will have this problem - document it clearly in the compatibility profile.

## Evaluation Criteria

Build a prototype Reactor service that:

1. Serves a manifest with a subset emoji mode and two custom reactions
2. Accepts JWT-verified reaction writes and enforces toggle semantics
3. Returns correct aggregated state on batch reads for 1,000 target IDs in under 50ms
4. Integrates with the Orbit desktop client: reaction picker shows manifest contents, reactions appear live via IRC tags, and reload from history fetches state from Reactor

Test with a real community deployment. Measure: write latency, batch read latency, storage growth rate per active channel over 30 days, and operator feedback on the manifest configuration UX.

## Dependencies

- MVP [Ground Control](../02-components/01-ground-control/01-overview.md) must be stable - `msgid` values are the reaction target keys.
- [Transponder](../02-components/04-transponder.md) (OIDC identity provider) should be deployed before Reactor - without it, all reaction writes are rejected (no JWT to verify). A NickServ-only deployment cannot use Reactor.
- The `+orbit/msg-react` tag must be added to the [Tag Namespace](../02-components/01-ground-control/02-tags/01-namespace.md) and the `services.json` schema updated to include the `reactor` key.
