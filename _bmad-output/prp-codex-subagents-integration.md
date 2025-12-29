---
stepsCompleted: [1, 2]
inputDocuments: []
date: 2025-12-29
author: Aip0rt
type: product-research-proposal
status: draft
assignedTeam: GPT-5 Pro / Claw Dev
version: 2.0
reviewedBy: Codex Team
---

# Product Research Proposal: Codex-Claude Subagent Integration

## Executive Summary

**Project:** codex-subagents
**Repository:** https://github.com/airplne/codex-subagents
**Description:** Orchestrate parallel Claude subagents from Codex using background terminal processes, with Codex subagents providing cross-model verification

### The Vision

Create a **dual-model orchestration system** where:
1. **Codex** acts as the primary orchestrator (CLI interface)
2. **Claude subagents** execute tasks (implementation, research, testing)
3. **Codex subagents** verify Claude's work (cross-model validation)

This creates a verification loop where different AI models check each other's output.

---

## Runtime Environment Specification

### Required Tools & Versions

| Tool | Minimum Version | Verification Command |
|------|-----------------|---------------------|
| **Codex CLI** | Latest stable | `codex --version` |
| **Claude Code CLI** | Latest stable | `claude --version` |
| **Node.js** | 18.x+ | `node --version` |
| **Git** | 2.x+ | `git --version` |

### Supported Operating Systems

| OS | Support Level | Notes |
|----|---------------|-------|
| macOS 13+ (Ventura) | Primary | Full support |
| Ubuntu 22.04+ | Primary | Full support |
| Windows 11 + WSL2 | Secondary | Requires WSL2 |
| Other Linux | Best effort | May require adjustments |

### What "Codex" Means in This Project

**Codex = OpenAI Codex CLI Tool**

- The command-line interface for OpenAI's Codex
- Invoked via `codex` command in terminal
- Has background process capabilities
- Can execute shell commands and manage sessions
- NOT the Codex API directly (though CLI may use it internally)

### Authentication Requirements

| Service | Auth Method | Storage |
|---------|-------------|---------|
| Codex CLI | OpenAI API key | `OPENAI_API_KEY` env var |
| Claude Code | Anthropic API key | `ANTHROPIC_API_KEY` env var |

---

## Architecture

### Dual-Model Orchestration Pattern

```
┌─────────────────────────────────────────────────────────────────┐
│                    CODEX (Primary Orchestrator)                 │
│                         CLI Interface                           │
└─────────────────────────────────────────────────────────────────┘
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
          ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ CLAUDE SUBAGENT │  │ CLAUDE SUBAGENT │  │ CLAUDE SUBAGENT │
│   Pool (Tasks)  │  │   Pool (Tasks)  │  │   Pool (Tasks)  │
│                 │  │                 │  │                 │
│ • Implement X   │  │ • Research Y    │  │ • Write tests Z │
│ • Code changes  │  │ • Analysis      │  │ • Validation    │
└────────┬────────┘  └────────┬────────┘  └────────┬────────┘
         │                    │                    │
         ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ CODEX SUBAGENT  │  │ CODEX SUBAGENT  │  │ CODEX SUBAGENT  │
│  (Verification) │  │  (Verification) │  │  (Verification) │
│                 │  │                 │  │                 │
│ • Verify impl   │  │ • Check research│  │ • Validate tests│
│ • Code review   │  │ • Fact check    │  │ • Coverage check│
└────────┬────────┘  └────────┬────────┘  └────────┬────────┘
         │                    │                    │
         └────────────────────┼────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │     AGGREGATION & DECISION    │
              │                               │
              │  Cross-model consensus:       │
              │  ✓ All verified → Ship        │
              │  ✗ Issues found → Loop back   │
              └───────────────────────────────┘
```

### Why Cross-Model Verification?

| Benefit | Description |
|---------|-------------|
| **Reduced bias** | Different models catch different errors |
| **Independent validation** | Codex reviewing Claude = true peer review |
| **Complementary strengths** | Each model excels at different tasks |
| **Higher confidence** | Multi-model agreement = stronger signal |

