# Orbit Web App & PWA

## Overview

The Orbit frontend is a single Vue + Vite + VUI application that runs in three contexts from one codebase:

1. **Inside Tauri** - the desktop client. The Rust backend provides OS integrations; the Vue frontend is otherwise identical to the web version. See [Desktop Client](01-desktop.md).
2. **As a standalone web app / PWA** - deployed to a static host. Full feature parity with the desktop client except for the small set of capabilities that require the Rust backend.
3. **As an embedded widget** - the same web app, loaded in an iframe and passed a `mode=widget` URL parameter. No separate bundle, no separate codebase. See [Widget Mode](03-widget.md).

This means there are effectively **two builds**: the Tauri binary and the web app. The widget is not a build - it is a runtime mode.

## The Platform Adapter

The Tauri desktop client and the web app share the entire Vue component tree. The only divergence is a thin **platform adapter** - a TypeScript interface the Vue app calls for capabilities that differ between environments:

```
src/platform/index.ts   ← re-exports tauri.ts or web.ts based on VITE_TARGET
src/platform/tauri.ts   ← calls @tauri-apps/api (desktop only)
src/platform/web.ts     ← uses browser APIs or gracefully stubs missing features
```

Vue components never import from `@tauri-apps/api` directly. They call the adapter. This keeps the component tree environment-agnostic and means the web app is a natural output of the same source, not a separate port.

The platform-specific surface is small:

| Capability                  | Desktop (tauri.ts)                                    | Web (web.ts)                                                            |
|-----------------------------|-------------------------------------------------------|-------------------------------------------------------------------------|
| System tray + badge         | Native OS tray via Tauri plugin                       | `document.title` badge count; favicon overlay                           |
| OS notifications            | Tauri notification plugin                             | Web Notifications API (with permission prompt)                          |
| Audio device management     | `cpal` via Rust IPC command                           | MediaDevices API (`navigator.mediaDevices.enumerateDevices()`)          |
| `orbit://` URI handling     | Registered OS scheme; Tauri single-instance focus     | Stubbed - not available in browser                                      |
| DNS SRV resolution          | Rust DNS resolver via IPC                             | Not available; user enters host directly or a server-side resolver endpoint is used |
| File I/O / large IPC        | Tauri custom protocol handler                         | Standard `fetch` + pre-signed S3 URLs                                   |

Everything outside this table - all IRC logic, all Satellite/WebRTC session handling, all VUI components, all Pinia stores, all message rendering - is shared and runs identically in both environments.

## Web App as a PWA

