# Orbit Design Specification

Orbit is a decentralized, open-source communication platform. It aims to be a viable alternative to platforms like Discord, prioritizing openness, user autonomy, and modern real-time communication standards.

## About This Repository

This repository houses the design specifications that guide the development of Orbit. Each document captures architectural decisions, technical choices, and product scope for different stages of the project.

## Documents

| Spec | Title | Description |
| ---- | ----- | ----------- |
| [0001](specs/0001-mvp-spec.md) | MVP Specification | The focused minimum viable product. Covers the core stack: Ergochat (IRCv3) for text and signaling, LiveKit (WebRTC) for voice and video, a Tauri v2 desktop client, an anonymous web widget, and custom URL deep linking. |
| [0002](specs/0002-research-roadmap.md) | Research & Development Roadmap | Longer-term experimental tracks including Media over QUIC (Iroh), Leptos/WASM frontend, Vulkan gaming overlays on Linux, federation, mobile clients, E2E encryption, and bot APIs. |

## Status

These are **living documents** and are subject to revision as the project evolves. Designs may change significantly based on prototyping results, community feedback, and shifting priorities.
