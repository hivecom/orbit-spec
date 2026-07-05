# Platform Comparison: Orbit vs Discord, Slack, and Teams

## Introduction

Discord, Slack, and Microsoft Teams are the three platforms most people reach for when evaluating a new communications tool. They serve different primary audiences (gaming communities, professional teams, and enterprise organizations respectively), but all three get cited when someone asks "why not just use X instead of Orbit?" This document answers that question.

The comparison isn't symmetrical. Orbit is built on IRC, uses a shared-server model rather than isolated community namespaces, and has an explicit philosophy of composability over completeness. Some of what looks like a gap in the table below is a model difference: the problem is solved a different way, or is out of scope by design. Those cases are marked separately from genuine gaps.

Where Orbit is behind, the table says so. The goal is to give operators, contributors, and prospective users an accurate picture of where things stand, what IRC hasn't standardized yet, and what isn't planned.

## Summary Comparison

**Legend:**
- `+` Supported
- `~` Not built yet: designed, or pending upstream IRC standardization
- `*` Model difference, covered differently by design
- `-` Not planned

| Category | Orbit | Discord | Slack | Teams |
|---|---|---|---|---|
| Text chat basics | + | + | + | + |
| Message editing | + | + | + | + |
| Message reactions | + | + | + | + |
| Message retractions | + | + | + | + |
| Replies and threads | + | + | + | + |
| Full-text search | + | + | + | + |
| Files and media | + | + | + | + |
| Group voice and video | + | + | + | + |
| Screen sharing | + | + | + | + |
| P2P 1:1 calls | + | + | + | + |
| Presence and status | + | + | + | + |
| Rich custom status | + | + | + | + |
| Permissions and moderation | * | + | + | + |
| Role hierarchies | * | + | + | + |
| Desktop app | + | + | + | + |
| Web app / PWA | + | + | + | + |
| Mobile (PWA) | + | + | + | + |
| Mobile native app | ~ | + | + | + |
| Embedded client | + | - | - | - |
| Push notifications | + | + | + | + |
| Bots and integrations | + | + | + | + |
| Webhook / REST gateway | ~ | + | + | + |
| OIDC / SSO / MFA | + | - | + | + |
| Guest / anonymous access | + | - | + | + |
| E2E encryption (1:1 DMs) | ~ | - | - | - |
| Self-hosting | + | - | - | - |
| Open protocol | + | - | - | - |
| Federation | - | - | - | - |
| Community / server model | * | + | + | + |
| Enterprise compliance tools | - | - | + | + |
| Application platform | - | + | + | + |
| Custom emoji | ~ | + | + | + |

## Category Breakdown

### Text chat basics

Orbit covers channels, DMs, group private conversations, persistent server-side history, read markers synced across devices, basic Markdown, emoji, and link/media previews. This is functionally on par with the competition. Channel hierarchy is rendered client-side using slash notation (`#dev/frontend`), an Orbit convention layered over standard IRC channel names.

### Message actions

**Message editing is covered by an interim mechanism.** IRC hasn't standardized in-place editing yet, and Orbit doesn't fork the server to add it. In the interim, Orbit clients edit via a client-side edit tag - that is the direction until a real solution lands at the protocol level, at which point Orbit adopts the standard and retires the tag. Two caveats: edits render between Orbit clients only (plain IRC clients keep seeing the original), and stored history isn't rewritten the way REDACT rewrites retractions. Reactions work today, handled client-side via message tags, with one concession: reactions can't be shown on messages surfaced purely from a search result. Retractions, replies (with inline excerpt), and client-managed threads are covered. Full-text search is available via Ergo's history backends plus an external indexer.

### Files and media

S3-compatible storage with inline image, audio, and video previews. Per-user quotas are supported in OIDC mode; open-access instances use rate limiting. An operator-controlled storage backend (MinIO, AWS S3, R2, B2) is a meaningful advantage over hosted platforms where storage costs and policies are opaque. File versioning and collaborative editing are not planned.

### Voice and video

Group voice/video with screen sharing (LiveKit SFU, Opus + VP9), P2P 1:1 calls (direct WebRTC with IRC signaling), push-to-talk, mute/deafen, per-user volume, session permissions, knocking, and ephemeral in-session chat. Standalone voice sessions via `satellite://` URI work without an IRC connection. Video call recording is not planned. Stage/streaming channels are not currently planned.

### Presence and status

