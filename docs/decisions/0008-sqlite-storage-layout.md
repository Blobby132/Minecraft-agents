# 0008. SQLite storage layout, migrations, backups, embeddings

- Status: Proposed
- Date: 2026-11-19
- Milestone: M1 (runtime DB), M4 (agent DB)

## Context

Persistent data must survive crashes and restarts. Each agent's memory must be isolated. better-sqlite3 13.2.0 ships N-API prebuilds (including win32-x64), and sqlite-vec 0.1.9 ships a windows-x64 build. Both were verified working together on Linux, including a real `vec0` KNN query. sqlite-vec needs `rowid` as BigInt.

## Decision

- `data/runtime/runtime.db` for the orchestrator. `data/agents/<agent-id>/agent.db` for each agent.
- WAL mode, foreign keys on, busy timeout, transactions for multi-row writes.
- Numbered SQL migrations per DB kind, tracked with `PRAGMA user_version`. **Automatic backup** (`db.backup`) before migrating. One transaction per migration. `quick_check` afterwards.
- Write before you act: a decision row is written before its action runs. Rows still `executing` at startup become `interrupted`.
- Vectors live in sqlite-vec `vec0` tables. Each embedding records its model ID and dimension. Changing the embedding model requires an explicit re-embedding migration.
- Fallback if sqlite-vec fails on Windows: brute-force cosine in TypeScript over BLOBs (adequate below ~10⁵ vectors).

## Consequences

- Simple, local, inspectable files. Backup is a copy. No database server.

## Alternatives considered

- One shared DB for all agents: weaker isolation and write contention in M8.
- A separate vector DB server: unnecessary at this scale.
