# Depot

Depot is the file storage component of an Orbit deployment: a thin S3/disk
policy-and-signing gateway, used for file uploads, avatars, and other binary
assets shared within communities. It holds the storage credentials, decides
who may upload what, signs (or proxies) the transfer, and gets out of the
way. It's a bespoke component Orbit builds - see
[Component Classes](01-overview.md#component-classes).

Existing self-hosted products like Zipline and Plik already do everything
Depot does and far more: web UIs, galleries, URL shorteners, dashboards.
Orbit doesn't want that. The Orbit client is the UI; Depot only needs to be
the authority that sits in front of the bytes.

Raw S3 alone is insufficient, which is why a thin authority is always needed
in front of it: S3 can't verify an OIDC token and attribute an upload to one
of *your* users, can't enforce per-user quotas, can't mint application API
keys, and can't scope a download to specific recipients. Those are
application-level concerns, and Depot is exactly that thin authority and
nothing more.

Depot is optional - text chat and real-time media function without it. File
sharing within Uplink channels requires a Depot instance.

The API table, upload flow, quotas, metadata schema, object keys, and
configuration live in [Implementation - Depot](../03-implementation/04-depot.md).
The file metadata tag shared alongside uploads is defined in
[Implementation - Tags](../03-implementation/02-tags.md).

## What S3 Does vs What Depot Does

Depot stays thin by leaning on everything S3-compatible storage already does
natively, and only implementing what storage can't.

S3-compatible storage provides, and Depot leans on: direct client transfer
via pre-signed URLs (bytes never touch Depot), per-upload constraints baked
into the signature (content type, size range, key prefix, expiry), object
metadata, lifecycle and expiry rules, bucket and IAM policies, CORS, and
range/multipart transfer.

Only Depot can: verify a [Transponder](07-transponder.md) identity token and
attribute the upload to a real account, enforce per-user storage quotas, mint
and manage application API keys for external tools, and scope downloads to
specific recipients.

Everything in the first list is configuration Depot hands to the storage
backend. Everything in the second list is the reason Depot exists at all.

## Storage Drivers

Where the bytes live is the first of two orthogonal axes an operator
composes. The storage driver contract is two operations: presign an upload
under constraints, and resolve a download. The client contract is identical
for both drivers - the client always asks Depot for an upload URL and
transfers to whatever it gets back; it never knows which backend is behind
Depot.

- **S3 driver**: uploads are direct pre-signed PUTs to the storage backend;
  bytes never touch Depot. Downloads are public object URLs, or short-lived
  pre-signed GETs when the object is private. Any S3-compatible backend
  works: MinIO, Amazon S3, Cloudflare R2, Backblaze B2, Garage.
- **Filesystem driver**: there's no pre-signed-URL equivalent for a local
  disk, so Depot proxies the transfer and serves downloads itself. No object
  storage is required to run.

The trade: the S3 driver keeps Depot stateless and bandwidth-free - the
backend carries all transfer load, which is the path to scale. The filesystem
driver puts Depot in the data path, consuming its bandwidth and CPU, in
exchange for needing no object storage at all. Filesystem for
single-box/homelab, S3 for scale.

## Accepted Credentials

The second axis is which credentials Depot accepts. This is a set of
capability flags the operator toggles in any combination, not a choice
between exclusive modes:

- **Anonymous** - accept unauthenticated upload requests.
- **OIDC** - accept a [Transponder](07-transponder.md) identity token and
  attribute the upload to the account.
- **API key** - accept a long-lived Depot-issued key, for external tools
  (ShareX, Puush, cURL) that can't run an interactive OIDC flow.

An operator can enable several at once: OIDC plus API keys for a normal Orbit
server, or anonymous plus OIDC for a community that wants quick anonymous
drops alongside attributed uploads.

When OIDC is enabled, Depot verifies the token signature against the
provider's published keys - the same verification pattern used by
[Satellite](05-satellite.md) and Uplink's auth bridge. No component contacts
any other component to check identity.

API keys are a separate auth path from the short-lived OIDC browser token:
minted by an already-authenticated user, stored only as a hash, revocable
instantly, and resolved back to the owner so a key-attributed upload reuses
all the identity machinery - quota, deletion rights, audit. Key lifecycle and
endpoints live in [Implementation - Depot](../03-implementation/04-depot.md).

### Statefulness Is Opt-In per Capability

Pure anonymous signing needs no state; Depot is fully stateless. State is
required only by the capabilities that genuinely need it - quotas, deletion,
audit, API keys, and recipient-scoping all require a small metadata store
(SQLite is plenty; Postgres is optional for operators who already run one).
An operator who enables only anonymous uploads runs a stateless Depot.

### Guest Users

Regardless of configuration, anonymous guests (SASL ANONYMOUS users) can't
upload files. In OIDC-only configurations they have no token; when anonymous
uploads are enabled, the Orbit client still doesn't present the upload UI to
guests. This is client-side enforcement - guests aren't the intended audience
for file uploads.

## Recipient-Scoped Uploads (Future Capability)

DMs need a file shared with a specific person, not the whole world. Only
Depot can provide this, because S3 has no concept of your accounts. Two
layers are designed:

- **Layer 1 - access-controlled downloads.** Files are stored private, with
  the allowed recipients recorded in Depot's metadata. Downloads go through
  Depot, which checks that the requester is on the list; the requester proves
  their own identity with their own valid token. At upload time the sender
  asserts the recipient identities; at download time the recipient proves
  their own - distinct steps. This forces the private/proxied download path
  even under the S3 driver, and it only works when OIDC is enabled -
  anonymous callers can't scope recipients.
- **Layer 2 - end-to-end encryption.** Supersedes Layer 1's trust assumption:
  the client encrypts the file before upload, and the URL points to
  ciphertext, useless without a key that never leaves the participants'
  clients. Depot stays a dumb store and the guarantee comes from the
  encryption layer. See [E2E Encryption](13-e2e.md).

Until these ship, operators of private communities should know that ordinary
Depot URLs are accessible to anyone who obtains them, and communicate this to
their users.

## Content Moderation

Every upload either passes through Depot (filesystem driver) or is authorized
by Depot (S3 pre-signed URL), so the gateway is the natural chokepoint for
optional automated content scanning: hash-matching against known-bad
databases, ML classification, or operator-defined rules. This is an operator
concern - Depot provides the hook point, not the scanner.

The capability scales with deployment size. A small friend-group instance
doesn't need scanning; a large public-facing one probably does. Depot's job
is to make both paths easy: no overhead when unconfigured, clean integration
points when needed.

### Storage-Driver Asymmetry

The two drivers create a real asymmetry for scanning:

- **Filesystem/proxy mode**: Depot receives the bytes directly, so scanning
  can happen inline, before the file is available to other users. The
  simplest and safest path for deployments that need hard content gating.
- **S3-direct mode**: bytes go straight to the backend and never touch Depot,
  so scanning is asynchronous, typically triggered by a storage event
  notification. Until the scan completes, the object exists in storage.

Deployments that need real content gating under the S3 driver should use a
private-by-default bucket policy with gateway-mediated downloads: Depot
issues a short-lived download URL only after the object's scan status is
clean. That sacrifices the bandwidth-free public-URL property in exchange for
content safety. The spectrum:

| Posture | Bucket policy | Download path | Scanning | Typical deployment |
|---------|--------------|---------------|----------|--------------------|
| No scanning | Public | Direct URL | None | Friend-group, homelab |
| Async scan-and-quarantine | Public | Direct URL | Async, best-effort | Mid-size community |
| Private-by-default, gateway-served | Private | Via Depot | Inline or pre-serve | Large/public-facing instance |

### Safe Anonymous Defaults

Anonymous users can join calls and chat without restriction - that's the
frictionless entry model - but they shouldn't be able to upload files to
public channels by default. The safe default is authenticated uploads only;
anonymous upload capability is an explicit operator opt-in, and when enabled
it carries tighter rate limits and smaller size caps than authenticated
uploads.

For shared content, every upload is attributed to an identity, content is
removable (uploaders can delete their own files; operators can always delete
at the backend level), and operators can scan at the gateway. The full
layered moderation posture, including the private-content side, lives in
[Messaging - Moderation](10-messaging.md#moderation).
