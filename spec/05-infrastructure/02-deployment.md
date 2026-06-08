# Deployment

This page documents the infrastructure components for an Orbit deployment: their technology stack, resource requirements, TLS policy, DNS configuration, and the reference Docker Compose setup.

For the full client-side DNS resolution algorithm and per-service discovery behaviour, see [DNS & Service Discovery](01-domain-discovery.md).

## Component Overview

| Component | Technology | Resource target (~100 users) | Required |
|---|---|---|---|
| Uplink (Ergo) | Go, single binary | 1 vCPU, 256 MB RAM | Yes |
| Satellite | Go + thin HTTP API | 1 vCPU, 512 MB RAM (scales with users) | No (optional) |
| Depot (storage gateway) | Go, thin gateway over S3-compatible storage (MinIO/AWS S3) or a local filesystem | Storage-dependent | No (only for file uploads) |
| coturn (STUN/TURN) | C, single binary | 1 vCPU, 128 MB RAM | No (only for NAT traversal) |
| Auth-script bridge | Rust or Go, single binary | Negligible (stateless JWT verification) | No (required for ES256/ECDSA providers; optional for RS256/EdDSA/HMAC with native jwt-auth) |
| **Total minimum (text-only)** | | **~$5/month VPS** | |
| **Total with voice** | | **~$10/month VPS** | |

A community can run Orbit text-only with just Uplink (Ergo). Satellite and Depot are optional components added as needed. Identity verification uses the auth-script bridge (general-purpose, any provider/algorithm) or Ergo's native `accounts.jwt-auth` (RS256/EdDSA/HMAC, static key). See [Transponder](../02-components/04-transponder.md#uplink-stock-ergo) for path selection.

See [../02-components/01-uplink/01-overview.md](../02-components/01-uplink/01-overview.md), [../02-components/02-satellite.md](../02-components/02-satellite.md), and [../02-components/03-depot.md](../02-components/03-depot.md) for per-component architecture detail.

## TLS

TLS is required on every connection. No plaintext IRC, no plaintext HTTP, no exceptions.

- Let's Encrypt for public deployments (automated via certbot or Caddy).
- Self-signed certificates are acceptable for local and development setups only.
- Ergo can terminate TLS for raw IRC connections directly (port 6697); a pure desktop-only, text-only deployment technically does not require a reverse proxy.
- A reverse proxy (Caddy in the reference deployment) terminates TLS for Ergo's WebSocket listener, Satellite token service endpoints, and Depot endpoints. This reverse proxy is a **practical requirement for any public deployment** - even text-only, if web clients are expected to connect. The same Caddy instance also serves `/.well-known/orbit/services.json` as a static file for web client service discovery, at no additional infrastructure cost. The reference Docker Compose always includes Caddy regardless of which other services are enabled.

## DNS Configuration

DNS is the primary discovery mechanism for all Orbit services. These records are advertisement pointers published under the community domain - they tell Orbit clients which services are associated with this community. Each SRV record resolves to the actual host and port of the service, which may be on entirely separate infrastructure. A missing record means the client will not attempt to discover that service.

### SRV Records

| Record | Type | Purpose | Add when |
|---|---|---|---|
| `_satellite._tcp.example.com` | SRV | Satellite discovery | If running Satellite |
| `_depot._tcp.example.com` | SRV | Depot (file storage) discovery | If running Depot |
| `_transponder._tcp.example.com` | SRV | Transponder (identity) discovery | If running Transponder |

### A / AAAA Records

| Record | Purpose | Add when |
|---|---|---|
| `irc.example.com` | Uplink (IRC + WebSocket) endpoint | If running IRC |
| `sat.example.com` | Satellite endpoint | If running Satellite |
| `depot.example.com` | Depot (storage gateway) endpoint | If running Depot |
| `turn.example.com` | TURN server | If running TURN |

A minimal text-only deployment needs only the `irc.example.com` A/AAAA record; the Orbit client connects to port 6697 (IRC over TLS) by default. A Satellite-only deployment (no IRC) needs only `_satellite._tcp` and the A/AAAA record for the Satellite. The client adapts based on which records are present.

Multiple `_satellite._tcp` SRV records can be configured to advertise multiple Satellites. SRV priority and weight control load distribution and failover.

For the full client-side resolution algorithm and service discovery behaviour, see [DNS & Service Discovery](01-domain-discovery.md).

## Reference Docker Compose

A reference `docker-compose.yml` is provided for self-hosters. It includes:

