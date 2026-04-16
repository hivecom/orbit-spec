# Research: Mobile Clients

**Status:** Post-MVP (Q2)

Cross-references:
- [Web App & PWA](../04-clients/02-web-app.md) - the PWA baseline that already covers mobile
- [Desktop Client](../04-clients/01-desktop.md) - shared monorepo structure and Tauri codebase
- [Beacon](08-beacon.md) - push notification relay required for full notification support
- [ADR: Vue Alternatives](../06-decisions/02-adr-vue-alternatives.md) - explains why Capacitor was not chosen

## Problem

A communication platform without mobile clients is not viable beyond early adopters. Users expect to receive messages and join voice on their phone. This is non-negotiable for long-term adoption.

## Baseline: PWA Already Covers Mobile

The web app is already a PWA and already works on mobile without any additional development:

- **Android**: Chrome and Chromium-based browsers support the full PWA install flow. The app installs to the home screen, runs in standalone mode, and WebRTC voice/video works.
- **iOS**: Safari supports PWA installation and push notifications since iOS 16.4. Core functionality - text chat, voice, video - works. Background Sync and some service worker behaviours are more restricted than Android.

The PWA is not a stopgap. It is a legitimate, supported mobile path that ships for free as part of the [web app](../04-clients/02-web-app.md). Users who don't want to install a native app are fully served by it.

## Chosen Approach: Tauri Mobile

For a native app store presence and deeper OS integration, **Tauri v2 mobile** is the chosen path. Tauri v2 added iOS and Android build targets, meaning the same Rust backend and the same `packages/core` Vue frontend used by the [desktop client](../04-clients/01-desktop.md) can be compiled to a native mobile app. In the monorepo this would be a new `apps/mobile` entrypoint - thin, like `apps/desktop` - importing from the same shared packages.

This is the right call for two reasons:

1. **By the time Orbit needs a native mobile app, Tauri mobile will be mature.** Mobile is explicitly post-MVP. Tauri mobile is improving rapidly, and the gap between its current rough edges and production-readiness is expected to close well within the timeframe before mobile becomes a priority.
2. **It preserves the single-codebase guarantee.** A bug fix or feature in `lib/core` propagates to desktop, web, and mobile simultaneously. Capacitor or native Swift/Kotlin would break this - Capacitor introduces its own plugin abstraction layer, and native builds require entirely separate frontend codebases.

**Why not Capacitor**: Capacitor wraps a web app in a native WebView and exposes native APIs via plugins - similar in spirit to Tauri. It is more mature on mobile today. However, it would introduce a second native abstraction layer alongside Tauri (one for desktop, one for mobile), splitting the platform adapter and the plugin ecosystem. Tauri's mobile targets are the cleaner long-term answer since they unify the Rust backend and the build toolchain across all native targets. See also [ADR: Vue Alternatives](../06-decisions/02-adr-vue-alternatives.md) for the broader decision context around the frontend framework and cross-platform approach.

**Why not native Swift/Kotlin**: Maximises platform integration but requires separate frontend codebases and platform-specific expertise. The development cost is not justified when Tauri mobile covers the same ground with shared code.

## Risks

- Tauri mobile is still maturing. The iOS and Android WebView abstractions have different capabilities and known rough edges compared to the desktop targets. This is acceptable risk given the post-MVP timeline.
- Mobile UX patterns (swipe gestures, bottom navigation, pull-to-refresh, haptic feedback) require significant UI work beyond porting the desktop layout. The mobile entrypoint will need its own layout shell and navigation model.
- Background voice connectivity is hard on mobile. Both iOS and Android aggressively kill background processes. Maintaining a voice connection while the app is backgrounded requires platform-specific handling - foreground services on Android, VOIP entitlements on iOS. Tauri mobile will need to expose hooks for this; if it cannot, a thin native wrapper may be needed for audio-only.
- Push notifications require a separate relay component. See [Beacon](08-beacon.md) for the proposed self-hostable push notification relay. Without Beacon, the mobile app has no push notifications; everything else works.

## Evaluation Criteria

When Tauri mobile is being evaluated for the mobile track, build a prototype that:

- Connects to Ground Control and authenticates via SASL
- Displays a channel list and chat messages with scrollback
- Joins a Satellite voice session with working audio
- Receives a push notification when mentioned (requires [Beacon](08-beacon.md) + FCM/APNs integration)
- Handles backgrounding gracefully (voice session survives switching apps)

Evaluate battery usage during a 1-hour voice session (target: not significantly worse than Discord mobile). Test background behaviour on both iOS and Android. If Tauri mobile cannot satisfy the background audio requirement, assess whether a thin native audio wrapper is viable before reconsidering Capacitor.

## Dependencies

- MVP desktop client and web app must be stable and the monorepo structure established before mobile work begins.
- Tauri mobile targets should be at a stable release (not alpha/beta) before committing to them for production.
- [Beacon](08-beacon.md) (push relay) is a dependency for full notification support; it can be scoped and built independently.

## Long-Term Consideration: Pure Native Applications

A fully native client - Swift/SwiftUI on iOS, Kotlin/Jetpack Compose on Android, GTK or Qt on Linux/Windows/macOS - represents the ceiling of platform integration quality. On mobile, native apps can use CallKit for system-level call UI, background audio entitlements without WebView constraints, and platform-native push handling. On desktop, a native client sidesteps the WebView layer entirely, giving direct access to the full OS graphics stack, tighter system tray integration, and lower baseline memory than any WebView-based approach.

The reason it is not the chosen path for the core Orbit project is scope, not capability. A native client per platform is effectively a separate frontend codebase for each target. None of them share the Vue component tree, the Pinia stores, or the VUI design language. Every feature built in `lib/core` would need to be re-implemented independently on each platform. Bug fixes, UI changes, and protocol updates would need to land in multiple places simultaneously. For a small team maintaining an open-source project, this maintenance burden is prohibitive - it would fragment effort away from the protocol and server infrastructure that all clients depend on.

The correct framing is: **native clients are something the community could build on top of Orbit's open infrastructure, not something the core Orbit project maintains.** Because Ground Control is standard IRCv3 and Satellite is LiveKit-compatible, a third-party developer could build a fully native client on any platform against the same backend with no changes to the server stack. This is the same relationship that WeeChat and irssi have to Orbit's Ground Control today - they are valid clients that the protocol supports, not clients the Orbit project ships.

If native community clients emerge - on any platform - the Orbit project should:

- Document the IRCv3 extensions and `+orbit/*` tag namespace thoroughly enough for third-party client authors to implement full compatibility.
- Define an **Orbit Client Compatibility Profile** - a checklist of capabilities (tag handling, Satellite signaling, identity verification) that a client must implement to be considered Orbit-compatible, regardless of the technology or platform it is built on.
- Avoid making protocol decisions that privilege the reference client over third-party implementations.

This is a long-term consideration and requires no immediate action. It is captured here to record that pure native clients are a deliberate out-of-scope decision for the core project, not an oversight - and that the open protocol is explicitly designed to make them possible.
