# Deployment

This page documents the infrastructure components for an Orbit deployment: their technology stack, resource requirements, TLS policy, DNS configuration, and the reference Docker Compose setup.

For the full client-side DNS resolution algorithm and per-service discovery behaviour, see [DNS & Service Discovery](01-domain-discovery.md).

## Component Overview

| Component | Technology | Resource target (~100 users) | Required |
|---|---|---|---|
| Ground Control (Ergochat) | Go, single binary | 1 vCPU, 256 MB RAM | Yes |
| Satellite | Go + thin HTTP API | 1 vCPU, 512 MB RAM (scales with users) | No (optional) |
| Depot (Object Storage) | MinIO or S3 | Storage-dependent | No (only for file uploads) |
| coturn (STUN/TURN) | C, single binary | 1 vCPU, 128 MB RAM | No (only for NAT traversal) |
| Auth-script bridge | Rust or Go, single binary | Negligible (stateless JWT verification) | No (only with OIDC identity provider) |
| **Total minimum (text-only)** | | **~$5/month VPS** | |
| **Total with voice** | | **~$10/month VPS** | |

A community can run Orbit text-only with just Ground Control (Ergochat). Satellite and Depot are optional components added as needed.

See [../02-components/01-ground-control/01-overview.md](../02-components/01-ground-control/01-overview.md), [../02-components/02-satellite.md](../02-components/02-satellite.md), and [../02-components/03-depot.md](../02-components/03-depot.md) for per-component architecture detail.

## TLS

TLS is required on every connection. No plaintext IRC, no plaintext HTTP, no exceptions.

- Let's Encrypt for public deployments (automated via certbot or Caddy).
- Self-signed certificates are acceptable for local and development setups only.
- Ergochat can terminate TLS for raw IRC connections directly (port 6697); a pure desktop-only, text-only deployment technically does not require a reverse proxy.
- A reverse proxy (Caddy in the reference deployment) terminates TLS for Ergochat's WebSocket listener, Satellite token service endpoints, and Depot endpoints. This reverse proxy is a **practical requirement for any public deployment** - even text-only, if web clients are expected to connect. The same Caddy instance also serves `/.well-known/orbit/services.json` as a static file for web client service discovery, at no additional infrastructure cost. The reference Docker Compose always includes Caddy regardless of which other services are enabled.

## DNS Configuration

DNS is the primary discovery mechanism for all Orbit services. These records are advertisement pointers published under the community domain - they tell Orbit clients which services are associated with this community. Each SRV record resolves to the actual host and port of the service, which may be on entirely separate infrastructure. A missing record means the client will not attempt to discover that service.

### SRV Records

| Record | Type | Purpose | Add when |
|---|---|---|---|
| `_satellite._tcp.example.com` | SRV | Satellite discovery | If running Satellite |
| `_depot._tcp.example.com` | SRV | Depot (file storage) discovery | If running Depot |
| `_transponder._tcp.example.com` | SRV | Transponder (identity) discovery | Post-MVP |

### A / AAAA Records

| Record | Purpose | Add when |
|---|---|---|
| `irc.example.com` | Ground Control (IRC + WebSocket) endpoint | If running IRC |
| `sat.example.com` | Satellite endpoint | If running Satellite |
| `depot.example.com` | Depot (object storage) endpoint | If running Depot |
| `turn.example.com` | TURN server | If running TURN |

A minimal text-only deployment needs only the `irc.example.com` A/AAAA record; the Orbit client connects to port 6697 (IRC over TLS) by default. A Satellite-only deployment (no IRC) needs only `_satellite._tcp` and the A/AAAA record for the Satellite. The client adapts based on which records are present.

Multiple `_satellite._tcp` SRV records can be configured to advertise multiple Satellites. SRV priority and weight control load distribution and failover.

For the full client-side resolution algorithm and service discovery behaviour, see [DNS & Service Discovery](01-domain-discovery.md).

## Reference Docker Compose

A reference `docker-compose.yml` is provided for self-hosters. It includes:

- **Ergochat** (Ground Control) - with WebSocket enabled, SASL configured, and chat history enabled.
- **LiveKit + token service** (Satellite) - **optional**; can be removed for text-only deployments.
- **MinIO** (Depot) - **optional**; only required if file uploads are needed.
- **coturn** (STUN/TURN) - for NAT traversal.
- **Auth-script bridge** (Transponder integration) - **optional (post-MVP)**; required only when an OIDC identity provider is configured. Verifies JWTs for Ergochat's `auth-script` SASL delegation. See [Transponder](../02-components/04-transponder.md).
- **Caddy** (reverse proxy) - terminates TLS via Let's Encrypt, routes WebSocket and API requests to Ergochat and Satellite, and serves `/.well-known/orbit/services.json` as a static file for web client service discovery.

One `docker compose up` produces a fully functional Orbit instance. Configuration is done via a single `.env` file and an `orbit.toml` for server-specific settings (domain, channel list). Satellite discovery is handled via DNS SRV records configured at the domain level - no IRC channel configuration is needed.

Satellite is optional. Running `docker compose up ergochat caddy` produces a minimal text-only Orbit server.

## CORS Configuration

The web app and widget connect to Ground Control (WebSocket), Satellite (HTTP + WebRTC), and Depot (HTTP) endpoints - potentially on different origins. The reverse proxy (Caddy) MUST be configured to set appropriate CORS headers on all API endpoints that web clients access:

- **Ground Control (WebSocket)**: WebSocket connections are not subject to CORS preflight, but the `Origin` header SHOULD be validated by the reverse proxy to reject connections from unexpected origins.
- **Satellite (token service)**: The `/session/create`, `/session/join`, `/session/knock`, `/session/admit`, `/session/lock`, and `/info` endpoints MUST return `Access-Control-Allow-Origin` headers matching the web app's origin. Preflight (`OPTIONS`) requests MUST be handled.
- **Depot (upload API)**: The presign endpoint MUST return appropriate CORS headers. S3 pre-signed upload URLs also require CORS configuration on the S3 bucket itself (MinIO or AWS S3 bucket CORS policy).

In the reference Caddy deployment, CORS headers are configured per-route in the Caddyfile. For development, a permissive `Access-Control-Allow-Origin: *` is acceptable; for production, restrict to the specific web app origin(s).
