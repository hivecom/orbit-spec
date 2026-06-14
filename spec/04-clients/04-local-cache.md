# Local History Cache & Storage

Every Orbit client keeps a persistent, on-device copy of the message history it has
seen. This is distinct from the in-memory **live window** described in
[Memory Discipline](01-desktop.md#memory-discipline): the live window is what is mounted in
the DOM right now (capped, evicted aggressively); the **local cache** is the durable archive
that sits underneath it and outlives the session.

This page covers why the cache exists, how it is keyed and paged, the progressive-rendering
path that makes large buffers feel instant, the platform seam that backs it (IndexedDB on the
web, SQLite on the desktop), the storage-management surface, and the honest limits in each
environment.

Cross-references:
- [Desktop - Memory Discipline](01-desktop.md#memory-discipline) - the in-DOM live window this cache feeds
- [Desktop - Reconnection Flow](01-desktop.md#reconnection-flow) - `CHATHISTORY` reconciliation the cache piggybacks on
- [Uplink - Interaction with Chat History](../02-components/01-uplink/01-overview.md#interaction-with-chat-history) - how retractions and replies replay through `chathistory`
- [Application Seams](../05-infrastructure/04-application-seams.md) - the capability-port pattern the cache is implemented as

## Why a Local Cache Is the Right Move for IRC

The IRC history model is unusually friendly to client-side caching, more so than the proprietary
APIs Orbit is measured against. Three properties make it close to ideal:

- **Records are immutable and append-only.** A `PRIVMSG` that has been delivered never changes.
  The only post-delivery mutations are retractions (which Orbit renders as a *tombstone* overlay,
  never a content edit - see [Retractions](../02-components/01-uplink/01-overview.md#retractions))
  and, post-MVP, message editing. The cache is therefore a write-once store with rare overlay
  events, not a cache-coherence problem. There is almost nothing to invalidate.
- **There is a stable, server-assigned dedup key.** Every message carries a `msgid`
  (`message-ids`) and an authoritative `server-time`. The `msgid` is the cache primary key and the
  deduplication key; `server-time` is the sort key. Live messages, `echo-message` self-copies, and
  `chathistory` replays all collapse to the same record by `msgid` with no client-side guessing.
- **Delivery is already batched.** `chathistory` responses arrive inside a `batch`, pre-chunked by
  the server. The cache writes a batch as a unit and the renderer consumes it incrementally.

### Offloading Uplink

Ergo's history retention is **operator-configured and intentionally bounded** - the reference
guidance is a window such as 7 days or 10,000 messages per channel (see
[Ergochat Configuration](../02-components/01-uplink/01-overview.md#ergochat-configuration)). The
server is the source of truth, but it is not a long-term archive, and every `chathistory` request
costs the server CPU, disk reads, and a round-trip.

A local cache changes the load profile in Orbit's favour:

- **Scrollback is served from disk, not the server.** Re-opening a channel and scrolling up a few
  hundred messages hits zero network round-trips when the data is already cached. On a `$5 VPS`
  serving a whole community, removing repeated `chathistory` paging from every client's normal
  navigation is a material reduction in server work.
- **The cache outlives server retention.** Messages that have aged out of Ergo's retention window
  remain readable on every device that saw them live. Each client becomes its own incremental
  archive of the conversations it participates in, without the operator having to provision
  unbounded server-side storage.
- **Reconnection fetches a delta, not a window.** On reconnect the client already knows the newest
  `msgid` it has cached per target, so it asks the server only for what it missed
  (`CHATHISTORY LATEST ... after the cached tip`) instead of re-pulling a full seed. See
  [Reconnection Flow](01-desktop.md#reconnection-flow).
- **It is the foundation for client-side search.** Full-text search ships in the MVP via Ergo's
  history backends (see the [Feature Map](../../FEATURES.md)). A populated local cache lets a client
  answer many searches locally - instant, offline-capable, and without hammering the server's FTS
  index for every keystroke. Server search remains authoritative for history the client never saw.

> The cache never weakens the trust model. It stores what the server delivered, keyed by the
> server-asserted `account-tag` and `msgid`. It is a performance and availability layer, not a new
> source of authority. See [Tag Integrity and Trust Model](../02-components/01-uplink/02-tags/02-trust-model.md).

## Storage Model

The cache is scoped **per account, per target**. "Target" (buffer) means any channel or DM query -
the same notion `CHATHISTORY TARGETS` uses. Keying by account keeps multiple identities on a shared
device (or multiple servers, in the multi-server client) from colliding.

Two logical stores:

| Store      | Key                          | Holds |
|------------|------------------------------|-------|
| `messages` | `msgid` (with a `[target, server_time]` index) | One record per delivered message |
| `buffers`  | `target` (channel/DM name)   | Per-target metadata: oldest/newest cached `msgid` and timestamp, cached count, last reconcile time, cap override |

A `messages` record stores exactly what the renderer and search need, and nothing the trust model
forbids reconstructing:

```ts
interface CachedMessage {
  msgid: string            // primary key, server-assigned (message-ids)
  target: string           // channel or DM this belongs to
  serverTime: number       // sort key, from server-time (epoch ms)
  account: string | null   // server-asserted author identity (account-tag), null for unauthenticated
  nick: string             // nick at send time (display only; account is authoritative)
  type: "privmsg" | "notice" | "action"
  text: string
  tags: Record<string, string> // surviving +orbit/* and +draft/* tags (reply ref, reactions, etc.)
  redacted?: boolean       // tombstone overlay; original text is NOT retained when set
}
```

Deduplication is always by `msgid`. The live socket path and the `chathistory` path both upsert into
the same store, so an overlap between a freshly received message and a cached one resolves to a
single record rather than a visible duplicate.

## Seeding, Paging, and the Sliding Window

Three tunables govern the data path. They are defaults, not protocol constants, and the desktop
client can run them higher than the web app (see [Environment Limits](#environment-limits-and-constraints)).

| Knob                | Role | Reference default |
|---------------------|------|-------------------|
| `CACHE_SEED_COUNT`  | How many recent messages are rendered immediately from cache when a target is opened | ~150 |
| `CACHE_PAGE_SIZE`   | How many older messages are pulled per scroll-up page | ~50 |
| `MAX_LIVE_MESSAGES` | Hard cap on messages kept in the reactive in-memory buffer (the DOM path) | 200 (matches [Memory Discipline](01-desktop.md#memory-discipline)) |

The cache is a strict superset of the live window: the live window is a `MAX_LIVE_MESSAGES`-sized
view that slides over a much larger cached (and ultimately server-side) history.

### Opening a Target (Prefill)

```mermaid
sequenceDiagram
    participant U as User
    participant V as Live Window (DOM)
    participant C as Local Cache
    participant S as Uplink (Ergo)

    U->>V: Open #channel
    V->>C: read seed (newest CACHE_SEED_COUNT)
    C-->>V: cached messages
    Note over V: Render instantly, no network wait
    V->>S: CHATHISTORY LATEST #channel <newest-cached-msgid> *
    S-->>V: delta batch (messages since tip)
    V->>C: upsert delta (dedupe by msgid)
    Note over V: Reconcile tail; cache and view converge
```

Prefill is the key UX win: the buffer paints from disk on the same frame the user clicks, and the
network reconciliation only ever fills the small gap between the cached tip and now. A cold target
(nothing cached) falls back to a normal `CHATHISTORY LATEST ... *` seed and populates the cache from
the response.

### Scrolling Back (Backward Paging)

`fetchOlderHistory()` is **cache-first**: when the user scrolls toward the top, it prepends the next
`CACHE_PAGE_SIZE` older messages from the cache. Only when the cache is exhausted at its oldest
cached `msgid` does it issue `CHATHISTORY BEFORE <oldest-cached-msgid>` to the server, then writes
that page back into the cache so the next scroll is local again.

### Scrolling Forward (Forward Paging)

When the live window has been trimmed at the tail (the user scrolled far up, so newer messages were
evicted from the DOM to honour `MAX_LIVE_MESSAGES`), scrolling back down calls
`fetchNewerFromCache()`: it appends newer messages from the cache and trims from the head, keeping a
sliding window of constant size. A `tailTrimmed` flag tracks whether the live window's newest message
is still the buffer's true tip; when it is, new live messages append directly and no forward fetch is
needed.

This backward/forward symmetry is what keeps a multi-thousand-message buffer navigable with a bounded
DOM: the user can scroll arbitrarily far in either direction and the renderer never holds more than
`MAX_LIVE_MESSAGES` nodes.

## Progressive Rendering

A naive "render the whole seed at once" approach janks on large seeds because layout and paint for
hundreds of message components block the main thread. Orbit clients render the seed **incrementally**:

- The seed is committed to the reactive buffer in `RENDER_CHUNK`-sized increments (e.g. 25-50
  messages), one chunk per animation frame (`requestAnimationFrame`), until the seed is fully
  mounted or `MAX_LIVE_MESSAGES` is reached.
- The first chunk is the visible viewport's worth of messages, so the user sees content on the first
  frame; subsequent chunks fill in above/below as frames are available.
- Scroll position is anchored to a stable message (by `msgid`) across chunk commits so the viewport
  does not jump while earlier messages mount in.

This staged approach (which superseded an earlier binary "render 50, then render the rest" scheme)
keeps the main thread responsive during the initial paint and during large scroll-driven page loads.

> For extremely large buffers, list virtualization (mounting only the visible range) is the
> end-state optimization. The incremental-commit approach above is the MVP mechanism; virtualization
> is tracked as a [performance follow-up](#evaluation-performance-and-limits).

## The Cache as a Platform Capability

Persistent storage is exactly the kind of environment-specific capability the
[Application Seams](../05-infrastructure/04-application-seams.md) model is built for. The cache is a
**capability port** on the `Platform` contract, not an `if (isTauri)` branch inside a store. `core`
calls the port; it never knows whether records land in IndexedDB or SQLite.

```ts
// packages/core/src/platform/index.ts (addition to the Platform contract)
export interface HistoryCachePort {
  seed(target: string, limit: number): Promise<CachedMessage[]>
  pageBefore(target: string, beforeMsgid: string, limit: number): Promise<CachedMessage[]>
  pageAfter(target: string, afterMsgid: string, limit: number): Promise<CachedMessage[]>
  upsert(messages: CachedMessage[]): Promise<void>     // batched, dedupes by msgid
  markRedacted(msgid: string): Promise<void>           // tombstone overlay
  // Storage management surface
  bufferStats(): Promise<BufferStats[]>
  prune(target: string, keepCount: number): Promise<void>
  export(target: string): Promise<CachedMessage[]>
  clear(): Promise<void>                               // wipe this account's cache
}

export interface BufferStats {
  target: string
  count: number
  estimatedBytes: number
  oldest: number   // server-time epoch ms
  newest: number
}
```

| Target            | Adapter            | Backing store |
|-------------------|--------------------|---------------|
| Web app / PWA     | `web.ts`           | IndexedDB (one object store per logical store, indexed on `[target, serverTime]`) |
| Desktop (Tauri)   | `tauri.ts`         | SQLite via the Rust backend (one table, indexed) - served over the custom protocol for bulk reads, not JSON IPC |
| Widget            | `web.ts`, degraded | IndexedDB if available, else `null` port -> ephemeral in-memory only (recent on load) |
| Mobile (Tauri)    | `tauri-mobile.ts`  | reuses the desktop SQLite adapter |

When the port is `null` (storage blocked, private-browsing IndexedDB denial, or widget mode), core
degrades explicitly: it runs with an in-memory buffer only and the seed/prefill path simply has
nothing to read. This is the same `null`-port degradation pattern used for `tray` and `deepLinks`.

Bulk history reads on the desktop go over Tauri's custom protocol handler rather than JSON-serialized
IPC, consistent with the [Large IPC payloads](01-desktop.md#memory-discipline) rule.

## Cache Lifecycle

### A Detached, App-Scoped Writer

The write path **must not be owned by a view component.** An early prototype scoped cache syncing to
the chat-surface component and only persisted the currently-mounted buffers (a "3 of 8 buffers
cached" bug appeared when the other targets were never mounted). The fix is to own the cache writer
at **app scope** - a store/composable that subscribes to the message stream for *all* joined targets
for the whole session, independent of which view is mounted. Component mount/unmount changes what is
*rendered*, never what is *persisted*.

### Write Path

- Live messages and `chathistory` batches are written through `upsert()` in **debounced batches**
  rather than one write per message, to amortize transaction overhead (IndexedDB transaction
  setup, SQLite write locks).
- Retractions call `markRedacted(msgid)`, which sets the tombstone flag and drops the stored `text`.
  The original content is never retained, matching the server contract.
- Edits (post-MVP) update the record's `text` in place, keyed by the edited message's `msgid`.

### Invalidation and Clearing

There is almost nothing to invalidate (records are immutable). The two clearing paths are:

- **User-initiated** - the [Storage management surface](#storage-management-surface) below.
- **Force refresh** - the client's hard-reset shortcut (Ctrl+Shift+R), which already clears
  `localStorage` for settings, is extended to also call `HistoryCachePort.clear()` so a force refresh
  wipes the on-device history archive and reloads from the server, scoped to the active account.

## Storage Management Surface

The cache is durable and can grow large, so it is a first-class surface in **Settings -> Storage**,
not a hidden implementation detail. It exposes:

- **Per-buffer stats** - target name, cached message count, estimated bytes, and the cached date
  range, via `bufferStats()`.
- **A live total** - aggregate message count and estimated size across all buffers, plus the
  environment quota and headroom (web) reported by `navigator.storage.estimate()`.
- **Eviction** - per-buffer "Evict old messages" (`prune(target, keepCount)`, oldest-first) and a
  global "Evict all". Eviction respects the live window: it never evicts what is currently mounted.
- **Export** - per-buffer "Export as JSON" (`export(target)`) so a user can keep their own archive
  independent of any server or device.
- **A configurable per-buffer cap** - a setting (e.g. `chat.cacheMaxMessagesPerBuffer`, default in the
  low tens of thousands) that triggers an oldest-first prune pass after writes exceed it. Changing the
  cap propagates to the cache layer and can trigger a one-time prune of existing over-cap buffers.

Eviction policy is **oldest-first within a target**, never cross-target, so heavily-used channels do
not evict each other. The cap is per-buffer rather than global to keep accounting simple and
predictable for the user.

## Environment Limits and Constraints

The honest constraint differences between surfaces drive the defaults above.

### Web App / PWA (IndexedDB)

- **Quota is browser-managed and shared.** Chromium grants an origin a large slice (commonly up to
  ~60% of free disk, shared across the origin's storage); Firefox uses group/eTLD+1 limits; Safari is
  the tightest. The client must treat quota as finite: read `navigator.storage.estimate()`, surface it
  in the Storage tab, and prune before writes when near the limit rather than letting a write throw
  `QuotaExceededError`.
- **Best-effort storage can be evicted by the browser.** Under storage pressure the browser may clear
  a non-persistent origin's IndexedDB. The client SHOULD request `navigator.storage.persist()` (granted
  more readily for installed PWAs and engaged origins) to opt into persistent storage and reduce
  surprise eviction.
- **WebKit time-based eviction.** Safari may evict script-writable storage for sites without sufficient
  user engagement after a period of inactivity. Installing the PWA and the `persist()` request mitigate
  this; the cache is designed to degrade gracefully (re-seed from `chathistory`) if it is wiped.
- **Structured-clone cost.** Large IndexedDB reads deserialize on the main thread. For big seeds this
  is mitigated by chunked reads and, as a follow-up, moving cache I/O to a Web Worker.
- **Practical posture.** Keep web defaults moderate (`CACHE_SEED_COUNT ~150`, per-buffer cap in the low
  tens of thousands). The web cache is an accelerator and a recent-history archive, not an unbounded
  one.

### Desktop / Mobile (SQLite)

- **No browser quota and no surprise eviction.** SQLite is bounded only by disk. The standalone clients
  do not suffer the web's quota or time-based eviction problems, so they can run a larger seed, a higher
  per-buffer cap, and effectively a complete personal archive of every conversation they have seen.
- **Bulk reads are cheap and off the WebView.** Paging and search queries run in Rust against an
  indexed table and stream results over the custom protocol, avoiding the WebView's main-thread
  deserialization cost entirely.
- **This asymmetry is intentional.** The same `HistoryCachePort` contract backs both; only the limits
  and tuning differ. A user who wants a permanent, searchable archive runs the desktop client; the web
  app gives most of the benefit within the browser's constraints.

## Evaluation: Performance and Limits

What the design gets right today:

- **Instant target switches** via cache prefill; the network only fills the tail delta.
- **Bounded DOM** regardless of buffer size via the sliding live window plus incremental rendering.
- **Reduced server load** - normal scrollback and reconnection are deltas, not full re-pulls.
- **Trivial coherence** - immutable, `msgid`-keyed records with tombstone overlays; no invalidation
  graph.

Where it can get faster (tracked follow-ups, not MVP blockers):

- **List virtualization** for very large mounted ranges, to push the effective DOM cost below
  `MAX_LIVE_MESSAGES` and make the live-window cap a soft target.
- **Worker-thread cache I/O** on the web, to keep structured-clone deserialization off the main
  thread for large reads.
- **Batch-write tuning** - larger debounce windows and "prune-before-write" under quota pressure to
  avoid `QuotaExceededError` round-trips.
- **A local search index** - an inverted index built over the cache (or SQLite FTS5 on the desktop)
  so the client answers most searches locally and only escalates to the server's history backend for
  history it never cached. This is the natural next layer on top of a populated cache and the biggest
  future server-offload win.

Known limits to keep in view:

- The web cache can be evicted by the browser; treat it as a fast, best-effort layer with the server
  as the durable fallback. The desktop cache is the durable one.
- Estimated byte sizes in the Storage tab are approximate (structured-clone/row overhead is not
  exact); they are for user guidance, not billing.
- The cache cannot recover content the server never delivered or that was retracted - it stores only
  what the client legitimately received.
