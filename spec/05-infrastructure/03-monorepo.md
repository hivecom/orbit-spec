# Monorepo Structure & Build Pipeline

The frontend, Tauri shell, and shared modules live in a single monorepo. This is the structural guarantee that a change to core logic - IRC parsing, Satellite session handling, a VUI component - propagates correctly to all build targets and runtime contexts without manual synchronisation.

There are two build outputs: the Tauri binary (desktop) and the web app / PWA (web). There are three runtime contexts: desktop, web, and widget. All three are served from the same `packages/core` source with a different platform adapter from `packages/platform`.

Cross-references:
- [04-platform.md](04-platform.md) - How the packages and entrypoints wire together at runtime
- [../04-clients/01-desktop.md](../04-clients/01-desktop.md) - Tauri desktop target
- [../04-clients/02-web-app.md](../04-clients/02-web-app.md) - Web / PWA target
- [../04-clients/03-widget.md](../04-clients/03-widget.md) - Widget mode

## Directory Structure

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

`apps/desktop`, `apps/web`, and `apps/mobile` are thin entrypoints. They import everything from the workspace packages defined in the `/packages` directory, which can import code the same way we'd do with npm packages.

If an app (or another package) wants to consume exports from a specific package, it simply needs to reference it inits own `package.json`.

```json
{
  "dependencies": {
    "platform": "workspace:*",
  }
}
```

Only exports defined within a package's `index.ts` will be available.

```ts
import { createWebPlatform } from "platform"
```



## Naming conventions

Directories and files should be named after the code they contain and export.

Each package should contain a single root-level index.ts file that serves as the package's public export. No other index.ts files should exist within the package.

Example structure

```
core
  src/
    components
    style
    lib/
      platform/
        types.ts
        composables.ts
        constants.ts
      environment.ts
```

1. If an internal module is complex and requires multiple files, those files should be named according to their purpose. Do not introduce an `index.ts` file within the library directory to re-export them.
2. If an internal module has a singular purpose, it may be implemented as a single file.
3. Re-exports should only occur from the package's root-level index.ts

When adding a new app or package, declare all workspace packages that it depends on in its dependencies (the CI will ask you). Only packages listed as dependencies should be imported.

```ts
// In this example, app/website should declare both platform and core as dependencies in package.json
import { usePlatform } from "platform"
import { createOrbitApp } from "core"
```

## Toolchain & Build Commands

The workspace uses **pnpm** (with a dependency `catalog:` in `pnpm-workspace.yaml`) and **[vite-plus](https://github.com/vitejs/vite-plus)** (`vp`) as the task runner. There is no npm; commands below assume pnpm + `vp`.

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

Widget mode requires no separate build command. It is the web app accessed with `?mode=widget`. CI deploys the web build; widget embedding follows automatically.

## CI Pipeline

A single CI run on every merge to `main` should:

1. **Run the shared test suite** against `packages/core` - unit tests for IRC logic, store logic, composables, and platform adapter interface contracts.
2. **Build `apps/web`** and deploy to the static host (Cloudflare Pages or equivalent).
3. **Build `apps/desktop`** for all three platforms (Linux, Windows, macOS) and produce release artifacts.

Because `packages/core` is tested independently of any build target, regressions are caught before either build runs. A failure in `packages/core` tests blocks both the web deploy and the desktop release.