The web app is shipped as a **Progressive Web App** using [`vite-plugin-pwa`](https://vite-pwa-org.netlify.app/), which wraps Workbox and handles service worker generation, manifest injection, and asset precaching automatically.

This gives the web app:

- **Installability** - browsers prompt users to "Add to Home Screen" / "Install App". On desktop, this opens the app in its own window with no browser chrome, making it visually indistinguishable from a native app.
- **Offline shell** - the app shell (HTML, JS, CSS, fonts) is precached by the service worker. On repeat visits, the shell loads instantly from cache even before the network responds.
- **Background sync** - the message outbox (messages composed during a disconnection) can be flushed via the Background Sync API when connectivity is restored, using the same outbox logic as the desktop client.
- **Push notifications (post-MVP)** - the Web Push API allows mention and DM notifications to be delivered even when the tab is not open, bridging the gap with the desktop client's OS notification integration.

PWA configuration in `vite.config.ts`:

```ts
VitePWA({
  registerType: 'autoUpdate',
  manifest: {
    name: 'Orbit',
    short_name: 'Orbit',
    theme_color: '#1a1a2e',
    icons: [ /* ... */ ],
    display: 'standalone',      // no browser chrome when installed
    start_url: '/',
    scope: '/',
  },
  workbox: {
    // Precache the app shell; runtime-cache API responses separately
    globPatterns: ['**/*.{js,css,html,woff2,svg,png}'],
    runtimeCaching: [
      {
        urlPattern: /^wss?:\/\/.*/,   // WebSocket - not cached, always live
        // Note: service workers cannot intercept WebSocket connections.
        // This rule is documentary only and has no runtime effect.
        handler: 'NetworkOnly',
      },
    ],
  },
})
```

The installed PWA and the Tauri desktop client can coexist. Users who install the PWA get a near-native experience; users who install the Tauri binary get the additional platform-native integrations (tray, `orbit://` URI scheme, Rust audio management). Both are valid, supported paths.

## Capability Matrix

The complete capability comparison across all three surfaces:

| Capability                               | Desktop (Tauri)                | Web App / PWA                               | Widget (web app, `mode=widget`)        |
|------------------------------------------|--------------------------------|---------------------------------------------|----------------------------------------|
| Text chat (full)                         | Yes                            | Yes                                         | Yes                                    |
| Message history / scrollback             | Yes                            | Yes                                         | Yes (limited to recent on load)        |
| Message editing and deletion             | Yes                            | Yes                                         | Yes                                    |
| Rich rendering (links, images, Markdown) | Yes                            | Yes                                         | Yes                                    |
| File uploads                             | Yes                            | Yes                                         | No (guests only; configurable)         |
| Voice & video (full)                     | Yes                            | Yes                                         | Yes (full, with browser permission)    |
| Voice & video (guest)                    | N/A                            | Receive-only in MVP ¹                       | Receive-only in MVP ¹                  |
| User list                                | Yes                            | Yes                                         | Yes (compact)                          |
| Unread indicators / mention highlights   | Yes                            | Yes                                         | Yes                                    |
| Offline message outbox                   | Yes                            | Yes (PWA Background Sync)                   | No                                     |
| System tray + badge                      | Yes (native OS)                | Tab title + favicon badge                   | N/A                                    |
| OS / push notifications                  | Yes (native)                   | Web Notifications API; Push API (post-MVP)  | No                                     |
| `orbit://` deep links                    | Yes (OS-registered)            | No                                          | "Open in Orbit" button                 |
| Audio device selection                   | Yes (Rust/cpal, fine control)  | Yes (MediaDevices API, device picker)       | No (uses browser default)              |
| DNS SRV discovery                        | Yes (Rust resolver)            | No (direct host entry or resolver endpoint) | No                                     |
| Installable (no browser chrome)          | Yes (native binary)            | Yes (PWA install prompt)                    | N/A                                    |
| Offline shell load                       | Yes                            | Yes (service worker precache)               | No                                     |
| Settings UI                              | Yes                            | Yes                                         | No (hidden in widget mode)             |
| Server browser                           | Yes                            | Yes                                         | No (hidden in widget mode)             |
| "Open in Orbit" escape hatch             | N/A                            | N/A                                         | Yes                                    |

¹ Whether guest users should ever be allowed to speak in voice channels is an unresolved decision. See [Open Questions](../06-decisions/03-open-questions.md#widget-voice-permissions).

## Rate Limiting

Rate limiting for guest connections is handled by Ergochat's built-in flood protection and per-IP connection limits, configured by the server operator in Ergochat's settings. No separate service or configuration is needed. The same limits apply to all connecting clients - desktop, web app, widget, or third-party IRC clients.

## MVP Priority and Rollout

For the MVP, the **Tauri desktop client is the primary target**. The web app and widget follow naturally from the same codebase with minimal additional work:

1. Implement the platform adapter interface with both `tauri.ts` and `web.ts` implementations.
2. Add `vite-plugin-pwa` with the manifest and Workbox config above.
3. Deploy the Vite build output to a static host (Cloudflare Pages, Netlify, or self-hosted via Caddy).
4. Widget mode is then available immediately at no additional cost - just pass `mode=widget` in the iframe `src`.

The PWA install flow, offline shell, and background sync are handled entirely by `vite-plugin-pwa` and Workbox. There is no custom service worker to maintain.

For the monorepo build structure, workspace layout, and CI pipeline, see [Monorepo & Build Pipeline](../05-infrastructure/03-monorepo.md).
