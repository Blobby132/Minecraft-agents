# 0006. All model calls via ModelProvider and one InferenceQueue

- Status: Proposed
- Date: 2026-11-19
- Milestone: M3 (spikes in M1 call Ollama directly, as throwaway code)

## Context

One GPU will serve every agent. The runtime may change (Ollama now; LM Studio or a llama.cpp server later). Urgent requests must not wait behind long planning calls.

## Decision

- `ModelProvider` interface: `health`, `listModels`, `loadedModels` (with GPU share), `generateStructured`, `embed`. Ollama is the first implementation.
- A single `InferenceQueue` in the orchestrator, with priorities P1 urgent … P5 background, aging, per-agent round-robin, deadlines (stale requests expire), cancellation through AbortController, configurable concurrency (default 1), and metrics.
- Model roles (`planner`, `critic`, `curriculum`, `embedder`) map to model IDs in config. All agents share the same base model.
- Structured output uses Ollama `format` with JSON Schema generated from zod, and `think` is set per task type.
- Every call is logged (prompt template version, model, timings, outcome) in the runtime DB.

## Consequences

- Switching model or runtime only touches the provider and config.
- Queue metrics are available before multi-agent load arrives.

## Alternatives considered

- Agents calling Ollama directly and relying on `OLLAMA_NUM_PARALLEL`: no priorities, no fairness, no deadlines, and VRAM thrashing.
