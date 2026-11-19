# Scope and milestone roadmap

Status: **Proposed** (first assignment, awaiting approval). Each milestone ends with a review gate. Work on the next milestone does not start until you approve the current one.

## 1. What is built now and what is postponed

**This first assignment (done):** documentation only. That means the version check, risks, architecture, AI design, roadmap, proposed ADRs, CLAUDE.md and a placeholder README. No application code.

**After approval, Milestone 1 only:** `@mca/shared`, `@mca/backend` (config, logging, health, event bus/store, gateway), `@mca/dashboard` (live connection, labelled empty panels, system panel), Spike A, Spike B, Windows start/stop scripts, README setup, the first ADRs marked Accepted.

**Explicitly postponed** (each to its own milestone; no stubs that pretend to work):

| Postponed item | Milestone |
|---|---|
| Agent process, Mineflayer adapter, primitive skills, reflexes, watchdog, reward-signal logging | M2 |
| ModelProvider, InferenceQueue, structured output pipeline, planner, critic, curriculum, decision records, model benchmark | M3 |
| Agent DB, memory stream, retrieval, reflection, facts, skill library, composite skills | M4 |
| Survival behaviours (shelter, food loop, night, combat, death recovery) | M5 |
| Python workspace, bandit, combat micro, policy registry | M6 |
| Progression stages | M7 |
| Multiple agents, profiles, inter-agent chat | M8 |
| Social systems | M9 |
| Desktop packaging / installer | after M5, by ADR |

## 2. Milestones and acceptance criteria

All milestones share these criteria:
- `npm ci && npm run build && npm run typecheck && npm run lint && npm test` pass **on your Windows 11 PC**.
- New major systems have tests.
- CLAUDE.md and the ADRs are up to date.
- The dashboard shows only real data.

Live tests (real Ollama, real Minecraft) run on your PC with `npm run test:live`. They are kept separate from the unit and integration tests, which run anywhere.

### M1: Foundation, dashboard and two de-risking spikes

- **Repo:** npm workspaces (`shared`, `backend`, `dashboard`), strict TypeScript, Vitest, lint, a dependency-rule test, `.gitattributes` with CRLF for `.bat` and `.ps1`.
- **Config:** a zod-validated `config/app.config.json` plus `.env`. Tests cover defaults, overrides and invalid config (the app exits non-zero with a clear message).
- **Logging:** pino JSON logs written to `data/logs/`, with rotation by size and day plus a retention limit. A test uses tiny limits.
- **Health checks:** backend, SQLite (with a real sqlite-vec KNN query), Ollama (version, configured models pulled, GPU share of loaded models), Minecraft server (status ping, version must equal `1.21.11`). Each reports `ok | degraded | down` with detail. Tests run against local stub servers.
- **Dashboard:** connects over Socket.IO with snapshot plus event sequence numbers, shows its connection state and reconnects automatically. It has Status, Objective, Activity, Memory and Learning panels. Each panel visibly says "No data yet — arrives in Milestone N". The System panel shows the real health checks live: stopping Ollama or the server shows `down` within one check interval.
- **Spike A (GPU benchmark):** `npm run spike:gpu -- --model qwen3:8b`. It records the Ollama version, model, processor split (`size_vram/size` from `/api/ps`), load time, TTFT, prompt and eval tokens/s, total dedicated GPU memory (Windows performance counters) and one structured-output probe. Results go to `data/benchmarks/gpu-<timestamp>.json`. **Done when** it has been run on your PC at least twice (Minecraft closed, then Minecraft open with you connected) and a summary is committed to `docs/benchmarks/`.
- **Spike B (connection test):** `npm run spike:connect`. A bot joins your 1.21.11 server, logs where it spawns, walks at least 5 blocks (checked by position change), disconnects cleanly and exits with code 0. **Done when** it has run on your PC, you saw the bot in your client, and the result JSON shows the walk.
- **Scripts:** `start.bat`/`start.ps1` (checks prerequisites, can start the MC server, starts the backend serving the dashboard) and `stop.bat`/`stop.ps1` (graceful shutdown, no orphan `node.exe`).
- **Docs:** README setup, ADRs Accepted, CLAUDE.md current.

### M2: Minecraft connection and basic bot control

- An agent process managed by the supervisor. It reconnects with backoff after a kick, a server restart or a network drop. Death leads to respawn. Every one of these is recorded as an event.
- WorldSnapshot extraction, tested on snapshots recorded from your real game.
- Primitive skills v1 with preconditions, expected effects, timeouts, abort and evidence: `go_to`, `collect_block`, `craft_item`, `place_block`, `equip`, `eat`, `attack_entity`, `flee`, `explore`.
- Reflexes v1 (lava/fire, drowning, flee, fight back, eat). Unit-tested on snapshots, and live-tested with scenarios set up over RCON.
- A watchdog for stuck states. Raw reward-signal logging starts. Chat is perceived and recorded.
- Developer controls in the dashboard, clearly labelled *manual*, to run a skill. They go through the **same validation path** the model will use later.
- **Acceptance:** live tests on the *test world* pass:
  - collect 5 logs within 120 s
  - craft a wooden pickaxe from logs
  - survive a summoned zombie through reflexes
  - recover automatically from `/kill`, `/kick` and a server restart
