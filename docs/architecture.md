# Architecture

Status: **Proposed** (first assignment, awaiting approval). Related: [ai-design.md](ai-design.md), [roadmap.md](roadmap.md), [risks.md](risks.md), ADRs in [decisions/](decisions/).

## 1. Goals that shape the architecture

1. **Real-time and interruptible.** Minecraft changes every 50 ms. A model call takes seconds. Nothing that keeps the bot alive may wait on a model.
2. **One GPU, many agents later.** All model traffic goes through one scheduler, which is shared by every agent.
3. **Clear module boundaries.** The modules are Minecraft control, AI reasoning, memory, learning, inference and the dashboard. Each has its own package and talks to the others only through typed contracts.
4. **Survives failure.** If a bot crashes, the backend and dashboard stay up. If the backend crashes, no persistent data is lost. Every long-running operation has a timeout and can be cancelled.
5. **Honest observability.** The dashboard only shows events and decision records that were actually recorded. Panels with no data say so.
6. **Ready for 5–10 agents without building it now.** Every agent-owned item already carries an agent ID: its process, database, identity, event stream and inference requests.

## 2. System diagram

```
┌──────────────────────────────────────── Windows 11 PC (everything local) ────────────────────────────────────────┐
│                                                                                                                   │
│  Browser ── Dashboard SPA (React + Vite) ◄──── Socket.IO + HTTP, bound to 127.0.0.1 only ───┐                     │
│                                                                                              │                     │
│  ┌──────────────────────────────── Orchestrator process  (@mca/backend) ────────────────────┴──────────────────┐  │
│  │  Gateway ─── HTTP API (/api/health, /api/shutdown, static dashboard) + Socket.IO (snapshot + event stream) │  │
│  │  Config (zod-validated file + .env) · Logging (pino → rotating files) · Health service (periodic checks)   │  │
│  │  Event bus ──► Event store (data/runtime/runtime.db)                                                       │  │
│  │  Agent supervisor: spawn / health-ping / restart with backoff / graceful stop                    [M2]      │  │
│  │  InferenceQueue (priority, aging, per-agent fairness, deadlines, cancel, metrics) ─► ModelProvider  [M3] ──┼──┼──► Ollama :11434
│  └──────────────────────────────────────────▲──────────────────────────────────────────────────────────────────┘  │     (GPU via Vulkan,
│                                             │ typed IPC messages (child_process IPC, zod-validated both ends)  │      verified in Spike A)
│  ┌─────────────────────────── Agent process, one per agent  (@mca/agent)  [M2+] ──────────────────────────────┐  │
│  │  Brain      planner · critic · curriculum · decision records · prompt templates            (@mca/brain)    │  │
│  │  Memory     memory stream · retrieval · reflection · facts · skill library     (@mca/memory) ──────────────┼──┼──► data/agents/<id>/agent.db
│  │  Learning   reward signal logger [M2/3] · policies (bandit etc.) [M6]          (@mca/learning)             │  │
│  │  Executor   skill runner · motor lock · watchdog                                                            │  │
│  │  Reflexes   deterministic, no model call                                                                     │  │
│  │  Minecraft  Mineflayer + plugins behind an adapter · WorldSnapshot · primitive skills  (@mca/minecraft) ───┼──┼──► MC server 1.21.11
│  └────────────────────────────────────────────────────────────────────────────────────────────────────────────┘  │     127.0.0.1:25565
│                                                                                                                   │
│  You watch with a normal Minecraft 1.21.11 client connected to localhost.                                        │
└───────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

Milestone tags such as `[M2]` mark when each part is first built. In Milestone 1 only the orchestrator's gateway, config, logging, health, event bus and event store exist, plus the dashboard.

## 3. Process model

| Process | Count | Owns | Why it is separate |
|---|---|---|---|
| Orchestrator (`@mca/backend`) | 1 | Gateway, health, config, runtime DB, InferenceQueue, agent supervisor | It must outlive any bot crash. The dashboard keeps working when a bot dies. |
| Agent (`@mca/agent`) | 1 per agent (1 until M8) | One Mineflayer bot and that agent's brain, memory DB and reflexes | Each Mineflayer bot runs its physics at 20 Hz on the event loop, and pathfinding can block that loop. A separate process stops one bot from stalling another or the gateway. An uncaught protocol error kills only that agent, and the supervisor restarts it. Each agent DB has exactly one writer. |
| Ollama | 1 (external) | Model weights, GPU | Managed by the user. We only check it and call it over HTTP. |
| Minecraft server | 1 (external) | The world | Optionally started and stopped by our scripts (see §8). |

The agent↔orchestrator protocol is a set of zod schemas in `@mca/shared`. For tests, the same protocol runs over an in-memory transport, so an agent can be tested without spawning a process. ADR 0004.

## 4. Module boundaries (packages)

Each package states what it may import. A dependency-rule test in the test suite fails if a forbidden import appears.

| Package | Responsibility | May depend on | Must NOT depend on | Built in |
|---|---|---|---|---|
| `@mca/shared` | Contracts: zod schemas and types for config, events, IPC messages, decisions, actions, WorldSnapshot, Socket.IO payloads | zod | anything else | M1 |
| `@mca/backend` | Orchestrator: config, logging, health, event bus/store, gateway, supervisor, hosts the InferenceQueue | shared, inference (M3) | minecraft, brain, memory (it talks to agents only through IPC) | M1 |
| `@mca/dashboard` | React SPA. Read-only views, plus explicit developer controls added in M2 | shared, socket.io-client, react | any backend package | M1 |
| `@mca/minecraft` | Mineflayer connection, plugin adapter, WorldSnapshot extraction, primitive skills, reflex rules | shared, mineflayer + plugins | brain, memory, inference | M2 |
| `@mca/inference` | `ModelProvider` interface, Ollama provider, InferenceQueue, structured-output pipeline (schema → call → parse → validate → repair → fallback) | shared | minecraft, brain, memory | M3 |
| `@mca/brain` | Planner, critic, curriculum, context builder, decision records, prompt templates | shared, plus *interfaces* for skills, memory and inference (injected) | mineflayer (it sees only WorldSnapshot and SkillCatalog) | M3 |
| `@mca/memory` | Agent DB schema and migrations, memory stream, retrieval, reflection, semantic facts, skill-library storage | shared, better-sqlite3, sqlite-vec | minecraft, brain | M4 |
| `@mca/learning` | Raw reward-signal logging (from M2/M3), then policies (bandit and so on), policy registry, offline evaluation | shared | minecraft, brain | M2/M3 logging, M6 learning |
| `@mca/agent` | Composition root for one agent process: wires minecraft + brain + memory + learning and talks IPC to the orchestrator | all of the above | backend | M2 |
| `python/` (not an npm package) | RL training and offline evaluation, newest stable Python | reads exported datasets | — | M6 |
| `spikes/*` | Throwaway de-risking scripts, clearly labelled | — | — | M1 |

Why the brain never imports Mineflayer: the brain reasons over a `WorldSnapshot` (plain data) and a `SkillCatalog` (names, descriptions, parameter schemas, preconditions). It produces a `Decision`. Only the executor in the agent process calls skills, and only after validation. So the model never moves the bot directly, and the brain is fully testable offline with snapshots recorded from the real game.

## 5. How modules communicate

| From → To | Mechanism | Content |
|---|---|---|
| Inside one process | Typed in-process event bus (`EventBus<EventMap>`) plus direct calls through injected interfaces | Domain events, service calls |
| Agent ↔ Orchestrator | `child_process.fork` IPC channel. Every message is a zod-validated envelope `{v, type, agentId, seq, ts, payload}`. Request/response pairs are correlated by `requestId`, with timeouts. | Agent → orch: events, decision records, health, `inference.request`. Orch → agent: `objective.set`, `command.run` (developer control), `inference.result`, `shutdown`. |
| Agent → Inference | Through the orchestrator's InferenceQueue only, never directly to Ollama | `{priority, deadline, taskType, schema, prompt, model role}` → `{status, value \| error, metrics}` |
| Orchestrator → Dashboard | Socket.IO. On connect the client asks for a **snapshot** (ack callback). After that it receives **events** with a monotonically increasing `seq`. If the client detects a gap it asks for a new snapshot. | Health, agent status, events, decision records, queue metrics |
| Dashboard → Orchestrator | Socket.IO events with zod-validated payloads (objective set, developer commands from M2) | — |
| Orchestrator/Agent → disk | SQLite (better-sqlite3, WAL mode) and pino log files | — |

Every event type is versioned and defined once in `@mca/shared`. Socket.IO payloads reuse the same schemas, so the dashboard and backend cannot drift apart.

## 6. Data and storage

```
data/                              (git-ignored; location set in config)
  runtime/runtime.db               orchestrator: event store, health history, inference call log, sessions, agent registry
  agents/<agent-id>/agent.db       one per agent: memory stream, facts, deaths, lessons, skills, skill runs,
                                   decision records, reward signals, goal stack
  agents/<agent-id>/backups/       automatic copy (better-sqlite3 db.backup) taken before every migration
  logs/                            rotating JSON logs: backend.log, agent-<id>.log
  benchmarks/                      raw spike and benchmark output (summaries are copied to docs/benchmarks/)
```

Rules:

- SQLite runs with `journal_mode=WAL`, `foreign_keys=ON`, `busy_timeout` set, and `synchronous=NORMAL`. Decision records and memory writes use `FULL` or explicit transactions where losing them would matter.
- Migrations are numbered SQL files, one sequence per DB kind, tracked with `PRAGMA user_version`. Before migrating, the DB is backed up. The migration then runs in a single transaction and is verified afterwards.
- Write before you act: a decision record is inserted with status `executing` **before** its action runs, and updated afterwards. On restart, anything still `executing` is marked `interrupted`. So a crash never silently loses a decision.
- Every embedding row stores the embedding model ID and vector dimension. Changing the embedding model means a recorded re-embedding migration, never mixed vectors.

## 7. Cross-cutting concerns

- **Config:** schema defaults in code < `config/app.config.json` (git-ignored, copied from `config/app.config.example.json`) < `.env` / environment variables, for machine-specific paths and ports. Validated with zod at startup. Invalid config gives a precise error message and a non-zero exit.
- **Logging:** pino writes structured JSON to `data/logs/` through pino-roll, rotating daily or at a size limit with a retention count. A human-readable copy goes to the console in dev. Logs are for debugging. **The event store is the source of truth for the dashboard.**
- **Health:** periodic checks (default 15 s) plus on demand. Each check returns `{status: ok|degraded|down, detail, checkedAt, latencyMs}`:
  - Backend: uptime, event-loop delay (`monitorEventLoopDelay`), memory.
  - SQLite: opens the runtime DB, `quick_check`, loads sqlite-vec, and runs a real KNN query on a scratch table.
  - Ollama: `/api/version`, `/api/tags` (are the configured models pulled?), `/api/ps` (loaded models and **GPU share = `size_vram / size`**).
  - Minecraft server: a status ping using minecraft-protocol. Reports version name, protocol, players and latency, and checks the version against the pinned `1.21.11`.
- **Security:** everything binds to `127.0.0.1`. CORS is limited to the dashboard origin. Model output is never executed as code (ADR 0007).
- **Shutdown:** `POST /api/shutdown` (localhost only) and SIGINT/SIGTERM both start one graceful path: stop agents (disconnect bots cleanly), flush logs, close DBs. Stop scripts call the endpoint first, then fall back to `taskkill /T` after a timeout.

## 8. Windows integration

- `scripts/windows/start.ps1` (with a `start.bat` wrapper) checks Node, Ollama reachability and the Minecraft server, optionally starts the server in its own console window, then starts the backend. The backend serves the built dashboard at `http://127.0.0.1:<port>`.
- `scripts/windows/stop.ps1` / `stop.bat` shuts down the backend gracefully and optionally stops the Minecraft server. Stopping the server cleanly needs either RCON on localhost or you typing `stop` in its console. Using RCON is a decision for you (see roadmap, "Decisions for you").
- `.gitattributes` forces CRLF for `.bat` and `.ps1`.

## 9. Repository structure

```
Minecraft-agents/
├─ CLAUDE.md                     rules + architecture summary for future sessions (kept current)
├─ README.md                     setup, run, credits (Voyager, Generative Agents, Mindcraft)
├─ package.json                  npm workspaces root, scripts: build, dev, test, lint, typecheck
├─ package-lock.json             committed; exact pins for Mineflayer + plugins
├─ tsconfig.base.json            shared compiler options (strict, erasableSyntaxOnly, project references)
├─ .node-version · .editorconfig · .gitattributes · .gitignore · .env.example
├─ config/
│  └─ app.config.example.json   documented example; the real app.config.json is git-ignored
├─ docs/
│  ├─ architecture.md · ai-design.md · roadmap.md · risks.md
│  ├─ research/                 version checks and option analyses
│  ├─ benchmarks/               committed summaries of real benchmark runs
│  └─ decisions/                ADRs: NNNN-title.md
├─ packages/
│  ├─ shared/                   M1  contracts (zod + types)
│  ├─ backend/                  M1  orchestrator
│  ├─ dashboard/                M1  React + Vite SPA
│  ├─ minecraft/                M2  Mineflayer adapter, snapshot, primitive skills, reflexes
│  ├─ agent/                    M2  agent process composition root
│  ├─ inference/                M3  ModelProvider, Ollama, InferenceQueue, structured output
│  ├─ brain/                    M3  planner, critic, curriculum, decisions, prompts/
│  ├─ memory/                   M4  agent DB, memory stream, retrieval, reflection, skill library
│  └─ learning/                 M2/3 reward logging · M6 policies
├─ python/                      M6  training + offline evaluation (newest stable Python)
├─ spikes/                      M1  throwaway: gpu-benchmark/, connection-test/
├─ scripts/
│  └─ windows/                  start/stop .ps1 + .bat wrappers, setup checks
└─ data/                        runtime data (git-ignored)
```

Each package has the same internal layout: `src/`, `test/` (Vitest), `package.json`, `tsconfig.json`. Packages are created **at their milestone**, not before. Milestone 1 creates only `shared`, `backend`, `dashboard`, `spikes/` and `scripts/`.
