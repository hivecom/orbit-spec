# Domain Discovery

Domain discovery is the mechanism by which Orbit clients locate services associated with a community domain. Users enter only a domain (e.g., `example.com`); the client resolves available services automatically using whichever discovery methods the client type supports. A domain can advertise any combination of services - IRC only, Satellite only, or the full stack - and the client adapts based on which records and endpoints are present.

Not all client types have equal discovery capabilities. Desktop clients (built with a Rust resolver) can perform full DNS SRV lookups. Web clients (browser-based PWAs or widgets) cannot - browsers do not expose a DNS SRV API. Web clients rely on well-known URL discovery, with manual entry as a last resort.

This page is the canonical reference for all discovery mechanisms and client-side resolution behaviour. Individual service pages link here rather than duplicating record definitions.

## Client Capability Matrix

| Discovery method | Desktop | Web App / PWA | Widget |
|---|---|---|---|
| DNS SRV records | Yes (Rust resolver) | No | No |
| A/AAAA direct lookup | Yes | No | No |
| Well-known URL (`/.well-known/orbit/services.json`) | Yes (as shortcut) | Yes | Yes |
| Manual host entry | Yes | Yes | No (pre-configured) |

Clients MUST degrade gracefully when a discovery method is unavailable. Missing services are silently absent from the UI; no error is shown to the user unless all applicable resolution paths have been exhausted.

## DNS Records

