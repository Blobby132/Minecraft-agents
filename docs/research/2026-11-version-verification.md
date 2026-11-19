# Version and compatibility check (2026-11-19)

Status: research note for the first assignment. Re-run it before each milestone that adds a dependency.

## How this was checked

- npm packages: `npm view <pkg> version time engines dependencies` against the public npm registry on 2026-11-19, and in some cases the published tarball was unpacked and read (`npm pack`).
- Node.js: `https://nodejs.org/dist/index.json` and `nodejs/Release/schedule.json`.
- Minecraft: Mojang's `version_manifest_v2.json` and each version's own JSON, which gives the required Java version.
- Ollama: `docs.ollama.com/gpu` and the ollama.com library pages. GitHub was blocked from the sandbox this ran in, so the Ollama release version comes from a release aggregator (releases.sh, which mirrors GitHub releases). Confirm it with `ollama --version` on the target PC.
- Run in a Linux sandbox: `better-sqlite3@13.2.0` with `sqlite-vec@0.1.9`. This was a real `vec0` KNN query. It succeeded.
- Not run here, because it needs the target PC: anything that needs Windows, the RX 9060 XT, Ollama itself or the Minecraft server. Those checks are the Milestone 1 health checks and spikes.

## Summary table

| Technology | Version found | Status for this project |
|---|---|---|
| Node.js | 24.22.3 "Krypton" (installed major) · 26.13.1 "Lithium" | Both are LTS. Node 24 went to **Maintenance LTS on 2026-10-20** (EOL 2028-04-30). Node 26 became **Active LTS on 2026-10-28** (EOL 2029-04-30). Everything below supports both. Recommendation: stay on 24 for now (see decisions). |
| TypeScript | **7.0.2** (`latest`, 2026-07-08, the native Go compiler) · 6.0.3 (last JS-based) | TS 7 works for compiling and type-checking. **typescript-eslint 8.73 only supports TS < 6.1**, so ESLint type-aware linting does not work with TS 7. Proposal: TS 7 plus oxlint (whose type-aware mode is built on the TS 7 engine). Fallback: TS 6.0.3 plus typescript-eslint. See ADR 0003. |
| Mineflayer | **4.39.0** (2026-09-06), engines `node >=22` | `testedVersions` ends `… '1.21.9', '1.21.11', '26.1'`. **1.21.11 is officially tested.** The brief said the newest Mineflayer stops at 1.21.11. That is no longer true: 4.38.0 (2026-08-27) added 26.1. |
| minecraft-data / minecraft-protocol | 3.119.0 / 1.68.0 | `pc` data includes 1.21.11 and 26.1. |
| mineflayer-pathfinder | 2.4.5 (last publish **2023-09-04**) | Driven by minecraft-data, so it runs on new versions. Barely maintained. Mindcraft ships a 12 KB patch for it (doors, trapdoors, lava, climbables). |
| mineflayer-collectblock | 1.6.0 (2025-01-24) | OK. It lists `mineflayer ^4.0.0` as a normal dependency, which npm dedupes. |
| mineflayer-pvp | 1.3.2 (last publish **2022-07-03**) | Unmaintained. It listens for `physicTick`, which Mineflayer 4.39 still emits but marks deprecated with a console warning and says "may be removed". Mindcraft patches it. |
| mineflayer-tool | 1.2.0 (last publish **2022-05-11**) | Unmaintained but small and simple. |
| better-sqlite3 | **13.2.0** (2026-11-14), engines `node >=22` | **Major de-risk:** 13.x is built on N-API (`node-addon-api`) and **ships prebuilt binaries inside the npm package, including `win32-x64.node`**. There is no install script, no node-gyp and no Visual Studio Build Tools, and the binary stays ABI-stable across Node majors. It bundles SQLite 3.54.1. |
| sqlite-vec | **0.1.9** (2026-05-18) · 0.1.10-alpha.4 | Has a `sqlite-vec-windows-x64` prebuilt. It is pre-1.0 and searches by brute-force KNN, which is fine at our scale (see risks). It worked with better-sqlite3 13.2.0 in the sandbox. Quirk: pass `rowid` as a **BigInt**. |
| Socket.IO (server + client) | 4.8.4 (2026-09-25) | OK. |
| React / react-dom | 19.3.1 (published 2026-11-18, one day before this check) | OK. Pin to the exact version. |
| Vite / @vitejs/plugin-react | 8.3.8 / 6.3.0 | OK. Needs Node `^20.19 \|\| >=22.12`. |
| Vitest | 5.1.1 | OK. Supports Node `^22.12 \|\| ^24 \|\| >=26`. |
| zod | 4.6.7 | Used for config, IPC, events and model-output validation. It has `z.toJSONSchema()`, which feeds Ollama's `format`, so one schema serves both purposes. |
| pino / pino-roll | 10.4.0 / 4.0.0 | Structured JSON logs that rotate by size or day. |
| Ollama | **v0.40.16** (2026-11-09, from the aggregator) | Supports structured outputs (`format` with a JSON schema) and `think` on/off. The docs say: **Windows ROCm list = RX 7000 series only. The RX 9060 XT is listed for ROCm on Linux only. Vulkan is "enabled by default" on Windows.** So the GPU path is Vulkan unless a newer release changes that. Milestone 1 Spike A measures it. |

