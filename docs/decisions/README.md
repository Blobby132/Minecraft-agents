# Architecture Decision Records

Short records of decisions that are hard to reverse. Copy `0000-template.md` to start a new one. Status values: Proposed → Accepted (after review) → Superseded/Deprecated. Never rewrite an Accepted ADR. Supersede it with a new one.

| ADR | Title | Status |
|---|---|---|
| [0001](0001-core-stack-and-no-agent-framework.md) | Core stack and no agent framework | Proposed |
| [0002](0002-pin-minecraft-1-21-11.md) | Pin Minecraft to 1.21.11 | Proposed |
| [0003](0003-typescript-7-and-oxlint.md) | TypeScript 7 with oxlint (fallback TS 6 + typescript-eslint) | Proposed |
| [0004](0004-orchestrator-and-agent-processes.md) | Orchestrator plus one process per agent, typed IPC | Proposed |
| [0005](0005-layered-agent-and-action-validation.md) | Layered agent; model never acts directly; validation pipeline | Proposed |
| [0006](0006-model-provider-and-inference-queue.md) | All model calls via ModelProvider and one InferenceQueue | Proposed |
| [0007](0007-composite-skills-as-json-programs.md) | Composite skills are validated JSON programs, not generated code | Proposed |
| [0008](0008-sqlite-storage-layout.md) | SQLite storage layout, migrations, backups, embeddings | Proposed |
| [0009](0009-memory-skills-training-separation.md) | Memory, skill acquisition and policy training are separate; log raw reward signals early | Proposed |
| [0010](0010-mineflayer-plugin-adapter.md) | Mineflayer plugins behind an adapter, exact pins, patch policy | Proposed |
| [0011](0011-dashboard-real-data-only.md) | Dashboard shows recorded data only; snapshot + sequenced events | Proposed |
