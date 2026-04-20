# Server Discovery Directory

## Overview

The Orbit project operates a public server directory at **`orbit.directory`** - a dedicated domain for discovering Orbit communities. Any server operator can list their community through a self-service registration flow that verifies domain ownership. The Orbit desktop and web clients include a built-in **"Browse Communities"** feature that queries `orbit.directory` to present a browsable, searchable list of public Orbit communities.

The directory is an Orbit project resource. It is not tied to any specific community or server operator.

## Registration Flow

Listing a server on `orbit.directory` is self-service:

1. **Submit**: The operator submits their domain to `orbit.directory` (via web form or API).
2. **Verify**: The directory service fetches `/.well-known/orbit/services.json` from the submitted domain. This serves two purposes:
   - **Domain ownership proof** - the operator demonstrates control of the domain by serving the well-known file, similar to how Let's Encrypt validates ownership via HTTP-01 challenges.
   - **Service discovery** - the directory reads the file to determine which services (IRC, Satellite, Depot) the server offers. See [Domain Discovery](../05-infrastructure/01-domain-discovery.md) for the `services.json` schema.
3. **Live**: Once verified, the listing goes live on `orbit.directory` and becomes visible in client "Browse Communities" views.

Operators can update or delist their server at any time.

## Listing Content

Each directory listing includes:

- **Domain** - the server's domain name
- **Community name** - human-readable display name
- **Description** - short summary of the community
- **User count** - approximate member count
- **Available services** - which Orbit services the server runs (IRC, Satellite, Depot), populated from `services.json`
- **Region / language tags** - operator-provided metadata for filtering and search

## Client Integration

The Orbit desktop and web clients ship with a built-in **"Browse Communities"** feature:

- Queries the `orbit.directory` API to retrieve and display public listings
- Supports search by name, description, tags, region, and language
- Allows users to connect to a listed community directly from the browser view

## Risks

- **Spam and abuse of self-service listings**: Because registration is open to any operator, the directory could attract spam or malicious listings. Domain verification is the primary mitigation - proving ownership of a real domain prevents throwaway or disposable listings. Additionally, a report mechanism allows users and operators to flag abusive or rule-violating listings for review and removal.
- **Content moderation**: Communities hosting illegal or harmful content should not be promoted. The directory will enforce a listing policy and act on reports.
- **Privacy**: The directory is strictly opt-in. Communities that do not register are not listed and are not affected in any way. The directory is a convenience layer, not a dependency - communities function normally without being listed.

## Other Orbit Project Domains

The Orbit project has additional domains available (e.g., for downloads, project homepage) that may be used in the future. These are out of scope for this specification.
