# Research: Leptos / WASM Frontend

**Status:** R&D (evaluate after MVP)

This track evaluates rewriting the Orbit desktop frontend in [Leptos](https://leptos.dev/) (Rust compiled to WebAssembly) as an alternative to the current Vue 3 + Vite stack. For the decision that led to Vue being chosen as the MVP frontend, see [ADR: Vue Alternatives](../06-decisions/02-adr-vue-alternatives.md). For the current frontend architecture, see [Desktop Client](../04-clients/01-desktop.md).

## Problem

The MVP desktop client uses Vue 3 + Vite compiled to JavaScript, rendered in the Tauri WebView. While this is efficient and productive, it still involves JS execution overhead and requires serialization across the Tauri IPC bridge for every backend call. Complex data structures (channel trees, user presence maps, message histories) pay a serialization tax on every update.

## Proposal

Evaluate rewriting the desktop frontend in [Leptos](https://leptos.dev/) (Rust -> WASM). Potential benefits:

- **Shared Rust types** between the Tauri backend and frontend - no serialization overhead for complex structures crossing the boundary
- **Lower memory footprint** from fine-grained reactive signals and no JS runtime overhead
- **Single-language codebase** - reduces context-switching cost for contributors and enables shared utility code

## Risks

- The Leptos ecosystem is immature. Component libraries are sparse, devtools are limited, and documentation has gaps. Building production UI will require writing a lot of primitives from scratch.
- WASM debugging is harder than JS debugging. Tooling has improved - Chrome DevTools now supports WASM/DWARF source-level debugging, and `console_error_panic_hook` helps surface Rust panics - but the ecosystem remains less mature than the JS debugging toolchain. Source maps can be less reliable, and stack traces are harder to read.
- Developer onboarding cost is high. Rust + Leptos is a niche skill set. This narrows the contributor pool considerably.
- The memory improvement may be marginal in practice. Vue 3 + Vite is also lightweight. A 5–10% memory reduction would not justify a rewrite.
- The web widget must remain JS-based regardless. WASM bundle sizes are too large for an embeddable widget. This means maintaining two frontend codebases - the opposite of simplification.

## Evaluation Criteria

Build a representative subset of the Orbit UI in Leptos:

- Channel list sidebar with collapsible categories
- Chat message view with scrollback loading
- Voice channel panel with participant list and controls

Benchmark against the Vue implementation:

- Memory usage (target: >30% reduction to justify the effort)
- Startup time (cold and warm)
- IPC overhead for frequent updates (typing indicators, presence changes)

Evaluate developer experience qualitatively: how long does a typical UI change take? How debuggable are issues? Only proceed if the improvement is substantial AND the DX is acceptable for a small team.

## Dependencies

MVP client (see [Desktop Client](../04-clients/01-desktop.md)) must be feature-complete to serve as a fair comparison target.
