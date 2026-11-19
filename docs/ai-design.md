# How the AI reasons, plans, acts, evaluates and learns

Status: **Proposed** (first assignment, awaiting approval).

Credits: the curriculum → skill → critic → skill-library loop is adapted from **Voyager** (Wang et al., 2023, MineDojo/Voyager, MIT). The memory stream, retrieval scoring, reflection and planning are adapted from **Generative Agents** (Park et al., 2023). Mineflayer action code may reference **Mindcraft** (mindcraft-bots/mindcraft, MIT).

## 1. Principles

1. **The model chooses. Code validates and acts.** The model returns a structured `Decision`. Code checks it against the schema and against the current game state, then runs a tested skill. The model never moves the bot directly.
2. **Speed layers.** Reflexes run in milliseconds, skills in seconds, the planner over seconds to tens of seconds, and reflection over minutes. A slower layer never blocks a faster one.
3. **Evidence over opinion.** Success is checked against game state first, by deterministic predicates. A model critic is used only where no predicate can express the check, and its verdict is recorded as model judgment.
4. **No scripted progression.** The game's *facts* (recipes, block hardness, item IDs) come from `minecraft-data`. *What to do next* is always chosen by the curriculum or planner from the objective, the current state and memory. Milestone 7 "stages" are acceptance checkpoints, not hardcoded behaviour.
5. **Three distinct kinds of learning, never mixed up:**

| Kind | What changes | Where it lives | Is it "training"? |
|---|---|---|---|
| **Memory** | Records of what happened and what is known | `agent.db`: memory stream, facts, deaths, lessons | **No** |
| **Skill acquisition** | New composite skills (data), plus success statistics | `agent.db`: skill library | **No** |
| **Policy learning** (M6) | Parameters of a small learned policy (for example bandit weights), versioned checkpoints | `data/policies/`, with an evaluation record per version | **Yes**, narrow and explicit |

The base LLM weights are never changed in this plan. The dashboard and the docs always use these exact terms.

## 2. The four layers

### Layer 1: Reflexes (deterministic, no model)

They run on a fixed evaluation tick (around 5–10 Hz) and on key Mineflayer events (`health`, `entityHurt`, `breath`, and so on). Each reflex is a pure function `evaluate(snapshot, recentEvents) → ReflexIntent | null`, which makes it unit-testable on recorded snapshots. An arbiter picks the highest-priority intent:

| Priority | Reflex | Trigger (thresholds from config) | Action |
|---|---|---|---|
| 1 | Escape lava / fire | in lava, or on fire | Path to the nearest safe block or water, jump |
| 2 | Avoid drowning | oxygen below threshold | Swim straight up, or head toward the nearest air |
| 3 | Flee | low health with a hostile mob nearby, or a creeper about to explode | Sprint away, out of line of sight |
| 4 | Fight back | attacked, health OK, weapon available | Equip best weapon, attack the attacker |
| 5 | Eat | food ≤ threshold and edible item in inventory, not in danger | Equip food, consume |

Reflexes take the **motor lock**. The running skill gets an `AbortSignal`, stops cleanly and reports `aborted(reason)`. When the reflex finishes, control returns and the planner is told what happened (for example "fled from zombie, health 6/20"). Every reflex activation is recorded as a real event.

### Layer 2: Skills (deterministic, tested TypeScript)

```ts
interface Skill<P> {
  name: string;                 // "collect_block"
  version: number;
  kind: 'primitive' | 'composite';
  description: string;          // shown to the model
  params: ZodType<P>;           // exported as JSON Schema for the model
  preconditions(s: WorldSnapshot, p: P): Check[];   // e.g. "has axe or can break by hand", "block type exists nearby"
  expectedEffects(s: WorldSnapshot, p: P): OutcomeCheck[];  // e.g. inventory_at_least(oak_log, n)
  timeoutMs(p: P): number;
  run(ctx: SkillContext, p: P, signal: AbortSignal): Promise<SkillResult>;
}
// SkillResult = { status: 'success'|'failure'|'aborted'|'timeout', reason?, evidence, durationMs }
```

- **Primitive skills** (hand-written in M2, extended in M5): `go_to`, `collect_block`, `craft_item` (places a crafting table if needed), `smelt_item`, `place_block`, `equip`, `eat`, `attack_entity`, `flee`, `explore`, `deposit_items`, `withdraw_items`, `sleep`, and so on.
- **Composite skills** (M4) are **declarative JSON programs** over existing skills, run by a small interpreter. They allow `sequence`, `repeat_until(condition, max)` and `if(condition)` over a fixed predicate vocabulary. They are *not* model-written JavaScript, which is a deliberate change from Voyager (ADR 0007). Small local models write invalid code often, and a JSON program can be fully validated before it ever runs: skill names exist, parameters match their schemas, loops have bounds.
- Every skill run writes a `skill_runs` row (inputs, result, duration, evidence). Success rates are calculated from those rows.

