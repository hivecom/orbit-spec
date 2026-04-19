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

> **Note**: Whether guest users should ever be allowed to speak in voice channels (rather than receive-only) is an unresolved decision. See [Open Questions](../06-decisions/03-open-questions.md#widget-voice-permissions).

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
  src="https://app.hivecom.net/?mode=widget&server=irc.hivecom.net&channel=general&theme[accent]=3b82f6&theme[background]=1e1e2e"
  width="400"
  height="600"
  allow="microphone; camera"
></iframe>
```

**Supported theme parameters:**

| Parameter | VUI token | Default |
|---|---|---|
| `theme[accent]` | `--color-accent` | VUI default |
| `theme[background]` | `--color-background` | VUI default |
| `theme[surface]` | `--color-surface` | VUI default |
| `theme[text]` | `--color-text` | VUI default |
| `theme[radius]` | `--border-radius-base` | VUI default |

Values are interpreted as hex color strings (without the `#`) for color tokens, and as CSS values for non-color tokens. The widget applies these as inline CSS custom property overrides on its root element, scoped to the iframe's document - they have no effect on the host page.

Only the tokens listed above are exposed as theme parameters. Full VUI token override is not supported - embedders who need deeper customization should self-host the web app and modify the VUI theme directly.

## Script-Tag Embed Option (Dropped)

The script-tag embed approach (previously referred to as Option B in earlier drafts) is dropped. An iframe is the correct primitive for embedding a full interactive application - it provides origin isolation, its own JS context, and browser-managed permissions. A script-tag approach would require injecting the app into the host page's DOM and sharing its JS environment, which creates security and compatibility surface that is not warranted.
