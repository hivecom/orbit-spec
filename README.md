# Orbit Design Specification

Orbit is a decentralized, open-source communication platform. It aims to be a viable alternative to platforms like Discord, prioritizing openness, user autonomy, and modern real-time communication standards.

## About This Repository

This repository houses the design specifications that guide the development of Orbit. Each document captures architectural decisions, technical choices, and product scope for different aspects of the project.

## Specification

The `spec/` directory contains the full set of Orbit design specifications, organized into focused pages:

| Section | Contents |
|---------|----------|
| [Architecture](spec/01-architecture/) | System overview, design philosophy, component glossary |
| [Components](spec/02-components/) | Ground Control (incl. tag namespace & trust model), Satellite, Depot, Transponder |
| [Identity & Auth](spec/03-identity/) | Authentication, permissions |
| [Clients](spec/04-clients/) | Desktop, web app, widget |
| [Infrastructure](spec/05-infrastructure/) | DNS discovery, deployment, monorepo |
| [Decisions](spec/06-decisions/) | ADRs, open questions, out-of-scope |
| [Research](spec/07-research/) | Post-MVP and R&D tracks |

Start with the [Architecture Overview](spec/01-architecture/01-overview.md) and [Component Glossary](spec/01-architecture/03-glossary.md).

## Status

These are **living documents** and are subject to revision as the project evolves. The spec is organized into focused, single-topic pages rather than monolithic documents, making it easier to update individual decisions without touching unrelated sections. Designs may change significantly based on prototyping results, community feedback, and shifting priorities.
