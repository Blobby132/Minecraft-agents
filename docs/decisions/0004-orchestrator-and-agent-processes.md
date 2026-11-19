# 0004. Orchestrator plus one process per agent, typed IPC

- Status: Proposed
- Date: 2026-11-19
- Milestone: M2 (decided now because it shapes the M1 contracts)

## Context

Each Mineflayer bot runs 20 Hz physics on the Node event loop, and pathfinding can block that loop. Protocol or plugin errors can throw uncaught exceptions. The final vision is 5–10 agents. The dashboard, health checks and the shared InferenceQueue must stay responsive when a bot misbehaves.

## Decision

- One **orchestrator** process (`@mca/backend`): gateway, health, config, runtime DB, InferenceQueue, agent supervisor.
- One **agent process per agent** (`@mca/agent`, started with `child_process.fork`). It owns the Mineflayer bot, reflexes, brain and its own agent DB (the single writer).
- They communicate through versioned, zod-validated message envelopes over the IPC channel. Requests are correlated by ID and have timeouts. The same protocol runs over an in-memory transport for tests.
- The supervisor restarts crashed agents with exponential backoff and records each restart.

## Consequences

- Failures stay contained, and the agent layout is already ready for multiple agents.
- Debugging spans two processes. Logs and events carry `agentId` and correlation IDs to compensate.

## Alternatives considered

- Everything in one process: simpler, but one bot can stall or crash everything.
- `worker_threads`: shares the process, so a native crash kills everything. Mineflayer's dependencies are not designed for worker threads.