---

## Problem Statement

### Current State

- Claude Code has a native subagent system (Claude orchestrates Claude)
- Codex has background terminal capabilities but no native multi-agent orchestration
- No established pattern for cross-model AI agent coordination
- No verification layer where one model checks another's work

### Opportunity

Build a system where:
1. Developers get parallel AI assistance from multiple models
2. Claude does the work, Codex verifies the work
3. Cross-model consensus increases output quality
4. Single Codex session manages the entire workflow

---

## Research Questions for GPT-5 Pro Team

### Q1: Architecture Pattern Selection

**Investigate:** What is the optimal architecture pattern for Codex-to-Claude AND Codex-to-Codex orchestration?

**Options to Evaluate:**

| Option | Mechanism | Pros | Cons |
|--------|-----------|------|------|
| A: Direct CLI | `claude --task "X"` in bg terminal | Simple, native | Limited control |
| B: API-based | Anthropic/OpenAI API calls | Full control | No file access |
| C: Message Queue | Redis/file coordination | Scalable, decoupled | Complex setup |
| D: Hybrid | CLI spawn + file communication | Flexible | More moving parts |

**Deliverable:** Architecture comparison matrix with weighted scores

---

### Q2: Background Terminal Process Mechanics

**Investigate:** What are the hard limits of Codex's background terminal feature?

**Measurement Methodology:**

| Metric | How to Measure | Repetitions | Definition |
|--------|----------------|-------------|------------|
| **Max concurrent processes** | Spawn N processes, observe failures | 10 runs | Count where spawn succeeds |
| **Spawn latency** | `time` from command to first output | 50 samples | Milliseconds to first byte |
| **Process timeout** | Long-running task, measure cutoff | 5 runs | Seconds until force-kill |
| **Memory per process** | Monitor RSS during execution | 10 samples | MB per background process |
| **Stdout buffer size** | Large output, check truncation | 5 runs | Bytes before truncation |

**Instrumentation Approach:**
```bash
# Wrapper script for measurement
#!/bin/bash
START=$(date +%s%N)
codex --background "claude --task '$TASK'"
END=$(date +%s%N)
echo "Spawn latency: $(( (END-START)/1000000 ))ms"
```

**Concurrency Definition:**
- "Concurrent" = processes running simultaneously (not queued)
- Measured by `ps aux | grep -c "claude\|codex"` during peak

**Deliverable:** Technical spec with hard numbers (tables, not prose)

---

### Q3: Context Synchronization

**Investigate:** How do we maintain context coherence across Codex + multiple Claude + multiple Codex subagents?

**Challenges:**
- Codex primary context vs Claude context vs Codex verifier context
- How do Claude subagents share information with each other?
- How do Codex verifiers access Claude's output?
- Avoiding context fragmentation and duplication

**Approaches to Evaluate:**

| Approach | Mechanism | Latency | Complexity |
|----------|-----------|---------|------------|
| Shared filesystem | All agents read/write common files | Low | Low |
| Context injection | Orchestrator summarizes, passes to each | Medium | Medium |
| Pub/sub | Agents broadcast updates via file/queue | Medium | High |
| Centralized state | Single state file, agents read/append | Low | Medium |

**Deliverable:** Context management strategy with data flow diagrams

---

### Q4: Task Distribution Patterns

**Investigate:** What patterns work for the dual-layer (execution + verification) architecture?

**Patterns to Prototype:**

```
PATTERN A: Fan-Out Execute → Fan-Out Verify → Aggregate
Codex → [Claude1, Claude2, Claude3] → [CodexV1, CodexV2, CodexV3] → Decision

PATTERN B: Execute-Verify Pairs
Codex → (Claude1 → CodexV1) || (Claude2 → CodexV2) → Aggregate

PATTERN C: Pipeline with Checkpoints
Codex → Claude1 → CodexV1 → Claude2 → CodexV2 → Done

PATTERN D: Supervisor with Spot-Checks
Codex → [Claude pool working] → Codex randomly verifies subset
```