- **Ergo** - the adopted IRC server filling the Uplink role - with WebSocket enabled, SASL configured, and chat history enabled.
- **LiveKit + token service** (Satellite) - **optional**; can be removed for text-only deployments.
- **Depot** (storage gateway) - **optional**; only required if file uploads are needed. The thin gateway runs against either an S3-compatible backend (a bundled MinIO container, or external AWS S3) or a local filesystem driver (a mounted volume). Single-box deployments can use the local filesystem driver and skip MinIO entirely; deployments that need scale or off-host storage point Depot at S3/MinIO.
- **coturn** (STUN/TURN) - for NAT traversal.
- **Auth-script bridge** - required when the OIDC provider signs with ES256 (or any algorithm not supported by Ergo's native `jwt-auth`); optional when using native `jwt-auth` with an RS256/EdDSA/HMAC provider. See [Transponder](../02-components/04-transponder.md#uplink-stock-ergo).
- **Caddy** (reverse proxy) - terminates TLS via Let's Encrypt, routes WebSocket and API requests to Ergo and Satellite, and serves `/.well-known/orbit/services.json` as a static file for web client service discovery.

One `docker compose up` produces a fully functional Orbit instance. Configuration is done via a single `.env` file and an `orbit.toml` for server-specific settings (domain, channel list). Satellite discovery is handled via DNS SRV records configured at the domain level - no IRC channel configuration is needed.

Satellite is optional. Running `docker compose up ergo caddy` produces a minimal text-only Orbit server.

## CORS Configuration

The web app and widget connect to Uplink (WebSocket), Satellite (HTTP + WebRTC), and Depot (HTTP) endpoints - potentially on different origins. The reverse proxy (Caddy) MUST be configured to set appropriate CORS headers on all API endpoints that web clients access:

- **Uplink (WebSocket)**: WebSocket connections are not subject to CORS preflight, but the `Origin` header SHOULD be validated by the reverse proxy to reject connections from unexpected origins.
- **Satellite (token service)**: The `/session/create`, `/session/join`, `/session/knock`, `/session/admit`, `/session/lock`, and `/info` endpoints MUST return `Access-Control-Allow-Origin` headers matching the web app's origin. Preflight (`OPTIONS`) requests MUST be handled.
- **Depot (upload API)**: The presign endpoint MUST return appropriate CORS headers. When Depot is backed by S3-compatible storage, pre-signed upload URLs also require CORS configuration on the bucket itself (MinIO or AWS S3 bucket CORS policy). With the local filesystem driver, uploads route through the Depot gateway and only the gateway's CORS headers apply.

In the reference Caddy deployment, CORS headers are configured per-route in the Caddyfile. For development, a permissive `Access-Control-Allow-Origin: *` is acceptable; for production, restrict to the specific web app origin(s).

## Backups

Operators are responsible for backing up their own data. The following components produce state that must be backed up:

| Component | What to back up | Notes |
|---|---|---|
| Uplink (Ergo) | Ergo's SQLite database (`ircd.db`) | Contains all user accounts, channel registrations, and message history. Location is configurable; default is the Ergo data directory. |
| Depot (storage gateway) | Stored objects + Depot metadata database | Back up the backing store: for S3-compatible backends, mirror the bucket via `mc mirror` (MinIO Client) or equivalent; for the local filesystem driver, back up the mounted data directory. Back up the Depot metadata (SQLite/Postgres) database separately. |
| Configuration | `.env` file, `orbit.toml`, Caddyfile, `docker-compose.yml` | Store in version control. Do not commit secrets - use a secrets manager or encrypted store. |
| Auth-script bridge | No state to back up | Stateless optional service; configuration is in the `.env` file. |

Ergo's always-on mode (required for registered users) means the database grows continuously with channel history. Operators should schedule regular backups and set a retention policy appropriate to their storage budget.

A simple backup approach for the reference Docker Compose deployment is a daily cron job that copies the Ergo data directory and the Depot backing store (the MinIO data directory, or the local filesystem data directory) to an off-site location (S3, rsync to a remote host, etc.).

## Monitoring and Health Checks

Each Orbit service exposes a health endpoint suitable for use with monitoring tools (Uptime Kuma, Prometheus, a simple `curl` cron job, etc.):

| Component | Health endpoint | What it checks |
|---|---|---|
| Uplink (Ergo) | IRC connect on port 6697 (or WebSocket on 6698) | TCP connectivity; Ergo does not expose an HTTP health endpoint by default |
| Satellite (token service) | `GET /health` | Token service reachability; optionally checks LiveKit connectivity |
| Depot API | `GET /health` | Depot gateway reachability and backing-store connectivity (S3 or local filesystem) |
| Auth-script bridge (optional) | `GET /healthz` | Bridge reachability and OIDC provider JWKS reachability; only present when the optional bridge is deployed |
| Caddy | `GET /` on the admin API (default port 2019) | Reverse proxy health |

**Recommended approach for self-hosters:** deploy [Uptime Kuma](https://github.com/louislam/uptime-kuma) alongside Orbit. It requires no configuration beyond pointing it at the health endpoints above, and provides a status page and alerting (email, Telegram, etc.) out of the box.

**Logging:** Ergo writes structured logs to stdout by default; redirect to a file or a log aggregator. When the optional auth-script bridge is deployed, it MUST emit structured logs for all authentication attempts (as specified in [Transponder - Auth-Script Bridge](../02-components/04-transponder.md#auth-script-bridge)). LiveKit and MinIO both produce structured JSON logs. In the reference Docker Compose deployment, all logs are available via `docker compose logs`.
