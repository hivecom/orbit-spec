# Research: Server Discovery and Directory

## Problem

If Orbit is decentralized (anyone can run a server), how do users find communities? Discord solves this with a centralized server directory and invite links. Orbit needs something equivalent.

## Proposal

A public directory service where server administrators can opt in to list their community. Searchable by name, description, tags, and approximate member count. Provides invite links or connection details.

## Approach

Start simple:

- A section on [hivecom.net](https://hivecom.net) backed by a database of server listings
- Server admins submit their listing via a web form or API call
- Listings include: server name, description, tags/categories, member count (self-reported or queried), connection URI, banner image
- Basic moderation: Hivecom team reviews listings to prevent abuse

As the directory matures, the following gaps must be addressed:

- **Listing verification**: The directory should require DNS-based proof of domain ownership to prevent fake or impersonation listings. This follows the same approach as existing web verification schemes - an operator proves they control the domain by publishing a DNS TXT record or serving a well-known token. Unverified listings should be visually distinguished from verified ones.
- **Automated health checks**: Listings should include an automated reachability check confirming the listed server is actually connectable (Ground Control IRC endpoint, optional Satellite endpoint). Stale or offline listings degrade the directory's usefulness and trust. Health check frequency and failure policy (hide vs. flag vs. remove) needs a defined policy.
- **Client-side integration**: Determine whether the directory is web-only (browser to hivecom.net) or also integrated as an in-app directory browser in the Orbit desktop and mobile clients. An in-app browser lowers the friction for discovering and joining communities but requires API stability and adds a surface to maintain. Both models should be prototyped before committing.

Evaluate decentralized alternatives later:

- DNS TXT records advertising Orbit server metadata (similar to how email uses MX records)
- A shared, replicated registry that multiple directory instances can sync
- Client-side crawling of known servers

## Risks

- A centralized directory run by Hivecom is a single point of control, which contradicts the decentralization ethos. But it's the pragmatic starting point - decentralized discovery is a hard problem and not worth solving before there are servers to discover.
- Content moderation of the directory itself is necessary. Malicious, illegal, or abusive communities should not be promoted. This requires policy decisions and human review.
- Privacy: some communities explicitly do not want to be discoverable. The directory must be strictly opt-in.

## Evaluation Criteria

Launch a minimal directory on hivecom.net once multiple independent Orbit communities exist. Track listing submissions and user traffic. If the centralized model works, keep it. If the community demands decentralization, revisit.

## Dependencies

Multiple Orbit communities must exist to populate the directory. This depends on the MVP being easy to deploy and operate.
