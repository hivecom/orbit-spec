# Research: Prioritization Matrix, Process, and Revision History

**Status:** Living document - updated as tracks are evaluated and promoted.

This page consolidates the prioritization matrix, the research lifecycle process, and the revision history for the Orbit R&D roadmap.

## Prioritization Matrix

The following table summarizes each track's expected impact, implementation feasibility, risk level, and suggested timeline. "Post-MVP" indicates work that could begin once the MVP ships. "R&D" indicates longer-term exploration without a committed timeline.

| Track | Impact | Feasibility | Risk | Suggested Phase |
|-------|--------|-------------|------|-----------------|
| [Bot & Integration API](09-bot-api.md) - Phase 0 (IRC docs + examples) | High | High | Low | MVP (day-one) |
| [Bot & Integration API](09-bot-api.md) - Phase 1+ (webhook bridge, REST API) | High | High | Low | Post-MVP (immediate) |
| Bot & Integration API - Orbit Extension API (orbit-app plugins) | High | Medium | Low | Post-MVP (Q2) |
| [Mobile Clients](07-mobile-clients.md) (Tauri Mobile) | High | Medium | Medium | Post-MVP (Q2) |
| [Linux Gaming Overlay - Tier 1](03-linux-overlay.md) (Layer-Shell / X11) | Medium | High | Low | Post-MVP (Q2–Q3) |
| [Federation](05-federation.md) - Phase 0 (Transponder) | High | High | Low | Post-MVP (immediate) |
| [Federation](05-federation.md) - Phase 1–2 (IRC linking, cross-org) | Medium | Medium | Medium | Post-MVP (Q3+) |
| [Server Discovery](10-server-discovery.md) Directory | Medium | High | Low | Post-MVP (Q3) |
| [E2E Encryption](06-e2e-encryption.md) (DMs) | Medium | Medium | High | Post-MVP (Q3–Q4) |
| [Media over QUIC (Iroh)](01-moq-iroh.md) | High | Low | High | R&D (ongoing) |
| [Leptos / WASM Frontend](02-leptos-wasm.md) | Low | Medium | Medium | R&D (evaluate after MVP) |
| [Linux Gaming Overlay - Tier 2](04-vulkan-overlay.md) (Vulkan) | High | Medium | Medium | Post-MVP (Q3–Q4) |

### Reading This Table

- **Impact** reflects how much the feature would matter to users if it shipped successfully.
- **Feasibility** reflects how realistic it is to build with current team size and technology maturity.
- **Risk** reflects the likelihood of the effort failing or producing unusable results.
- Items with High Impact but Low Feasibility and High Risk ([Media over QUIC](01-moq-iroh.md)) are worth researching but should not be depended on. They are bets, not plans.
- The [Vulkan Overlay (Tier 2)](04-vulkan-overlay.md) has been rescoped to speaker indicator + webcam pip only - comparable rendering complexity to MangoHud - which makes it a realistic Post-MVP deliverable rather than a long-term R&D bet.
- Items with High Feasibility and Low Risk ([Bot API](09-bot-api.md), [Server Discovery](10-server-discovery.md), [Transponder / Federation Phase 0](05-federation.md)) should be prioritized early because they deliver value cheaply. Bot API Phase 0 (IRC docs and example bots) ships day-one with the MVP at zero engineering cost - Phase 1+ (webhook bridge, REST API) follows immediately post-MVP.
- Federation is now split: Phase 0 (Transponder - identity bridging via signed assertions) is high-feasibility foundational work that improves single-server Satellite auth immediately. It is also optional - deployments without it degrade gracefully. Phases 1–2 (IRC linking and cross-org federation) carry the traditional federation risks and are deferred until Phase 0 is proven.

## Process

Each research track follows the same lifecycle:

1. **Research**: Gather information, read specs, study prior art. Write up findings.
2. **Prototype**: Build the smallest possible thing that tests the core hypothesis. Time-box this - if a prototype takes longer than 4 weeks for a single developer, the scope is too large.
3. **Evaluate**: Measure the prototype against the evaluation criteria defined in the track's spec page. Be honest about results. "It kind of works" is not good enough to justify shipping.
4. **Decide**: Based on evaluation, either promote the track to a full specification (with a proper design spec and implementation plan), defer it (revisit later when conditions change), or abandon it (document why, so future contributors don't repeat the work).
5. **Spec**: If promoted, write a dedicated specification covering architecture, implementation plan, testing strategy, and rollout.

Abandoned tracks are not failures. They are information. Document what was tried, what was learned, and why it didn't work out. This is more valuable than the prototype code itself.

## Revision History

| Date | Change |
|------|--------|
| 2025-06 | Initial draft. All tracks at Research stage. |
| 2025-07 | Expanded Federation track: added Transponder (identity bridging via signed assertions), verified/unverified user model, graceful degradation, phased approach, federation trust chain. Transponder is a standalone, optional component - zero IRC server modifications required. |
