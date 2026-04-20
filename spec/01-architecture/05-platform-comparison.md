# Platform Comparison: Orbit vs Discord, Slack, and Teams

## Introduction

Discord, Slack, and Microsoft Teams are the three platforms most people reach for when evaluating a new communications tool. They serve different primary audiences - gaming communities, professional teams, and enterprise organizations respectively - but all three get cited when someone asks "why not just use X instead of Orbit?" This document answers that question honestly.

The comparison is not symmetrical. Orbit is built on IRC, uses a shared-uplink model rather than isolated community namespaces, and has an explicit philosophy of composability over completeness. Some of what looks like a gap in the table below is a deliberate model difference: the problem is solved a different way, or is considered out of scope by design. Those cases are marked separately from genuine gaps.

Where Orbit is behind, this document says so plainly. The goal is to give operators, contributors, and prospective users an accurate picture of where things stand today (MVP), where they are going (Uplink fork / NEXT), and what is not planned.

## Summary Comparison

**Legend:**
- `+` Supported
- `~` Planned - Uplink fork or post-MVP NEXT track
- `*` Model difference - covered differently by design
- `-` Not planned

| Category | Orbit MVP | Orbit + Fork | Discord | Slack | Teams |
|---|---|---|---|---|---|
| Text chat basics | + | + | + | + | + |
| Message editing | - | ~ | + | + | + |
| Message reactions | - | ~ | + | + | + |
| Message retractions | + | + | + | + | + |
| Replies and threads | + | + | + | + | + |
| Full-text search | - | ~ | + | + | + |
| Files and media | + | + | + | + | + |
| Group voice and video | + | + | + | + | + |
| Screen sharing | + | + | + | + | + |
| P2P 1:1 calls | + | + | + | + | + |
| Presence and status | + | + | + | + | + |
| Rich custom status | + | + | + | + | + |
| Permissions and moderation | * | * | + | + | + |
| Role hierarchies | * | * | + | + | + |
| Desktop app | + | + | + | + | + |
| Web app / PWA | + | + | + | + | + |
| Mobile (PWA) | + | + | + | + | + |
| Mobile native app | - | ~ | + | + | + |
| Embeddable widget | + | + | - | - | - |
| Push notifications | - | ~ | + | + | + |
| Bots and integrations | + | + | + | + | + |
| Webhook / REST gateway | - | ~ | + | + | + |
| OIDC / SSO / MFA | + | + | - | + | + |
| Guest / anonymous access | + | + | - | + | + |
| E2E encryption (1:1 DMs) | - | ~ | - | - | - |
| Self-hosting | + | + | - | - | - |
| Open protocol | + | + | - | - | - |
| Federation | - | ~ | - | - | - |
| Community / server model | * | * | + | + | + |
| Enterprise compliance tools | - | - | - | + | + |
| Application platform | - | - | + | + | + |
| Custom emoji | - | ~ | + | + | + |

## Category Breakdown

### Text chat basics

Orbit covers channels, DMs, group private conversations, persistent server-side history, read markers synced across devices, basic Markdown, emoji, and link/media previews. This is functionally on par with the competition at MVP. Channel hierarchy is rendered client-side using slash notation (`#dev/frontend`), which is an Orbit convention layered over standard IRC channel names - not a server-enforced namespace.

### Message actions

**This is the most significant honest gap at MVP.** Editing and reactions are not present yet - both are Uplink fork milestones. Retractions, replies (with inline excerpt), and client-managed threads are in MVP. Full-text search is a fork milestone. Operators running stock Ergochat today accept this trade-off; the Uplink fork closes it.

### Files and media

S3-compatible storage with inline image, audio, and video previews is in MVP. Per-user quotas are supported in OIDC mode; open-access instances use rate limiting. Operator-controlled storage backend (MinIO, AWS S3, R2, B2) is a meaningful advantage over hosted platforms where storage costs and policies are opaque. No file versioning or collaborative editing - that is not planned.

### Voice and video

Group voice/video with screen sharing (LiveKit SFU, Opus + VP9), P2P 1:1 calls (direct WebRTC with IRC signaling), push-to-talk, mute/deafen, per-user volume, session permissions, knocking, and ephemeral in-session chat are all in MVP. Standalone voice sessions via `satellite://` URI work without an IRC connection. Video call recording is not planned. Stage/streaming channels are not currently planned.

### Presence and status

