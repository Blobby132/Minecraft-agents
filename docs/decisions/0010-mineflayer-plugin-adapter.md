# 0010. Mineflayer plugins behind an adapter, exact pins, patch policy

- Status: Proposed
- Date: 2026-11-19
- Milestone: M2 (Spike B in M1 uses Mineflayer directly, as throwaway code)

## Context

mineflayer-pathfinder (last published 2023-09), mineflayer-pvp (2022-07) and mineflayer-tool (2022-05) are effectively unmaintained. pvp uses the deprecated `physicTick` event. Mindcraft (MIT) carries patches for pathfinder and pvp. Mineflayer 4.39 already includes the core fixes Mindcraft had patched.

## Decision

- Only `@mca/minecraft` imports Mineflayer or its plugins. Everything else uses our adapter interfaces and plain-data `WorldSnapshot`.
- Pin exact versions for `mineflayer`, `minecraft-data`, `minecraft-protocol` and every plugin. Commit the lockfile.
- A local fix uses `patch-package`, and only with an automated test that reproduces the bug and an ADR entry. Code copied from Mindcraft keeps its MIT notice.
- Plan to replace mineflayer-pvp with our own combat module by M5–M6.

## Consequences

- Plugins can be replaced without touching the brain or skills' public contracts.

## Alternatives considered

- Forking the plugins now: premature. We fork only if patches grow large.
