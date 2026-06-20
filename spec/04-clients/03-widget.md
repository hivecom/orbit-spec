# Widget Mode

## Overview

The widget is the Orbit web app running with `?mode=widget` (or `?mode=widget&server=...&channel=...`) passed as URL parameters. The app reads these on startup and activates a constrained presentation layer. There is no separate widget bundle - the widget iframe loads the same deployed web app with different URL parameters. This means widget users receive the same bug fixes, the same performance improvements, and the same WebRTC stack as full web app users automatically.

See [Web App & PWA](02-web-app.md) for the full application this builds on.

## Constrained Presentation Layer

When `mode=widget` is active:

- The full sidebar, server browser, and settings are hidden.
- A compact single-channel view is rendered.
- A **"Open in Orbit"** button is shown. This button tries `orbit://` first (opens the desktop client if installed - see [Desktop Client](01-desktop.md) for the `orbit://` scheme) and falls back to the full web app URL.

## Voice and Video

Voice and video work normally in the widget. The iframe just needs `allow="microphone; camera"` from the embedder. No special configuration is required beyond the standard iframe permission attributes.

> **Resolved**: Guests can speak in voice by default. This is configurable per Satellite instance by the operator - operators can restrict guests to receive-only if needed. See [Open Questions](../0A-decisions/03-open-questions.md#widget-voice-permissions).

## Embedding

```html
<!-- Recommended: iframe -->
<iframe
  src="https://app.hivecom.net/?mode=widget&server=irc.hivecom.net&channel=general"
  width="400"
  height="600"
  allow="microphone; camera"
></iframe>
```

## No Separate Bundle

Widget mode requires no separate build command. It is the web app accessed with `?mode=widget`. CI deploys the web build; widget embedding follows automatically. There is no separate codebase, no separate deployment, and no version drift between the widget and the full web app.

## Code Splitting

The widget loads the same web app bundle as the full client, but only renders a single-channel view. To avoid loading code for features the widget never uses (server browser, settings, full voice UI, etc.), the app uses Vite's route-based code splitting.

When `mode=widget` is active on startup, a lazy boundary is established at the application shell level. Features outside the widget surface - the server browser, settings panels, the full Satellite session management UI - are placed behind dynamic `import()` calls and are never fetched unless the user navigates to them. In widget mode, they are never navigated to, so they are never downloaded.

This means the widget's initial bundle is significantly smaller than the full web app load. The trade-off is a slightly more complex chunk graph in the Vite build, but `vite-plugin-pwa`'s Workbox precaching handles this automatically - only the chunks that widget mode actually needs are precached for the widget's initial load path.

## Theming

Widget embedders can customize the widget's appearance to match their site by passing VUI CSS custom property overrides as `data-` attributes on the iframe, or as URL query parameters. The widget reads these on startup and applies them over VUI's default token values before rendering.

**URL query parameters (recommended):**

```html
<iframe
  src="https://app.hivecom.net/?mode=widget&server=irc.hivecom.net&channel=general&theme[accent]=3b82f6&theme[light-mode]=dark"
  width="400"
  height="600"
  allow="microphone; camera"
></iframe>
```

**Supported theme parameters:**

| Parameter | Default | Type |
|---|---|--|
| `theme[color-mode]` | `system` | `light`, `dark`, `system` |
| `theme[light-accent]` | `#93EFD8` | `CSSHexColor` |
| `theme[dark-accent]` | `#93EFD8`| `CSSHexColor` |
| `theme[radius]` | `8px`| `CSSValue` | 

Values are interpreted as hex color strings (without the `#`) for color tokens, and as CSS values for non-color tokens.

Only the tokens listed above are exposed as theme parameters. Full VUI token override is not supported - embedders who need deeper customization should self-host the web app and modify the VUI theme directly.
