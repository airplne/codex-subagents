---
stepsCompleted: [1, 2, 3]
inputDocuments: []
date: 2025-12-29
author: Aip0rt
type: product-research-proposal
status: draft
assignedTeam: GPT-5 Pro / Claw Dev
version: 3.0
reviewedBy: Codex Team
inspirations:
  - https://github.com/coleam00/Linear-Coding-Agent-Harness
---

# Product Research Proposal: Codex-Claude Subagent Integration

## Executive Summary

**Project:** codex-subagents
**Repository:** https://github.com/airplne/codex-subagents
**Description:** Orchestrate parallel Claude subagents from Codex using background terminal processes, with Codex subagents providing cross-model verification, managed by a structured harness

### The Vision

Create a **dual-model orchestration harness** where:
1. **Harness Core** manages task queues, security, progress tracking, and configuration
2. **Codex** acts as the primary orchestrator (CLI interface)
3. **Claude subagents** execute tasks (implementation, research, testing)
4. **Codex subagents** verify Claude's work (cross-model validation)
5. **Aggregator** calculates consensus and drives decisions

Inspired by the [Linear Coding Agent Harness](https://github.com/coleam00/Linear-Coding-Agent-Harness) pattern.

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

## Research Constraints & Tooling Preferences

### Architecture Tooling (Q1, Q8)

**Preference: Low-dependency first, graduate if needed**

| Tier | Tools | When to Use |
|------|-------|-------------|
| **Tier 1 (Default)** | Bash, Node.js, filesystem (JSON/YAML) | Start here - proves pattern with minimal deps |
| **Tier 2 (If needed)** | SQLite, Redis | Only if file-based coordination proves insufficient |
| **Tier 3 (Production)** | Docker, K8s | NOT for POC - save for production hardening |

**Rationale:** The POC should prove the *pattern* works. Infrastructure choices come after we know the pattern is viable. Document why if you graduate to Tier 2.

### Measurement Tooling (Q2)

**Preference: Yes, use third-party tools for accurate measurement**

| Tool | Purpose | Status |
|------|---------|--------|
| `htop` / `top` | Process monitoring | ✅ Approved |
| `pidstat` | Per-process CPU/memory | ✅ Approved |
| `time` / `/usr/bin/time -v` | Execution timing | ✅ Approved |
| `hyperfine` | Statistical benchmarking | ✅ Recommended |
| Node.js `perf_hooks` | JS-level profiling | ✅ Approved |
| Custom instrumentation | Harness-specific metrics | ✅ Approved |

**Requirement:** Document which tools were used and how measurements were taken (for reproducibility).

### Security Evaluation Scope (Q6)

**Preference: CLI permissions only for research phase**

| Scope | Coverage | Phase |
|-------|----------|-------|
| **CLI-only (Research)** | Permission flags, env vars, file permissions, audit logging | ✅ Now |
| **OS-level (Production)** | Containers, namespaces, cgroups, seccomp | Document for later |

**Rationale:** Prove the pattern first, harden later. Research SHOULD document where OS-level isolation would be needed for production.

---

## Architecture

### Harness-Based Dual-Model Orchestration

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      CODEX SUBAGENTS HARNESS                            │
│         (Inspired by Linear Coding Agent Harness Pattern)               │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           HARNESS CORE                                  │
├──────────────────┬──────────────────┬──────────────────┬───────────────┤
│   TASK QUEUE     │  SECURITY LAYER  │  PROGRESS STORE  │  CONFIG MGR   │
│                  │                  │                  │               │
│ • Task specs     │ • Cmd allowlist  │ • Task status    │ • Agent types │
│ • Dependencies   │ • FS boundaries  │ • Agent logs     │ • Timeouts    │
│ • Priority queue │ • API rate limit │ • Metrics/timing │ • Retry rules │
│ • Retry state    │ • Secret masking │ • Audit trail    │ • MCP config  │
└────────┬─────────┴────────┬─────────┴────────┬─────────┴───────┬───────┘
         │                  │                  │                 │
         └──────────────────┼──────────────────┼─────────────────┘
                            │                  │
                            ▼                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    ORCHESTRATOR (Primary Codex)                         │
│                                                                         │
│  1. Parse task specification (from file, CLI, or GitHub Issue)          │
│  2. Break into subtasks, populate task queue                            │
│  3. Spawn Claude subagents for execution (via background terminals)     │
│  4. Spawn Codex subagents for verification (via background terminals)   │
│  5. Aggregate results, check consensus                                  │
│  6. Update progress store, handle retries                               │
└─────────────────────────────────────────────────────────────────────────┘
                            │
         ┌──────────────────┼──────────────────┐
         │                  │                  │
         ▼                  ▼                  ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ CLAUDE EXECUTOR │ │ CLAUDE EXECUTOR │ │ CLAUDE EXECUTOR │
│   (bg term 1)   │ │   (bg term 2)   │ │   (bg term 3)   │
├─────────────────┤ ├─────────────────┤ ├─────────────────┤
│ • Pull task     │ │ • Pull task     │ │ • Pull task     │
│ • Read context  │ │ • Read context  │ │ • Read context  │
│ • Execute work  │ │ • Execute work  │ │ • Execute work  │
│ • Write output  │ │ • Write output  │ │ • Write output  │
│ • Update status │ │ • Update status │ │ • Update status │
└────────┬────────┘ └────────┬────────┘ └────────┬────────┘
         │                   │                   │
         ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ CODEX VERIFIER  │ │ CODEX VERIFIER  │ │ CODEX VERIFIER  │
│   (bg term 4)   │ │   (bg term 5)   │ │   (bg term 6)   │
├─────────────────┤ ├─────────────────┤ ├─────────────────┤
│ • Read Claude   │ │ • Read Claude   │ │ • Read Claude   │
│   output        │ │   output        │ │   output        │
│ • Verify work   │ │ • Verify work   │ │ • Verify work   │
│ • Score quality │ │ • Score quality │ │ • Score quality │
│ • Flag issues   │ │ • Flag issues   │ │ • Flag issues   │
│ • Update status │ │ • Update status │ │ • Update status │
└────────┬────────┘ └────────┬────────┘ └────────┬────────┘
         │                   │                   │
         └───────────────────┼───────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           AGGREGATOR                                    │
│                                                                         │
│  • Collect all Claude outputs + Codex verifications                     │
│  • Calculate cross-model consensus score                                │
│  • Decision: SHIP (all verified) | RETRY (issues found) | ESCALATE     │
│  • Update task queue, trigger next iteration or complete                │
└─────────────────────────────────────────────────────────────────────────┘
```

### Harness Core Components

| Component | Purpose | Implementation |
|-----------|---------|----------------|
| **Task Queue** | Manage work items with dependencies and priority | `tasks/queue.json` |
| **Security Layer** | Command allowlist, filesystem sandboxing, rate limiting | `config/security.yaml` |
| **Progress Store** | Track task status, agent logs, metrics | `tasks/*.json` + `logs/` |
| **Config Manager** | Agent types, timeouts, retry rules, MCP config | `config/agents.yaml` |

### Why Harness Pattern?

| Without Harness | With Harness |
|-----------------|--------------|
| Ad-hoc agent spawning | Structured task queue with priorities |
| No visibility into progress | Full observability (logs, metrics, status) |
| Security unclear | Explicit allowlist and boundaries |
| Results scattered across stdout | Centralized aggregation and storage |
| Manual retry on failure | Automatic retry with backoff |
| No audit trail | Complete audit logging |

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
- No harness pattern for managing multi-agent workflows with observability

### Opportunity

Build a harness-managed system where:
1. Developers get parallel AI assistance from multiple models
2. Claude does the work, Codex verifies the work
3. Cross-model consensus increases output quality
4. Full observability into task progress and agent behavior
5. Production-grade orchestration with retry logic and security

---

## Research Questions for GPT-5 Pro Team

### Q1: Architecture Pattern Selection

**Investigate:** What is the optimal architecture pattern for Codex-to-Claude AND Codex-to-Codex orchestration within the harness?

**Options to Evaluate:**

| Option | Mechanism | Pros | Cons |
|--------|-----------|------|------|
| A: Direct CLI | `claude --task "X"` in bg terminal | Simple, native | Limited control |
| B: API-based | Anthropic/OpenAI API calls | Full control | No file access |
| C: Message Queue | Redis/file coordination | Scalable, decoupled | Complex setup |
| D: Hybrid | CLI spawn + file communication | Flexible | More moving parts |

**Constraint:** Start with Tier 1 tools (Bash/Node.js, filesystem). Only move to Tier 2 if file-based proves insufficient.

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

**Approved Tools:** `htop`, `pidstat`, `time`, `/usr/bin/time -v`, `hyperfine`, Node.js `perf_hooks`

**Instrumentation Approach:**
```bash
#!/bin/bash
# Wrapper script for measurement
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

**Investigate:** What patterns work for the dual-layer (execution + verification) architecture within the harness?

**Patterns to Prototype:**

```
PATTERN A: Fan-Out Execute → Fan-Out Verify → Aggregate
Harness → [Claude1, Claude2, Claude3] → [CodexV1, CodexV2, CodexV3] → Aggregator

PATTERN B: Execute-Verify Pairs
Harness → (Claude1 → CodexV1) || (Claude2 → CodexV2) → Aggregator

PATTERN C: Pipeline with Checkpoints
Harness → Claude1 → CodexV1 → Claude2 → CodexV2 → Done

PATTERN D: Supervisor with Spot-Checks
Harness → [Claude pool working] → Codex randomly verifies subset → Aggregator
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

**Scope:** CLI permissions only for research phase. Document OS-level needs for production.

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

**Investigate:** What DX patterns make the harness intuitive?

**Areas to Define:**

| Question | Answer Format Needed |
|----------|---------------------|
| How to spawn Claude subagents? | Example command/API call |
| How to spawn Codex verifiers? | Example command/API call |
| How are results presented? | Output format specification |
| How are errors surfaced? | Error handling specification |
| How to configure agent count? | Config file format |
| How to customize verification rules? | Rule definition format |
| How to view progress? | Dashboard/CLI output spec |

**Deliverable:** DX specification with complete example workflow

---

### Q8: Harness Architecture (NEW)

**Investigate:** How should we structure the orchestration harness?

**Reference Implementation:** [Linear Coding Agent Harness](https://github.com/coleam00/Linear-Coding-Agent-Harness)

**Components to Design:**

| Component | Purpose | Questions to Answer |
|-----------|---------|---------------------|
| **Task Queue** | Manage work items | Format? Priority logic? Dependency handling? |
| **Security Layer** | Command allowlist, FS boundaries | What's allowed? How enforced? |
| **Progress Store** | Track status, logs, metrics | Schema? Update frequency? Retention? |
| **Config Manager** | Agent types, timeouts, retry rules | Config format? Hot reload? |
| **Orchestrator** | Main loop, spawning, aggregation | Event-driven or polling? |
| **Executor Interface** | Standard Claude agent protocol | Input/output contract? |
| **Verifier Interface** | Standard Codex verifier protocol | Verification criteria? Scoring? |
| **Aggregator** | Consensus logic, decisions | Threshold for consensus? Retry logic? |

**Harness File Structure:**

```
harness/
├── config/
│   ├── agents.yaml          # Agent type definitions
│   ├── security.yaml        # Allowlist, boundaries
│   └── harness.yaml         # Main harness config
├── tasks/
│   ├── queue.json           # Pending tasks
│   ├── in-progress.json     # Currently executing
│   ├── completed.json       # Done tasks with results
│   └── failed.json          # Failed tasks for retry/review
├── logs/
│   ├── orchestrator.log     # Main orchestrator log
│   ├── claude-agent-{id}.log
│   └── codex-verifier-{id}.log
├── metrics/
│   ├── timing.json          # Latency measurements
│   ├── consensus.json       # Verification scores
│   └── throughput.json      # Tasks per time period
├── context/
│   ├── shared-state.json    # Cross-agent shared state
│   └── task-{id}/           # Per-task context files
└── output/
    └── task-{id}/           # Final outputs per task
```

**Deliverable:** Harness architecture specification with:
- Component interfaces (input/output contracts)
- File format schemas
- State machine diagram for task lifecycle
- Error handling and retry logic

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
│       ├── q8-harness-architecture.md         # Q8 deliverable (NEW)
│       └── adr-final-architecture.md          # Final ADR
├── harness/                                   # Harness implementation (NEW)
│   ├── config/
│   │   ├── agents.yaml
│   │   ├── security.yaml
│   │   └── harness.yaml
│   ├── tasks/
│   ├── logs/
│   ├── metrics/
│   ├── context/
│   └── output/
├── poc/
│   ├── README.md                              # POC documentation
│   ├── run-poc.sh                             # Single command to run
│   ├── src/
│   │   ├── orchestrator.js                    # Main orchestrator
│   │   ├── claude-spawner.js                  # Claude agent spawner
│   │   ├── codex-verifier.js                  # Codex verification
│   │   └── harness/                           # Harness core (NEW)
│   │       ├── task-queue.js
│   │       ├── security.js
│   │       ├── progress.js
│   │       └── aggregator.js
│   └── results/
│       └── benchmark-results.json             # Measured metrics
└── docs/
    └── research-log.md                        # Daily research notes
```

### File Naming Convention

- Research outputs: `q{N}-{topic}.md` (e.g., `q1-architecture-matrix.md`)
- POC code: `{function}.js` (e.g., `orchestrator.js`)
- Config files: `{component}.yaml` (e.g., `security.yaml`)
- Task files: `{status}.json` (e.g., `queue.json`)
- Logs: `{component}-{id}.log` (e.g., `claude-agent-1.log`)
- Benchmarks: `{metric}.json` (e.g., `timing.json`)

### Reproduction Commands

```bash
# Clone and setup
git clone https://github.com/airplne/codex-subagents.git
cd codex-subagents

# Set required environment variables
export OPENAI_API_KEY="your-key"
export ANTHROPIC_API_KEY="your-key"

# Initialize harness
./harness/init.sh

# Run POC (single command)
./poc/run-poc.sh

# Run specific benchmark
./poc/run-poc.sh --benchmark q2-concurrency

# Run harness with example task
./harness/run.sh --task examples/simple-task.yaml

# View progress
./harness/status.sh

# View results
cat poc/results/benchmark-results.json
cat harness/metrics/consensus.json
```

---

## Success Criteria

### Research Phase Complete When:

| Criterion | Verification |
|-----------|--------------|
| All 8 research questions documented | Files exist in `research/` |
| Architecture Decision Record produced | `adr-final-architecture.md` exists |
| Harness POC implemented | `harness/` directory functional |
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
| **Harness overhead** | < 500ms | Time added by harness layer |

---

## Risks and Mitigations

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Codex background process limit too low | High | Medium | Discover early in Q2, design for limit |
| Context sync too complex | High | Medium | Start with simple file-based approach |
| Verification adds too much latency | Medium | Medium | Parallelize, sample-based verification |
| API rate limits hit | Medium | High | Implement queuing, backoff in harness |
| Auth complexity across models | Medium | Low | Use env vars consistently |
| Harness overhead significant | Medium | Low | Benchmark early, optimize hot paths |

---

## Timeline & Phases

| Phase | Duration | Activities | Outputs |
|-------|----------|------------|---------|
| **Discovery** | Days 1-2 | Document Codex + Claude capabilities | Technical assessments |
| **Analysis** | Days 3-5 | Answer Q1-Q4 | Comparison matrices |
| **Design** | Days 6-9 | Answer Q5-Q8, draft ADR | Technical specs, harness design |
| **Validation** | Days 10-13 | Build POC + harness, run benchmarks | Working prototype |
| **Synthesis** | Days 14-16 | Finalize ADR, document findings | Final deliverables |

---

## Stakeholders

| Role | Who | Responsibility |
|------|-----|----------------|
| **Project Owner** | Aip0rt | Final decisions, approval |
| **Research Team** | GPT-5 Pro | Execute Q1-Q8, build POC + harness |
| **Implementation** | Claw Dev (TBD) | Build production system |
| **Review** | BMAD Party Agents | Brainstorming, validation |

---

## Next Steps

1. **GPT-5 Pro Team:** Confirm PRP scope and artifact contract
2. **GPT-5 Pro Team:** Set up development environment per Runtime spec
3. **GPT-5 Pro Team:** Execute research phases sequentially
4. **GPT-5 Pro Team:** Build harness POC alongside research
5. **GPT-5 Pro Team:** Submit findings via PR using template
6. **Handoff:** Return ADR to Aip0rt for implementation decision
7. **BMAD Workflow:** Route to `create-architecture` or `tech-spec`

---

## Appendix A: Related Resources

- **Repository:** https://github.com/airplne/codex-subagents
- **Claude Code Source:** https://github.com/anthropics/claude-code
- **Claude Code Docs:** https://docs.anthropic.com/claude-code
- **Linear Coding Agent Harness:** https://github.com/coleam00/Linear-Coding-Agent-Harness
- **BMAD Framework:** `_bmad/` directory in repository

## Appendix B: Glossary

| Term | Definition |
|------|------------|
| **Codex CLI** | OpenAI's command-line tool for Codex |
| **Claude Code** | Anthropic's CLI tool for Claude |
| **Harness** | Orchestration framework managing task queue, security, progress |
| **Orchestrator** | Primary Codex instance managing the workflow |
| **Subagent** | Spawned agent (Claude or Codex) doing subtask |
| **Executor** | Claude subagent performing task execution |
| **Verifier** | Codex subagent checking Claude's output |
| **Aggregator** | Component that collects results and calculates consensus |
| **Background terminal** | Codex feature to run processes asynchronously |
| **Cross-model verification** | One AI model checking another's work |
| **Task Queue** | Harness component managing pending work items |
| **Progress Store** | Harness component tracking status and metrics |

## Appendix C: Harness Config Examples

### agents.yaml
```yaml
agents:
  claude-executor:
    type: claude
    role: executor
    timeout: 300000  # 5 minutes
    retries: 2
    commands:
      - claude
      - --task

  codex-verifier:
    type: codex
    role: verifier
    timeout: 180000  # 3 minutes
    retries: 1
    commands:
      - codex
      - --verify

orchestrator:
  max_concurrent_claude: 3
  max_concurrent_codex: 3
  consensus_threshold: 0.8
  retry_backoff: exponential
```

### security.yaml
```yaml
allowed_commands:
  - node
  - npm
  - git
  - ls
  - cat
  - mkdir
  - cp
  - mv

blocked_commands:
  - rm -rf
  - sudo
  - curl  # unless explicitly needed
  - wget

filesystem:
  allowed_paths:
    - ./src
    - ./tests
    - ./harness/output
    - ./harness/context
  blocked_paths:
    - /etc
    - /usr
    - ~/.ssh
    - ~/.aws

api_limits:
  openai_rpm: 60
  anthropic_rpm: 60

audit:
  enabled: true
  log_path: ./harness/logs/audit.log
  include_outputs: false  # for privacy
```

### task.yaml (example)
```yaml
id: task-001
name: Implement user authentication
priority: high
dependencies: []

spec: |
  Implement JWT-based user authentication with:
  - Login endpoint POST /auth/login
  - Token validation middleware
  - Refresh token support

context_files:
  - ./src/routes/index.js
  - ./src/middleware/auth.js

expected_outputs:
  - ./src/routes/auth.js
  - ./src/middleware/jwt.js
  - ./tests/auth.test.js

verification_criteria:
  - All tests pass
  - No hardcoded secrets
  - Proper error handling
  - Token expiration configured
```

---

*PRP Generated: 2025-12-29*
*PRP Updated: 2025-12-29 (v3.0 - Harness architecture + tooling constraints)*
*BMAD Framework Version: 6.0.0-alpha.21*
