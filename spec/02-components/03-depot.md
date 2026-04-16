# Depot

Depot is the file storage component of an Orbit deployment. It provides S3-compatible object
storage for file uploads, user avatars, and other binary assets shared within Orbit communities.

Depot is an optional component - text chat and real-time media function without it. File sharing
within Ground Control channels requires a Depot instance.

A Depot deployment consists of two parts:

- **S3-compatible backend**: MinIO, Amazon S3, or any compatible service that stores the actual
  objects and serves them via public URLs.
- **Depot API**: A thin HTTP service that sits in front of the S3 backend. This is the only piece
  that Orbit clients interact with directly - the S3 backend is never accessed by clients except
  via pre-signed upload URLs issued by the Depot API.

For backend configuration (MinIO setup, S3 bucket policy, TLS), see
[Infrastructure & Deployment](../05-infrastructure/02-deployment.md).

For DNS-based discovery of a domain's Depot instance, see
[DNS & Service Discovery](../05-infrastructure/01-domain-discovery.md).

## Upload Flow

1. User selects a file in the Orbit client.
2. The client sends a pre-signed URL request to the Depot API. The Depot API authenticates the
   request, enforces rate limits and maximum file size, and - if all checks pass - generates a
   pre-signed S3 upload URL.
3. The client uploads the file directly to the S3 backend via the pre-signed URL. The upload
   does not pass through the Depot API.
4. On successful upload, the client posts a `PRIVMSG` to the channel containing the public file
   URL as the message body.
5. The client attaches file metadata as message tags on the same `PRIVMSG`: `+orbit/file-name`,
   `+orbit/file-size`, `+orbit/file-type`.
6. The Orbit client renders an inline preview (images, audio, video) or a download card. Pure IRC
   clients see a plain URL.

## The Depot API

The Depot API is a thin HTTP service that is part of the Depot component - not a separate service.
It is deployed co-located with (or in front of) the S3 backend.

Responsibilities:

- **Authentication**: Reject requests from unauthenticated (guest) users before issuing any
  pre-signed URL (see [Authentication and Rate Limiting](#authentication-and-rate-limiting)).
- **Pre-signed URL issuance**: Generate time-limited S3 pre-signed upload URLs for authenticated
  requests.
- **Rate limiting**: Enforce per-IP upload rate limits.
- **Size enforcement**: Reject requests that exceed the configurable maximum file size per upload.

The Depot API does **not** proxy file downloads. Downloads are served directly from the S3 backend
via public object URLs.

## Download Model

Downloads are public. Anyone with the file URL can fetch the file directly from the S3 backend. The
URL is the access control - if you don't want someone to access a file, don't share the URL. This
is the same model as Imgur, public S3 buckets, or paste services.

## Authentication and Rate Limiting

Uploads require authentication. The Depot API rejects requests from unauthenticated (guest) users.
Uploads are rate-limited per IP.

The following are deferred to post-MVP:

- Per-user upload quotas.
- IRC-account-tied authentication (binding upload access to a verified NickServ/SASL account via
  `account-tag`).
- File deletion by uploaders.

## File Immutability

Files cannot be deleted by uploaders. Once a file is uploaded it is immutable from the client's
perspective. Server operators can purge files via S3 admin tools (e.g., the MinIO admin console or
AWS S3 management interface). There is no client-facing delete API in the MVP.

## File Metadata Tags

When a file is shared in a channel, the client attaches the following tags to the `PRIVMSG`:

| Tag                 | Content                                      |
|---------------------|----------------------------------------------|
| `+orbit/file-name`  | Original filename (e.g., `screenshot.png`)   |
| `+orbit/file-size`  | File size in bytes                           |
| `+orbit/file-type`  | MIME type (e.g., `image/png`)                |

These tags are defined in the [Orbit Tag Namespace](01-ground-control/02-tags/01-namespace.md).

**These are client-asserted metadata.** The tags are set by the sending client and can contain any
value - they are not validated by the IRC server. Orbit clients SHOULD verify file metadata
independently by checking HTTP response headers on download, rather than trusting the tags blindly.
For the full verification rules governing client-asserted tags, see
[Tag Integrity and Trust Model](01-ground-control/02-tags/02-trust-model.md).

## Service Discovery

Orbit clients discover a domain's Depot instance via a `_depot._tcp` DNS SRV record. For record
format and the full client resolution algorithm, see
[DNS & Service Discovery](../05-infrastructure/01-domain-discovery.md).
