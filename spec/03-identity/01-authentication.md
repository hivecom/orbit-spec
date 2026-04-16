# Authentication

Authentication in Orbit is handled at the [Ground Control](../02-components/01-ground-control/01-overview.md) layer - Ergochat's built-in SASL mechanisms are the sole authentication path in the MVP. There are no session tokens, no JWTs issued at login, and no middleware proxies between the Orbit client and the IRC server.

For channel-level permissions that derive from authentication state, see [Permissions](02-permissions.md).

## Registered Users

- Authenticate via SASL (PLAIN or SCRAM-SHA-256) over TLS to Ground Control (Ergochat).
- NickServ (Ergochat's built-in service) handles account registration, password changes, email verification (optional), and nickname enforcement.
- Registered nicknames are reserved. Unregistered users attempting to use a registered nickname are forced to rename.
- Account data is stored in Ergochat's internal database.

For Ergochat SASL configuration detail - including the WebSocket listener, nickname reservation patterns, and connection limits - see [Ground Control](../02-components/01-ground-control/01-overview.md).

## Anonymous Web Widget Users

The Orbit web client (whether embedded as a widget or deployed as a full web app) connects directly to Ergochat's WebSocket endpoint - the same path as the desktop client. There is no middleware proxy.

```mermaid
sequenceDiagram
    participant BR as Browser (web client / widget)
    participant GC as Ground Control

    BR->>GC: Connect WSS
    BR->>GC: SASL ANONYMOUS (auto guest-* nick)
    GC-->>BR: IRC session established (direct)
```

- Guest users connect via SASL ANONYMOUS. Ergochat assigns a `guest-*` nickname automatically.
- No account creation, no backend, no JWT, no session tokens required.
- Guest nicknames are prefixed with `guest-` and cannot be registered via NickServ.
- Any IRC client - including third-party web UIs - can connect the same way. This is intentional: Orbit does not gatekeep access to a standard IRC server.
