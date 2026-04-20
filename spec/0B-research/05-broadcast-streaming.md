# Research: Broadcast Streaming

This track explores integrating one-to-many broadcast livestreaming into the Orbit ecosystem as an optional add-on service, distinct from [Satellite](../02-components/02-satellite.md). The reference implementation under consideration is [qwer](https://github.com/hivecom/qwer) - a minimalistic Rust livestreaming server that powers [qwer.ee](https://qwer.ee), built by the same team.

## Problem

Satellite is a bidirectional, low-latency media layer built on LiveKit SFU. It is designed for interactive use cases: voice calls, group video rooms, and screen sharing between participants. It is not designed for broadcast - the "one streamer, many passive viewers" model common to platforms like Twitch or YouTube Live.

Forcing broadcast use cases through an SFU has real costs:

- LiveKit is optimized for room-scale participant counts, not high-fan-out viewer counts
- Every viewer holds a WebRTC peer connection with associated ICE, DTLS, and SRTP overhead
- There is no concept of ingest authentication separate from room participation
- SFU resources scale with participants, not with viewer bandwidth alone

Broadcast streaming is also architecturally different in nature: it is unidirectional, tolerates higher latency (2–10s is acceptable for most live content), and benefits from CDN delivery for large audiences. These properties call for a different component entirely.

## Reference Implementation: qwer

qwer (`qw-ingest`) is a single-process Rust streaming server. Its architecture:

- Accepts RTMP ingest (from OBS and similar tools), authenticates via a separate gRPC `StreamAuth` service
- Converts RTMP → fragmented MP4 (fMP4) in-process using a filter graph pipeline
- Fans out the live stream to viewers via HTTP chunked transfer and WebSocket (MSE transport)
- Captures periodic snapshots (thumbnails) of active streams
- Exposes a gRPC `StreamInfo` service that emits stream lifecycle events (started, stopped, viewer join/leave, bandwidth stats)

The fan-out mechanism is an in-memory `MediaFrameQueue`: a `Vec` of bounded async channels (1024 frames each), one per connected viewer. Each incoming frame is cloned by reference (Arc) and pushed to every viewer channel via `try_send`. Slow or disconnected viewers are evicted immediately on overflow.

This is a clean, efficient design for the single-node case.

## Scaling

qwer does **not** have a horizontal scaling path. It is a single-process, single-node server. All viewers connect to the same process; the process ships the full bitstream to each viewer individually over its own outbound connection. There is no:

- HLS/DASH segmentation or CDN push
- Relay/edge tier for fan-out distribution
- Shared segment store between nodes
- External pub/sub for multi-instance coordination

For modest concurrent viewership (tens to low hundreds of viewers per stream) on a well-provisioned machine, this is entirely workable. A 3 Mbps stream with 100 viewers consumes ~300 Mbps of egress from one host - achievable on a modern VPS. The bottleneck is network bandwidth, not CPU or memory, and Tokio handles the async I/O efficiently.

For larger audiences, the standard scaling path would need to be built on top of qwer:

1. **HLS/DASH segmentation** - write fMP4 segments to an S3-compatible store (Depot), serve via CDN; ingest stays on one node, viewers pull from edge
2. **Relay tier** - secondary qwer instances subscribe to the primary ingest node and re-fan-out to local viewer pools, reducing per-stream egress on the origin
3. **Segment store** - once segments are in object storage, any edge node can serve them without coordinating with the ingest node

None of this exists in qwer today.

## Relationship to Orbit

Broadcast streaming is **not part of Orbit's core architecture**. Satellite handles interactive media; there is no gap in the MVP that broadcast streaming fills. The two serve fundamentally different use cases:

| | Satellite | Broadcast (qwer) |
|---|---|---|
| Direction | Bidirectional | Unidirectional |
| Latency target | <300ms | 2–10s acceptable |
| Participant model | Room members | Streamer + passive viewers |
| Transport | WebRTC | RTMP ingest / HTTP+WS egress |
| Scale model | Per-room SFU | Per-stream fan-out |
| Identity | Orbit/IRC identity | Stream key |

A broadcast service would be an optional, independently deployed add-on - not a named core component. Operators who want to offer livestreaming to their communities could deploy it alongside Orbit; operators who don't need it simply don't. It would integrate loosely via IRC signaling (a `TAGMSG` when a stream goes live, analogous to `orbit/sat-invite`) and stream key authentication against Transponder or Uplink identity.

## Open Questions

- **Naming.** If this becomes a documented optional service, what is it called in Orbit's vocabulary? Candidates: Beacon, Relay, Broadcast.
- **Auth integration.** qwer's `StreamAuth` gRPC service is a seam point - it could be implemented to validate stream keys against Transponder JWTs, without modifying qwer's ingest logic.
- **IRC signaling.** A `TAGMSG` carrying stream metadata (node URL, stream name, viewer count) fits the existing Orbit signaling pattern and requires no changes to Uplink.
- **Segment storage.** If HLS segmentation is added, Depot (S3-compatible) is the natural home for segments. This would be the first cross-component dependency between a streaming service and Depot. Probably not a great use-case for Depot, but could use the same underlying S3-compatible storage (MinIO, R2, etc.) without the Depot API layer.
- **Scaling investment.** Is it worth extending qwer toward a relay/CDN model, or is single-node sufficient for Orbit's target communities (small to mid-sized, not Twitch-scale)?

## Evaluation Criteria

Before promoting this to a named component or investing in scaling work, the following should be assessed:

- Is there actual demand for livestreaming among Orbit's target communities?
- Does single-node qwer cover the realistic concurrent viewership ceiling for those communities?
- Can stream key auth be wired to Transponder/Uplink without forking qwer's core?
- Is the IRC signaling integration clean enough to implement without touching Uplink internals?

## Dependencies

- [Satellite](../02-components/02-satellite.md) - defines what Orbit's interactive media layer already covers and what it does not
- [Depot](../02-components/03-depot.md) - natural segment store if HLS segmentation is pursued
- [Transponder](../02-components/04-transponder.md) - identity layer for stream key validation
- [Uplink](../02-components/01-uplink/01-overview.md) - IRC signaling path for stream-live notifications