Online, away, and offline states via `away-notify`, `extended-monitor`, and `draft/pre-away`. Rich status text and avatars via `draft/metadata-2`. This is equivalent to what Discord and Slack offer for basic presence. Teams' calendar-integrated presence (automatically setting status from meeting state) is out of scope.

### Permissions and moderation

This is a model difference, not a gap. Orbit uses IRC channel modes (`+o`, `+v`, `+b`) plus services bots for moderation. Operators have SAMODE, ban lists, quiet masks, and the full IRC management toolset. There is no layered role hierarchy in the core; bots can implement that on top if needed. Discord's role system is richer by default and requires no additional setup. Orbit's model is more composable but requires more operator intention. For communities already comfortable with IRC operations, this is not a regression.

### Platform reach

Desktop (Tauri v2, Linux/Windows/macOS, ~20 MB binary), web app, and PWA. An embedded iframe client with anonymous guest access, something none of the three competitors offer. Native mobile apps (iOS/Android via Tauri v2 Mobile) are designed but not built yet; the PWA works today on mobile but isn't a substitute for a native app in terms of notifications and background behavior.

### Notifications

Push notifications are available now: stock Ergo ships native Web Push via `draft/webpush`. In-app notifications also work. Native mobile delivery (FCM, APNs, UnifiedPush) pairs with the native mobile apps and isn't built yet, but Web Push covers the web and desktop surfaces today.

### Bots and integrations

Any IRC bot, in any language, on any framework works with Orbit on day one. This is a meaningful advantage: the IRC bot ecosystem is decades old and large. A webhook bridge and REST API gateway for Discord/Slack-style integrations are designed but not built. Until the webhook gateway exists, teams relying heavily on Slack-style workflow automations will need to adapt or bridge.

### Identity and auth

SASL PLAIN, SCRAM-SHA-256, SASL ANONYMOUS (guests), and OIDC via any compliant provider (Keycloak, Authentik, etc.) through the optional Transponder role. SSO, passkeys, and MFA are handled by the OIDC provider, the operator's choice. This is comparable to or better than Slack and Teams for operators who run their own identity stack. Discord has no SSO support.

### Self-hosting and openness

Orbit is fully self-hostable. The IRC transport is an open, standardized protocol. Any IRCv3-compatible client can connect. No vendor lock-in on the message store, identity provider, file storage, or voice infrastructure. Discord, Slack, and Teams are closed SaaS platforms with no self-hosting path.

### Community / server model

Discord and Slack use isolated community namespaces ("servers" / "workspaces"). Orbit doesn't. A single server instance hosts many communities simultaneously as sets of channels and permissions. There are no walled-off namespaces in the core. For operators running a single community, this is invisible. For operators running a network of communities, it's an advantage. For users accustomed to Discord's server-per-community model, it requires a small mental shift. Server discovery via `orbit.directory` is designed but not built.

### Enterprise compliance

DLP, eDiscovery, audit log exports, and regulatory compliance tooling are not planned. Slack and Teams have mature enterprise compliance products. Orbit is not a replacement for them in regulated industries.

### Application platform

Discord's Activities, Slack's app marketplace, and Teams' tab/app ecosystem are not planned for Orbit. Orbit doesn't embed third-party applications. Bots and bridges cover automation use cases; a full application platform is out of scope.

## What This Means in Practice

**Right now:** Orbit is a good fit for technical communities, open-source projects, gaming groups, and operators who want full control over their infrastructure. It covers text, voice, video, files, presence, identity, push notifications, reactions, and search well enough for daily use. Message editing works through an interim client-side tag: edits show between Orbit clients but not on plain IRC clients, and stored history isn't rewritten. Communities where full-fidelity editing is table stakes should weigh that until IRC standardizes editing.

**As IRC evolves:** when an editing standard lands in stock Ergo or IRCv3, Orbit adopts it without a fork and retires the interim edit tag, closing the last text gap. Orbit is already a credible general-purpose alternative for communities that prioritize self-hosting, open protocols, and operator control over convenience features. Federation is not a goal for now: it requires server-to-server linking that stock IRC servers don't provide.

**Who Orbit is not for:** Organizations that require enterprise compliance tooling (DLP, eDiscovery, audit exports), teams that depend heavily on embedded third-party applications, and anyone who needs a fully managed hosted service with commercial SLAs. Orbit is infrastructure, not a SaaS product.
