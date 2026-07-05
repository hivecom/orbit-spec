# Deployment

Operator-facing deployment reference: DNS records, the well-known services file, the client
resolution algorithms, the reference Docker Compose, TLS specifics, CORS, monitoring, and
backups. The topology, resource shape, TLS policy, and backup responsibility model live in
[Infrastructure architecture](../02-architecture/12-infrastructure.md).

## DNS Records

DNS records are advertisement pointers published under the community domain - they tell Orbit
clients which services are associated with this community. Each record resolves to the actual
host and port of the service, which may be on entirely separate infrastructure. A missing record
means the client will not attempt to discover that service; it degrades gracefully with no error
shown to the user.

### SRV Records

| Record | Type | Purpose | Add when |
|---|---|---|---|
| `_satellite._tcp.example.com` | SRV | Satellite discovery | If running Satellite |
| `_depot._tcp.example.com` | SRV | Depot (file storage) discovery | If running Depot |
| `_transponder._tcp.example.com` | SRV | Identity provider (OIDC) discovery | If running Transponder |

> **Uplink (IRC)** does not use an SRV record. It is reached via a conventional
> `irc.example.com` A/AAAA record on port 6697. See [Uplink Resolution](#uplink-resolution)
> below.

Multiple `_satellite._tcp` SRV records can be configured to advertise multiple Satellites. SRV
priority and weight control load distribution and failover.

### A / AAAA Records

| Record | Purpose | Add when |
|---|---|---|
| `irc.example.com` | Uplink (IRC + WebSocket) endpoint | If running IRC |
| `sat.example.com` | Satellite endpoint | If running Satellite |
| `depot.example.com` | Depot (storage gateway) endpoint | If running Depot |
| `turn.example.com` | TURN server | If running TURN |

A minimal text-only deployment needs only the `irc.example.com` A/AAAA record; the Orbit client
connects to port 6697 (IRC over TLS) by default. A Satellite-only deployment (no IRC) needs only
`_satellite._tcp` and the A/AAAA record for the Satellite. The client adapts based on which
records are present.

DNS SRV lookup is a desktop-client capability; browsers cannot do it. Web clients rely on the
well-known file below. The capability matrix and operator guidance are in
[Infrastructure architecture](../02-architecture/12-infrastructure.md).

## Well-known Services File

Operators MAY publish a static JSON file at:

```
https://example.com/.well-known/orbit/services.json
```

Web clients (and desktop clients as a convenience shortcut, bypassing DNS) fetch this file via
plain HTTPS to discover available services. The file format is:

```json
{
  "irc": { "host": "irc.example.com", "port": 6697, "ws_port": 6698 },
  "satellite": [
    { "host": "sat.example.com", "port": 7880, "name": "US East" }
  ],
  "depot": { "host": "depot.example.com", "port": 3000 }
}
```

Missing keys mean that service is not available - the client treats absent keys identically to
absent DNS records. This mechanism is **optional but RECOMMENDED** for any community that wants
web client users to benefit from automatic discovery.

**Field reference:**

| Key | Type | Description |
|---|---|---|
| `irc.host` | string | Uplink hostname |
| `irc.port` | number | IRC over TLS port (typically 6697) |
| `irc.ws_port` | number | WebSocket port for web clients (typically 6698) |
| `satellite` | array | One object per Satellite, in priority order |
| `satellite[].host` | string | Satellite hostname |
| `satellite[].port` | number | Satellite port |
| `satellite[].name` | string | Human-readable label (e.g., region name) |
| `depot.host` | string | Depot hostname |
| `depot.port` | number | Depot port |

**Serving requirements:** the file MUST be served over HTTPS with an appropriate
`Content-Type: application/json` header. It MAY be served from a CDN, static file host, or
reverse proxy - no dynamic backend is required. In the reference deployment, Caddy is already
present to terminate TLS for the Ergochat WebSocket listener and route HTTPS traffic to Satellite
and Depot, so serving this file is a two-line static route addition to the Caddy config - no
extra service, port, or process. If the community domain does not serve HTTP(S) itself, the
operator SHOULD configure a redirect or proxy at that domain pointing to the file, or rely on DNS
records for desktop discovery and manual entry for web client fallback.

## Per-Service Resolution

Resolution behaviour differs by client type. Clients MUST degrade gracefully when a discovery
method is unavailable: missing services are silently absent from the UI, and no error is shown
unless all applicable resolution paths have been exhausted.

### Uplink Resolution

**Desktop clients** resolve Uplink in this order:

1. **Well-known URL** - fetch `/.well-known/orbit/services.json` and use the `irc.host` and `irc.port` values if present.
2. **Conventional subdomain** - attempt `irc.example.com` on port `6697` (IRC over TLS). This is the expected path for well-configured deployments.
3. **Bare domain fallback** - attempt `example.com` on port `6697` as a last resort (covers operators who skipped the subdomain entirely).
4. **Manual entry** - if all automatic attempts fail, prompt the user to enter a host:port manually.

**Web clients** resolve Uplink in this order:

1. **Well-known URL** - fetch `/.well-known/orbit/services.json` and use the `irc.host` and `irc.ws_port` values if present. Web clients MUST connect via WebSocket using `ws_port`, not the raw IRC port.
2. **Manual entry** - if the well-known URL is absent or does not contain `irc`, prompt the user to enter a host manually.

Operators SHOULD publish an `irc.example.com` A/AAAA record. The bare domain fallback (desktop
step 3) exists only as a defensive measure and SHOULD NOT be relied upon.

### Satellite Resolution

**Desktop clients:**

1. Resolve `_satellite._tcp.example.com` SRV records.
2. For each discovered record, query the node `GET /info` metadata endpoint to retrieve name, region, capacity, and version (see [Satellite](03-satellite.md#discovery-metadata)).
3. Present discovered nodes as **Server Nodes** with a verified badge. DNS records under the domain are the operator assertion that these nodes are official.
4. SRV priority and weight are respected for load balancing and failover.

**Web clients:**

1. Fetch `/.well-known/orbit/services.json` and read the `satellite` array. Each entry is a node in priority order.
2. For each entry, query `GET /info` to retrieve node metadata and verify availability.
3. If no well-known file is available, prompt the user for manual entry.

**If no Satellites are discovered by any applicable mechanism:** Voice features degrade
gracefully - P2P calls still work (no Satellite is required for P2P), and BYOS Satellites remain
available, but group voice via server-operated Satellites is unavailable. The client hides
Satellite selection UI rather than showing an error.

**Multiple nodes** may be advertised via multiple SRV records (desktop) or multiple entries in
the `satellite` array (all clients). Priority order follows SRV priority/weight for DNS, and
array index for well-known discovery.

### Depot Resolution

**Desktop clients:**

1. Resolve `_depot._tcp.example.com` SRV.
2. Use the returned host and port for all file upload and download requests.

**Web clients:**

1. Fetch `/.well-known/orbit/services.json` and read the `depot` object.
2. If no well-known file is available, file sharing is unavailable until a host is entered manually.

**If no Depot endpoint is discovered by any applicable mechanism:** File sharing is unavailable.
The client hides the upload UI rather than presenting a broken state.

### Identity Provider Resolution

The Orbit client discovers the server's OIDC identity provider (the Transponder role) through the
following mechanisms in priority order:

1. **Well-known URL** - fetch `/.well-known/orbit/oidc` on the server's domain. This returns a JSON document containing the OIDC issuer URL. The client then fetches the standard `/.well-known/openid-configuration` from that issuer to discover all endpoints. (All clients.)
2. **DNS SRV** - resolve `_transponder._tcp.example.com`. Use the returned host and port as the OIDC issuer base URL. (Desktop only.)

**If no identity provider is discovered:** all Satellite participants are treated as unverified.
Voice sessions continue normally; identity verification is absent. See
[Identity](05-identity.md) for the OIDC flows that follow discovery.

## TLS Specifics

TLS is required on every connection - the policy is in
[Infrastructure architecture](../02-architecture/12-infrastructure.md). The mechanics:

- Let's Encrypt for public deployments (automated via certbot or Caddy).
- Self-signed certificates are acceptable for local and development setups only.
- Ergo can terminate TLS for raw IRC connections directly (port 6697); a pure desktop-only, text-only deployment technically does not require a reverse proxy.
- A reverse proxy (Caddy in the reference deployment) terminates TLS for Ergo's WebSocket listener, Satellite token service endpoints, and Depot endpoints. This reverse proxy is a **practical requirement for any public deployment** - even text-only, if web clients are expected to connect. The same Caddy instance also serves `/.well-known/orbit/services.json` as a static file for web client service discovery, at no additional infrastructure cost. The reference Docker Compose always includes Caddy regardless of which other services are enabled.

## Reference Docker Compose

A reference `docker-compose.yml` is provided for self-hosters. It includes:

- **Ergo** - the adopted IRC server filling the Uplink role - with WebSocket enabled, SASL configured, and chat history enabled. Set `history.retention.allow-individual-delete: true` to enable message retractions; Ergo only advertises `draft/message-redaction` (and only accepts `REDACT`) when this is on. See [Uplink - Message Retractions](01-uplink.md#message-retractions-redact). Operators with data-protection obligations (GDPR and similar) should also review `history.retention.enable-account-indexing` for account-wide erasure - see [Uplink - Data Protection Levers](01-uplink.md#data-protection-levers).
- **LiveKit + token service** (Satellite) - **optional**; can be removed for text-only deployments.
- **Depot** (storage gateway) - **optional**; only required if file uploads are needed. The thin gateway runs against either an S3-compatible backend (a bundled MinIO container, or external AWS S3) or a local filesystem driver (a mounted volume). Single-box deployments can use the local filesystem driver and skip MinIO entirely; deployments that need scale or off-host storage point Depot at S3/MinIO.
- **coturn** (STUN/TURN) - for NAT traversal.
- **Auth-script bridge** - required when the OIDC provider signs with ES256 (or any algorithm not supported by Ergo's native `jwt-auth`); optional when using native `jwt-auth` with an RS256/EdDSA/HMAC provider. See [Identity](05-identity.md#uplink-verification-paths).
- **Caddy** (reverse proxy) - terminates TLS via Let's Encrypt, routes WebSocket and API requests to Ergo and Satellite, and serves `/.well-known/orbit/services.json` as a static file for web client service discovery.

One `docker compose up` produces a fully functional Orbit instance. Configuration is done via a
single `.env` file and an `orbit.toml` for server-specific settings (domain, channel list).
Satellite discovery is handled via DNS SRV records configured at the domain level - no IRC
channel configuration is needed.

Satellite is optional. Running `docker compose up ergo caddy` produces a minimal text-only Orbit
server.

## CORS Configuration

The web app and embedded client connect to Uplink (WebSocket), Satellite (HTTP + WebRTC), and
Depot (HTTP) endpoints - potentially on different origins. The reverse proxy (Caddy) MUST be
configured to set appropriate CORS headers on all API endpoints that web clients access:

- **Uplink (WebSocket)**: WebSocket connections are not subject to CORS preflight, but the `Origin` header SHOULD be validated by the reverse proxy to reject connections from unexpected origins.
- **Satellite (token service)**: The `/session/create`, `/session/join`, `/session/knock`, `/session/admit`, `/session/lock`, and `/info` endpoints MUST return `Access-Control-Allow-Origin` headers matching the web app's origin. Preflight (`OPTIONS`) requests MUST be handled.
- **Depot (upload API)**: The presign endpoint MUST return appropriate CORS headers. When Depot is backed by S3-compatible storage, pre-signed upload URLs also require CORS configuration on the bucket itself (MinIO or AWS S3 bucket CORS policy). With the local filesystem driver, uploads route through the Depot gateway and only the gateway's CORS headers apply.
- In the reference Caddy deployment, CORS headers are configured per-route in the Caddyfile. For development, a permissive `Access-Control-Allow-Origin: *` is acceptable; for production, restrict to the specific web app origin(s).

## Monitoring and Health Checks

Each Orbit service exposes a health endpoint suitable for use with monitoring tools (Uptime Kuma,
Prometheus, a simple `curl` cron job, etc.):

| Component | Health endpoint | What it checks |
|---|---|---|
| Uplink (Ergo) | IRC connect on port 6697 (or WebSocket on 6698) | TCP connectivity; Ergo does not expose an HTTP health endpoint by default |
| Satellite (token service) | `GET /health` | Token service reachability; optionally checks LiveKit connectivity |
| Depot API | `GET /health` | Depot gateway reachability and backing-store connectivity (S3 or local filesystem) |
| Auth-script bridge (optional) | `GET /healthz` | Bridge reachability and OIDC provider JWKS reachability; only present when the optional bridge is deployed |
| Caddy | `GET /` on the admin API (default port 2019) | Reverse proxy health |

**Recommended approach for self-hosters:** deploy
[Uptime Kuma](https://github.com/louislam/uptime-kuma) alongside Orbit. It requires no
configuration beyond pointing it at the health endpoints above, and provides a status page and
alerting (email, Telegram, etc.) out of the box.

**Logging:** Ergo writes structured logs to stdout by default; redirect to a file or a log
aggregator. When the optional auth-script bridge is deployed, it MUST emit structured logs for
all authentication attempts (as specified in
[Identity - Auth-Script Bridge Requirements](05-identity.md#auth-script-bridge-requirements)).
LiveKit and MinIO both produce structured JSON logs. In the reference Docker Compose deployment,
all logs are available via `docker compose logs`.

## Backup Mechanics

Operators are responsible for backing up their own data (the responsibility model is in
[Infrastructure architecture](../02-architecture/12-infrastructure.md)). The following components
produce state that must be backed up:

| Component | What to back up | Notes |
|---|---|---|
| Uplink (Ergo) | Ergo's SQLite database (`ircd.db`) | Contains all user accounts, channel registrations, and message history. Location is configurable; default is the Ergo data directory. |
| Depot (storage gateway) | Stored objects + Depot metadata database | Back up the backing store: for S3-compatible backends, mirror the bucket via `mc mirror` (MinIO Client) or equivalent; for the local filesystem driver, back up the mounted data directory. Back up the Depot metadata (SQLite/Postgres) database separately. |
| Configuration | `.env` file, `orbit.toml`, Caddyfile, `docker-compose.yml` | Store in version control. Do not commit secrets - use a secrets manager or encrypted store. |
| Auth-script bridge | No state to back up | Stateless optional service; configuration is in the `.env` file. |

Ergo's always-on mode (required for registered users) means the database grows continuously with
channel history. Operators should schedule regular backups and set a retention policy appropriate
to their storage budget.

A simple backup approach for the reference Docker Compose deployment is a daily cron job that
copies the Ergo data directory and the Depot backing store (the MinIO data directory, or the
local filesystem data directory) to an off-site location (S3, rsync to a remote host, etc.).

## Cross-References

- [Infrastructure architecture](../02-architecture/12-infrastructure.md) - topology, discovery model, TLS policy, resource shape
- [Uplink](01-uplink.md) - Ergo configuration detail
- [Satellite](03-satellite.md) - `/info` and the session endpoints behind CORS
- [Depot](04-depot.md) - storage gateway configuration
- [Identity](05-identity.md) - identity provider discovery and the auth-script bridge
