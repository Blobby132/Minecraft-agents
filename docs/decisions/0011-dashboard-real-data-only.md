# 0011. Dashboard shows recorded data only; snapshot plus sequenced events

- Status: Proposed
- Date: 2026-11-19
- Milestone: M1

## Context

The brief forbids fake data, invented thoughts and mocked results in the product. Panels without data must be labelled empty. Socket.IO connections can drop and reconnect.

## Decision

- Every dashboard view renders from the event store, decision records or live health results. Test fixtures exist only under `test/`, and the build fails if a dashboard module imports from a test directory.
- Empty panels show "No data yet — arrives in Milestone N".
- The model's `rationale` is shown as "model's stated reason" (verbatim output), never as hidden thoughts.
- Protocol: on connect, the client requests a snapshot (ack). Then it receives events carrying a monotonically increasing `seq`. On a gap or a reconnect, it requests a new snapshot.
- The server binds to 127.0.0.1 only.

## Consequences

- What you see is what happened. Resyncing after a reconnect is simple and correct.

## Alternatives considered

- Polling a REST API: higher latency and more load. Socket.IO is already in the stack.
