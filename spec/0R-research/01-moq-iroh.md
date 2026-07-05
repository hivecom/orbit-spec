# Research: Media over QUIC (Iroh)

This track explores replacing WebRTC with Media over QUIC (MoQ) using the [Iroh](https://iroh.computer/) networking stack for native desktop clients. See [Satellite](../02-architecture/05-satellite.md) for the LiveKit/WebRTC baseline this track is evaluated against.

## Problem

WebRTC carries legacy complexity. ICE negotiation is unreliable, SRTP adds overhead, SDP is a bloated signaling format, and NAT traversal success rates are inconsistent across network configurations. For native desktop clients where we control both endpoints, we can do better.

## Proposal

Replace WebRTC with MoQ over Iroh (Rust) for native desktop clients. Iroh provides:

- QUIC-native transport with built-in stream multiplexing and congestion control
- Relay-assisted NAT hole punching with high success rates in practice
- Poll-based streaming: frames are encoded and transmitted only while a viewer is connected, which cuts idle resource usage
- Sub-50ms latency for screen broadcasting in controlled test environments

## Browser Fallback

Web clients would receive the same media streams via WebTransport over HTTP/3, which in principle avoids WebRTC on the browser side entirely.

In practice, WebTransport browser support is incomplete as of mid-2025. Safari doesn't support it, so a WebRTC fallback would still be needed for Safari users, partially defeating the purpose of the migration. Check [caniuse.com/webtransport](https://caniuse.com/webtransport) for current support rather than any point-in-time statement in this document.

The bridging between Iroh (a native QUIC stack) and WebTransport (a browser API over HTTP/3) is the least-specified part of this proposal. It isn't clear yet whether Iroh can act directly as a WebTransport server, or whether a separate gateway process would have to translate between Iroh's wire format and what browsers expect. Any prototype needs to answer this early, since it weighs heavily in the feasibility assessment.

## Risks

- MoQ is still an active IETF working group (draft stage), not a finished standard. The wire format and semantics may change.
- Iroh's live media libraries are experimental and not battle-tested at scale.
- WebTransport support is incomplete (see Browser Fallback above); the Safari gap alone forces a WebRTC fallback.
- Maintaining two media transports (WebRTC for compatibility, MoQ for native) increases operational and debugging complexity significantly.

## Evaluation Criteria

Build a standalone prototype that streams screen capture from one native client to another via Iroh. Measure:

- End-to-end latency (target: <50ms on LAN, <150ms over WAN)
- NAT traversal success rate across residential, corporate, and mobile networks
- CPU and bandwidth usage under sustained load
- Reliability over degraded network conditions (packet loss, jitter)

Compare directly against the [Satellite](../02-architecture/05-satellite.md) LiveKit/WebRTC baseline. Only proceed if MoQ delivers measurable improvements that justify the added complexity.

## Dependencies

[Satellite](../02-architecture/05-satellite.md) infrastructure must be stable and benchmarked to provide a comparison baseline.
