---
stepsCompleted: [1]
inputDocuments: []
date: 2025-12-29
author: Aip0rt
type: product-research-proposal
status: draft
assignedTeam: GPT-5 Pro / Claw Dev
---

# Product Research Proposal: Codex-Claude Subagent Integration

## Executive Summary

**Project:** codex-subagents
**Repository:** https://github.com/airplne/codex-subagents
**Description:** Orchestrate parallel Claude subagents from Codex using background terminal processes

### The Vision

Invert the traditional AI orchestration paradigm by enabling **Codex to spawn and coordinate Claude subagents** via background terminal processes. This creates a cross-model collaboration pattern where Codex serves as the conductor and Claude instances form the orchestra.

---

## Problem Statement

### Current State

- Claude Code has a native subagent system where Claude orchestrates other Claude instances
- Codex has background terminal process capabilities but lacks native multi-agent orchestration
- No established pattern exists for cross-model AI agent coordination

### Opportunity

Developers using Codex could benefit from parallel AI assistance without context fragmentation. By leveraging Codex's background terminal feature to spawn Claude subagents, we create:

1. **Cross-model synergy** - Combine Codex's strengths with Claude's capabilities
2. **Parallel execution** - Multiple Claude instances working concurrently
3. **Unified orchestration** - Single Codex session managing the swarm

---

## Research Questions for GPT-5 Pro Team

### Q1: Architecture Pattern Selection

**Investigate:** What is the optimal architecture pattern for Codex-to-Claude orchestration?

**Options to Evaluate:**
- Direct CLI invocation (`claude-code` CLI in background terminals)
- API-based communication (Anthropic API calls from Codex)
- Message queue pattern (Redis/file-based coordination)
- Hybrid approach (CLI for spawning, API for communication)

**Deliverable:** Architecture comparison matrix with trade-offs

---

### Q2: Background Terminal Process Mechanics

**Investigate:** How should Codex's background terminal feature be leveraged?

**Areas to Research:**
- Maximum concurrent background processes supported
- Process lifecycle management (spawn, monitor, terminate)
- Output streaming and aggregation patterns
- Error handling and recovery mechanisms
- Resource constraints and limitations

**Deliverable:** Technical specification for process management layer

---

### Q3: Context Synchronization

**Investigate:** How do we maintain context coherence across Codex + multiple Claude instances?

**Challenges:**
- Codex context vs Claude context (different models, different state)
- Inter-agent communication (how do Claude subagents share information?)
- Task result aggregation back to Codex orchestrator
- Avoiding context fragmentation and duplication

**Deliverable:** Context management strategy document

---

### Q4: Task Distribution Patterns

**Investigate:** What task distribution patterns are most effective?

**Patterns to Evaluate:**
- **Fan-out/Fan-in:** Codex distributes subtasks, aggregates results
- **Pipeline:** Sequential handoff between specialized Claude agents
- **Swarm:** Autonomous agents with shared objective
- **Supervisor:** Codex monitors and redirects as needed

**Deliverable:** Task distribution pattern recommendations with use cases

---

### Q5: Integration Points with Claude Code

**Investigate:** How should we interface with Claude Code's existing capabilities?

**Areas to Explore:**
- Claude Code CLI invocation patterns
- Session management and persistence
- Tool access and permissions inheritance
- Hook integration possibilities
- MCP server coordination

**Deliverable:** Claude Code integration specification

---

### Q6: Security and Sandboxing

**Investigate:** What security model should govern cross-model agent execution?

**Considerations:**
- Sandboxing Claude subagents appropriately
- Credential and secret management across agents
- Permission boundaries between Codex and Claude
- Audit logging for multi-agent actions

**Deliverable:** Security architecture recommendations

---

### Q7: Developer Experience

**Investigate:** What DX patterns would make this intuitive for developers?

**Areas to Define:**
- How developers request subagent spawning
- How results are presented back
- How errors and conflicts are surfaced
- Configuration and customization options

**Deliverable:** DX specification and example workflows

---

## Success Criteria

### Research Phase Complete When:

1. All 7 research questions have documented answers
2. Architecture decision record (ADR) is produced
3. Proof-of-concept code validates core assumptions
4. Risk assessment identifies blockers and mitigations
5. Recommendation for implementation approach is clear

### Key Metrics to Inform:

- **Latency:** Time to spawn subagent + receive first response
- **Throughput:** Maximum useful concurrent subagents
- **Reliability:** Success rate of task completion across agents
- **Context efficiency:** Tokens used vs task complexity

---

## Proposed Research Timeline

| Phase | Activity | Output |
|-------|----------|--------|
| Discovery | Document Codex background terminal capabilities | Technical assessment |
| Discovery | Analyze Claude Code subagent implementation | Pattern documentation |
| Analysis | Evaluate architecture options (Q1-Q4) | Comparison matrices |
| Design | Define integration approach (Q5-Q7) | Technical specifications |
| Validation | Build minimal proof-of-concept | Working prototype |
| Synthesis | Compile findings into ADR | Architecture decision |

---

## Technical Context

### Codex Capabilities (to validate)

```
- Background terminal processes
- File system access
- Git operations
- CLI tool execution
- Persistent session state
```

### Claude Code Capabilities (known)

```
- Task tool for subagent spawning
- Background agent execution
- Agent types: Explore, Plan, code-reviewer, debugger, etc.
- Session persistence and resumption
- Hook system for automation
- MCP server integration
```

### Target Architecture (hypothesis)

```
Codex (Orchestrator)
  |
  +-- Background Terminal 1 --> claude-code --task "Research X"
  |
  +-- Background Terminal 2 --> claude-code --task "Implement Y"
  |
  +-- Background Terminal 3 --> claude-code --task "Test Z"
  |
  +-- Result Aggregation <-- stdout/files from all terminals
```

---

## Risks and Unknowns

| Risk | Impact | Mitigation |
|------|--------|------------|
| Codex background process limits | High | Research actual limits early |
| Context sync complexity | High | Design simple coordination protocol |
| Latency overhead | Medium | Benchmark and optimize |
| Authentication complexity | Medium | Leverage existing Claude auth |
| Model capability gaps | Low | Document capabilities clearly |

---

## Stakeholders

- **Requesting:** Aip0rt (Project Owner)
- **Research Team:** GPT-5 Pro / Claw Dev Team
- **Implementation:** TBD after research phase
- **Review:** BMAD Party Mode Agents (for brainstorming)

---

## Next Steps

1. **GPT-5 Pro Team:** Review this PRP and confirm scope
2. **GPT-5 Pro Team:** Execute research across Q1-Q7
3. **GPT-5 Pro Team:** Produce Architecture Decision Record
4. **Handoff:** Return findings to Aip0rt for implementation planning
5. **BMAD Workflow:** Route to `create-architecture` or `tech-spec` based on complexity

---

## Appendix A: Related Resources

- **Repository:** https://github.com/airplne/codex-subagents
- **Claude Code Docs:** https://docs.anthropic.com/claude-code
- **BMAD Framework:** `_bmad/` directory in repository

---

*PRP Generated: 2025-12-29*
*BMAD Framework Version: 6.0.0-alpha.21*
