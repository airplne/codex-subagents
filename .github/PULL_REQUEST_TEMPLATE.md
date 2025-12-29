## Summary

<!-- Describe what this PR does in 2-3 sentences. Focus on the WHY, not just the WHAT. -->

## Change Type

<!-- Check all that apply -->

- [ ] **Research Finding** - New insights from investigation (GPT-5 Pro team)
- [ ] **Architecture Decision** - ADR or architectural pattern selection
- [ ] **Implementation** - New feature or capability
- [ ] **Bug Fix** - Corrects existing behavior
- [ ] **Documentation** - Updates to docs, README, or inline comments
- [ ] **Infrastructure** - CI/CD, tooling, configuration
- [ ] **Refactor** - Code improvement without behavior change

## PRP Research Question Addressed

<!-- If this PR addresses one of the 7 research questions from the PRP, check it -->

- [ ] **Q1: Architecture Pattern Selection** - Codex-to-Claude orchestration patterns
- [ ] **Q2: Background Terminal Mechanics** - Process lifecycle, limits, streaming
- [ ] **Q3: Context Synchronization** - Cross-model context coherence
- [ ] **Q4: Task Distribution Patterns** - Fan-out/fan-in, pipeline, swarm, supervisor
- [ ] **Q5: Claude Code Integration** - CLI invocation, session management, hooks
- [ ] **Q6: Security & Sandboxing** - Permissions, credentials, audit logging
- [ ] **Q7: Developer Experience** - UX patterns, configuration, error handling
- [ ] **N/A** - Not related to PRP research questions

## Technical Details

### Codex Orchestration Impact

<!-- How does this affect Codex's role as the orchestrator? -->

- Spawning mechanism: <!-- e.g., background terminal, API call, etc. -->
- Process management: <!-- e.g., lifecycle changes, monitoring, etc. -->
- Resource implications: <!-- e.g., concurrent process limits, memory, etc. -->

### Claude Subagent Impact

<!-- How does this affect Claude subagent behavior? -->

- Agent types affected: <!-- e.g., Explore, Plan, code-reviewer, custom, etc. -->
- Context handling: <!-- e.g., context passing, isolation, sharing, etc. -->
- Communication pattern: <!-- e.g., stdout, files, API, message queue, etc. -->

### Cross-Model Coordination

<!-- How do Codex and Claude coordinate in this change? -->

```
Codex (Orchestrator)
  └─→ [Describe the flow]
       └─→ Claude Subagent(s)
            └─→ [Result aggregation path]
```

## Testing & Validation

<!-- How was this tested? What validation was performed? -->

- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Manual testing performed
- [ ] Proof-of-concept validated
- [ ] Benchmarks run (latency, throughput, reliability)
- [ ] No testing required (documentation only)

### Test Evidence

<!-- Paste relevant test output, benchmarks, or screenshots -->

```
# Test results or benchmark data
```

## Breaking Changes

<!-- List any breaking changes and migration path -->

- [ ] **No breaking changes**
- [ ] **Breaking changes** (describe below)

<!-- If breaking, describe:
1. What breaks
2. Who is affected
3. Migration path
-->

## Documentation

- [ ] README updated
- [ ] Architecture docs updated
- [ ] API documentation updated
- [ ] Inline code comments added
- [ ] PRP findings documented
- [ ] No documentation needed

## BMAD Framework Alignment

<!-- Check the phase this PR relates to -->

- [ ] **Phase 1: Analysis** - Research, discovery, product brief
- [ ] **Phase 2: Planning** - PRD, tech-spec, UX design
- [ ] **Phase 3: Solutioning** - Architecture, epics, stories
- [ ] **Phase 4: Implementation** - Development, testing, review

## Checklist

- [ ] Code follows project conventions
- [ ] Self-review completed
- [ ] No secrets or credentials committed
- [ ] No unnecessary files included
- [ ] Commit messages are descriptive
- [ ] PR title is clear and descriptive

## Reviewer Guidance

<!-- Help reviewers focus on what matters most -->

**Pay special attention to:**
-

**Less critical / straightforward:**
-

## Related Issues/PRs

<!-- Link related issues or PRs -->

- Closes #
- Related to #
- Depends on #

---

<!--
Keywords for indexing: codex-subagents, claude, anthropic, openai, multi-agent,
orchestration, background-terminal, parallel-execution, swarm, context-sync,
task-distribution, cross-model, ai-agents, developer-tools, bmad-framework
-->
