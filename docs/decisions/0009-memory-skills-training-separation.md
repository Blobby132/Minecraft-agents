# 0009. Memory, skill acquisition and policy training are separate; log raw reward signals early

- Status: Proposed
- Date: 2026-11-19
- Milestone: all

## Context

The brief requires that saving memories or skills is never described as training model weights. Milestone 6 needs real data for narrow RL.

## Decision

- Three named concepts in code, the database, the dashboard and the docs:
  1. **Memory**: recorded experience and facts.
  2. **Skill acquisition**: new composite skills plus statistics.
  3. **Policy training**: parameter updates of small learned policies, versioned, starting in M6.
- The base LLM weights are not changed in this plan.
- From M2, log **raw reward signals** (health/food deltas, damage, deaths, item gains and losses, skill results, verifier verdicts, time) linked to decisions. Do not log a single pre-computed scalar, so reward functions can be defined and versioned later.

## Consequences

- An honest UI and terminology. M6 starts with an existing dataset.

## Alternatives considered

- Logging only a scalar reward: it freezes the reward design too early.
