# 0007. Composite skills are validated JSON programs, not model-generated code

- Status: Proposed
- Date: 2026-11-19
- Milestone: M4

## Context

Voyager has GPT-4 write JavaScript skills, which it then runs. Our models are small local ones (8–20B), which produce invalid or unsafe code much more often. Running generated code also risks the process and the PC.

## Decision

- Composite skills are **declarative JSON programs** over existing skills: `sequence`, bounded `repeat_until`, and `if` over a fixed predicate vocabulary. A small interpreter runs them.
- Before saving, a program is statically validated: skills exist, parameters type-check against each skill's schema, loops have bounds, and it is not a duplicate. It is saved **only after it succeeds in the game**, as confirmed by the deterministic verifier.
- Each skill tracks attempts and successes. Skills with low success are flagged and excluded from automatic selection.
- New *primitive* skills are only ever written by humans (or by Claude Code in a reviewed commit), with tests.

## Consequences

- Safe, fully checkable skill growth. Expressiveness is limited to what composition of primitives allows. That is acceptable, because primitives can be added by hand.

## Alternatives considered

- Model-written JS in a sandbox (`ses`, `isolated-vm`): possible later if composition proves too limiting. It would need its own ADR and a security review.