DNS discovery applies to desktop clients only. Web clients skip this section entirely and proceed to [Web Client Compatibility](#web-client-compatibility).

### SRV Records

| Record | Type | Purpose | Add when |
|---|---|---|---|
| `_satellite._tcp.example.com` | SRV | Satellite discovery | If running Satellite |
| `_depot._tcp.example.com` | SRV | Depot (file storage) discovery | If running Depot |
| `_transponder._tcp.example.com` | SRV | Identity provider (OIDC) discovery | If running Transponder |

> **Uplink (IRC)** does not use an SRV record. It is reached via a conventional `irc.example.com` A/AAAA record on port 6697. See [Uplink Resolution](#uplink-resolution) below.

### A / AAAA Records

| Record | Purpose | Add when |
|---|---|---|
| `irc.example.com` | Uplink (IRC + WebSocket) endpoint | If running IRC |
| `sat.example.com` | Satellite endpoint | If running Satellite |
| `depot.example.com` | Depot (storage gateway) endpoint | If running Depot |
| `turn.example.com` | TURN server | If running TURN |

These records are the actual host addresses that SRV targets and direct connections resolve to. They may point to entirely separate infrastructure from the community root domain.

## Web Client Compatibility

Browser-based clients cannot perform DNS SRV lookups. The well-known URL covers this gap; manual entry is the fallback when the file is absent.

### Well-known URL

Operators MAY publish a static JSON file at:

```
https://example.com/.well-known/orbit/services.json
```

Web clients (and desktop clients as a convenience shortcut, bypassing DNS) fetch this file via plain HTTPS to discover available services. The file format is:

```json
{
  "irc": { "host": "irc.example.com", "port": 6697, "ws_port": 6698 },
  "satellite": [
    { "host": "sat.example.com", "port": 7880, "name": "US East" }
  ],
  "depot": { "host": "depot.example.com", "port": 3000 }
}
```

This is analogous to `/.well-known/matrix/client` for Matrix servers. Operators serving static files can publish this with no additional backend. Missing keys mean that service is not available - the client treats absent keys identically to absent DNS records.

This mechanism is **optional but RECOMMENDED** for any community that wants web client users to benefit from automatic discovery.

> **No additional infrastructure cost.** Every practical Orbit deployment already includes a reverse proxy (Caddy in the reference setup) because the Ergochat WebSocket listener requires TLS termination and Satellite/Depot API endpoints require HTTPS routing. Since Caddy is already present, serving `/.well-known/orbit/services.json` as a static file is a two-line addition to the Caddy config - no extra service, port, or process is needed. Operators running a public deployment are already running the infrastructure required to serve this file.

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

## Per-Service Resolution

### Uplink Resolution

Resolution behaviour differs by client type.

**Desktop clients** resolve Uplink in this order:

1. **Well-known URL** - fetch `/.well-known/orbit/services.json` and use the `irc.host` and `irc.port` values if present.
2. **Conventional subdomain** - attempt `irc.example.com` on port `6697` (IRC over TLS). This is the expected path for well-configured deployments.
3. **Bare domain fallback** - attempt `example.com` on port `6697` as a last resort (covers operators who skipped the subdomain entirely).
4. **Manual entry** - if all automatic attempts fail, prompt the user to enter a host:port manually.

**Web clients** resolve Uplink in this order:

1. **Well-known URL** - fetch `/.well-known/orbit/services.json` and use the `irc.host` and `irc.ws_port` values if present. Web clients MUST connect via WebSocket using `ws_port`, not the raw IRC port.
2. **Manual entry** - if the well-known URL is absent or does not contain `irc`, prompt the user to enter a host manually.

Operators SHOULD publish an `irc.example.com` A/AAAA record. The bare domain fallback (desktop step 3) exists only as a defensive measure and SHOULD NOT be relied upon.

See [../02-components/01-uplink/01-overview.md](../02-components/01-uplink/01-overview.md) for Uplink architecture.

### Satellite Resolution

**Desktop clients:**

1. Resolve `_satellite._tcp.example.com` SRV records.
2. For each discovered record, query the node `GET /info` metadata endpoint to retrieve name, region, capacity, and version.
3. Present discovered nodes as **Server Nodes** with a verified badge. DNS records under the domain are the operator assertion that these nodes are official.
4. SRV priority and weight are respected for load balancing and failover.

**Web clients:**

1. Fetch `/.well-known/orbit/services.json` and read the `satellite` array. Each entry is a node in priority order.
2. For each entry, query `GET /info` to retrieve node metadata and verify availability.
3. If no well-known file is available, prompt the user for manual entry.

**If no Satellites are discovered by any applicable mechanism:** Voice features degrade gracefully - P2P calls still work (no Satellite is required for P2P), and BYOS Satellites remain available, but group voice via server-operated Satellites is unavailable. The client hides Satellite selection UI rather than showing an error.

**Multiple nodes** may be advertised via multiple SRV records (desktop) or multiple entries in the `satellite` array (all clients). Priority order follows SRV priority/weight for DNS, and array index for well-known discovery.

See [../02-components/02-satellite.md](../02-components/02-satellite.md) for Satellite architecture.

### Depot Resolution

**Desktop clients:**

1. Resolve `_depot._tcp.example.com` SRV.
2. Use the returned host and port for all file upload and download requests.

**Web clients:**

1. Fetch `/.well-known/orbit/services.json` and read the `depot` object.
2. If no well-known file is available, file sharing is unavailable until a host is entered manually.

**If no Depot endpoint is discovered by any applicable mechanism:** File sharing is unavailable. The client hides the upload UI rather than presenting a broken state.

See [../02-components/03-depot.md](../02-components/03-depot.md) for Depot architecture.

### Identity Provider Resolution

The Orbit client discovers the server's OIDC identity provider (the Transponder role) through the following mechanisms in priority order:

1. **Well-known URL** - fetch `/.well-known/orbit/oidc` on the server's domain. This returns a JSON document containing the OIDC issuer URL. The client then fetches the standard `/.well-known/openid-configuration` from that issuer to discover all endpoints. (All clients.)
2. **DNS SRV** - resolve `_transponder._tcp.example.com`. Use the returned host and port as the OIDC issuer base URL. (Desktop only.)

**If no identity provider is discovered:** all Satellite participants are treated as unverified. Voice sessions continue normally; identity verification is absent.

See [../02-components/04-transponder.md](../02-components/04-transponder.md) for the full identity provider specification.

## Operator Guidance

DNS records and the well-known file are **advertisement pointers** published under the community domain. They tell Orbit clients which services are associated with this community, not that those services run on `example.com` directly. Each record or entry resolves to the actual host and port of the service, which may be on entirely separate infrastructure.

**Publish only records for services you are actually running.** A missing record or absent key means the client will not attempt to discover that service - it degrades gracefully with no error shown to the user.

### DNS (Desktop Clients)

| Deployment type | Required records |
|---|---|
| Text-only (IRC only) | `irc.example.com` A/AAAA |
| IRC + voice | `irc.example.com` A/AAAA, `_satellite._tcp` SRV, `sat.example.com` A/AAAA |
| IRC + voice + files | All of the above, plus `_depot._tcp` SRV, `depot.example.com` A/AAAA |
| Satellite-only (no IRC) | `_satellite._tcp` SRV, `sat.example.com` A/AAAA |

### Well-known URL (All Clients)

Publish `/.well-known/orbit/services.json` at the community domain. This file MUST be served over HTTPS with an appropriate `Content-Type: application/json` header. The file MAY be served from a CDN, static file host, or reverse proxy - no dynamic backend is required.

In the reference deployment (Docker Compose with Caddy), serving this file requires only adding a static file route to the Caddy config - no additional service, container, or port is needed. Caddy is already present in the reference setup to terminate TLS for the Ergochat WebSocket listener and route HTTPS traffic to Satellite and Depot, so this is a zero-cost addition for any operator following the reference deployment.

If the community domain does not serve HTTP(S) itself, the operator SHOULD configure a redirect or proxy at that domain pointing to the file, or rely on DNS records for desktop discovery and manual entry for web client fallback.

### Summary by Deployment Type

| Deployment type | DNS records (desktop) | Well-known JSON (all clients) |
|---|---|---|
| Text-only (IRC only) | `irc.example.com` A/AAAA | `irc` key only |
| IRC + voice | `irc.example.com`, `_satellite._tcp` SRV, `sat.example.com` | `irc` + `satellite` |
| IRC + voice + files | All of the above + `_depot._tcp` SRV, `depot.example.com` | `irc` + `satellite` + `depot` |
| Satellite-only (no IRC) | `_satellite._tcp` SRV, `sat.example.com` | `satellite` only |

For the complete operator setup guide including Docker Compose configuration, see [02-deployment.md](02-deployment.md).