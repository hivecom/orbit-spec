# Infrastructure

The deployment shape of an Orbit instance: topology and resource footprint,
service discovery, TLS policy, backup responsibility, and the community
directory. DNS record examples, the discovery file schema, resolution
algorithms, Docker Compose, CORS, and monitoring live in
[Implementation - Deployment](../03-implementation/09-deployment.md).

## Topology and Resource Shape

| Component | Technology | Resource target (~100 users) | Required |
|---|---|---|---|
| Uplink (Ergo) | Go, single binary | 1 vCPU, 256 MB RAM | Yes |
| Satellite | Go + thin HTTP API | 1 vCPU, 512 MB RAM (scales with users) | No |
| Depot | Go, thin gateway over S3-compatible storage or a local filesystem | Storage-dependent | No (only for file uploads) |
| coturn (STUN/TURN) | C, single binary | 1 vCPU, 128 MB RAM | No (only for NAT traversal) |
| Auth bridge | Single stateless binary | Negligible | No (required for providers Ergo can't verify natively - see [Transponder](07-transponder.md#uplink)) |
| **Total minimum (text-only)** | | **~$5/month VPS** | |
| **Total with voice** | | **~$10/month VPS** | |

A community can run Orbit text-only with just Uplink. Satellite and Depot are
added as needed, and a reverse proxy (Caddy in the reference deployment) ties
the public surface together. One `docker compose up` produces a functional
instance; the reference compose file lives in
[Implementation - Deployment](../03-implementation/09-deployment.md).

## Service Discovery

Users enter only a domain; the client resolves available services
automatically. A domain can advertise any combination of services - IRC only,
Satellite only, or the full stack - and the client adapts to which records
and endpoints are present.

Not all client types have equal discovery capabilities. Desktop clients
(with a native resolver) can perform DNS SRV lookups; browser-based clients
can't, because browsers expose no DNS SRV API, so they rely on a well-known
discovery URL with manual entry as the last resort.

| Discovery method | Desktop | Web App / PWA | Embedded |
|---|---|---|---|
| DNS SRV records | Yes | No | No |
| A/AAAA direct lookup | Yes | No | No |
| Well-known URL (`/.well-known/orbit/services.json`) | Yes (as shortcut) | Yes | Yes |
| Manual host entry | Yes | Yes | No (pre-configured) |

The discovery surface:

- **SRV records** advertise Satellite, Depot, and Transponder under the
  community domain. Uplink doesn't use an SRV record - it's reached via a
  conventional `irc.` subdomain on the standard IRC TLS port, with a
  bare-domain attempt as a defensive fallback.
- **A/AAAA records** are the actual host addresses those pointers resolve to,
  and may point at entirely separate infrastructure from the community root
  domain.
- **The well-known file** is a static JSON document served over HTTPS on the
  community domain, listing the same services for clients that can't do DNS -
  analogous to Matrix's client well-known. It's optional but RECOMMENDED for
  any community that wants web users to get automatic discovery, and it costs
  nothing extra: every practical deployment already runs a reverse proxy for
  TLS, and serving one static file is a two-line config addition.
- **Identity provider discovery** follows the same pattern: a well-known URL
  on the community domain returns the OIDC issuer (all clients), or an SRV
  record advertises it (desktop).

Missing records mean missing services, and clients MUST degrade gracefully:
a service that isn't advertised is silently absent from the UI - no
Satellite records means voice UI is hidden (P2P and BYOS still work), no
Depot means upload UI is hidden, no identity provider means everyone is
unverified. No error is shown unless every applicable resolution path is
exhausted.

### Operator Guidance

DNS records and the well-known file are advertisement pointers: they tell
Orbit clients which services are associated with this community, not that
those services run on the community domain itself. Publish only records for
services you actually run. If the community domain doesn't serve HTTPS at
all, rely on DNS for desktop discovery and manual entry for web clients, or
proxy the well-known path to somewhere that does.

## TLS

TLS is required on every connection. No plaintext IRC, no plaintext HTTP, no
exceptions.

- Let's Encrypt for public deployments, automated by the reverse proxy or
  certbot. Self-signed certificates are acceptable for local development
  only.
- Ergo can terminate TLS for raw IRC connections directly, so a desktop-only,
  text-only deployment technically doesn't require a reverse proxy.
- In practice a reverse proxy is a requirement for any public deployment: it
  terminates TLS for Ergo's WebSocket listener (web clients), routes HTTPS to
  Satellite and Depot, and serves the well-known discovery file. The
  reference deployment always includes it.

## Backups

Operators are responsible for backing up their own data. What carries state:

| Component | What to back up |
|---|---|
| Uplink (Ergo) | Ergo's database - accounts, channel registrations, and message history |
| Depot | The backing store (bucket or data directory) plus Depot's metadata database |
| Configuration | Deployment configuration files, kept in version control without secrets |
| Auth bridge | Nothing - stateless |

Always-on mode means the Ergo database grows continuously with history;
operators should schedule regular backups and set a retention policy
appropriate to their storage budget. Concrete backup mechanics live in
[Implementation - Deployment](../03-implementation/09-deployment.md).

## Community Directory

The Orbit project operates a public server directory at `orbit.directory`, a
dedicated domain for discovering Orbit communities. It's an Orbit project
resource, not tied to any specific community or operator, and the Orbit
clients query it for their community browsing surface (the experience is
specified in [Product - Experience](../01-product/02-experience.md)).

Listing is self-service and verified by domain ownership:

1. **Submit**: the operator submits their domain.
2. **Verify**: the directory fetches the domain's well-known discovery file.
   Serving it proves control of the domain - the same shape as an HTTP-01
   challenge - and tells the directory which services the community offers.
3. **Live**: the listing appears in the directory and in client browse views.
   Operators can update or delist at any time.

A listing carries the domain, a display name, a description, an approximate
user count, the available services (populated from the discovery file), and
operator-provided region and language tags for filtering and search.

Risks and their handling:

- **Spam and abuse of self-service listings.** Domain verification is the
  primary mitigation - proving ownership of a real domain prevents throwaway
  listings. A report mechanism lets users flag abusive listings for review
  and removal.
- **Content moderation.** Communities hosting illegal or harmful content
  shouldn't be promoted; the directory enforces a listing policy and acts on
  reports.
- **Privacy.** The directory is strictly opt-in. Communities that don't
  register aren't listed and aren't affected - it's a convenience layer, not
  a dependency.
