# Orbit - Research & Development Roadmap

## 1. Purpose

This document tracks experimental and future-facing work that extends beyond the [MVP specification](0001-mvp-spec.md). Each track listed here is independent - it may be pursued, deferred, or abandoned outright based on prototype results, resource availability, and community need. Nothing in this document is committed. These are research directions, not promises.

Tracks are roughly ordered by expected impact and feasibility. A prioritization matrix at the end summarizes the landscape.

---

## 2. Track: Media over QUIC (Iroh)

### Problem

WebRTC carries legacy complexity. ICE negotiation is unreliable, SRTP adds overhead, SDP is a bloated signaling format, and NAT traversal success rates are inconsistent across network configurations. For native desktop clients where we control both endpoints, we can do better.

### Proposal

Replace WebRTC with Media over QUIC (MoQ) using the [Iroh](https://iroh.computer/) networking stack (Rust) for native desktop clients. Iroh provides:

- **QUIC-native transport** with built-in stream multiplexing and congestion control
- **Advanced NAT traversal** via relay-assisted hole punching with high success rates in practice
- **Poll-based streaming** - encode and transmit frames only when a viewer is actively connected, reducing idle resource usage
- **Sub-50ms latency** for screen broadcasting in controlled test environments

### Browser Fallback

Web clients would receive the same media streams via WebTransport over HTTP/3. This avoids the need for WebRTC entirely on the browser side, provided browser support is sufficient.

### Risks

- MoQ is still an active IETF working group (draft stage), not a finished standard. The wire format and semantics may change.
- Iroh's live media libraries are experimental and not battle-tested at scale.
- WebTransport browser support is incomplete. Safari does not support it as of mid-2025. This would require a WebRTC fallback for Safari users, partially defeating the purpose.
- Maintaining two media transports (WebRTC for MVP compatibility, MoQ for native) increases operational and debugging complexity significantly.

### Evaluation Criteria

Build a standalone prototype that streams screen capture from one native client to another via Iroh. Measure:

- End-to-end latency (target: <50ms on LAN, <150ms over WAN)
- NAT traversal success rate across residential, corporate, and mobile networks
- CPU and bandwidth usage under sustained load
- Reliability over degraded network conditions (packet loss, jitter)

Compare directly against the Satellite (LiveKit/WebRTC) baseline from the MVP. Only proceed if MoQ delivers measurable, meaningful improvements that justify the added complexity.

### Dependencies

MVP Satellite infrastructure must be stable and benchmarked to provide a comparison baseline.

---

## 3. Track: Leptos / WASM Frontend

### Problem

The MVP desktop client uses Svelte compiled to JavaScript, rendered in the Tauri WebView. While this is efficient and productive, it still involves JS execution overhead and requires serialization across the Tauri IPC bridge for every backend call. Complex data structures (channel trees, user presence maps, message histories) pay a serialization tax on every update.

### Proposal

Evaluate rewriting the desktop frontend in [Leptos](https://leptos.dev/) (Rust → WASM). Potential benefits:

- **Shared Rust types** between the Tauri backend and frontend - no serialization overhead for complex structures crossing the boundary
- **Lower memory footprint** from fine-grained reactive signals and no JS runtime overhead
- **Single-language codebase** - reduces context-switching cost for contributors and enables shared utility code

### Risks

- The Leptos ecosystem is immature. Component libraries are sparse, devtools are limited, and documentation has gaps. Building production UI will require writing a lot of primitives from scratch.
- WASM debugging is meaningfully harder than JS debugging. Source maps exist but are less reliable. Stack traces are less readable. `console.log` still works but structured devtools inspection does not.
- Developer onboarding cost is high. Rust + Leptos is a niche skill set. This narrows the contributor pool considerably.
- The memory improvement may be marginal in practice. Svelte is already lightweight. A 5-10% memory reduction would not justify a rewrite.
- The web widget must remain JS-based regardless. WASM bundle sizes are too large for an embeddable widget. This means maintaining two frontend codebases - the opposite of simplification.

### Evaluation Criteria

Build a representative subset of the Orbit UI in Leptos:

- Channel list sidebar with collapsible categories
- Chat message view with scrollback loading
- Voice channel panel with participant list and controls

Benchmark against the Svelte implementation:

- Memory usage (target: >30% reduction to justify the effort)
- Startup time (cold and warm)
- IPC overhead for frequent updates (typing indicators, presence changes)

Evaluate developer experience qualitatively: how long does a typical UI change take? How debuggable are issues? Only proceed if the improvement is substantial AND the DX is acceptable for a small team.

### Dependencies

MVP client must be feature-complete to serve as a fair comparison target.

---

## 4. Track: Linux Gaming Overlay

The overlay scope is deliberately narrow: a **speaker indicator** (avatar thumbnails with a speaking ring showing who is active in voice) and an optional **webcam pip** (a small floating video tile, similar to Discord's picture-in-picture). No chat window, no notification feed, no interactive UI. This constraint is what makes the Vulkan approach tractable.

### Tier 1: Wayland Layer-Shell + X11 Composite Overlay

#### Problem

Gamers need to see who is speaking in a voice channel without alt-tabbing. A speaker indicator — avatar thumbnails with a coloured ring toggling on voice activity — covers this. An optional webcam pip covers the video case. Both are passive, non-interactive displays. Linux display server fragmentation (X11 vs. Wayland, multiple compositors) is the main engineering challenge, not the UI complexity.

#### Proposal

For windowed and borderless-fullscreen games (which still go through the compositor):

- **X11**: Create a composite overlay window using Xlib. Set `_NET_WM_WINDOW_TYPE_NOTIFICATION` (or `_NET_WM_WINDOW_TYPE_DOCK`) to hint that the window should float above others. Use `XShapeCombineRectangles` to define input passthrough regions so mouse clicks fall through to the game underneath. Render the overlay using a lightweight renderer (e.g., tiny-skia) — a handful of pre-rasterized avatar images and coloured rings is trivial to draw.
- **Wayland**: Use the `wlr-layer-shell` protocol (supported by wlroots-based compositors: Sway, Hyprland, river, etc.) to request the `Overlay` layer. Disable exclusive zone so the overlay doesn't push application windows. Set keyboard interactivity to `none` so the overlay never steals focus from the game.

#### Risks

- **Compositor fragmentation**: `wlr-layer-shell` is not supported by GNOME's Mutter or KDE's KWin (though KWin has been adding support). This limits Wayland coverage to wlroots-based compositors, which skews toward power users.
- **Performance**: Overlay rendering must be extremely lightweight. Even a few milliseconds of additional frame time is noticeable in competitive gaming. The overlay must not cause dropped frames.
- **Maintenance burden**: Supporting both X11 and Wayland codepaths is double the platform code, double the testing, double the bugs.

#### Evaluation Criteria

Prototype both overlay approaches showing a static speaker indicator (3 avatar thumbnails, one with an active speaking ring). Test with 5+ popular Linux-native and Proton games in windowed and borderless-fullscreen modes. Measure frame time impact (target: <0.5ms added per frame). Test on at least three compositors (Sway, Hyprland, GNOME if feasible).

---

### Tier 2: Vulkan Explicit Layer

#### Problem

Tier 1 overlays fail when a game takes exclusive fullscreen via direct scanout, because the compositor is bypassed entirely. The overlay window simply isn't visible — the game owns the display output directly. This is common with competitive games and VR applications.

#### Proposal

Write a custom Vulkan implicit layer (a shared object / `.so` file) in Rust that:

1. **Registers** via a JSON manifest file placed in the Vulkan loader's layer search path
2. **Intercepts** `vkQueuePresentKHR` — the Vulkan call that submits a frame for display
3. **Composites** the speaker indicator and optional webcam pip onto the game's framebuffer before presentation, using pre-rasterized avatar textures and coloured border quads

This is architecturally similar to how [MangoHud](https://github.com/flightlessmango/MangoHud) works. The rendering scope — a small number of static avatar images, a coloured ring per avatar, and optionally one or two decoded video frames for the webcam pip — is comparable to MangoHud's GPU graph and frame time history in complexity. This is not a full UI toolkit problem.

#### Risks

- **Driver compatibility**: Must work across AMD (RADV, AMDVLK), NVIDIA (proprietary driver), and Intel (ANV) Vulkan implementations. Each has quirks in layer interception, memory allocation, and synchronization. This is the primary engineering risk, not the rendering complexity.
- **Translation layer interop**: Many Linux games run through DXVK or VKD3D-Proton. The layer must not interfere with these translation layers. MangoHud navigates this successfully, so it is solved territory — but requires testing.
- **Webcam video texture**: Uploading decoded video frames as a Vulkan texture each frame requires careful synchronization (CPU → GPU transfer, format negotiation). This is the most novel piece relative to MangoHud's scope, but it is a well-understood GPU programming pattern.
- **Input handling**: The overlay is passive (no interaction), so input passthrough is not a concern. A hotkey to toggle visibility is the only input needed, and that can be handled out-of-band by the Orbit client process via a shared memory flag or signal.
- **Stability**: A crashing Vulkan layer crashes the game. Zero tolerance for bugs. Incremental rollout with opt-in is essential.

#### Evaluation Criteria

Build a Vulkan layer prototype that:

1. Successfully intercepts `vkQueuePresentKHR`
2. Composites 3 pre-rasterized avatar images with a coloured border in a corner of the screen
3. Works on AMD (RADV) and NVIDIA (proprietary) drivers without crashes or visual artifacts in at least 10 test games, including 2+ running under Proton/DXVK
4. Adds less than 0.5ms of frame time overhead

If stable, add webcam pip: upload a decoded video frame as a texture and composite it into a second corner. If that works cleanly, Tier 2 is shippable.

#### Dependencies

Tier 1 must be shipped first. Tier 2 is not a replacement — it covers the exclusive-fullscreen case that Tier 1 cannot reach. Both ship the same visual feature set; only the rendering path differs.

---

## 5. Track: Federation

### Problem

The MVP is single-server. All users in a community connect to one Orbit server instance. Users on server A cannot communicate with users on server B. Communities are isolated islands.

A deeper problem sits underneath federation itself: **identity bridging**. IRC handles user authentication within its own protocol boundary (SASL, NickServ, `account-tag`), but those assertions are server-scoped. They don't travel outside the IRC connection. When a user connects to a Satellite node (a completely separate service), the Satellite has no native way to verify that this person is the same `zealsprince` who is authenticated on IRC. The MVP punts on this with a public join key model (see [0001 §5.7](0001-mvp-spec.md#57-satellite-authentication)), but that model cannot support federation, cross-server trust, or even basic identity display in voice sessions.

Critically, the solution must not require modifications to the IRC server. Orbit's design philosophy is explicit: **Orbit is a layer on top of existing IRC.** Any IRCv3 server that supports message tags can become Orbit-enabled. The identity bridging mechanism must be a standalone component that works alongside any compliant IRC server, not a patch to one specific implementation. And it must be **optional** — if the component isn't deployed, everything else still works. The experience degrades gracefully, not catastrophically.

### Proposal

Two layers:

1. **IRC network linking** for text-layer federation (the control plane).
2. **Signed identity assertions** for decoupling Satellite nodes from IRC while preserving verifiable identity.

#### Layer 1: IRC Network Linking

Leverage IRC's native server linking. IRC was designed for multi-server networks from the very beginning - this is one of the reasons we chose it as the protocol foundation. [Ergo](https://ergo.chat/) (Ground Control's IRC server) supports server-to-server linking out of the box.

However, basic IRC linking (multiple servers forming one logical network) is not the same as true federation (independent organizations running independent servers that interoperate as peers). True federation raises hard questions:

- **Identity portability**: How does a user registered on server A prove their identity when interacting with server B? Federated identity is a deep problem (see: email, Matrix, ActivityPub - all have different trade-offs).
- **Channel namespacing**: How do you distinguish `#general` on server A from `#general` on server B? IRC traditionally uses a flat namespace. Federation requires hierarchical or scoped naming.
- **Media routing**: If users from two federated servers are in the same voice channel, which Satellite node handles the media? Do you relay between nodes? Who pays for the bandwidth?
- **Moderation boundaries**: Who has moderation authority in a federated channel? Can server A's admins moderate users from server B? What about content policies that differ between servers?
- **History synchronization**: Do federated servers share message history? How much? What happens when a server goes offline and comes back - does it backfill?

Start with the simplest model: IRC network linking where multiple Ground Control (Ergo) instances form a single logical network under shared administration. This is well-understood IRC infrastructure with decades of operational experience. True cross-organization federation (more analogous to email or Matrix) is a separate, later problem. Do not attempt to design a general federation protocol until the simpler model is deployed and its limitations are understood in practice.

#### Layer 2: Signed Identity Assertions (Identity Bridging)

IRC identity today is **server-asserted**. SASL authenticates a user to the IRC server; the `account-tag` (IRCv3) lets other clients on the same server see which account sent a message. But none of this produces a portable, cryptographically verifiable token that a separate service can check. The Satellite is a separate service. It needs a bridge.

The proposal: **a standalone service called Transponder signs short-lived identity tokens that Satellite nodes can independently verify.** This is a minimal OIDC-like pattern, stripped down to what we actually need — no OAuth dance, no redirect flows, no token refresh complexity. Just a signed assertion. The IRC server is not modified in any way.

**Transponder** is a small, standalone service — part of the Orbit ecosystem, not the IRC ecosystem. It is deployed alongside the IRC server (e.g., in the same `docker-compose.yml`) but has no code-level dependency on any specific IRC server implementation. It has two components: a lightweight **IRC bot** connected to Ground Control, and an HTTP endpoint that publishes its signing public key. It needs exactly one thing from the IRC server: **server-asserted `account-tag` on messages** — a standard IRCv3 feature that cannot be forged by clients.

How it verifies identity: **Transponder trusts the IRC server's own identity assertions.** If you're authenticated to the IRC server via SASL, the server attaches your `account-tag` to every message you send. Transponder's IRC bot reads this tag directly — no separate authentication required, no access to auth backends, no password replay. The IRC server is the single source of truth, and Transponder trusts it.

This is the key distinction from approaches that require Transponder to access the IRC server's auth backend directly. Transponder doesn't need to know whether Ground Control uses Ergochat's internal accounts, LDAP, SQL-backed auth, client certificates, or an external OIDC provider. It doesn't care. It only sees the `account-tag` that the IRC server asserts after authenticating the user through whatever backend it uses. This makes Transponder truly IRC-server-agnostic — it works with any IRCv3 server that supports `account-tag`, which is all of them.

**How it works:**

1. **Transponder has a signing keypair** (Ed25519). Generated at deployment, stored in its own configuration. The IRC server knows nothing about this key.

2. **Transponder publishes its public key** via one or more of:
   - `/.well-known/orbit/keys.json` on the server's domain (served by Transponder itself or a reverse proxy)
   - A DNS TXT record (analogous to DKIM: `orbit._keys.example.com`)
   - A DNS SRV record `_transponder._tcp.example.com` pointing to the Transponder service (consistent with how Satellite nodes and Ground Control are discovered)

3. **When a user wants to join a Satellite session**, the Orbit client requests an identity token from Transponder via IRC — no separate HTTP authentication. The client sends a `TAGMSG` to the Transponder bot's nickname with a `+orbit/token-request` tag containing a nonce. The Transponder bot receives this message, reads the server-asserted `account-tag` (which cannot be forged by clients — the IRC server attaches it based on SASL authentication), and responds with a signed identity token via `TAGMSG` back to the client. No second login. No password replay. The IRC server is the authority, and Transponder trusts its `account-tag` assertion.

```
Orbit Client        Ground Control (any IRCv3)         Transponder Bot        Satellite
    │                       │                              │                       │
    │  SASL auth            │                              │                       │
    │──────────────────────►│                              │                       │
    │  Connected, authed    │                              │                       │
    │◄──────────────────────│                              │                       │
    │                       │                              │                       │
    │  TAGMSG transponder-bot                              │                       │
    │  +orbit/token-request={nonce}                        │                       │
    │──────────────────────►│  (relayed with account-tag)  │                       │
    │                       │─────────────────────────────►│                       │
    │                       │                              │  reads account-tag,   │
    │                       │                              │  signs identity token  │
    │  TAGMSG from transponder-bot                         │                       │
    │  +orbit/identity-token={signed_jwt}                  │                       │
    │◄──────────────────────│◄─────────────────────────────│                       │
    │                                                                              │
    │  POST /session/join {identity_jwt}                                           │
    │─────────────────────────────────────────────────────────────────────────────►│
    │  LiveKit JWT (verified: true)                                                │
    │◄─────────────────────────────────────────────────────────────────────────────│
```

4. **The identity token contains:**
   - `sub`: IRC account name (e.g., `zealsprince`)
   - `iss`: server identifier (e.g., `irc.hivecom.net`)
   - `aud`: target Satellite session or node (optional, can be broad)
   - `iat`: issued-at timestamp
   - `exp`: expiration (short-lived — 5 minutes is sufficient, the token is only used to obtain a LiveKit JWT)

5. **The client presents this token to the Satellite token service** in the `/session/join` request.

6. **The Satellite verifies the signature** against Transponder's known public key (configured at node setup or fetched from `.well-known`). If valid, it issues a LiveKit JWT with the verified account name baked into the participant identity.

This keeps **both the IRC server and the Satellite completely unchanged**. The IRC server does what IRC servers do — authenticate users, relay messages, manage channels. The Satellite does what media servers do — route audio and video. Transponder is the only Orbit-specific server component, and it's a thin, stateless bridge between the IRC server's identity assertions and a signing key. It requires nothing from the IRC server beyond standard IRCv3 `account-tag` support — no patches, no plugins, no access to internal databases.

#### Verified and Unverified Users

A Satellite node is a media server. There is no requirement that every participant be an IRC user. Legitimate use cases exist for non-IRC participants: someone shares a voice room link with a friend who doesn't have an account, a BYON node operator wants open access, a website embeds a voice widget for anonymous visitors.

The Satellite token service can issue tokens in two modes:

| Mode | How they authenticate | LiveKit JWT contains | Orbit UI treatment |
|------|----------------------|---------------------|--------------------|
| **Verified** | Signed identity token from Transponder | `account: "zealsprince"`, `server: "irc.hivecom.net"`, `verified: true` | Display name + verified indicator (e.g., checkmark, badge) |
| **Unverified** | Join key, room password, or other node-level auth | `display_name: "some-name"`, `verified: false` | Display name shown, no verification badge, clear "unverified" indicator |

This is the same pattern as Jitsi (anyone can join, some users are logged in) or Bluesky (verified domain handles vs. unverified accounts). No gatekeeping — just transparency. The Orbit client displays the distinction clearly so users can make informed trust decisions.

Unverified users:
- Can join voice/video sessions if the node's auth policy allows it (join key, password, or open access).
- Have self-asserted display names. These are **not trustworthy** — the UI must never present them as equivalent to verified identities.
- Cannot impersonate a verified user. If a verified `zealsprince` is in the session, an unverified participant claiming the same name should be visually distinguishable (e.g., suffixed with a tag, different color, or simply lacking the badge).
- Are subject to the same moderation controls as anyone else in the session (mute, kick, etc.).

#### Graceful Degradation

Transponder is **optional**. If a server operator doesn't deploy it, nothing breaks:

| Feature | With Transponder | Without Transponder |
|---------|-----------------|-------------------|
| Text chat | Works (pure IRC) | Works (pure IRC) |
| Group voice / video | Works, participants verified | Works, all participants unverified |
| BYON | Works, IRC users verified | Works, everyone unverified |
| Web widget | Works (has its own JWT flow via widget gateway) | Works (has its own JWT flow via widget gateway) |
| P2P calls | Works, caller identity verified | Works, caller identity unverified |

The Orbit client detects whether a Transponder is available for the current domain (via DNS SRV `_transponder._tcp`, `/.well-known/orbit/keys.json`, or DNS TXT `orbit._keys`). If none is found, the client skips the identity token step and joins Satellite sessions as an unverified participant. The UI reflects this — no verification badges for anyone, but everything functions.

This is the correct default for the "Orbit works on any IRCv3 server" promise. Two people running Orbit on Libera.Chat with no Transponder and no server-operated Satellite can still use BYON voice. They both show up unverified because nobody's running the service. The experience is honest, not broken.

Transponder becomes valuable when a server operator wants a cohesive, identity-aware deployment — their own Ground Control, their own Satellites, and now verified identities tying them together. But it's an upgrade, not a prerequisite.

#### Federation Trust Chain

The signed identity model scales naturally to federation:

- **Same-server (MVP)**: Satellite trusts one Transponder's public key. Auto-configured at deployment — Transponder ships alongside Ground Control and the key is shared via local config.
- **Linked network**: Multiple Ground Control instances in a linked Ergo network share a Transponder (or run separate instances with cross-signed keys). All Satellites in the network trust the same key set.
- **True federation**: Satellites maintain a trust store of public keys from federated Transponder instances. Trust establishment can follow one of several models (to be evaluated):
  - **Manual**: server operator explicitly adds keys they trust (like SSH `known_hosts`). Most secure, highest friction.
  - **TOFU (Trust On First Use)**: accept a new server's key the first time it's encountered, warn on key change. Good balance of security and usability for small-scale federation.
  - **Directory-based**: a shared directory (e.g., hivecom.net's discovery service from [Track 9](#9-track-server-discovery-and-directory)) vouches for key-to-server bindings. Centralizes trust but simplifies onboarding.
  - **DNS-based**: publish the public key in a DNS TXT record, verified via DNSSEC. Decentralized, leverages existing infrastructure. Analogous to DKIM for email.

This is the same spectrum that email traversed with SPF/DKIM/DMARC and that ActivityPub is navigating now with HTTP Signatures. We have the advantage of learning from those efforts and choosing the simplest model that works for our scale.

### Approach

1. **Phase 0 (pre-federation)**: Implement Transponder and token verification in the Satellite token service for single-server deployments. This replaces the "public join key" model from the MVP for verified users while keeping join key / password access for unverified users. Transponder connects to Ground Control as an IRC bot and uses `account-tag` for identity verification — no auth backend adapters needed, works with any IRCv3 server out of the box. This is valuable even without federation — it gives Satellite nodes real identity without touching the IRC server.
2. **Phase 1 (IRC linking)**: Set up a two-server Ergo linked network. Test text federation. Both servers share the same Transponder (or run separate instances with cross-signed keys). Satellites trust both.
3. **Phase 2 (cross-org federation)**: Independent servers with independent Transponder instances and independent keys. Implement trust store management in the Satellite. Evaluate TOFU vs. directory vs. DNS-based trust models via prototype.

Do not jump to Phase 2 until Phase 1 is deployed and its limitations are understood in practice.

### Risks

- IRC server linking is historically fragile under netsplits (network partitions). When the link between servers drops and reconnects, state reconciliation (channel membership, modes, bans) can produce surprising results.
- Media federation (routing voice/video across Satellite node boundaries) is a non-goal by design. Satellite nodes do **not** communicate with each other — there is no inter-node media routing or relay. A server operator advertises one or more Satellite nodes via DNS SRV records (e.g., regional "NA" and "EU" nodes). Users choose which node to join based on region or preference. All participants in a voice session connect to the same Satellite node. Sessions are designed for small pods: **32–64 concurrent participants maximum** per session. This is a communication tool for communities, not a broadcasting platform. If a community needs more capacity, the operator runs more Satellite nodes and different groups or channels use different nodes. This dramatically simplifies the federation story: federation is about *identity* (Transponder) and *text* (IRC linking), not media. Media stays local to a single node per session.
- The complexity-to-value ratio may be unfavorable. If most Orbit communities are self-contained (like most Discord servers are), federation may serve a small minority of users at significant engineering cost.
- Key management adds operational burden. Server operators need to protect signing keys, rotate them, and handle key compromise. This is table-stakes for any cryptographic identity system, but it's still work.
- The verified/unverified model could create a two-tier user experience that feels exclusionary if not handled carefully in the UI. The design must frame verification as informational, not as a status hierarchy.

### Evaluation Criteria

**Identity bridging (Phase 0):**

- Implement Transponder: Ed25519 keypair generation, IRC bot with `account-tag` verification, TAGMSG-based token issuance, public key publication endpoint.
- Implement token verification in the Satellite token service.
- Verify that a user authenticated on IRC receives a verified LiveKit JWT, and that a user with only a join key receives an unverified JWT.
- Verify that the Orbit client displays verified and unverified participants distinctly.
- Measure latency overhead of the token request flow (should be negligible — one additional round-trip).

**IRC linking (Phase 1):**

- Set up a two-server Ergo linked network. Test:
  - Text chat across the link (message delivery, ordering, latency)
  - Media signaling relay (can a user on server A join a voice session hosted on server B's Satellite node, using server A's Transponder token?)
  - History synchronization (what happens to message history when the link drops and reconnects?)
  - Failure modes (what breaks during a netsplit? how does it recover?)

**Federation trust (Phase 2):**

- Test cross-server identity verification: user on server A presents a token to a Satellite that only trusts server B's key. Verify rejection. Add server A's key to the trust store. Verify acceptance.
- Evaluate TOFU, directory, and DNS-based trust establishment. Document trade-offs in operational complexity, security guarantees, and UX friction.

Identify where each phase breaks and document the gaps before proceeding to the next.

### Dependencies

- The MVP must be stable on single-server deployments. Federation on a shaky foundation is a recipe for compounding bugs.
- Phase 0 (Transponder) should be implemented early post-MVP — it improves single-server Satellite auth and is a prerequisite for all federation phases. It requires no IRC server changes. It is also optional — deployments without Transponder degrade gracefully to fully-unverified Satellite sessions.
- Phase 2 depends on [Track 9 (Server Discovery)](#9-track-server-discovery-and-directory) if directory-based trust is pursued.

---

## 6. Track: End-to-End Encryption

### Problem

The MVP provides transport encryption (TLS for IRC, DTLS-SRTP for WebRTC media). The server can read all text messages and could theoretically inspect media streams. For many communities this is fine - they trust their own server. But some users and organizations require true end-to-end encryption where the server is unable to read message content even in principle.

### Options

Two established approaches:

- **Double Ratchet (Signal Protocol)**: Well-proven for 1-on-1 messaging with forward secrecy and post-compromise security. Mature Rust implementations exist (`libsignal-protocol`). The challenge is extending to group chats - Sender Keys (as used by Signal's group messaging) work but have weaker security properties than pairwise ratchets. Scales to ~1000 members before performance degrades.
- **MLS (Messaging Layer Security)**: [IETF RFC 9420](https://www.rfc-editor.org/rfc/rfc9420.html). Purpose-built for group E2E encryption with efficient key management via ratchet trees. Designed to scale to large groups. [OpenMLS](https://openmls.tech/) is a Rust implementation. More complex than Signal but architecturally better suited for group channels.

### Risks

E2E encryption fundamentally conflicts with several server-side features:

- **Search**: The server can't index messages it can't read. Search must happen client-side, which means downloading and decrypting potentially large amounts of history.
- **History for new members**: When a new user joins an E2E-encrypted channel, they can't read messages sent before they joined (by design). This is a UX surprise for users coming from Discord.
- **Link previews**: Server-side link unfurling doesn't work. Previews must be generated client-side.
- **Moderation**: Server administrators cannot review reported messages in E2E channels. Content scanning for abuse prevention is impossible. This creates real trust & safety challenges.
- **Key management UX**: Key verification (ensuring you're talking to the right person), multi-device key synchronization, and key recovery after device loss are notoriously hard UX problems. Signal has invested years in making this seamless. We would be starting from scratch.
- **Anonymous users**: The MVP anonymous/guest access model and E2E encryption are architecturally at odds. E2E encryption requires stable identity (persistent key pairs). Anonymous users are, by definition, transient.

### Evaluation Criteria

Prototype E2E encrypted direct messages using the Signal Protocol (via `libsignal-protocol` Rust crate):

- Implement key exchange and ratcheting for 1-on-1 DMs
- Evaluate UX for key verification (QR code scanning, safety number comparison)
- Test multi-device scenarios (user has desktop + mobile - how do keys sync?)
- Measure performance impact on message send/receive latency

Only extend to group encryption (via MLS) if DM encryption ships successfully and there is demonstrated user demand.

### Dependencies

Stable identity system from the MVP (account registration, authentication, device management).

---

## 7. Track: Mobile Clients

### Problem

A communication platform without mobile clients is not viable beyond early adopters. Users expect to receive messages and join voice on their phone. This is non-negotiable for long-term adoption.

### Options

Three approaches, each with different trade-offs:

1. **Tauri Mobile**: Tauri v2 supports iOS and Android build targets. The existing Svelte frontend could theoretically be reused with platform-specific adaptations. Maximizes code reuse. However, Tauri mobile is less mature than the desktop targets - expect rough edges, missing APIs, and platform-specific bugs.

2. **Native mobile**: Swift (iOS) and Kotlin (Android) apps consuming the same Ground Control + Satellite infrastructure. Best possible platform integration (notifications, background audio, OS-level permissions). But this doubles (or triples) the frontend development cost and requires platform-specific expertise.

3. **Progressive Web App (PWA)**: The web widget, expanded to full functionality, served as an installable PWA. Lowest development cost by far. But limited OS integration: no reliable push notifications on iOS without workarounds, no background audio reliability, no deep OS integration for calls.

### Recommendation

Start with approach 1 (Tauri Mobile) since it maximizes code reuse with the desktop client. Fall back to approach 2 (native) for platform-specific features that Tauri can't handle adequately (push notifications, background audio, call UI integration).

Approach 3 (PWA) is worth maintaining as a baseline for users who don't want to install an app, but should not be the primary mobile strategy.

### Risks

- Tauri mobile is young. The iOS and Android WebView abstractions have different capabilities and bugs compared to their desktop counterparts.
- Mobile UX patterns (swipe gestures, bottom navigation, pull-to-refresh, haptic feedback) require significant UI rework beyond simply porting the desktop layout to a smaller screen.
- Background voice connectivity on mobile is hard. Both iOS and Android aggressively kill background processes to save battery. Maintaining a voice connection while the app is backgrounded requires platform-specific workarounds (foreground services on Android, VOIP entitlements on iOS).
- Push notifications require a push relay server (FCM for Android, APNs for iOS). This introduces a centralized dependency - Google or Apple must be in the loop for notifications to work. This conflicts with decentralization goals but is pragmatically unavoidable. The proposed solution: a **server-operator-controlled push notification relay service** called **Beacon** — similar in spirit to Transponder. Beacon is a small, self-hostable component that connects to Ground Control as an IRC client, monitors for mentions and DMs directed at offline users, and fires push notifications via FCM/APNs. This keeps the push relay under the server operator's control rather than depending on a centralized Hivecom service. On Android, Beacon should support **UnifiedPush** as an alternative to FCM for users who prefer FOSS infrastructure — UnifiedPush is well-established in the open-source Android ecosystem (used by Element, Tusky, and others). For iOS, APNs is unavoidable, but the relay service itself remains self-hosted. Beacon is optional — if not deployed, no push notifications, but everything else works.

### Evaluation Criteria

Build a Tauri mobile prototype that:

- Connects to Ground Control (Ergo) and authenticates via SASL
- Displays a channel list and chat messages
- Joins a Satellite voice session with working audio
- Receives a push notification when mentioned (requires FCM/APNs integration)

Evaluate battery usage during a 1-hour voice session (target: not significantly worse than Discord mobile). Test background behavior when switching apps. Assess notification reliability.

### Dependencies

MVP desktop client and web widget must be stable. Mobile is not the place to discover backend bugs.

---

## 8. Track: Bot and Integration API

### Problem

Discord's bot ecosystem is one of its strongest retention mechanisms. Communities build custom moderation tools, music playback, game stat tracking, welcome flows, and hundreds of other automations via bots. Orbit needs an equivalent story.

### Current State

IRC already has a natural bot model - bots are just IRC clients. Any program that can speak the IRC protocol can connect to Ground Control (Ergo), join channels, and respond to messages. There are mature IRC bot libraries in every major language. For the MVP, this works.

### Proposal

A formal Orbit Bot API would layer additional capabilities on top of the IRC foundation:

- **Standardized event webhooks**: HTTP callbacks fired when specific events occur (message posted, user joined, channel created). This lets bot developers use any HTTP-capable language/framework without implementing IRC protocol handling.
- **REST API for actions**: HTTP endpoints for sending messages, managing channels, querying message history, updating user metadata. Wraps IRC commands in a more accessible interface.
- **Bot-specific authentication**: API keys or OAuth2 client credentials rather than SASL user accounts. Scoped permissions per channel or server.
- **Rate limiting and permission scoping**: Bots get specific, auditable permissions. Prevents a misbehaving bot from spamming or performing unauthorized actions.
- **Registry / marketplace**: Long-term, a directory where communities can discover and install bots. Very long-term.

### Approach

Phase this incrementally to avoid over-engineering:

1. **Phase 0 (day-one, ships with MVP)**: Write documentation on how to build IRC bots against Ergo. Provide example bots in Rust, Python, and JavaScript. This is zero engineering cost and enables the community immediately — it should launch alongside the MVP, not after it.
2. **Phase 1**: Build an HTTP webhook bridge - a lightweight service that connects to Ground Control as an IRC client, listens for configurable events, and fires HTTP POST requests to registered webhook endpoints. This is a small, self-contained service.
3. **Phase 2**: Build a REST API gateway that wraps common IRC commands (send message, join channel, set topic, kick user, query WHOIS) in HTTP endpoints with JSON request/response bodies.
4. **Phase 3**: Formalize as a versioned API specification (OpenAPI). Add authentication scoping, rate limiting, and developer documentation portal.

A **client plugin system** for the Orbit desktop and web clients — allowing third-party UI extensions that integrate with IRC bots and Satellite services — is considered high-value and should be designed early in the post-MVP phase. The combination of an IRC bot (server-side logic) and a client plugin (UI integration) is the Orbit equivalent of a Discord bot with slash commands and embeds. This is the path to a rich integration ecosystem without requiring Orbit itself to implement every feature.

### Risks

- Building an HTTP API layer on top of IRC is building a new abstraction over an existing protocol. Leaky abstractions are inevitable - some IRC behaviors won't map cleanly to REST semantics (e.g., IRC's event-driven nature vs. REST's request-response model).
- Webhook reliability requires queuing and retry infrastructure. A naive implementation that fires HTTP requests and hopes they arrive will lose events under load or when endpoints are temporarily down.
- Security scoping for bots needs careful design. A bot with `send_message` permission in `#general` should not be able to escalate to operator privileges. IRC's permission model is coarse - the API layer must enforce finer-grained access control on top of it.

### Evaluation Criteria

Ship the webhook bridge (Phase 1) as an optional, self-hostable service. Announce it to the community. If bot developers actually adopt it and build useful integrations, that validates further investment in Phase 2+. Let adoption data drive the roadmap, not speculation.

### Dependencies

MVP Ground Control deployment must be stable. The webhook bridge is a client of Ergo - if Ground Control is unstable, the bridge inherits that instability.

---

## 9. Track: Server Discovery and Directory

### Problem

If Orbit is decentralized (anyone can run a server), how do users find communities? Discord solves this with a centralized server directory and invite links. Orbit needs something equivalent.

### Proposal

A public directory service where server administrators can opt in to list their community. Searchable by name, description, tags, and approximate member count. Provides invite links or connection details.

### Approach

Start simple:

- A section on [hivecom.net](https://hivecom.net) backed by a database of server listings
- Server admins submit their listing via a web form or API call
- Listings include: server name, description, tags/categories, member count (self-reported or queried), connection URI, banner image
- Basic moderation: Hivecom team reviews listings to prevent abuse

Evaluate decentralized alternatives later:

- DNS TXT records advertising Orbit server metadata (similar to how email uses MX records)
- A shared, replicated registry that multiple directory instances can sync
- Client-side crawling of known servers

### Risks

- A centralized directory run by Hivecom is a single point of control, which contradicts the decentralization ethos. But it's the pragmatic starting point - decentralized discovery is a hard problem and not worth solving before there are servers to discover.
- Content moderation of the directory itself is necessary. Malicious, illegal, or abusive communities should not be promoted. This requires policy decisions and human review.
- Privacy: some communities explicitly do not want to be discoverable. The directory must be strictly opt-in.

### Evaluation Criteria

Launch a minimal directory on hivecom.net once multiple independent Orbit communities exist. Track listing submissions and user traffic. If the centralized model works, keep it. If the community demands decentralization, revisit.

### Dependencies

Multiple Orbit communities must exist to populate the directory. This depends on the MVP being easy to deploy and operate.

---

## 10. Prioritization Matrix

The following table summarizes each track's expected impact, implementation feasibility, risk level, and suggested timeline. "Post-MVP" indicates work that could begin once the MVP ships. "R&D" indicates longer-term exploration without a committed timeline.

| Track | Impact | Feasibility | Risk | Suggested Phase |
|-------|--------|-------------|------|-----------------|
| Bot & Integration API Phase 0 (IRC docs + examples) | High | High | Low | MVP (day-one) |
| Bot & Integration API Phase 1+ (webhook bridge, REST API) | High | High | Low | Post-MVP (immediate) |
| Client Plugin System | High | Medium | Low | Post-MVP (Q2) |
| Mobile Clients (Tauri Mobile) | High | Medium | Medium | Post-MVP (Q2) |
| Layer-Shell / X11 Overlay (Tier 1) | Medium | High | Low | Post-MVP (Q2–Q3) |
| Federation Phase 0 (Transponder) | High | High | Low | Post-MVP (immediate) |
| Federation Phase 1–2 (IRC linking, cross-org) | Medium | Medium | Medium | Post-MVP (Q3+) |
| Server Discovery Directory | Medium | High | Low | Post-MVP (Q3) |
| E2E Encryption (DMs) | Medium | Medium | High | Post-MVP (Q3–Q4) |
| Media over QUIC (Iroh) | High | Low | High | R&D (ongoing) |
| Leptos / WASM Frontend | Low | Medium | Medium | R&D (evaluate after the MVP) |
| Vulkan Gaming Overlay (Tier 2) | High | Medium | Medium | Post-MVP (Q3–Q4) |

**Reading this table**:

- **Impact** reflects how much the feature would matter to users if it shipped successfully.
- **Feasibility** reflects how realistic it is to build with current team size and technology maturity.
- **Risk** reflects the likelihood of the effort failing or producing unusable results.
- Items with High Impact but Low Feasibility and High Risk (Media over QUIC) are worth researching but should not be depended on. They are bets, not plans.
- The Vulkan Overlay (Tier 2) has been rescoped to speaker indicator + webcam pip only — comparable rendering complexity to MangoHud — which makes it a realistic Post-MVP deliverable rather than a long-term R&D bet.
- Items with High Feasibility and Low Risk (Bot API, Server Directory, Transponder) should be prioritized early because they deliver value cheaply. Bot API Phase 0 (IRC docs and example bots) ships day-one with the MVP at zero engineering cost — Phase 1+ (webhook bridge, REST API) follows immediately post-MVP.
- Federation is now split: Phase 0 (Transponder — identity bridging via signed assertions) is high-feasibility foundational work that improves single-server Satellite auth immediately. It is also optional — deployments without it degrade gracefully. Phases 1–2 (IRC linking and cross-org federation) carry the traditional federation risks and are deferred until Phase 0 is proven.

---

## 11. Process

Each track follows the same lifecycle:

1. **Research**: Gather information, read specs, study prior art. Write up findings.
2. **Prototype**: Build the smallest possible thing that tests the core hypothesis. Time-box this - if a prototype takes longer than 4 weeks for a single developer, the scope is too large.
3. **Evaluate**: Measure the prototype against the evaluation criteria defined in this document. Be honest about results. "It kind of works" is not good enough to justify shipping.
4. **Decide**: Based on evaluation, either promote the track to a full specification (with a proper design spec and implementation plan), defer it (revisit later when conditions change), or abandon it (document why, so future contributors don't repeat the work).
5. **Spec**: If promoted, write a dedicated specification (similar to [0001](0001-mvp-spec.md)) covering architecture, implementation plan, testing strategy, and rollout.

Abandoned tracks are not failures. They are information. Document what was tried, what was learned, and why it didn't work out. This is more valuable than the prototype code itself.

---

## 12. Revision History

| Date | Change |
|------|--------|
| 2025-06 | Initial draft. All tracks at Research stage. |
| 2025-07 | Expanded Federation track: added Transponder (identity bridging via signed assertions), verified/unverified user model, graceful degradation, phased approach, federation trust chain. Transponder is a standalone, optional component — zero IRC server modifications required. |
