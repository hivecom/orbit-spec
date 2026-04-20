# Research: Linux Gaming Overlay - Tier 2 (Vulkan Explicit Layer)

This is Tier 2 of the Linux Gaming Overlay. Tier 1 ([Layer-Shell / X11 Overlay](03-linux-overlay.md)) must ship first - Tier 2 covers the exclusive-fullscreen case that Tier 1 cannot reach. Both tiers expose the same visual feature set (speaker indicator + optional webcam pip); only the rendering path differs.

For the overlay scope definition and the preamble shared by both tiers, see [03-linux-overlay.md](03-linux-overlay.md).

## Problem

Tier 1 overlays fail when a game takes exclusive fullscreen via direct scanout, because the compositor is bypassed entirely. The overlay window simply isn't visible - the game owns the display output directly. This is common with competitive games and VR applications.

## Proposal

Write a custom Vulkan implicit layer (a shared object / `.so` file) in Rust that:

1. **Registers** via a JSON manifest file placed in the Vulkan loader's layer search path
2. **Intercepts** `vkQueuePresentKHR` - the Vulkan call that submits a frame for display
3. **Composites** the speaker indicator and optional webcam pip onto the game's framebuffer before presentation, using pre-rasterized avatar textures and coloured border quads

This is architecturally similar to how [MangoHud](https://github.com/flightlessmango/MangoHud) works. The rendering scope - a small number of static avatar images, a coloured ring per avatar, and optionally one or two decoded video frames for the webcam pip - is comparable to MangoHud's GPU graph and frame time history in complexity. This is not a full UI toolkit problem.

## Risks

- **Driver compatibility**: Must work across AMD (RADV, AMDVLK), NVIDIA (proprietary driver), and Intel (ANV) Vulkan implementations. Each has quirks in layer interception, memory allocation, and synchronization. This is the primary engineering risk, not the rendering complexity.
- **Translation layer interop**: Many Linux games run through DXVK or VKD3D-Proton. The layer must not interfere with these translation layers. MangoHud navigates this successfully, so it is solved territory - but requires testing.
- **Webcam video texture**: Uploading decoded video frames as a Vulkan texture each frame requires careful synchronization (CPU -> GPU transfer, format negotiation). This is the most novel piece relative to MangoHud's scope, but it is a well-understood GPU programming pattern.
- **Input handling**: The overlay is passive (no interaction), so input passthrough is not a concern. A hotkey to toggle visibility is the only input needed, and that can be handled out-of-band by the Orbit client process via a shared memory flag or signal.
- **Stability**: A crashing Vulkan layer crashes the game. Zero tolerance for bugs. Incremental rollout with opt-in is essential.

## Evaluation Criteria

Build a Vulkan layer prototype that:

1. Successfully intercepts `vkQueuePresentKHR`
2. Composites 3 pre-rasterized avatar images with a coloured border in a corner of the screen
3. Works on AMD (RADV) and NVIDIA (proprietary) drivers without crashes or visual artifacts in at least 10 test games, including 2+ running under Proton/DXVK
4. Adds less than 0.5ms of frame time overhead

If stable, add webcam pip: upload a decoded video frame as a texture and composite it into a second corner. If that works cleanly, Tier 2 is shippable.

## Dependencies

[Tier 1 (Layer-Shell / X11 Overlay)](03-linux-overlay.md) must be shipped first. Tier 2 is not a replacement - it covers the exclusive-fullscreen case that Tier 1 cannot reach. Both tiers ship the same visual feature set; only the rendering path differs.