### Layer 3: Planner (the model)

The planner is **event-driven, not per-tick**. It runs when:
- a skill finishes (success, failure, timeout)
- a reflex hands control back
- a new objective arrives from you
- a significant event occurs (nightfall, inventory full, a new structure seen, a death)
- a re-plan timer fires as a safety net

**Context builder.** Small models need short, focused prompts. The budget is about 3–6K tokens, measured in the M3 benchmark:
- Objective and active goal stack.
- A compact observation: position, dimension, biome, time of day, health, food, armour, equipment, inventory (aggregated), nearby notable blocks (counts and nearest positions), nearby entities (type, distance, hostile?).
- Top-k retrieved memories (§4).
- Candidate skills: always the core set, plus the top-k library skills by embedding similarity to the current goal. Each comes with its description, parameter schema and recent success rate.
- The last N decision outcomes, including critic feedback on failures. This is Voyager's "execution errors / critique" feedback.

**Output: one `Decision`**, constrained by Ollama's `format` JSON schema, which is generated from the zod schema:

```jsonc
{
  "goal": { "id": "g-17", "text": "Get 3 oak logs for a crafting table" },
  "rationale": "No wood in inventory; oak trees 12 blocks east.",   // the model's stated reason, recorded verbatim
  "plan": ["collect 3 oak_log", "craft oak_planks", "craft crafting_table"],
  "action": { "skill": "collect_block", "params": { "block": "oak_log", "count": 3 } },
  "expected_outcome": {
    "description": "3 oak logs in inventory",
    "checks": [ { "type": "inventory_at_least", "item": "oak_log", "count": 3 } ]
  },
  "confidence": 0.7
}
```

The `skill` field is an **enum generated at runtime from the skill registry**, and item and block names are checked against `minecraft-data`. So the model cannot name a skill or item that does not exist. On the dashboard, `rationale` is labelled as the *model's stated reason*. It is real model output, never an invented "thought".

**Validation pipeline** (in `@mca/inference` + `@mca/brain`):

