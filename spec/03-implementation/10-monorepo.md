# Monorepo Structure & Build Pipeline

The frontend, the Tauri shell, and the wasm core live in a single monorepo. This is the structural guarantee that a change to shared logic (IRC parsing in the core, a store in the app package, a VUI component) propagates correctly to all build targets and runtime contexts without manual synchronisation.

There are two build outputs: the Tauri binary (desktop) and the web app / PWA (web). There are three runtime contexts: desktop, web, and embedded. All three run the same `packages/app` frontend on the same wasm core, differing only in the platform adapter injected from `packages/platform`.

Cross-references:
- [06-platform.md](06-platform.md) - How the packages and entrypoints wire together at runtime
- [../02-architecture/11-clients.md](../02-architecture/11-clients.md) - The client family these targets serve

## Directory Structure

```
orbit
  apps/
    desktop/          # Tauri project; src-tauri/ lives here
    web/              # Vite entrypoint for the web app / PWA build
    mobile/           # Tauri v2 mobile entrypoint (reuses the desktop adapter)
  packages/
    app/              # The Vue application: components, stores, composables, routing
    core/             # Rust workspace compiled to WebAssembly: protocol and state logic
    platform/         # The Platform contract and capability adapters: web.ts, desktop.ts
  package.json        # Workspace root (pnpm workspaces)
  pnpm-workspace.yaml # Workspace + dependency catalog
```

`apps/desktop`, `apps/web`, and `apps/mobile` are thin entrypoints. They import everything from the workspace packages, which consume each other the same way we'd consume npm packages.

## The Core is Rust

`packages/core` is a Cargo workspace, not a TypeScript package. `core-shared` holds the protocol and state logic; `core-wasm` wraps it with wasm-bindgen and generates TypeScript types via tsify, so the TS side consumes typed bindings like any other workspace dependency:

```ts
import type { IrcServer, ServerList } from "core-wasm"
```

The wasm artifact is built from the Rust source; nothing hand-written crosses the boundary, and CI builds it from source rather than consuming a committed artifact. Building it requires the Rust toolchain. The TS packages build with pnpm alone.

The Vue side lives in `packages/app`: the `OrbitApp` component tree, Pinia stores, composables, and routing. Stores wrap the core's controllers and hand state to the UI; components never parse protocol data themselves.

If an app (or another package) wants to consume exports from a specific package, it references it in its own `package.json`:

```json
{
  "dependencies": {
    "platform": "workspace:*",
  }
}
```

Only exports defined within a TS package's `index.ts` are available.

```ts
import { createWebPlatform } from "platform"
```

## Naming conventions

These conventions apply to the TypeScript packages (`app`, `platform`); the core follows Cargo conventions.

Directories and files should be named after the code they contain and export.

Each package should contain a single root-level index.ts file that serves as the package's public export. No other index.ts files should exist within the package.

Example structure

```
app
  src/
    components/
    stores/
    lib/
      windows.ts
      logger.ts
```

1. If an internal module is complex and requires multiple files, those files should be named according to their purpose. Do not introduce an `index.ts` file within the library directory to re-export them.
2. If an internal module has a singular purpose, it may be implemented as a single file.
3. Re-exports should only occur from the package's root-level index.ts

When adding a new app or package, declare all workspace packages that it depends on in its dependencies (the CI will ask you). Only packages listed as dependencies should be imported.

```ts
// apps/web declares both app and platform in its package.json
import { createOrbitApp } from "app"
import { createWebPlatform } from "platform"
```

## Toolchain & Build Commands

The workspace uses **pnpm** (with a dependency `catalog:` in `pnpm-workspace.yaml`) and **[vite-plus](https://github.com/vitejs/vite-plus)** (`vp`) as the task runner. There is no npm; commands below assume pnpm + `vp`. Building `core-wasm` additionally needs the Rust toolchain.

From the workspace root:

```sh
pnpm ready              # gate: vp check && vp run -r test && vp run -r build
pnpm dev                # web dev server (Vite, runs apps/web)
vp run -r build         # build every package and app
vp run -r test          # run every test suite
vp create               # create a new app/package
```

Per target, build/dev are scoped to the workspace package:

```sh
vp run web#dev          # Vite dev server (web development)
vp run web#build        # SPA + PWA (web app + widget mode)
vp run desktop#dev      # Tauri dev mode (desktop development; lands with the Tauri app)
vp run desktop#build    # Tauri binary, all platforms
```

Embedded mode requires no separate build command. It is the web app accessed with `?mode=widget`. CI deploys the web build; embedding follows automatically.

## CI Pipeline

A single CI run on every merge to `main` should:

1. **Build `core-wasm` from source** with the Rust toolchain installed in CI; committed wasm artifacts are never trusted.
2. **Run the test suites**: the core's Rust tests, and the TS suites in `packages/app` and `packages/platform` against a mock platform adapter.
3. **Build `apps/web`** and deploy to the static host (Cloudflare Pages or equivalent).
4. **Build `apps/desktop`** for all three platforms (Linux, Windows, macOS) and produce release artifacts.

Because the core and app packages are tested independently of any build target, regressions are caught before either build runs. A failure in steps 1 or 2 blocks both the web deploy and the desktop release.
