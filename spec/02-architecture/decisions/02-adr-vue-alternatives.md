# ADR: Vue 3 over Alternatives

**Status:** Decided

## Context

The Orbit client needs a frontend framework for both the Tauri desktop shell and the web app / PWA. The framework drives UI rendering, state management, and component architecture across all three runtime contexts: desktop, web, and embedded.

See [Clients](../11-clients.md) for the client architecture.

## Decision

**Vue 3 + Vite + VUI (`@dolanske/vui`).**

## Alternatives Considered

### Leptos (Rust / WASM)

Leptos is a Rust-based reactive framework that compiles to WebAssembly. It eliminates the JavaScript runtime entirely and shares the type system with the Tauri backend. In theory this produces the most memory-efficient frontend possible and maximises code sharing between backend and frontend.

In practice, the team has no existing Leptos experience, VUI does not target WASM, and the Leptos ecosystem is young enough that missing libraries would need to be built from scratch. The estimated ~10% memory premium of Vue over a Leptos/WASM approach is an acceptable trade-off for development velocity.

Leptos remains an active research track. See [Research: Leptos/WASM](../../0R-research/02-leptos-wasm.md).

### Svelte

In a clean-room evaluation, Svelte is a strong contender. It compiles to minimal vanilla JavaScript with no Virtual DOM overhead, produces small bundles, and has no runtime to speak of. For a team with no prior investment in either framework, it would be a reasonable pick.

The practical gap is narrowing. Vue 3.5 delivered a -56% memory reduction in the reactivity system through a major internal refactor, closing much of the runtime overhead difference. Vue's Vapor Mode (currently in development) will bypass the Virtual DOM entirely and compile components to fine-grained imperative DOM operations - similar in spirit to how Svelte works. This will bring Vue's performance and memory profile much closer to Svelte without sacrificing the existing ecosystem, tooling, or the VUI component library.

Given that trajectory, the practical performance delta is already small and shrinking. The team's existing Vue expertise and the shared toolchain outweigh the residual advantage Svelte holds today.

### Quasar

Quasar Framework is a Vue-native meta-framework with first-class build modes for SPA, PWA, Electron, and Tauri - all driven from a single `quasar.config.js`. Its `quasar build -m pwa`, `quasar build -m spa`, and `quasar build -m tauri` commands produce each target from the same source tree without manual Vite configuration.

Quasar was evaluated as an alternative **build system**, not as a framework replacement. The trade-off: Quasar ships its own component library (Quasar Components) which conflicts with VUI's design language, and its opinionated project structure adds a layer of abstraction over raw Vite that complicates custom configuration.

Raw Vite + `vite-plugin-pwa` keeps the toolchain minimal and VUI intact. Quasar may be revisited if monorepo build complexity grows to a point where its orchestration value outweighs the component library conflict.

## Decision Rationale

1. **Team expertise.** The team has extensive experience with Vue 3 + Vite, making it the lowest-friction choice for the desktop frontend.
2. **VUI coverage.** The team already maintains VUI (`@dolanske/vui`), a custom component library built for Vue that covers most of the needed component surface: layout primitives, buttons, inputs, modals, tabs, and more. This accelerates UI development significantly.
3. **Shared toolchain.** Vue + Vite + VUI is used across the team's other projects, meaning shared tooling, shared components, and faster onboarding for new contributors.
4. **Composition API.** Vue 3's Composition API with `<script setup>` provides a productive, ergonomic development pattern that aligns with the team's existing code style.
