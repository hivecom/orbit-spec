# Monorepo Structure & Build Pipeline

The frontend, Tauri shell, and shared modules live in a single monorepo. This is the structural guarantee that a change to core logic - IRC parsing, Satellite session handling, a VUI component - propagates correctly to all build targets and runtime contexts without manual synchronisation.

There are two build outputs: the Tauri binary (desktop) and the web app / PWA (web). There are three runtime contexts: desktop, web, and widget. All three are served from the same `lib/core` source with a different platform adapter.

Cross-references:
- [../04-clients/01-desktop.md](../04-clients/01-desktop.md) - Tauri desktop target
- [../04-clients/02-web-app.md](../04-clients/02-web-app.md) - Web / PWA target
- [../04-clients/03-widget.md](../04-clients/03-widget.md) - Widget mode

## Recommended Directory Structure

```
orbit-app
  apps/
    desktop/          # Tauri project; src-tauri/ lives here
    web/              # Vite entrypoint for the web app / PWA build
  lib/
    core/             # Shared Vue components, Pinia stores, composables, IRC logic
    platform/         # Platform adapter (tauri.ts + web.ts)
  package.json        # Workspace root (npm workspaces)
```

> **Naming note:** The directory tree above is canonical. The original spec prose referenced `packages/core` and `packages/platform`; those references are incorrect. The correct paths are `lib/core` and `lib/platform`.

`apps/desktop` and `apps/web` are thin entrypoints. They import everything from `lib/core` and `lib/platform`. The actual application logic lives entirely in `lib/core`. This means:

- A bug fix in the IRC message parser in `lib/core` is automatically present in the next desktop build, the next web deploy, and the next widget session - no porting required.
- The platform adapter in `lib/platform` is the only file that differs between targets.
- `apps/web` is a `vite build` with `vite-plugin-pwa` configured. `apps/desktop` is `tauri build`, which calls `vite build` internally against the same `lib/core` source with a different platform adapter export.

## Build Commands

From the workspace root:

```sh
npm run build:web       # SPA + PWA (web app + widget mode)
npm run build:desktop   # Tauri binary (all platforms)
npm run dev:web         # Vite dev server (web development)
npm run dev:desktop     # Tauri dev mode (desktop development)
```

Widget mode requires no separate build command. It is the web app accessed with `?mode=widget`. CI deploys the web build; widget embedding follows automatically.

## CI Pipeline

A single CI run on every merge to `main` should:

1. **Run the shared test suite** against `lib/core` - unit tests for IRC logic, store logic, composables, and platform adapter interface contracts.
2. **Build `apps/web`** and deploy to the static host (Cloudflare Pages or equivalent).
3. **Build `apps/desktop`** for all three platforms (Linux, Windows, macOS) and produce release artifacts.

Because `lib/core` is tested independently of any build target, regressions are caught before either build runs. A failure in `lib/core` tests blocks both the web deploy and the desktop release.