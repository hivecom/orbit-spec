# Research: Media over QUIC (Iroh)

**Status:** R&D (ongoing)

This track explores replacing WebRTC with Media over QUIC (MoQ) using the [Iroh](https://iroh.computer/) networking stack for native desktop clients. See [Satellite](../02-components/02-satellite.md) for the current LiveKit/WebRTC baseline that this track is evaluated against.

## Problem

WebRTC carries legacy complexity. ICE negotiation is unreliable, SRTP adds overhead, SDP is a bloated signaling format, and NAT traversal success rates are inconsistent across network configurations. For native desktop clients where we control both endpoints, we can do better.

## Proposal

Replace WebRTC with Media over QUIC (MoQ) using the [Iroh](https://iroh.computer/) networking stack (Rust) for native desktop clients. Iroh provides:

- **QUIC-native transport** with built-in stream multiplexing and congestion control
- **Advanced NAT traversal** via relay-assisted hole punching with high success rates in practice
- **Poll-based streaming** - encode and transmit frames only when a viewer is actively connected, reducing idle resource usage
- **Sub-50ms latency** for screen broadcasting in controlled test environments

## Browser Fallback

Web clients would receive the same media streams via WebTransport over HTTP/3. This avoids the need for WebRTC entirely on the browser side - in principle.

In practice, WebTransport browser support is incomplete as of mid-2025. Safari does not support WebTransport, meaning a WebRTC fallback would still be required for Safari users, partially defeating the purpose of the migration. For living, up-to-date browser support status, refer to [caniuse.com/webtransport](https://caniuse.com/webtransport) rather than any point-in-time statement in this document.

The exact bridging architecture between Iroh (a native QUIC stack) and WebTransport (a browser API over HTTP/3) is also the least-specified part of this proposal. It is not yet clear whether Iroh can act directly as a WebTransport server, or whether a separate gateway process would be required to translate between Iroh's wire format and what browsers expect. This bridging question must be resolved early in any prototype work - it affects the feasibility assessment significantly.

## Risks

- MoQ is still an active IETF working group (draft stage), not a finished standard. The wire format and semantics may change.
- Iroh's live media libraries are experimental and not battle-tested at scale.
- WebTransport browser support is incomplete. Safari does not support it as of mid-2025. This would require a WebRTC fallback for Safari users, partially defeating the purpose.
- Maintaining two media transports (WebRTC for MVP compatibility, MoQ for native) increases operational and debugging complexity significantly.

## Evaluation Criteria

Build a standalone prototype that streams screen capture from one native client to another via Iroh. Measure:

- End-to-end latency (target: <50ms on LAN, <150ms over WAN)
- NAT traversal success rate across residential, corporate, and mobile networks
- CPU and bandwidth usage under sustained load
- Reliability over degraded network conditions (packet loss, jitter)

Compare directly against the [Satellite](../02-components/02-satellite.md) (LiveKit/WebRTC) baseline from the MVP. Only proceed if MoQ delivers measurable, meaningful improvements that justify the added complexity.

## Dependencies

MVP [Satellite](../02-components/02-satellite.md) infrastructure must be stable and benchmarked to provide a comparison baseline.