1. JSON parse.
2. Zod schema check.
3. Semantic check against the current `WorldSnapshot`: skill exists, preconditions pass, items and blocks are real, coordinates are within range, the plan is not identical to one that just failed N times.
4. If a step fails, **repair**: one retry that sends the precise validation errors back to the model.
5. If repair fails, **fallback**: a safe deterministic choice (`observe` / `idle_safely` / continue the current goal's last valid step). It is recorded as `fallback`.

Malformed replies are normal input, never crashes. Each decision record stores whether it was `valid`, `repaired`, `fallback` or `rejected`. That is the "valid-action rate" metric.

**Curriculum (Voyager's automatic curriculum, adapted).** When the goal stack is empty or the current goal is finished, the model proposes the next goal. It sees the user objective, the current state, completed goals, failed goals (with reasons) and relevant memories. The prompt follows Voyager's curriculum prompt, rewritten for a small model: shorter, schema-constrained, and with explicit "already completed" and "recently failed" lists. Nothing in code encodes "wood → stone → iron".

### Layer 4: Critic

1. **Deterministic verifier (always runs).** It evaluates `expected_outcome.checks` against before and after snapshots and the skill's evidence. The predicate vocabulary includes `inventory_at_least`, `inventory_delta`, `position_within`, `block_at`, `entity_absent`, `health_at_least`, `equipped`, `is_in_dimension`, and more. The result is `success | partial | failure` plus the evidence.
2. **Model critic (only when needed).** This is for goal-level outcomes that no predicate can express (for example "is this shelter enclosed and lit?"). It is given structured evidence (block scans around the base, light levels, before/after) and returns `{success, reason, suggestion}`, following Voyager's self-verification prompt adapted for a small model. It is recorded as **model judgment**, kept separate from the verifier's result, and shown that way in the dashboard.

Critic output feeds back into the next planner call, into memory (as an episodic event, and as a lesson after repeated failures), and into the reward-signal log.

## 3. The decision loop end to end

```
Mineflayer events ─► Perception: WorldSnapshot + event stream ─► Reflex arbiter ── preempts ──┐
                              │                                                               ▼
                              ▼                                                   Executor (motor lock, watchdog)
                  planner trigger? ─► Context builder ─► InferenceQueue (priority P2) ─► model ─► Decision JSON
                                       ▲  memory retrieval, skill retrieval                         │
                                       │                                                            ▼
                                       │                                   Validate (schema → state) ─► repair / fallback
                                       │                                                            │
                                       │                                    Decision record (status=executing, written first)
                                       │                                                            ▼
                                       │                                         Executor runs skill (AbortSignal, timeout)
                                       │                                                            ▼
                                       └──── memory + reward signals + dashboard ◄── Verifier ─► (Model critic if needed)
```

**Decision record** (one row per decision, written before execution and updated after):
`id, agentId, createdAt, trigger, objectiveId, goal, rationale, plan, action, expectedOutcome, observationRef, retrievedMemoryIds, candidateSkills, promptTemplate@version, model, rawOutput, parseStatus (valid|repaired|fallback|rejected), validationErrors, execution {status, reason, durationMs, evidence}, verifier {verdict, evidence}, critic {verdict, reason}?, rewardSignalIds, latency {queueWaitMs, inferenceMs, ttftMs, executionMs}`.
These records feed the dashboard Activity panel, the memory stream, the benchmark and later training.

**Watchdog and stuck states.** Every skill has a timeout. A progress monitor tracks position change, inventory change and dig/place progress. If nothing progresses for N seconds, the skill is aborted with reason `stuck`. If the same action and parameters fail K times in a row, they enter a cooldown and the validator rejects them, which forces a different plan. If reflexes oscillate (flee, return, flee again), a "danger zone" memory is created so the planner avoids that area.

**Deaths, disconnects and crashes.**
- Death: record a death event (cause, killer, position, inventory lost), respawn, invalidate the goal state that depended on inventory, and let the planner decide whether to go back for the items.
- Disconnect or kick: reconnect with exponential backoff, and record each attempt.
- Agent process crash: the supervisor restarts it. On startup the agent reloads its goal stack and marks any `executing` decision as `interrupted`.
- Orchestrator crash: nothing is lost (WAL plus write-before-act). The start script brings everything back.

## 4. Memory (per agent, `data/agents/<id>/agent.db`)

| Store | Content | Written by |
|---|---|---|
| **Memory stream (episodic)** | Observations, actions, outcomes, chat, reflections. Each has `createdAt` (wall clock and game tick), `lastAccessedAt`, `importance (1–10)`, `embedding`, and source refs (decision ID, event ID). | Executor, critic, perception, reflection |
| **Semantic facts** | Named locations (base, chests, villages, portals, ore sightings: coordinates, dimension, last seen, confidence), resources seen, recipes the agent has *verified* by crafting (recipe data itself comes from minecraft-data) | Perception, critic |
| **Deaths** | Time, position, dimension, cause, killer, inventory lost, linked lesson | Death handler |
| **Lessons and reflections** | Short insight text, links to evidence memories, times retrieved, measured usefulness | Reflection, repeated-failure detector |
| **Skill library** | Skill definition (composite JSON), description and embedding, version, origin decision, attempts, successes, average duration, last used | Skill acquisition (§5) |
| **Goal stack and plans** | Current objective, goals and status, for crash recovery | Planner |
| **Relationships** *(M8–9)* | Per other agent: interactions, trust, sentiment, debts | Social layer later |

**Retrieval (Generative Agents).** `score = w_rec · recency + w_imp · importance + w_rel · relevance`, with each term normalised to [0, 1]:
- recency = `decay ^ hoursSinceLastAccess`, with decay 0.995 per hour as in the paper, tunable
- importance = `importance / 10`
- relevance = cosine similarity of the query embedding and the memory embedding

sqlite-vec does a KNN pre-filter (top ~200 by relevance). TypeScript then re-ranks those by the full score. Retrieved memories get their `lastAccessedAt` updated. The weights live in config.

**Importance scoring** avoids one model call per event. Common event types get a deterministic score (for example death = 10, first diamond = 9, a new location = 5, a routine step = 1). Events with no rule are batched and scored by the model at background priority.

**Reflection.** When the summed importance of events since the last reflection crosses a threshold (the paper uses 150; ours is tuned), or after a death or a repeated failure, a low-priority job runs. It asks the model for the 2–3 most salient questions about recent memories, retrieves evidence for each, and produces insights that cite evidence memory IDs. Those are stored as `reflection` memories, and as `lessons` when they say how to act.

**Embeddings.** `nomic-embed-text` through the same `ModelProvider` and `InferenceQueue`, at background priority. Each vector row records its model ID and dimension (ADR 0008).

## 5. Skill acquisition

1. **Start (M2–M3):** only hand-written primitive skills.
2. **Propose (M4+):** after a goal succeeds through a sequence of primitive skills, or when asked to "write a skill for X", the model may propose a **composite skill**: name, description, parameter schema and a JSON program.
3. **Static validation:** the program parses. Every referenced skill exists. Parameters are type-checked against the referenced skill schemas. Loops have bounds. Conditions use the known predicate vocabulary. Nothing duplicates an existing skill (checked by embedding similarity and name).
4. **Trial:** the skill runs in the game for the current goal. It is **saved only if the verifier confirms success.** A failed trial is recorded with the critic's reason, and the model may revise it a limited number of times (Voyager's iterative refinement, capped).
5. **Use:** saved skills are embedded and become candidates in the planner context. Each tracks attempts and successes. If a skill's success rate falls below a threshold over a minimum number of runs, it is flagged, shown with a warning, and excluded from automatic selection until revised.

## 6. Reward signals now, learning later (M6)

From Milestone 2, **raw signals** are logged per agent: health and food deltas, damage taken and source, deaths, items gained and lost, skill results, verifier verdicts, time per goal, and distance travelled. We store raw signals, not a single scalar reward, so the reward function can be defined and versioned later without losing history. Each decision record links to the signals observed during its execution. That gives M6 a real dataset of (context, action, outcome) tuples.

Narrow RL options for Milestone 6 are presented as an analysis before anything is built:

- **A. Contextual bandit for skill selection** (recommended first). Context = features of the WorldSnapshot and goal. Arms = candidate skills or parameters. Reward = verifier outcome minus time and damage cost. Trained offline from logged decisions, with offline evaluation by inverse-propensity or doubly-robust estimators, then online with exploration. It works at real-time game speed because one decision is one sample. The planner proposes and the bandit re-ranks.
- **B. Combat micro.** Real-time Mineflayer runs at 20 ticks/s and is too slow for end-to-end RL. Options to compare:
  - (i) A fast abstract combat simulator in Python, trained at thousands of steps per second and validated in a flat test world. There is a sim-to-real gap.
  - (ii) Several bots in parallel arenas on a dedicated test server.
  - (iii) Imitation from logged reflex and skill episodes.
  - Note: vanilla `/tick rate` and `/tick sprint` speed up the server but **not** Mineflayer's client-side physics, which runs on a 50 ms timer. So they do not speed up movement-based training.
- **C. Offline RL or behaviour cloning** on the logged decision dataset, as a later option.

These policies are tiny and train on the Ryzen CPU. PyTorch GPU support on AMD under Windows is a known risk, and we would avoid depending on it.

## 7. Inference scheduling (one GPU, many agents)

`InferenceQueue` lives in the orchestrator and serves every agent:

| Priority | Use | Typical deadline |
|---|---|---|
| P1 urgent | A danger decision the reflexes could not settle (for example fight or flee when outnumbered) | ~3 s, otherwise dropped and the reflex default is used |
| P2 act | Next-step decision after a skill finishes | ~20 s |
| P3 verify | Model critic | ~30 s |
| P4 plan | Curriculum or goal proposal, skill composition | ~60 s |
| P5 background | Reflection, importance scoring, embeddings | none |

Features:
- Strict priority with **aging**, so low priorities are not starved.
- **Round-robin across agents** within a priority.
- **Deadlines**: stale requests are dropped with a recorded `expired` status.
- **Cancellation** through AbortController, which closes the HTTP stream so Ollama stops generating.
- Concurrency of 1 by default. It is configurable once `OLLAMA_NUM_PARALLEL` and VRAM headroom have been measured.
- Metrics for the dashboard: queue depth, wait time, TTFT, tokens/s, outcome per priority.

`ModelProvider` interface (Ollama first; LM Studio or llama.cpp server later):

```ts
interface ModelProvider {
  readonly id: string;                                  // "ollama"
  health(): Promise<ProviderHealth>;                    // version, reachable
  listModels(): Promise<ModelInfo[]>;
  loadedModels(): Promise<LoadedModel[]>;               // incl. GPU share (size_vram/size)
  generateStructured<T>(req: StructuredRequest<T>, signal: AbortSignal): Promise<StructuredResult<T>>;
  embed(req: EmbedRequest, signal: AbortSignal): Promise<EmbedResult>;
}
```

Model roles (`planner`, `critic`, `curriculum`, `embedder`) map to model IDs in config. Every agent uses the same base model. Identity and personality come only from each agent's stored profile and prompts.

## 8. Multi-agent readiness (not built now)

These are already in place before Milestone 8:
- An agent ID in every event, IPC message, DB path, log file and inference request.
- One process and one DB per agent.
- A per-agent profile file for identity and personality (introduced in M3, with only one agent).
- A fair InferenceQueue.
- Chat perception goes into the memory stream as `chat` events from M2.

Still to build in Milestones 8–9: an inter-agent messaging protocol, relationship memory, trading, and dashboard views across agents.
