# CLAUDE.md: guide for future sessions

Autonomous Minecraft learning AI. A standalone, local-only Windows 11 app that runs AI players in Minecraft Java Edition 1.21.11. You give a high-level goal; the AI plans, acts through validated skills, checks results against game state and adapts. Long-term: 5–10 independent agents in one world (not built yet).

**Current phase:** first assignment delivered (design docs + proposed ADRs). **Waiting for the owner's approval before Milestone 1.** No application code exists yet.

## Hard rules (from the project brief; never break them)

1. Clean, new repo. Do not reuse, copy from or modify the owner's other projects (Jarvis / Mark-LIV, Youtube_help / Shorts Creator, etc.).
2. Everything runs locally. No paid cloud APIs or subscriptions at runtime.
3. **No fake data, no invented "thoughts", no mocked results** in the product. The dashboard shows only real recorded events and decision records. Empty panels are labelled empty. Test fixtures live only in `test/` directories.
4. **Memory ≠ skill acquisition ≠ training.** Saving memories or skills is not training model weights. Never describe it that way (ADR 0009).
5. Never hardcode the gameplay progression. Game *facts* come from minecraft-data. *What to do next* comes from the curriculum/planner.
6. Validate every model-proposed action against a schema **and** the current game state before executing it (ADR 0005).
7. Handle errors, crashes, disconnects, deaths and stuck states. Persistent data must survive restarts and crashes.
8. Maintainable code over shortcuts. Do not build features before their milestone.
9. On a technical limitation: stop and explain it with alternatives. Never silently work around it.
10. Record important decisions as ADRs in `docs/decisions/` (template `0000-template.md`).
11. Write tests for every major system. Python arrives only in M6 (newest stable version only).
12. Keep this file current.
13. Milestones are gated: finish, get the owner's review, then start the next one.

## Pinned decisions (see ADRs, research note `docs/research/2026-11-version-verification.md`)

- Minecraft **1.21.11** for server, client and bot (ADR 0002). The server is at `C:\MinecraftAI\server`, `online-mode=false`, `server-ip=127.0.0.1`. Java 21.
- Node.js 24 LTS · TypeScript 7 + oxlint (ADR 0003) · npm workspaces · zod everywhere · pino logs · Vitest.
- React + Vite dashboard · Socket.IO · SQLite with better-sqlite3 13 (N-API prebuilds) + sqlite-vec (`rowid` must be BigInt).
- Mineflayer 4.39 + pathfinder / collectblock / pvp / tool, exact pins, behind an adapter (ADR 0010).
- Ollama on the local HTTP API (`127.0.0.1:11434`). The GPU path on the RX 9060 XT is Vulkan. **Never assume GPU acceleration. Check `/api/ps` `size_vram`.**
- All model calls go through `ModelProvider` + one priority `InferenceQueue` (ADR 0006). Structured JSON output via Ollama `format`. Malformed replies → repair → fallback, never a crash.
- No agent framework. One base model for all agents. Identity comes from per-agent data and prompts.

## Architecture in one screen (details: `docs/architecture.md`, `docs/ai-design.md`)

- **Orchestrator process** `@mca/backend`: config, logging, health, event bus/store, Socket.IO gateway, agent supervisor, InferenceQueue.
- **Agent process per agent** `@mca/agent` (M2+): Mineflayer bot, reflexes, executor, brain, memory DB. Talks to the orchestrator through zod-validated IPC (ADR 0004).
- **Four layers:** reflexes (no model) → skills (tested TS, abortable) → planner (model returns a `Decision` JSON) → critic (deterministic verifier first, model critic only when needed).
- **Packages and allowed dependencies:** `shared` (contracts) ← everyone. `minecraft` must not import brain/memory/inference. `brain` never imports Mineflayer (it sees `WorldSnapshot` + `SkillCatalog`). `dashboard` imports only `shared`. `backend` talks to agents only through IPC.
- **Storage:** `data/runtime/runtime.db`, `data/agents/<id>/agent.db` (+ `backups/`), `data/logs/`, `data/benchmarks/`. WAL mode, numbered migrations, backup before migrating, write-before-act decision records (ADR 0008).
- **Composite skills** are validated JSON programs over existing skills, saved only after verified in-game success. No model-written code is ever executed (ADR 0007).
- **Credits required** in the README and file headers: Voyager (MIT), Generative Agents (Park et al. 2023), Mindcraft (MIT; keep its notice on copied code).

## Repository layout

```
packages/{shared,backend,dashboard}   M1     packages/{minecraft,agent}   M2
packages/{inference,brain}            M3     packages/memory              M4
packages/learning                     M2/3 logging, M6 policies          python/   M6
spikes/   M1 throwaway                scripts/windows/   start/stop      docs/   design, ADRs, research, benchmarks
```

## Working conventions

- Develop on the assigned feature branch. Commit only when asked or at a milestone gate. Never force-push.
- Use exact versions for Minecraft-related packages and commit the lockfile.
- Unit and integration tests run anywhere. Live tests (real Ollama, real Minecraft) are opt-in via `npm run test:live` and run on the owner's PC. Automated live tests must use a separate test world, never the owner's main world.
- Everything binds to 127.0.0.1.
- Windows scripts: `.ps1` with `.bat` wrappers, CRLF line endings.

## Milestones

1. Foundation, dashboard, Spike A (GPU benchmark) + Spike B (connection test) ← **next, after approval**
2. Minecraft connection and basic bot control
3. Local model, benchmark, autonomous decisions
4. Persistent memory and skill library
5. Autonomous survival
6. Advanced learning and narrow RL
7. Progression stages (iron → diamonds → Nether → pearls/blaze rods → stronghold → dragon)
8. Multiple independent agents
9. AI civilization

Acceptance criteria: `docs/roadmap.md`.
