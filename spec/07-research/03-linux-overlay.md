# Research: Linux Gaming Overlay - Tier 1 (Layer-Shell / X11)

This is Tier 1 of the Linux Gaming Overlay track. It covers windowed and borderless-fullscreen games that still go through the compositor. For exclusive-fullscreen support where the compositor is bypassed, see [Vulkan Overlay Layer (Tier 2)](04-vulkan-overlay.md).

## Overlay Scope

The overlay scope is deliberately narrow: a **speaker indicator** (avatar thumbnails with a speaking ring showing who is active in voice) and an optional **webcam pip** (a small floating video tile, similar to Discord's picture-in-picture). No chat window, no notification feed, no interactive UI. This constraint is what makes both tiers tractable and keeps rendering complexity bounded.

## Problem

Gamers need to see who is speaking in a voice channel without alt-tabbing. A speaker indicator - avatar thumbnails with a coloured ring toggling on voice activity - covers this. An optional webcam pip covers the video case. Both are passive, non-interactive displays. Linux display server fragmentation (X11 vs. Wayland, multiple compositors) is the main engineering challenge, not the UI complexity.

## Proposal

For windowed and borderless-fullscreen games (which still go through the compositor):

**X11 approach:** Create a composite overlay window using Xlib. Set `_NET_WM_WINDOW_TYPE_NOTIFICATION` (or `_NET_WM_WINDOW_TYPE_DOCK`) to hint that the window should float above others. Use `XShapeCombineRectangles` to define input passthrough regions so mouse clicks fall through to the game underneath. Render the overlay using a lightweight renderer (e.g., tiny-skia) - a handful of pre-rasterized avatar images and coloured rings is trivial to draw.

**Wayland approach:** Use the `wlr-layer-shell` protocol (supported by wlroots-based compositors: Sway, Hyprland, river, etc.) to request the `Overlay` layer. Disable exclusive zone so the overlay doesn't push application windows. Set keyboard interactivity to `none` so the overlay never steals focus from the game.

## Risks

- **Compositor fragmentation**: `wlr-layer-shell` is not supported by GNOME's Mutter or KDE's KWin (though KWin has been adding support). This limits Wayland coverage to wlroots-based compositors, which skews toward power users.
- **Performance**: Overlay rendering must be extremely lightweight. Even a few milliseconds of additional frame time is noticeable in competitive gaming. The overlay must not cause dropped frames.
- **Maintenance burden**: Supporting both X11 and Wayland codepaths is double the platform code, double the testing, double the bugs.

## Evaluation Criteria

Prototype both overlay approaches showing a static speaker indicator (3 avatar thumbnails, one with an active speaking ring). Test with 5+ popular Linux-native and Proton games in windowed and borderless-fullscreen modes. Measure frame time impact (target: <0.5ms added per frame). Test on at least three compositors (Sway, Hyprland, GNOME if feasible).

> **Tier 2**: For exclusive-fullscreen support where the compositor is bypassed entirely, see [Vulkan Overlay Layer (Tier 2)](04-vulkan-overlay.md).
