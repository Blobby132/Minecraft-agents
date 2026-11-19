# 0005. Layered agent; the model never acts directly; validation pipeline

- Status: Proposed
- Date: 2026-11-19
- Milestone: M2–M3

## Context

Model calls take seconds and small models produce invalid output. The game needs millisecond responses to danger. The brief requires that every proposed action is validated against a schema and the game state before it runs.

## Decision

- Four layers: **reflexes** (deterministic, preempt everything), **skills** (tested TypeScript, abortable, with timeouts), **planner** (model, event-driven), **critic** (deterministic verifier first, model critic only when needed). Details in `docs/ai-design.md`.
- The model outputs one `Decision` JSON. The pipeline is: parse → zod schema → semantic validation against the current WorldSnapshot (skill exists, preconditions, real item names, ranges, not a recently failed repeat) → at most one repair retry with the errors → safe deterministic fallback. Every step is recorded.
- Skill names and item/block names are enums generated at runtime from the skill registry and minecraft-data.
- A decision record is written before execution and updated after it.

## Consequences

- The bot stays alive while the model thinks. Every action can be traced to a validated decision.
- The skill library must exist before the model can do anything useful (M2 before M3).

## Alternatives considered

- The model emitting low-level controls or code: too slow, unsafe and unverifiable.
