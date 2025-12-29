# codex-subagents

> Orchestrate parallel Claude subagents from Codex using background terminal processes

## Vision

Invert the traditional AI orchestration paradigm: **Codex conducts, Claude agents perform.**

This project enables Codex to spawn and coordinate multiple Claude subagents via background terminal processes, creating a cross-model collaboration pattern for parallel AI-assisted development.

```
Codex (Orchestrator)
  │
  ├─→ Background Terminal 1 ──→ Claude Subagent (Research)
  │
  ├─→ Background Terminal 2 ──→ Claude Subagent (Implementation)
  │
  ├─→ Background Terminal 3 ──→ Claude Subagent (Testing)
  │
  └─→ Result Aggregation ←──── stdout/files from all terminals
```

## Why?

- **Cross-model synergy** - Combine Codex's strengths with Claude's capabilities
- **Parallel execution** - Multiple Claude instances working concurrently
- **Unified orchestration** - Single Codex session managing the swarm
- **No context fragmentation** - Coordinated multi-agent assistance

## Status

**Phase: Research & Discovery**

The GPT-5 Pro / Claw Dev team is investigating the optimal integration approach. See the [Product Research Proposal](_bmad-output/prp-codex-subagents-integration.md) for details.

### Research Questions

| # | Question | Status |
|---|----------|--------|
| Q1 | Architecture Pattern Selection | Investigating |
| Q2 | Background Terminal Mechanics | Investigating |
| Q3 | Context Synchronization | Investigating |
| Q4 | Task Distribution Patterns | Investigating |
| Q5 | Claude Code Integration | Investigating |
| Q6 | Security & Sandboxing | Investigating |
| Q7 | Developer Experience | Investigating |

## Project Structure

```
codex-subagents/
├── _bmad/                    # BMAD Framework (methodology)
├── _bmad-output/             # Project artifacts
│   └── prp-codex-subagents-integration.md  # Research proposal
├── .github/
│   └── PULL_REQUEST_TEMPLATE.md
└── README.md
```

## Getting Started

> Coming after research phase completes

## Contributing

1. Review the [PRP](_bmad-output/prp-codex-subagents-integration.md) for context
2. Pick a research question (Q1-Q7) to investigate
3. Submit findings via PR using our [PR template](.github/PULL_REQUEST_TEMPLATE.md)
4. Follow BMAD framework conventions

## Tech Stack

- **Orchestrator:** OpenAI Codex
- **Subagents:** Claude (via Claude Code CLI)
- **Coordination:** Background terminal processes
- **Framework:** BMAD (BMad Method)

## License

TBD

---

**Keywords:** codex, claude, anthropic, openai, multi-agent, orchestration, background-terminal, parallel-execution, swarm, context-synchronization, task-distribution, cross-model, ai-agents, developer-tools, subagents