Online, away, and offline states are in MVP via `away-notify`, `extended-monitor`, and `draft/pre-away`. Rich status text and avatars are in MVP via `draft/metadata-2`. This is equivalent to what Discord and Slack offer for basic presence. Teams' calendar-integrated presence (automatically setting status from meeting state) is out of scope.

### Permissions and moderation

This is a model difference, not a gap. Orbit uses IRC channel modes (`+o`, `+v`, `+b`) plus services bots for moderation. Operators have SAMODE, ban lists, quiet masks, and the full IRC management toolset. There is no layered role hierarchy in the core - bots can implement that on top if needed. Discord's role system is richer by default and requires no additional setup; Orbit's model is more composable but requires more operator intention. For communities already comfortable with IRC operations, this is not a regression.

### Platform reach

Desktop (Tauri v2, Linux/Windows/macOS, ~10-15 MB binary), web app, and PWA are in MVP. An embeddable iframe widget with anonymous guest access is in MVP - something none of the three competitors offer. Native mobile apps (iOS/Android via Tauri v2 Mobile) are a NEXT-track item; the PWA works today on mobile but is not a substitute for a native app in terms of notifications and background behavior.

### Notifications

**A genuine gap.** Push notifications (FCM, APNs, UnifiedPush, Web Push) are a fork milestone. In-app notifications work today. Running Orbit as a primary community platform without push notifications is workable for desktop-first communities, but is a real limitation for mobile users. This is on the roadmap and is not a model difference.

### Bots and integrations

Any IRC bot, in any language, on any framework works with Orbit on day one - no Orbit-specific API required. This is a meaningful advantage: the IRC bot ecosystem is decades old and large. A webhook bridge and REST API gateway for Discord/Slack-style integrations are NEXT-track items. Until the webhook gateway ships, teams relying heavily on Slack-style workflow automations will need to adapt or bridge.

### Identity and auth

SASL PLAIN, SCRAM-SHA-256, SASL ANONYMOUS (guests), and OIDC via any compliant provider (Keycloak, Authentik, etc.) via the optional Transponder component are in MVP. SSO, passkeys, and MFA are handled by the OIDC provider - operator's choice. This is comparable to or better than Slack and Teams for operators who run their own identity stack. Discord has no SSO support.

### Self-hosting and openness

Orbit is fully self-hostable. The IRC transport is an open, standardized protocol. Any IRCv3-compatible client can connect. No vendor lock-in on the message store, identity provider, file storage, or voice infrastructure. Discord, Slack, and Teams are closed SaaS platforms with no self-hosting path. This is a structural difference, not a feature comparison.

### Community / server model

Discord and Slack use isolated community namespaces ("servers" / "workspaces"). Orbit does not. A single Uplink instance hosts many communities simultaneously as sets of channels and permissions. There are no walled-off namespaces in the core. For operators running a single community, this is invisible. For operators running a network of communities, it is an advantage. For users accustomed to Discord's server-per-community model, it requires a small mental shift. Server discovery via `orbit.directory` is a NEXT-track item.

### Enterprise compliance

DLP, eDiscovery, audit log exports, and regulatory compliance tooling are not planned. Slack and Teams have mature enterprise compliance products. Orbit is not a replacement for them in regulated industries.

### Application platform

Discord's Activities, Slack's app marketplace, and Teams' tab/app ecosystem are not planned for Orbit. Orbit does not embed third-party applications. Bots and bridges cover automation use cases; a full application platform is out of scope.

## What This Means in Practice

**Right now (MVP):** Orbit is a good fit for technical communities, open-source projects, gaming groups, and operators who want full control over their infrastructure. It covers text, voice, video, files, presence, and identity well enough for daily use. The missing pieces - editing, reactions, search, push notifications - are real, and communities where those features are table stakes should wait for the fork milestones or evaluate whether the trade-offs are acceptable.

**After the fork:** The Uplink fork closes the most significant functional gaps - editing, reactions, search, push notifications, mobile native apps, and federation. At that point, Orbit is a credible general-purpose alternative for communities that prioritize self-hosting, open protocols, and operator control over convenience features.

**Who Orbit is not for:** Organizations that require enterprise compliance tooling (DLP, eDiscovery, audit exports), teams that depend heavily on embedded third-party applications, and anyone who needs a fully managed hosted service with commercial SLAs. Orbit is infrastructure, not a SaaS product.