**Deliverable:** Pattern recommendations with "use this when..." matrix

---

### Q5: Integration Points with Claude Code

**Investigate:** How should we interface with Claude Code CLI?

**Research Constraints:**
- Claude Code is **open source** - internal inspection IS allowed
- Source: https://github.com/anthropics/claude-code
- Can read implementation details, not just black-box testing

**Areas to Explore:**

| Area | Question | Method |
|------|----------|--------|
| CLI invocation | Best flags for non-interactive use | Test + source review |
| Session management | Can sessions be shared/resumed? | Source review |
| Tool inheritance | Do subagents inherit tool access? | Experiment |
| Hook integration | Can orchestrator inject hooks? | Source review |
| MCP coordination | Multi-agent MCP server access? | Experiment |

**Deliverable:** Claude Code integration specification with code examples

---

### Q6: Security and Sandboxing

**Investigate:** What security model governs cross-model agent execution?

**Mandatory Security Guardrails:**

| Rule | Rationale |
|------|-----------|
| **NEVER use `--dangerously-skip-permissions`** | Shared environments must have permission checks |
| **API keys via env vars ONLY** | Never hardcode, never commit |
| **Logs must be sanitized** | Strip secrets before any output |
| **Subagent scope limits** | Define what each agent CAN'T do |
| **Audit trail required** | Log which agent did what, when |

**Permission Matrix to Define:**

| Action | Claude Subagent | Codex Verifier | Orchestrator |
|--------|-----------------|----------------|--------------|
| Read files | Scoped to project | Scoped to project | Full |
| Write files | Scoped to project | Output only | Full |
| Execute commands | Sandboxed | Read-only verify | Full |
| Network access | None | None | As needed |
| Access secrets | Via env var | Via env var | Via env var |

**Deliverable:** Security architecture with permission matrix and audit spec

---

### Q7: Developer Experience

**Investigate:** What DX patterns make this intuitive?

**Areas to Define:**

| Question | Answer Format Needed |
|----------|---------------------|
| How to spawn Claude subagents? | Example command/API call |
| How to spawn Codex verifiers? | Example command/API call |
| How are results presented? | Output format specification |
| How are errors surfaced? | Error handling specification |
| How to configure agent count? | Config file format |
| How to customize verification rules? | Rule definition format |

**Deliverable:** DX specification with complete example workflow

---

## Artifact Contract

### Output Directory Structure

```
codex-subagents/
├── _bmad-output/
│   ├── prp-codex-subagents-integration.md    # This document
│   └── research/
│       ├── q1-architecture-matrix.md          # Q1 deliverable
│       ├── q2-background-terminal-spec.md     # Q2 deliverable
│       ├── q3-context-sync-strategy.md        # Q3 deliverable
│       ├── q4-task-patterns.md                # Q4 deliverable
│       ├── q5-claude-integration-spec.md      # Q5 deliverable
│       ├── q6-security-architecture.md        # Q6 deliverable
│       ├── q7-dx-specification.md             # Q7 deliverable
│       └── adr-final-architecture.md          # Final ADR
├── poc/
│   ├── README.md                              # POC documentation
│   ├── run-poc.sh                             # Single command to run
│   ├── src/
│   │   ├── orchestrator.js                    # Main orchestrator
│   │   ├── claude-spawner.js                  # Claude agent spawner
│   │   └── codex-verifier.js                  # Codex verification
│   └── results/
│       └── benchmark-results.json             # Measured metrics
└── docs/
    └── research-log.md                        # Daily research notes
```

### File Naming Convention

- Research outputs: `q{N}-{topic}.md` (e.g., `q1-architecture-matrix.md`)
- POC code: `{function}.js` (e.g., `orchestrator.js`)
- Benchmarks: `benchmark-{metric}.json`
- Logs: `YYYY-MM-DD-{activity}.md`

### Reproduction Commands