- The dashboard shows live health, food, position, inventory and the real event feed.

### M3: Local model, benchmark and autonomous decisions

- `@mca/inference`: Ollama provider, InferenceQueue (priority, aging, fairness, deadlines, cancellation, metrics), structured output with repair and fallback.
- `@mca/brain`: context builder, planner, deterministic verifier, model critic, curriculum, decision records, versioned prompt templates (Voyager prompts adapted and credited).
- **Model benchmark:** `qwen3.5:9b`, `gemma4:12b` (QAT and standard), `gpt-oss:20b`, `Sweaterdog/Andy-4`, with `qwen3:8b` as a reference. It runs an offline scenario set (recorded real snapshots, each with a set of acceptable actions) and short live episodes, with Minecraft running. Metrics: valid-action rate, decision quality, TTFT and total latency, tokens/s, VRAM, plus `think` on and off where it applies. The result is an ADR choosing the default model.
- **Acceptance:**
  - Given "collect 10 logs and craft a wooden pickaxe", the agent succeeds autonomously in at least 8 of 10 trials on the test world within the time limit.
  - The fuzz test confirms zero crashes from malformed model output.
  - Every executed action has a validation record.
  - The dashboard shows real decision records (goal, stated reason, plan, expected outcome, actual outcome, verdict).

### M4: Persistent memory and skill library

- Agent DB with migrations and automatic backups. Memory stream with embeddings, retrieval scoring, reflection, facts (locations, deaths, lessons), skill library with composite skills and success statistics.
- **Acceptance:**
  - Memory survives a forced kill and a restart (integrity check passes).
  - Retrieval scoring has unit tests with known expected rankings.
  - After a restart the agent uses a remembered fact (for example it returns to a chest it saw before).
  - A composite skill is saved only after verified success, reused in a later goal, and its success rate updates.
  - A migration creates a backup first.
  - The dashboard's Memory and Learning panels show real records, labelled *memory* and *skill library*, not *training*.

### M5: Autonomous survival

- Night and shelter, a food loop, combat with gear, tool upkeep, inventory management, death recovery, exploration with location memory.
- **Acceptance:** on a fresh normal-difficulty survival world, the agent survives 3 in-game days with no input in at least 4 of 5 runs. No stuck state lasts longer than the watchdog limit without being resolved. Metrics are recorded per run.

### M6: Advanced learning and narrow RL

- An options analysis comes first (bandit, combat micro, sim vs real speed-up). Then a Python workspace on the newest stable Python, a dataset export, and a contextual bandit for skill selection with offline evaluation and then online use. Optionally, combat micro.
- **Acceptance:** the learned policy beats the baseline skill-choice success rate on held-out situations, with a statistically meaningful margin. Policies are versioned and can be rolled back. The UI clearly separates policy training from memory and skills.

### M7: Progression in stages (each a separate checkpoint)

1. Iron tools
2. Diamonds
3. Enter the Nether
4. Ender pearls and blaze rods
5. Locate a stronghold
6. Defeat the Ender Dragon

- **Acceptance per stage:** from a defined start state in a fresh world, the agent reaches the stage autonomously in k of n runs (k and n agreed at the stage). Time is logged. The curriculum chose the steps, with no scripted progression. Each stage gets its own review.

### M8: Multiple independent AI players

- Profiles (identity, personality), per-agent processes and DBs, fair queue scheduling, inter-agent chat perception.
- **Acceptance:** 5 agents run concurrently for 1 hour on one PC, each pursuing its own goals. There is no memory cross-contamination (tested). P1 inference latency stays under the agreed limit, and the server keeps acceptable TPS.

### M9: AI civilization and social interaction

- Communication protocol, relationship memory, trading, alliances, competition and conflict that emerge rather than follow a script. Dashboard analysis views.
- Acceptance criteria to be agreed at the M8 review.

## 3. Decisions for you (needed before or during M1)

| # | Decision | Recommendation |
|---|---|---|
| D1 | Minecraft version. Mineflayer now supports 26.1, but 26.x needs Java 25. | **Stay on 1.21.11** (ADR 0002). |
| D2 | Node 24 (installed, Maintenance LTS until 2028-04) or Node 26 (Active LTS). | **Stay on Node 24** for now. Moving later is cheap because better-sqlite3 is now N-API. |
| D3 | TypeScript 7 with oxlint, or TypeScript 6 with typescript-eslint. | **TS 7 + oxlint**, with TS 6 as the recorded fallback (ADR 0003). |
| D4 | One process per agent from M2. | **Yes** (ADR 0004). |
| D5 | Enable **RCON** on your server (localhost only, password in `.env`) for test setup and a clean stop. | **Yes**, needed from M2. Optional in M1. |
| D6 | A **separate test server and world** (for example `C:\MinecraftAI\test-server`, flat or seeded, port 25566) for automated live tests, so tests never touch your main world. | **Yes**, from M2. |
| D7 | Composite skills as validated JSON programs instead of model-written JavaScript (a change from Voyager). | **Yes** (ADR 0007). |
| D8 | Project licence for this repo. Needed if code is copied from MIT projects, since their notices must be kept either way. | Your choice (MIT would match the sources). |
