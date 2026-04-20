# ADR: Tauri v2 vs Electron

**Status:** Decided

## Context

The Orbit desktop client requires a native application shell: a binary that runs on Linux, Windows, and macOS, hosts a web-based UI, and provides a backend capable of handling IRC protocol parsing, file I/O, audio device management, and IPC. The two primary candidates for this role are Tauri v2 and Electron.

See [../04-clients/01-desktop.md](../04-clients/01-desktop.md) for the full desktop client specification.

## Decision

**Tauri v2 with a Rust backend.**

## Rationale

### Comparison Table

| Metric | Tauri v2 | Electron |
|---|---|---|
| Binary size | ~10–15 MB | ~150+ MB |
| Idle RAM | ~30–50 MB | ~200–500 MB |
| Rendering | OS WebView | Bundled Chromium |
| Backend | Rust | Node.js |
| Auto-update | Built-in | Built-in |

### Key Points

**Binary size and idle memory.** Tauri uses the native WebView of the operating system - WebKitGTK on Linux, WebView2 on Windows, WKWebView on macOS - rather than shipping an entire Chromium browser. This produces a binary an order of magnitude smaller than Electron and dramatically lower idle memory consumption.

**Rust backend for performance-critical work.** The Rust backend handles IRC protocol parsing, file I/O, audio device management, and IPC. Rust provides memory safety without a garbage collector, predictable performance, and direct access to system APIs. These characteristics are well-suited to the always-on, low-overhead nature of an IRC client.

**Shared frontend toolchain.** Tauri renders a standard web view, so the Vue 3 + Vite + VUI frontend runs identically in the desktop shell and in the web browser. The same codebase serves both targets with only the platform adapter differing.

## Trade-offs

- The Rust backend requires contributors familiar with Rust, which has a steeper learning curve than Node.js.
- Platform-specific WebView rendering means minor visual inconsistencies are possible across operating systems (font rendering, scrollbar appearance, focus rings).
- Testing the UI layer requires native WebView environments; headless Chromium-based tests used for Electron are not directly applicable.

## Caveats

**Windows - WebView2:** WebView2 is a separate runtime (~150 MB) that must be installed on the user machine. On Windows 11 and recent Windows 10 builds it is pre-installed. Older systems may need an installer step. The Tauri installer can bundle a WebView2 bootstrapper to handle this automatically, but it adds complexity to the Windows distribution.

**Linux - WebKitGTK:** WebKitGTK memory usage varies by distribution and GTK version. The 30–50 MB idle estimate applies to typical builds; some configurations may exceed this. The debounced resize strategy described in the memory discipline section is specifically motivated by a known WebKitGTK memory leak on rapid resize cycles.

**macOS - WKWebView:** WKWebView is well-behaved. No known platform-specific caveats for the MVP.
