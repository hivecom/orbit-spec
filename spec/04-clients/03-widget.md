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

## Script-Tag Embed Option (Dropped)

The script-tag embed approach (previously referred to as Option B in earlier drafts) is dropped. An iframe is the correct primitive for embedding a full interactive application - it provides origin isolation, its own JS context, and browser-managed permissions. A script-tag approach would require injecting the app into the host page's DOM and sharing its JS environment, which creates security and compatibility surface that is not warranted.