## Minecraft version: finding and recommendation

Verified facts:

- Latest Minecraft release: **26.3** (2026-09-15). Mojang now numbers versions by year.
- Newest version Mineflayer supports: **26.1**. 26.2 and 26.3 are not supported.
- **Minecraft 26.1 and later need Java 25.** 1.21.11 needs Java 21, which is what you have installed.
- 1.21.11 is still in Mineflayer's `testedVersions`.

Recommendation: **keep 1.21.11 pinned**. Reasons:

1. It is officially supported, and your server and client already run it.
2. Moving to 26.1 needs Java 25, a new server, a new client profile, and a world that cannot be downgraded afterwards. It would still leave us two releases behind.
3. 26.1 support in Mineflayer is about three months old. The 4.38 and 4.39 changelogs still contain fixes for flaky 26.1 tests. The old plugins (pathfinder, pvp, tool) have far more real-world use on 1.21.x, including through Mindcraft.
4. None of our milestones depend on anything added after 1.21.11.

We will look at this again at a milestone boundary, recorded as an ADR, and only if a concrete need appears. The brief's condition ("don't upgrade until Mineflayer officially supports a newer version") is now technically met for 26.1. This recommendation chooses not to act on it yet. ADR 0002.

## Mineflayer plugins: what we will do

- Pin exact versions. The lockfile is committed.
- Wrap every plugin behind our own `minecraft` adapter, so the rest of the code never imports a plugin. Then any plugin can be patched or replaced without touching skills or the brain.
- Fix problems with `patch-package` only where an automated test reproduces the problem. Each patch gets an ADR. Mindcraft's patches (MIT) are a reference only.
- Mineflayer 4.39 already includes the core-library fixes that Mindcraft used to patch (the use_item rotation and the dig-time-in-water bug). Its remaining patches are pathfinder behaviour, and we will add those only if our tests show we need them.
- Combat: start with mineflayer-pvp behind the adapter. Plan to replace it with our own combat module by Milestone 5–6, which is also where the narrow RL combat work sits.

## Ollama model shortlist (tags verified on ollama.com 2026-11-19)

| Candidate | Tag | Download size | Context | Notes |
|---|---|---|---|---|
| Qwen 3.5 9B | `qwen3.5:9b` (= `latest`) | 6.6–7.6 GB | 256K | Also `9b-q4_K_M`, `9b-q8_0`. Thinking model: we will benchmark with `think` off and on. |
| Gemma 4 12B | `gemma4:12b` · `gemma4:12b-it-qat` | 7.7–8.0 GB · 7.2 GB | 256K | The QAT build is the one to try first for VRAM. |
| gpt-oss-20b | `gpt-oss:20b` | **14 GB** | 128K | **Probably will not fit on a 16 GB card next to the Minecraft client.** Ollama will offload part of it to the CPU. The benchmark will measure that rather than assume it. |
| Andy-4 (Mindcraft baseline) | `Sweaterdog/Andy-4` (default ≈ q4, `q5_k_m`, `q8_0`) | 4.9 GB (default) | 128K | Trained on Mindcraft's `!command` text format, not our JSON. Under schema-constrained decoding it will produce valid JSON, but how good its decisions are is an open question. That makes it a fair baseline. |
| Reference (already pulled) | `qwen3:8b` | 5.2 GB | 40K | Your installed model. Useful as a reference point. |
| Embeddings (already pulled) | `nomic-embed-text` (v1.5, 137M) | ~0.3 GB | — | 768-dimension vectors. |

Newer Qwen generations exist (`qwen3.6`, `qwen3.8`), but only in 27B or larger and MoE sizes. Those do not fit next to Minecraft on 16 GB, so Qwen 3.5 9B stays the Qwen candidate. Model licences (especially Andy-4's) will be checked before the Milestone 3 benchmark.

## Reference projects

- **Voyager** (MineDojo/Voyager): MIT. We port ideas and adapted prompts, with credit in file headers and the README.
- **Mindcraft** (mindcraft-bots/mindcraft): **MIT** (© 2024 Kolby Nottingham). We may copy code from it if we keep the copyright and licence notice in the copied files and in the README credits. Its `package.json` shows the same plugin set we chose (mineflayer ^4.33, pathfinder 2.4.5, pvp 1.3.2, collectblock) plus `patch-package`.
- **Generative Agents** (Park et al., 2023): a paper. We implement it ourselves and cite it.

## Still to verify on the target PC (Milestone 1)

- That Ollama actually runs on the RX 9060 XT through Vulkan: the processor split from `/api/ps` (`size_vram` / `size`), tokens/s, and time to first token, with Minecraft closed and then open.
- That `better-sqlite3` and `sqlite-vec` load on Windows 11 under Node 24. The health check runs a real KNN query.
- That Mineflayer 4.39.0 joins your 1.21.11 server, walks, and disconnects cleanly (Spike B).
- That the Ollama version on your PC is recent enough for structured outputs and `think` control.
