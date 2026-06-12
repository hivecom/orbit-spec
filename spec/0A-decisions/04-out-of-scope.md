# Out of Scope for the MVP

The following are not part of the MVP release. Each item is tracked in the [Research Tracks](../0B-research/) where applicable.

Items are deferred because they require significant research, have unresolved design questions, or are non-essential enhancements. Nothing below is permanently rejected - only deferred.

| Feature | Reason for deferral | Research track / next step |
|---|---|---|
| Media over QUIC / Iroh transport | Experimental; WebRTC is proven and sufficient for the MVP | [MoQ / Iroh](../0B-research/01-moq-iroh.md) |
| Leptos / WASM frontend rewrite | Research track; Vue + VUI delivers faster for now | [Leptos / WASM](../0B-research/02-leptos-wasm.md) |
| Vulkan gaming overlay (Linux) | Requires Vulkan layer injection - complex and fragile | [Vulkan Overlay](../0B-research/04-vulkan-overlay.md) |
| Wayland layer-shell / X11 overlays | Platform-specific complexity; needs prototyping | [Linux Overlay](../0B-research/03-linux-overlay.md) |
| End-to-end encryption | Key management UX is unsolved for group chat at scale | [E2E Encryption](../06-next/05-e2e-encryption.md) |
| Federation between Orbit instances | Requires server-to-server linking absent from stock Ergo; deferred and not a planned track, not a fork reason | [Federation](../06-next/01-federation.md) |
| Mobile clients (iOS / Android) | Tauri v2 mobile support exists but is immature | [Mobile Clients](../06-next/02-mobile-clients.md) |
| Server directory / discovery service | Centralization concern; needs careful design | [Server Discovery](../06-next/06-server-discovery.md) |
| Custom emoji / sticker packs | Nice-to-have; not essential for launch | - |
| Custom role / permission system | Use IRC modes; extend via extensions if needed | - |
| Message editing (amend) | Not yet standardized in IRC; Orbit handles editing at the client layer until Ergo or IRCv3 ships a standard | Message Editing (next) |
| Message retractions | Available in MVP via native `draft/message-redaction` (`REDACT`, shipped in stock Ergo) | - |
| Push notifications | Available in MVP via native `draft/webpush` (stock Ergo v2.15.0+); additional mobile delivery backends are NEXT | [Push Notifications](../06-next/04-push-notifications.md) |
| Orbit extension API (Orbit application plugins) | Extension architecture is defined; the client-side plugin host and API surface are post-MVP | [Bot API](../06-next/03-bot-api.md) |
