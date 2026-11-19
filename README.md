# Minecraft Agents

An autonomous, fully local AI player for Minecraft Java Edition 1.21.11, running on Windows 11. You give it a high-level goal. It plans, acts through tested skills, checks its progress against the real game state and changes strategy when things go wrong.

> **Status: design phase.** No application code yet. The architecture and plan are waiting for review. Setup instructions arrive with Milestone 1.

## Documentation

- [Architecture](docs/architecture.md): processes, modules, communication, storage, repository layout
- [AI design](docs/ai-design.md): reflexes, skills, planner, critic, memory, skill library, learning
- [Roadmap](docs/roadmap.md): scope, milestones, acceptance criteria, open decisions
- [Risks](docs/risks.md): top technical risks and how each is detected early
- [Version check (2026-11-19)](docs/research/2026-11-version-verification.md)
- [Architecture Decision Records](docs/decisions/)

## Credits

This project borrows ideas, and possibly code, from the work below. Each relevant file header credits its source.

- **Voyager**: Wang et al., 2023, *Voyager: An Open-Ended Embodied Agent with Large Language Models*. [MineDojo/Voyager](https://github.com/MineDojo/Voyager), MIT License. The automatic curriculum → skill execution → self-verification → embedding-indexed skill library loop. Its ideas and prompts are ported to TypeScript and adapted for small local models.
- **Generative Agents**: Park et al., 2023, *Generative Agents: Interactive Simulacra of Human Behavior*. The memory stream, retrieval by recency, importance and relevance, reflection and planning. Implemented here from the paper.
- **Mindcraft**: [mindcraft-bots/mindcraft](https://github.com/mindcraft-bots/mindcraft), MIT License, © 2024 Kolby Nottingham. A reference for Mineflayer action code. Any copied code keeps its licence notice.
- **Mineflayer** and the PrismarineJS ecosystem: the Minecraft bot library this project is built on.