```bash
# Clone and setup
git clone https://github.com/airplne/codex-subagents.git
cd codex-subagents

# Set required environment variables
export OPENAI_API_KEY="your-key"
export ANTHROPIC_API_KEY="your-key"

# Run POC (single command)
./poc/run-poc.sh

# Run specific benchmark
./poc/run-poc.sh --benchmark q2-concurrency

# View results
cat poc/results/benchmark-results.json
```

---

## Success Criteria

### Research Phase Complete When:

| Criterion | Verification |
|-----------|--------------|
| All 7 research questions documented | Files exist in `research/` |
| Architecture Decision Record produced | `adr-final-architecture.md` exists |
| POC validates core assumptions | `run-poc.sh` exits 0 |
| Risk assessment complete | Risks section in ADR |
| Recommendation is actionable | ADR has "Build This" section |

### Key Metrics (POC Must Measure):

| Metric | Target | Measurement |
|--------|--------|-------------|
| **Spawn latency** | < 2000ms | Time to first Claude output |
| **Concurrent agents** | ≥ 3 | Simultaneous Claude + Codex |
| **Verification latency** | < 3000ms | Codex verifier response time |
| **Success rate** | > 95% | Tasks completed without error |
| **Cross-model agreement** | Measurable | % where Codex confirms Claude |

---

## Risks and Mitigations

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Codex background process limit too low | High | Medium | Discover early in Q2, design for limit |
| Context sync too complex | High | Medium | Start with simple file-based approach |
| Verification adds too much latency | Medium | Medium | Parallelize, sample-based verification |
| API rate limits hit | Medium | High | Implement queuing, backoff |
| Auth complexity across models | Medium | Low | Use env vars consistently |

---

## Timeline & Phases

| Phase | Duration | Activities | Outputs |
|-------|----------|------------|---------|
| **Discovery** | Days 1-2 | Document Codex + Claude capabilities | Technical assessments |
| **Analysis** | Days 3-5 | Answer Q1-Q4 | Comparison matrices |
| **Design** | Days 6-8 | Answer Q5-Q7, draft ADR | Technical specs |
| **Validation** | Days 9-11 | Build POC, run benchmarks | Working prototype |
| **Synthesis** | Days 12-14 | Finalize ADR, document findings | Final deliverables |

---

## Stakeholders

| Role | Who | Responsibility |
|------|-----|----------------|
| **Project Owner** | Aip0rt | Final decisions, approval |
| **Research Team** | GPT-5 Pro | Execute Q1-Q7, build POC |
| **Implementation** | Claw Dev (TBD) | Build production system |
| **Review** | BMAD Party Agents | Brainstorming, validation |

---

## Next Steps

1. **GPT-5 Pro Team:** Confirm PRP scope and artifact contract
2. **GPT-5 Pro Team:** Set up development environment per Runtime spec
3. **GPT-5 Pro Team:** Execute research phases sequentially
4. **GPT-5 Pro Team:** Submit findings via PR using template
5. **Handoff:** Return ADR to Aip0rt for implementation decision
6. **BMAD Workflow:** Route to `create-architecture` or `tech-spec`

---

## Appendix A: Related Resources

- **Repository:** https://github.com/airplne/codex-subagents
- **Claude Code Source:** https://github.com/anthropics/claude-code
- **Claude Code Docs:** https://docs.anthropic.com/claude-code
- **BMAD Framework:** `_bmad/` directory in repository

## Appendix B: Glossary

| Term | Definition |
|------|------------|
| **Codex CLI** | OpenAI's command-line tool for Codex |
| **Claude Code** | Anthropic's CLI tool for Claude |
| **Orchestrator** | Primary Codex instance managing the workflow |
| **Subagent** | Spawned agent (Claude or Codex) doing subtask |
| **Verifier** | Codex subagent checking Claude's output |
| **Background terminal** | Codex feature to run processes asynchronously |
| **Cross-model verification** | One AI model checking another's work |

---

*PRP Generated: 2025-12-29*
*PRP Updated: 2025-12-29 (v2.0 - Codex team feedback incorporated)*
*BMAD Framework Version: 6.0.0-alpha.21*
