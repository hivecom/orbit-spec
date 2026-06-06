# Presence

Orbit presence is built on top of Ergo's existing IRCv3 extensions. The MVP uses a layered
approach: standard IRC `AWAY` for basic online/offline state, and `draft/metadata-2` for richer
status and user profile data.

## Online / Offline

Ergo's `away-notify` and `extended-monitor` extensions provide real-time online/offline presence:

- **`away-notify`**: When a user sets or clears `AWAY` status, all clients who share a channel
  with that user (and have negotiated `away-notify`) receive an immediate notification. Orbit
  clients use this to update the online indicator in the user list.
- **`extended-monitor`**: Clients can monitor specific users for presence changes (JOIN, QUIT,
  AWAY) even without sharing a channel with them. Orbit clients use this to track DM contacts.
- **`draft/pre-away`**: Clients signal they are about to disconnect or go away before it happens,
  enabling smoother presence transitions (e.g., showing "going offline" briefly before the
  indicator changes).

Basic presence states derived from these extensions:

| State   | IRC primitive              | How Orbit clients detect it        |
|---------|----------------------------|------------------------------------|
| Online  | No AWAY set, connected     | Connected + no AWAY flag           |
| Away    | AWAY set                   | AWAY flag via `away-notify`        |
| Offline | QUIT / disconnected        | QUIT event or monitor update       |

## Rich Status and User Metadata (`draft/metadata-2`)

`draft/metadata-2` (shipped stable in Ergo 2.17.0+) provides a native key/value store per
user. Orbit defines the following metadata keys:

| Key             | Content                                     | Example                                                         |
|-----------------|---------------------------------------------|-----------------------------------------------------------------|
| `avatar`        | URL of the user's avatar image (Orbit uses a Depot URL) | `https://depot.example.com/avatars/abc123/avatar.webp` |
| `display-name`  | Human-readable display name (separate from nick) | `Alice`                                                    |
| `orbit.status`  | Short presence status string                | `dnd`, `in a meeting`, `streaming`                              |

`avatar` and `display-name` are the IRCv3 quasi-standard keys (unprefixed) so other
metadata-aware clients interoperate; `orbit.status` is vendor-prefixed because it has no
standard equivalent. The `avatar` value is a plain URL - Orbit stores a Depot URL there, but
any client simply fetches it.

> **Trust:** metadata is user-set and unverified. Avatar URLs are attacker-controlled for
> arbitrary users. Clients MUST sanitize and SHOULD proxy them - validate the content-type,
> never leak referrers to arbitrary hosts, and never treat any metadata value as an identity
> claim (see [IRC Services Abstraction - Metadata Is Not an Identity Signal](../05-services.md#metadata-is-not-an-identity-signal)).

### How it works

Orbit clients negotiate `draft/metadata-2` at connect time and subscribe to the keys they
understand:

```
METADATA * SUB avatar display-name orbit.status
```

When any user in a shared channel (or a monitored user) updates one of these keys, the client
receives a live `METADATA` notification and updates the UI immediately. No polling required.

Users set their own metadata:

```
METADATA * SET avatar :https://depot.example.com/avatars/abc123/avatar.webp
METADATA * SET display-name :Alice
METADATA * SET orbit.status :dnd
```

On joining a channel, Ergo sends the current metadata for all users in the channel (within the
client's subscription list) as part of the join burst. Clients immediately have profile data for
all visible users without additional requests.

### Avatar Upload Flow

1. User selects an avatar image in Orbit settings.
2. Client requests a pre-signed URL from Depot (`POST /upload/presign`).
3. Client uploads the image to S3 via the pre-signed URL.
4. Client sets the metadata key: `METADATA * SET avatar :<depot-url>`
5. All clients sharing a channel with this user receive the update immediately via
   `draft/metadata-2` subscription.

Avatars are stored in Depot under a conventional path: `avatars/{account_hash}/avatar.webp`. Old
avatars are overwritten - one avatar per account.

### Server Configuration

`draft/metadata-2` is **disabled by default**. Ergo only advertises it when the operator adds an
`accounts.metadata` block to the config; if the block is absent, metadata is off and `SUB`/`SET`
commands fail. Operators enabling it should set the relevant limits:

```yaml
accounts:
  metadata:
    # max-keys: keys a client may set on itself
    # max-subs: keys a client may subscribe to
    # max-value-bytes: maximum value size
    # operator-only-modification: false   # leave off for self-service profiles
```

`operator-only-modification` (Ergo 2.18.0+) is a *global* switch - it gates all metadata writes
behind operator privileges. There is no per-key write ACL, so a deployment cannot, with stock
Ergo, allow users to set `orbit.status` while reserving `avatar` for a privileged service. Leave
it off for self-service profiles.

### Availability

`draft/metadata-2` shipped stable in Ergo 2.17.0. When it is unavailable (older Ergo, or the
`accounts.metadata` block is absent):

- Orbit clients fall back to displaying initials or a generated avatar (based on account name
  hash) when no `avatar` metadata is available.
- Display names fall back to the IRC nickname.
- Status indicators fall back to the basic AWAY/online model.

The Orbit client SHOULD negotiate `draft/metadata-2` if the server advertises it, and fall back
gracefully when it does not.

## Read Markers (`draft/read-marker`)

`draft/read-marker` is already stable in Ergo. It tracks the user's read position per channel on
the server, synchronized across all connected client sessions:

- When the user reads messages in a channel, the client sends a `MARKREAD` command with the latest
  `msgid` they have seen.
- When the user connects from another device, the server provides the stored read marker for each
  channel, allowing the client to show accurate unread counts from the correct position.
- This replaces client-side unread tracking, which is device-local and resets on each new
  connection.

Orbit clients MUST negotiate `draft/read-marker` and use it for all unread tracking.

## Future Upstream Directions

Orbit does not fork Ergo. Richer presence capabilities are tracked as potential IRCv3 upstream
contributions rather than as an Orbit-owned server, so every IRC client benefits and compatibility
is preserved. Candidates include:

- Richer status states with structured semantics (not just a string).
- Presence history (when was a user last online).

See [Design Philosophy - Where Orbit's Value Lives](../../01-architecture/02-philosophy.md#where-orbits-value-lives)
for the no-fork rationale.

## Cross-References

- [Uplink Overview](01-overview.md) - required IRCv3 extensions
- [Depot](../03-depot.md) - avatar image storage
- [Tag Namespace](02-tags/01-namespace.md) - client-only tags used alongside metadata
