# Out of Scope for the MVP

The following are not part of the MVP release. Each item is tracked in the [Research Tracks](../research/).

Items are deferred because they require significant research, have unresolved design questions, or are non-essential enhancements. Nothing below is permanently rejected - only deferred.

| Feature | Reason for deferral | Research track |
|---|---|---|
| Media over QUIC / Iroh transport | Experimental; WebRTC is proven and sufficient for the MVP | [01-moq-iroh.md](../07-research/01-moq-iroh.md) |
| Leptos / WASM frontend rewrite | Research track; Vue + VUI delivers faster for now | [02-leptos-wasm.md](../07-research/02-leptos-wasm.md) |
| Vulkan gaming overlay (Linux) | Requires Vulkan layer injection - complex and fragile | [04-vulkan-overlay.md](../07-research/04-vulkan-overlay.md) |
| Wayland layer-shell / X11 overlays | Platform-specific complexity; needs prototyping | [03-linux-overlay.md](../07-research/03-linux-overlay.md) |
| End-to-end encryption | Key management UX is unsolved for group chat at scale | [06-e2e-encryption.md](../07-research/06-e2e-encryption.md) |
| Federation between Orbit instances | Requires protocol extensions beyond IRCv3 server linking | [05-federation.md](../07-research/05-federation.md) |
| Mobile clients (iOS / Android) | Tauri v2 mobile support exists but is immature | [07-mobile-clients.md](../07-research/07-mobile-clients.md) |
| Server directory / discovery service | Centralization concern; needs careful design | [10-server-discovery.md](../07-research/10-server-discovery.md) |
| Screen sharing | Voice and video are in the MVP; screen sharing is a post-MVP fast-follow | - |
| Multi-server support in client | Connect to one server at a time in the MVP | - |
| Custom emoji / sticker packs | Nice-to-have; not essential for launch | - |
| Message threads / forums | IRC doesn't have a native threading model; needs design work | - |
| Custom role / permission system | Use IRC modes; extend via extensions if needed | - |
| Message reactions | Reactions require a dedicated service (Reactor) for aggregated state, custom emoji, and sticker packs; IRC's append-only history cannot serve counts reliably | [12-reaction-service.md](../07-research/12-reaction-service.md) |
| Orbit extension API (orbit-app plugins) | Extension architecture is defined; the client-side plugin host and API surface are post-MVP | [09-bot-api.md](../07-research/09-bot-api.md) |
