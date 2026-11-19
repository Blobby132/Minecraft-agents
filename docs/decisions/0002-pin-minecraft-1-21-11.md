# 0002. Pin Minecraft to 1.21.11

- Status: Proposed
- Date: 2026-11-19
- Milestone: M1

## Context

The brief pins the server, client and bot to 1.21.11 because "the newest Mineflayer supports up to 1.21.11". Checked on 2026-11-19:

- Mineflayer 4.39.0 `testedVersions` now ends with `1.21.11`, `26.1`. Support for 26.1 was added in 4.38.0 (2026-08-27).
- The latest Minecraft release is 26.3. Mineflayer does not support 26.2 or 26.3.
- Minecraft 26.1+ requires **Java 25**. 1.21.11 requires Java 21, which is installed.
- The 26.1 support is recent, and 4.38 and 4.39 still contain fixes for flaky 26.1 tests. The plugins are old and well-proven on 1.21.x.

## Decision

Keep server, client, `mineflayer` and `minecraft-data` pinned to **1.21.11**. The health check rejects any other server version. We will reconsider only at a milestone boundary, with a concrete need, through a new ADR.

## Consequences

- No Java 25 install, no world migration, no new client profile.
- We stay behind the latest game version. None of the planned milestones need post-1.21.11 content.

## Alternatives considered

- Move to 26.1: Mineflayer officially supports it now. But it adds a Java 25 requirement, a world that cannot be downgraded, newer and less proven bot support, and it would still be two releases behind the latest.
