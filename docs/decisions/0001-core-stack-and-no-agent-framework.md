# 0001. Core stack and no agent framework

- Status: Proposed
- Date: 2026-11-19
- Milestone: M1

## Context

The brief fixes the stack: Node.js LTS + TypeScript, React + Vite, Socket.IO, SQLite (better-sqlite3 + sqlite-vec), Mineflayer and plugins, and Ollama over local HTTP. Everything runs locally with no paid APIs. The game loop is real-time and must be interruptible. The versions were checked on 2026-11-19 (see `docs/research/2026-11-version-verification.md`).

## Decision

- Runtime is Node.js 24 LTS. The `engines` field is `>=24 <27`, so Node 26 LTS is allowed later.
- npm workspaces monorepo. Packages are added at their milestone (`docs/architecture.md` §4).
- zod is the single source of schemas: config, IPC, events, Socket.IO payloads, model output. JSON Schema for Ollama is generated from the zod schemas.
- pino + pino-roll for logs, Vitest for tests.
- **No agent framework** (Mastra, LangGraph or similar). The planner/executor loop, scheduling and preemption are our own code.
- The model is called over Ollama's HTTP API with `fetch` behind our own `ModelProvider`, not the `ollama` npm client. That gives full control of timeouts, abort and streaming metrics.

## Consequences

- We write and test the loop, the scheduler and preemption ourselves. That is more code, but we can see and control all of it.
- One schema library everywhere keeps types and validation from drifting apart.

## Alternatives considered

- Agent frameworks: they assume request/response turns, not a preemptible 20 Hz loop, and they hide scheduling we need to control.
- The `ollama` npm client: fine, but a thin `fetch` wrapper gives more control over abort, stream timing and TTFT measurement.
