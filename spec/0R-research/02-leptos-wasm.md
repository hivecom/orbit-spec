# Research: Leptos / WASM Frontend

This track evaluates rewriting the Orbit desktop frontend in [Leptos](https://leptos.dev/) (Rust compiled to WebAssembly) as an alternative to the current Vue 3 + Vite stack. For the decision that led to Vue, see [ADR: Vue Alternatives](../02-architecture/decisions/02-adr-vue-alternatives.md). For the current frontend architecture, see [Clients](../02-architecture/11-clients.md).

## Problem

The desktop client uses Vue 3 + Vite compiled to JavaScript, rendered in the Tauri WebView. While this is efficient and productive, it still involves JS execution overhead and requires serialization across the Tauri IPC bridge for every backend call. Complex data structures (channel trees, user presence maps, message histories) pay a serialization tax on every update.

## Proposal

Evaluate rewriting the desktop frontend in [Leptos](https://leptos.dev/) (Rust -> WASM). Potential benefits:

- Shared Rust types between the Tauri backend and frontend, so complex structures cross the boundary without serialization
- Lower memory footprint from fine-grained reactive signals and no JS runtime
- A single-language codebase, which cuts context-switching for contributors and allows shared utility code

## Risks

- The Leptos ecosystem is immature. Component libraries are sparse, devtools are limited, and documentation has gaps. Building production UI means writing a lot of primitives from scratch.
- WASM debugging is harder than JS debugging. Tooling has improved (Chrome DevTools supports WASM/DWARF source-level debugging, and `console_error_panic_hook` surfaces Rust panics) but it remains behind the JS toolchain. Source maps can be less reliable and stack traces harder to read.
- Developer onboarding cost is high. Rust + Leptos is a niche skill set, which narrows the contributor pool considerably.
- The memory improvement may be marginal in practice. Vue 3 + Vite is also lightweight, and a 5-10% reduction wouldn't justify a rewrite.
- The embedded web client must stay JS-based regardless; WASM bundles are too large for an embed. That means maintaining two frontend codebases, the opposite of simplification.

## Evaluation Criteria

Build a representative subset of the Orbit UI in Leptos:

- Channel list sidebar with collapsible categories
- Chat message view with scrollback loading
- Voice channel panel with participant list and controls

Benchmark against the Vue implementation:

- Memory usage (target: >30% reduction to justify the effort)
- Startup time (cold and warm)
- IPC overhead for frequent updates (typing indicators, presence changes)

Evaluate developer experience qualitatively: how long does a typical UI change take? How debuggable are issues? Only proceed if the improvement is substantial and the DX is acceptable for a small team.

## Dependencies

The desktop client (see [Clients](../02-architecture/11-clients.md)) must be feature-complete to serve as a fair comparison target.
