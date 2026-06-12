# Monorepo Structure & Build Pipeline

The frontend, Tauri shell, and shared modules live in a single monorepo. This is the structural guarantee that a change to core logic - IRC parsing, Satellite session handling, a VUI component - propagates correctly to all build targets and runtime contexts without manual synchronisation.

There are two build outputs: the Tauri binary (desktop) and the web app / PWA (web). There are three runtime contexts: desktop, web, and widget. All three are served from the same `packages/core` source with a different platform adapter from `packages/platform`.

Cross-references:
- [04-application-seams.md](04-application-seams.md) - How the packages and entrypoints wire together at runtime
- [../04-clients/01-desktop.md](../04-clients/01-desktop.md) - Tauri desktop target
- [../04-clients/02-web-app.md](../04-clients/02-web-app.md) - Web / PWA target
- [../04-clients/03-widget.md](../04-clients/03-widget.md) - Widget mode

## Recommended Directory Structure

```
orbit
  apps/
    desktop/          # Tauri project; src-tauri/ lives here
    web/              # Vite entrypoint for the web app / PWA build
    mobile/           # Tauri v2 mobile entrypoint (NEXT-track; reuses the desktop adapter)
  packages/
    core/             # Shared Vue components, Pinia stores, composables, IRC logic + the Platform contract
    platform/         # Capability adapters: web.ts, tauri.ts, tauri-mobile.ts
  package.json        # Workspace root (pnpm workspaces)
  pnpm-workspace.yaml # Workspace + dependency catalog
```

> **Naming note:** The directory tree above is canonical and matches the on-disk scaffold. Earlier drafts proposed a `lib/core` / `lib/platform` layout; that has been dropped in favour of `packages/`, which is the standard pnpm workspace convention and avoids fighting the toolchain. The platform adapters live alongside core as `packages/platform`.

`apps/desktop`, `apps/web`, and `apps/mobile` are thin entrypoints. They import everything from `packages/core` and wire in a concrete platform adapter from `packages/platform`. The actual application logic lives entirely in `packages/core`. This means:

- A bug fix in the IRC message parser in `packages/core` is automatically present in the next desktop build, the next web deploy, and the next widget session - no porting required.
- The platform adapter is the only code that differs between targets, and it is selected at the app boot of each entrypoint.
- `apps/web` is a `vite build` with `vite-plugin-pwa` configured. `apps/desktop` is `tauri build`, which calls `vite build` internally against the same `packages/core` source with a different platform adapter export.

## Toolchain & Build Commands

The workspace uses **pnpm** (with a dependency `catalog:` in `pnpm-workspace.yaml`) and **[vite-plus](https://github.com/vitejs/vite-plus)** (`vp`) as the task runner. There is no npm; commands below assume pnpm + `vp`.

From the workspace root:

```sh
pnpm ready              # gate: vp check && vp run -r test && vp run -r build
pnpm dev                # web dev server (Vite, runs apps/web)
vp run -r build         # build every package and app
vp run -r test          # run every test suite
```

Per target, build/dev are scoped to the workspace package:

```sh
vp run web#dev          # Vite dev server (web development)
vp run web#build        # SPA + PWA (web app + widget mode)
vp run desktop#dev      # Tauri dev mode (desktop development; lands with the Tauri app)
vp run desktop#build    # Tauri binary, all platforms
```

Widget mode requires no separate build command. It is the web app accessed with `?mode=widget`. CI deploys the web build; widget embedding follows automatically.

## CI Pipeline

A single CI run on every merge to `main` should:

1. **Run the shared test suite** against `packages/core` - unit tests for IRC logic, store logic, composables, and platform adapter interface contracts.
2. **Build `apps/web`** and deploy to the static host (Cloudflare Pages or equivalent).
3. **Build `apps/desktop`** for all three platforms (Linux, Windows, macOS) and produce release artifacts.

Because `packages/core` is tested independently of any build target, regressions are caught before either build runs. A failure in `packages/core` tests blocks both the web deploy and the desktop release.
