# Platform

Platform defines a target-agnostic API interface, which the `core` can consume. Core does not
conditionally implement logic based on the platform - it simply calls the method. If said API is
not available? Then it's a noop. Never perform conditional logic like that inside `core` - though
in some cases it's necessary, like not rendering a part of UI if the platform does not support
it. The adapter concept and the three-context client model are described in
[Clients architecture](../02-architecture/11-clients.md).

Here's an example of the Platform interface (subject to change):

```ts
export interface Platform {
  readonly target: "web" | "desktop" | "mobile"
  readonly notifications: NotificationPort
  readonly tray: TrayPort | null          // null on web/widget
  readonly audioDevices: AudioDevicePort
  readonly deepLinks: DeepLinkPort | null  // null in the browser
  readonly fileTransfer: FileTransferPort
  readonly dns: DnsPort | null             // null in the browser
  readonly historyCache: HistoryCachePort | null // IndexedDB (web) / SQLite (desktop); null when storage is blocked
}
```

The `core` can call the `usePlatform` composable anywhere, which exposes the globally injected
interface based on the active platform. This is what keeps the component tree
environment-agnostic and `core` headlessly testable.

The factory adapter simply nulls out what the current platform cannot do:

```ts
export function createWebPlatform(): Platform {
  return {
    target: "web",
    notifications: createNotificationPort(),
    tray: null,           // no native tray in the browser
    audioDevices: createAudioDevicePort(),
    deepLinks: null,      // no orbit:// handler in the browser
    fileTransfer: createFileTransferPort(),
    dns: null,            // resolver endpoint used instead
    historyCache: createIndexedDbCachePort(), // null if IndexedDB is unavailable
  }
}
```

The `HistoryCachePort` contract and its per-environment adapters are specified in
[Local Cache](07-local-cache.md).

## Port Implementations per Environment

Each capability port maps to a concrete implementation per environment:

| Capability                  | Desktop (tauri.ts)                                    | Web (web.ts)                                                            |
|-----------------------------|-------------------------------------------------------|-------------------------------------------------------------------------|
| System tray + badge         | Native OS tray via Tauri plugin                       | `document.title` badge count; favicon overlay                           |
| OS notifications            | Tauri notification plugin                             | Web Notifications API (with permission prompt)                          |
| Audio device management     | `cubeb` via Rust IPC command                          | MediaDevices API (`navigator.mediaDevices.enumerateDevices()`)          |
| `orbit://` URI handling     | Registered OS scheme; Tauri single-instance focus     | Stubbed - not available in browser                                      |
| DNS SRV resolution          | Rust DNS resolver via IPC                             | Not available; user enters host directly or a server-side resolver endpoint is used |
| File I/O / large IPC        | Tauri custom protocol handler                         | Standard `fetch` + pre-signed S3 URLs                                   |

Everything outside this table - all IRC logic, all Satellite/WebRTC session handling, all VUI
components, all Pinia stores, all message rendering - is shared and runs identically in both
environments. The mobile adapter (`tauri-mobile.ts`) reuses the desktop adapter and overrides
only what differs.

## Targets

The platform adapters allow applications to differ only in their initialization - the `main.ts`
file.

```ts
// apps/web/src/main.ts
import { createOrbitApp } from "core"
import { createWebPlatform } from "platform"
import App from "./App.vue"

// Platform exposes the create web platform factory
const platform = createWebPlatform()

// Orbit core (or ui) exposes an enhanced Vue app creation composable which automatically provides the platform, routing, and globals
const app = createOrbitApp(App, platform)

// This is where the application starts, for safety we define it explicitly
app.mount("#app")

```

The desktop and mobile entrypoints are the same four lines with a different adapter
(`createTauriPlatform()`, `createTauriMobilePlatform()`). The adapter is provided **once, before
mount**, so every component and composable downstream can `usePlatform()` synchronously without
guards.

The app component itself is a pass-through into `core`:

```vue
<!-- apps/web/src/App.vue -->
<script setup lang="ts">
import { OrbitApp } from "core"
import { usePlatform } from "platform"

const platform = usePlatform() 
</script>

<template>
  <OrbitApp />
</template>
```

## Environment

`platform.target` is the **single source of truth** for "which environment am I in". Consequences:

- Components and stores that need to branch on environment read `usePlatform().target`
- The `<div class="ob-root" :style="`--ob-platform: ${platform.target}`">` exposes the target to CSS for environment-specific styling without any JS branching.

## Import Discipline

- **No platform imports inside `core`.** A lint boundary forbidding `@tauri-apps/api`, and discouraging raw `navigator.*`/`window.*` capability access, inside `packages/core` is the cheap mechanical guard. If core needs a capability, add a port to the contract.
- **No deep imports across packages.** Consume `core` and `platform` through their package entrypoints (`packages/*/src/index.ts`)
- **Adapters own all the messy parts.** Permission prompts, `enumerateDevices()`, anchor-click downloads, Tauri IPC - all of it lives in their respective packages

## Testability

Because the only environment dependency is the injected `Platform`, `core` is tested headlessly
against a mock adapter:

```ts
const platform: Platform = {
  target: "web",
  notifications: { requestPermission: async () => true, notify: async () => {} },
  tray: null,
  audioDevices: { enumerate: async () => [], onChange: () => () => {} },
  deepLinks: null,
  fileTransfer: { download: async () => {} },
  dns: null,
  historyCache: null,
}
```

## Adding a New Target

The seam reduces "support a new platform" to a mechanical checklist:

1. Add `packages/platform/src/<target>.ts` exporting a factory that returns a `Platform` with the right `target` and ports (reuse an existing adapter and override only what differs - `tauri-mobile.ts` extends `tauri.ts`).
2. Add `apps/<target>/` with a four-line `main.ts` that injects the new adapter.
3. If the target unlocks a brand-new capability, add a port to the contract in `core` and implement it across the existing adapters (`null` where unavailable).

No change to the component tree, stores, or routing is required.

## Cross-References

- [Clients architecture](../02-architecture/11-clients.md) - the client family and adapter concept
- [Local Cache](07-local-cache.md) - the `HistoryCachePort` contract and per-environment backing stores
- [Monorepo](10-monorepo.md) - where the packages and entrypoints live
