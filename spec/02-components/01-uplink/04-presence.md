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

`draft/metadata-2` (in Ergo's git, pending stable release) provides a native key/value store per
user. Orbit defines the following metadata keys:

| Key             | Content                                     | Example                                                         |
|-----------------|---------------------------------------------|-----------------------------------------------------------------|
| `orbit.avatar`  | Depot URL for the user's avatar image       | `https://depot.example.com/avatars/abc123/avatar.webp`          |
| `display-name`  | Human-readable display name (separate from nick) | `Alice`                                                    |
| `orbit.status`  | Short presence status string                | `dnd`, `in a meeting`, `streaming`                              |

### How it works

Orbit clients negotiate `draft/metadata-2` at connect time and subscribe to the keys they
understand:

```
METADATA * SUB orbit.avatar display-name orbit.status
```

When any user in a shared channel (or a monitored user) updates one of these keys, the client
receives a live `METADATA` notification and updates the UI immediately. No polling required.

Users set their own metadata:

```
METADATA * SET orbit.avatar :https://depot.example.com/avatars/abc123/avatar.webp
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
4. Client sets the metadata key: `METADATA * SET orbit.avatar :<depot-url>`
5. All clients sharing a channel with this user receive the update immediately via
   `draft/metadata-2` subscription.

Avatars are stored in Depot under a conventional path: `avatars/{account_hash}/avatar.webp`. Old
avatars are overwritten - one avatar per account.

### Availability

`draft/metadata-2` is currently in Ergo's git and pending a stable release. Until it ships:

- Orbit clients fall back to displaying initials or a generated avatar (based on account name
  hash) when no `orbit.avatar` metadata is available.
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

## The Uplink Fork

In the Uplink fork (planned next step), presence becomes a first-class server feature:

- Richer status states with structured semantics (not just a string).
- Presence history (when was a user last online).
- Server-managed avatar storage (Depot dependency removed for avatars).

## Cross-References

- [Uplink Overview](01-overview.md) - required IRCv3 extensions
- [Depot](../../03-depot.md) - avatar image storage
- [Tag Namespace](02-tags/01-namespace.md) - client-only tags used alongside metadata
