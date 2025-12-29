---
stepsCompleted: [1, 2, 3, 4]
inputDocuments:
  - prp-codex-subagents-integration.md (v3.0)
  - GPT-5 Pro Research Findings (Q1-Q8 + ADR)
date: 2025-12-29
author: Aip0rt
type: implementation-prp
status: ready-for-dev
assignedTeam: Claw Dev
version: 1.0
phase: Implementation
researchBy: GPT-5 Pro Team
---

# Implementation PRP: Codex-Claude Subagent Harness

## Executive Summary

This document translates the GPT-5 Pro team's research findings into actionable implementation specifications for the Claw Dev team. The research phase is complete. This is the **build spec**.

**Architecture Decision:** Harness-Orchestrated Dual-Agent System using:
- **Codex CLI** as orchestrator and verifier
- **Claude Code CLI** as executor
- **Filesystem** as communication layer
- **Node.js** as harness runtime

**Repository:** https://github.com/airplne/codex-subagents

---

## Research Findings Summary

### Architecture Selection: Option D (Hybrid CLI + File)

| Option | Weighted Score | Selected |
|--------|---------------|----------|
| A: Direct CLI | 168 | No |
| B: API-Based | 162 | No |
| C: Queue Workers | 158 | No |
| **D: Hybrid CLI + File** | **188** | **Yes** |

**Rationale:** Best balance of simplicity, control, and Tier-1 compliance. CLI retains file/code execution capabilities. File-based coordination provides observability without external dependencies.

### Measured Performance Baselines

| Metric | Value | Notes |
|--------|-------|-------|
| **Max Concurrent Processes** | 5+ | No hard limit; resource-constrained |
| **Spawn Latency (avg)** | ~1,200ms | Range: 800ms - 2,100ms |
| **Spawn Latency (p95)** | ~2,100ms | Under max load: ~1,500ms |
| **Process Timeout** | None | 5-hour OpenAI session limit only |
| **Memory per Process** | 100-200MB | Peaks at ~180MB under load |
| **Stdout Buffer** | No truncation | Tested up to 100KB+ |

### Task Distribution Pattern: Execute-Verify Pairs (Pattern B)

**Default Pattern:** For each task, spawn Claude executor → immediately spawn Codex verifier when Claude finishes.

**Sequential Pattern:** For dependent tasks, use Pipeline with Checkpoints (Pattern C).

---

## Implementation Specifications

### 1. Project Structure

```
codex-subagents/
├── harness/
│   ├── bin/
│   │   ├── run.sh              # Main entry point
│   │   ├── status.sh           # Query progress
│   │   ├── init.sh             # Initialize harness
│   │   └── stop.sh             # Graceful shutdown
│   ├── config/
│   │   ├── agents.yaml         # Agent definitions
│   │   ├── security.yaml       # Allowlist/blocklist
│   │   └── harness.yaml        # Main config
│   ├── src/
│   │   ├── orchestrator.js     # Main loop
│   │   ├── task-queue.js       # Queue management
│   │   ├── executor.js         # Claude spawner
│   │   ├── verifier.js         # Codex spawner
│   │   ├── aggregator.js       # Consensus logic
│   │   ├── security.js         # Security checks
│   │   ├── progress.js         # Status tracking
│   │   └── config.js           # Config loader
│   ├── tasks/
│   │   ├── queue.json          # Pending tasks
│   │   ├── in-progress.json    # Active tasks
│   │   ├── completed.json      # Done tasks
│   │   └── failed.json         # Failed tasks
│   ├── logs/
│   │   ├── orchestrator.log
│   │   ├── claude-agent-{id}.log
│   │   └── codex-verifier-{id}.log
│   ├── metrics/
│   │   ├── timing.json
│   │   ├── consensus.json
│   │   └── throughput.json
│   ├── context/
│   │   ├── shared-state.json
│   │   └── task-{id}/          # Per-task context
│   └── output/
│       └── task-{id}/          # Per-task outputs
├── examples/
│   └── simple-task.yaml        # Example task spec
├── docs/
│   └── research-log.md
└── package.json
```

---

### 2. Component Specifications

#### 2.1 Orchestrator (`orchestrator.js`)

**Responsibilities:**
1. Load config and initialize task queue
2. Main event loop: monitor queue, spawn agents, handle completions
3. Enforce concurrency limits (`max_concurrent_claude`, `max_concurrent_codex`)
4. Trigger verification after execution completes
5. Handle retries and aggregation decisions
6. Cleanup on exit (kill child processes)

**State Machine:**

```
┌─────────┐    spawn     ┌────────────┐   complete   ┌───────────┐
│ PENDING │─────────────→│ EXECUTING  │─────────────→│ VERIFYING │
└─────────┘              └────────────┘              └───────────┘
     ↑                         │                          │
     │                         │ fail                     │ pass
     │                         ↓                          ↓
     │                   ┌──────────┐              ┌───────────┐
     └───────────────────│  RETRY   │              │ COMPLETED │
        (if retries)     └──────────┘              └───────────┘
                               │ no retries
                               ↓
                         ┌──────────┐
                         │  FAILED  │
                         └──────────┘
```

**Implementation:**

```javascript
// orchestrator.js - Core structure
const { spawn } = require('child_process');
const EventEmitter = require('events');

class Orchestrator extends EventEmitter {
  constructor(config) {
    super();
    this.config = config;
    this.taskQueue = new TaskQueue();
    this.security = new Security(config.security);
    this.progress = new Progress();
    this.activeAgents = { claude: [], codex: [] };
  }

  async start() {
    this.running = true;
    while (this.running) {
      await this.tick();
      await this.sleep(100); // 100ms polling interval
    }
  }

  async tick() {
    // Check for completed processes
    this.checkCompletions();

    // Spawn new agents if slots available
    while (this.canSpawnClaude() && this.taskQueue.hasReadyTask()) {
      const task = this.taskQueue.nextTask();
      await this.spawnClaudeExecutor(task);
    }
  }

  canSpawnClaude() {
    return this.activeAgents.claude.length < this.config.orchestrator.max_concurrent_claude;
  }

  async spawnClaudeExecutor(task) {
    const executor = new Executor(task, this.config, this.security);
    const proc = await executor.spawn();

    this.activeAgents.claude.push({ task, proc, executor });
    this.taskQueue.markInProgress(task.id);
    this.progress.update(task.id, 'executing');

    proc.on('exit', (code) => this.onClaudeComplete(task, code));
  }

  async onClaudeComplete(task, exitCode) {
    // Remove from active
    this.activeAgents.claude = this.activeAgents.claude.filter(a => a.task.id !== task.id);

    if (exitCode === 0) {
      // Spawn verifier
      await this.spawnCodexVerifier(task);
    } else {
      this.handleFailure(task, 'executor_crash');
    }
  }

  async spawnCodexVerifier(task) {
    const verifier = new Verifier(task, this.config, this.security);
    const proc = await verifier.spawn();

    this.activeAgents.codex.push({ task, proc, verifier });
    this.progress.update(task.id, 'verifying');

    proc.on('exit', (code) => this.onVerifierComplete(task, code));
  }

  async onVerifierComplete(task, exitCode) {
    this.activeAgents.codex = this.activeAgents.codex.filter(a => a.task.id !== task.id);

    const result = await this.aggregator.analyze(task);

    if (result.passed) {
      this.taskQueue.markCompleted(task.id, result);
      this.progress.update(task.id, 'completed');
    } else if (task.attempts < this.config.agents['claude-executor'].retries) {
      this.taskQueue.requeueForRetry(task.id, result.feedback);
    } else {
      this.taskQueue.markFailed(task.id, result);
      this.progress.update(task.id, 'failed');
    }
  }

  async shutdown() {
    this.running = false;
    // Kill all active processes
    [...this.activeAgents.claude, ...this.activeAgents.codex].forEach(a => {
      a.proc.kill('SIGTERM');
    });
  }
}
```

---

#### 2.2 Executor (`executor.js`) - Claude Agent Spawner

**Claude CLI Invocation Pattern:**

```bash
claude -p "<prompt>" \
  --allowedTools "Bash,Read,Edit" \
  --output-format text
```

**Implementation:**

```javascript
// executor.js
const { spawn } = require('child_process');
const fs = require('fs');
const path = require('path');

class Executor {
  constructor(task, config, security) {
    this.task = task;
    this.config = config;
    this.security = security;
    this.logPath = `harness/logs/claude-agent-${task.id}.log`;
  }

  buildPrompt() {
    // Load task spec
    const spec = fs.readFileSync(this.task.specPath, 'utf8');

    // Load any context files
    let context = '';
    if (this.task.contextFiles) {
      this.task.contextFiles.forEach(f => {
        if (fs.existsSync(f)) {
          context += `\n--- ${f} ---\n${fs.readFileSync(f, 'utf8')}\n`;
        }
      });
    }

    // Include retry feedback if this is a retry
    let retryContext = '';
    if (this.task.attempts > 0 && this.task.lastFeedback) {
      retryContext = `\n\n**PREVIOUS ATTEMPT FAILED. FIX THESE ISSUES:**\n${this.task.lastFeedback}\n`;
    }

    return `You are a software engineer. Complete the following task.

## Task Specification
${spec}

## Context Files
${context}
${retryContext}
## Instructions
1. Read and understand the requirements
2. Implement the solution
3. Write any necessary tests
4. Ensure code is clean and well-documented

Output your work to the designated output directory.`;
  }

  async spawn() {
    const prompt = this.buildPrompt();
    const workDir = `harness/context/task-${this.task.id}`;

    // Ensure work directory exists
    fs.mkdirSync(workDir, { recursive: true });

    const args = [
      '-p', prompt,
      '--allowedTools', 'Bash,Read,Edit',
      '--output-format', 'text'
    ];

    const proc = spawn('claude', args, {
      cwd: workDir,
      env: {
        ...process.env,
        ANTHROPIC_API_KEY: process.env.ANTHROPIC_API_KEY
      }
    });

    // Log stdout and stderr
    const logStream = fs.createWriteStream(this.logPath);
    proc.stdout.pipe(logStream);
    proc.stderr.pipe(logStream);

    // Security monitoring
    proc.stdout.on('data', (data) => {
      const output = data.toString();
      this.security.scanOutput(output, this.task.id);
    });

    return proc;
  }
}
```

---

#### 2.3 Verifier (`verifier.js`) - Codex Verification Agent

**Codex CLI Invocation Pattern:**

```bash
codex exec "<verification prompt>" \
  --allowedTools "Read,Bash"
```

**Implementation:**

```javascript
// verifier.js
const { spawn } = require('child_process');
const fs = require('fs');

class Verifier {
  constructor(task, config, security) {
    this.task = task;
    this.config = config;
    this.security = security;
    this.logPath = `harness/logs/codex-verifier-${task.id}.log`;
    this.resultPath = `harness/output/task-${task.id}/verification.json`;
  }

  buildPrompt() {
    const outputDir = `harness/output/task-${this.task.id}`;
    const specPath = this.task.specPath;

    // Load verification criteria
    const criteria = this.task.verificationCriteria || [
      'All tests pass',
      'No hardcoded secrets',
      'Proper error handling',
      'Code follows project conventions'
    ];

    return `You are a code reviewer. Verify the implementation in ${outputDir} against the task specification.

## Task Specification
${fs.readFileSync(specPath, 'utf8')}

## Verification Criteria
${criteria.map((c, i) => `${i + 1}. ${c}`).join('\n')}

## Instructions
1. Read the implementation files in ${outputDir}
2. Run any tests if present (npm test or similar)
3. Check each verification criterion
4. Report your findings

## Output Format
Respond with a JSON object:
{
  "passed": true/false,
  "score": 0.0-1.0,
  "issues": ["issue1", "issue2"],
  "comments": "Overall assessment"
}`;
  }

  async spawn() {
    const prompt = this.buildPrompt();

    const args = [
      'exec', prompt,
      '--allowedTools', 'Read,Bash'
    ];

    const proc = spawn('codex', args, {
      cwd: process.cwd(),
      env: {
        ...process.env,
        OPENAI_API_KEY: process.env.OPENAI_API_KEY
      }
    });

    // Capture output for result parsing
    let output = '';
    const logStream = fs.createWriteStream(this.logPath);

    proc.stdout.on('data', (data) => {
      output += data.toString();
      logStream.write(data);
    });
    proc.stderr.pipe(logStream);

    proc.on('exit', () => {
      // Try to parse JSON result from output
      try {
        const jsonMatch = output.match(/\{[\s\S]*"passed"[\s\S]*\}/);
        if (jsonMatch) {
          const result = JSON.parse(jsonMatch[0]);
          fs.writeFileSync(this.resultPath, JSON.stringify(result, null, 2));
        }
      } catch (e) {
        // If no JSON, create a basic result
        fs.writeFileSync(this.resultPath, JSON.stringify({
          passed: false,
          score: 0,
          issues: ['Could not parse verification output'],
          raw: output
        }, null, 2));
      }
    });

    return proc;
  }
}
```

---

#### 2.4 Task Queue (`task-queue.js`)

**Task Schema:**

```javascript
{
  id: "task-001",
  name: "Implement user authentication",
  specPath: "examples/auth-task.yaml",
  priority: "high",  // high, medium, low
  dependencies: [],  // task IDs that must complete first
  status: "pending", // pending, in-progress, completed, failed
  attempts: 0,
  maxRetries: 2,
  contextFiles: ["src/routes/index.js"],
  verificationCriteria: ["All tests pass", "No hardcoded secrets"],
  createdAt: "2025-12-29T10:00:00Z",
  startedAt: null,
  completedAt: null,
  lastFeedback: null  // Verifier feedback for retries
}
```

**Implementation:**

```javascript
// task-queue.js
const fs = require('fs');

class TaskQueue {
  constructor() {
    this.queuePath = 'harness/tasks/queue.json';
    this.inProgressPath = 'harness/tasks/in-progress.json';
    this.completedPath = 'harness/tasks/completed.json';
    this.failedPath = 'harness/tasks/failed.json';
    this.load();
  }

  load() {
    this.queue = this.readJson(this.queuePath, []);
    this.inProgress = this.readJson(this.inProgressPath, []);
    this.completed = this.readJson(this.completedPath, []);
    this.failed = this.readJson(this.failedPath, []);
  }

  readJson(path, defaultValue) {
    try {
      return JSON.parse(fs.readFileSync(path, 'utf8'));
    } catch {
      return defaultValue;
    }
  }

  save() {
    fs.writeFileSync(this.queuePath, JSON.stringify(this.queue, null, 2));
    fs.writeFileSync(this.inProgressPath, JSON.stringify(this.inProgress, null, 2));
    fs.writeFileSync(this.completedPath, JSON.stringify(this.completed, null, 2));
    fs.writeFileSync(this.failedPath, JSON.stringify(this.failed, null, 2));
  }

  hasReadyTask() {
    return this.queue.some(t => this.isDependenciesMet(t));
  }

  isDependenciesMet(task) {
    if (!task.dependencies || task.dependencies.length === 0) return true;
    return task.dependencies.every(depId =>
      this.completed.some(c => c.id === depId)
    );
  }

  nextTask() {
    // Sort by priority (high > medium > low), then by creation time
    const priorityOrder = { high: 0, medium: 1, low: 2 };
    const ready = this.queue
      .filter(t => this.isDependenciesMet(t))
      .sort((a, b) => {
        const pDiff = priorityOrder[a.priority] - priorityOrder[b.priority];
        if (pDiff !== 0) return pDiff;
        return new Date(a.createdAt) - new Date(b.createdAt);
      });

    return ready[0] || null;
  }

  markInProgress(taskId) {
    const task = this.queue.find(t => t.id === taskId);
    if (task) {
      task.status = 'in-progress';
      task.startedAt = new Date().toISOString();
      task.attempts++;
      this.queue = this.queue.filter(t => t.id !== taskId);
      this.inProgress.push(task);
      this.save();
    }
  }

  markCompleted(taskId, result) {
    const task = this.inProgress.find(t => t.id === taskId);
    if (task) {
      task.status = 'completed';
      task.completedAt = new Date().toISOString();
      task.result = result;
      this.inProgress = this.inProgress.filter(t => t.id !== taskId);
      this.completed.push(task);
      this.save();
    }
  }

  markFailed(taskId, result) {
    const task = this.inProgress.find(t => t.id === taskId);
    if (task) {
      task.status = 'failed';
      task.completedAt = new Date().toISOString();
      task.result = result;
      this.inProgress = this.inProgress.filter(t => t.id !== taskId);
      this.failed.push(task);
      this.save();
    }
  }

  requeueForRetry(taskId, feedback) {
    const task = this.inProgress.find(t => t.id === taskId);
    if (task) {
      task.status = 'pending';
      task.lastFeedback = feedback;
      this.inProgress = this.inProgress.filter(t => t.id !== taskId);
      this.queue.push(task);
      this.save();
    }
  }

  addTask(taskSpec) {
    const task = {
      id: `task-${Date.now()}`,
      ...taskSpec,
      status: 'pending',
      attempts: 0,
      createdAt: new Date().toISOString()
    };
    this.queue.push(task);
    this.save();
    return task;
  }
}
```

---

#### 2.5 Security (`security.js`)

**Implementation:**

```javascript
// security.js
const fs = require('fs');
const yaml = require('js-yaml');

class Security {
  constructor(configPath = 'harness/config/security.yaml') {
    this.config = yaml.load(fs.readFileSync(configPath, 'utf8'));
    this.auditLog = fs.createWriteStream('harness/logs/audit.log', { flags: 'a' });
  }

  isCommandAllowed(command) {
    const cmd = command.trim().split(' ')[0];

    // Check blocklist first
    for (const blocked of this.config.blocked_commands) {
      if (command.includes(blocked)) {
        return { allowed: false, reason: `Blocked command: ${blocked}` };
      }
    }

    // Check allowlist
    if (this.config.allowed_commands.includes(cmd)) {
      return { allowed: true };
    }

    return { allowed: false, reason: `Command not in allowlist: ${cmd}` };
  }

  isPathAllowed(filePath) {
    // Check blocked paths
    for (const blocked of this.config.filesystem.blocked_paths) {
      if (filePath.startsWith(blocked)) {
        return { allowed: false, reason: `Blocked path: ${blocked}` };
      }
    }

    // Check allowed paths
    for (const allowed of this.config.filesystem.allowed_paths) {
      if (filePath.startsWith(allowed)) {
        return { allowed: true };
      }
    }

    return { allowed: false, reason: `Path not in allowlist: ${filePath}` };
  }

  scanOutput(output, taskId) {
    // Look for command executions in output
    const bashMatches = output.match(/(?:Bash|bash|shell|terminal):\s*(.+)/gi);
    if (bashMatches) {
      bashMatches.forEach(match => {
        const cmd = match.replace(/(?:Bash|bash|shell|terminal):\s*/i, '');
        const check = this.isCommandAllowed(cmd);

        if (!check.allowed) {
          this.logViolation(taskId, 'command', cmd, check.reason);
          // In production, could kill the process here
        } else {
          this.logAction(taskId, 'command', cmd);
        }
      });
    }
  }

  sanitizeSecrets(text) {
    // Mask API keys and common secret patterns
    return text
      .replace(/sk-[a-zA-Z0-9]{32,}/g, '[REDACTED_API_KEY]')
      .replace(/ANTHROPIC_API_KEY=\S+/g, 'ANTHROPIC_API_KEY=[REDACTED]')
      .replace(/OPENAI_API_KEY=\S+/g, 'OPENAI_API_KEY=[REDACTED]');
  }

  logAction(taskId, type, action) {
    const entry = {
      timestamp: new Date().toISOString(),
      taskId,
      type,
      action: this.sanitizeSecrets(action),
      status: 'allowed'
    };
    this.auditLog.write(JSON.stringify(entry) + '\n');
  }

  logViolation(taskId, type, action, reason) {
    const entry = {
      timestamp: new Date().toISOString(),
      taskId,
      type,
      action: this.sanitizeSecrets(action),
      status: 'blocked',
      reason
    };
    this.auditLog.write(JSON.stringify(entry) + '\n');
    console.error(`[SECURITY] Violation in ${taskId}: ${reason}`);
  }
}
```

---

#### 2.6 Aggregator (`aggregator.js`)

**Implementation:**

```javascript
// aggregator.js
const fs = require('fs');

class Aggregator {
  constructor(config) {
    this.config = config;
    this.consensusThreshold = config.orchestrator.consensus_threshold || 0.8;
  }

  async analyze(task) {
    const resultPath = `harness/output/task-${task.id}/verification.json`;

    try {
      const result = JSON.parse(fs.readFileSync(resultPath, 'utf8'));

      // Log to metrics
      this.logConsensus(task.id, result);

      // Decision logic
      if (result.passed && result.score >= this.consensusThreshold) {
        return {
          passed: true,
          score: result.score,
          feedback: null,
          decision: 'SHIP'
        };
      } else if (result.issues && result.issues.length > 0) {
        return {
          passed: false,
          score: result.score,
          feedback: result.issues.join('\n'),
          decision: task.attempts < task.maxRetries ? 'RETRY' : 'ESCALATE'
        };
      } else {
        return {
          passed: false,
          score: result.score || 0,
          feedback: result.comments || 'Verification failed without specific issues',
          decision: 'ESCALATE'
        };
      }
    } catch (e) {
      return {
        passed: false,
        score: 0,
        feedback: `Failed to parse verification result: ${e.message}`,
        decision: 'ESCALATE'
      };
    }
  }

  logConsensus(taskId, result) {
    const metricsPath = 'harness/metrics/consensus.json';
    let metrics = [];

    try {
      metrics = JSON.parse(fs.readFileSync(metricsPath, 'utf8'));
    } catch {}

    metrics.push({
      timestamp: new Date().toISOString(),
      taskId,
      score: result.score,
      passed: result.passed,
      issueCount: result.issues?.length || 0
    });

    fs.writeFileSync(metricsPath, JSON.stringify(metrics, null, 2));
  }
}
```

---

#### 2.7 Progress Store (`progress.js`)

**Implementation:**

```javascript
// progress.js
const fs = require('fs');

class Progress {
  constructor() {
    this.progressPath = 'harness/metrics/progress.json';
    this.load();
  }

  load() {
    try {
      this.data = JSON.parse(fs.readFileSync(this.progressPath, 'utf8'));
    } catch {
      this.data = {
        started: new Date().toISOString(),
        tasks: {}
      };
    }
  }

  save() {
    fs.writeFileSync(this.progressPath, JSON.stringify(this.data, null, 2));
  }

  update(taskId, status, details = {}) {
    this.data.tasks[taskId] = {
      status,
      updatedAt: new Date().toISOString(),
      ...details
    };
    this.save();
  }

  getSummary() {
    const tasks = Object.values(this.data.tasks);
    return {
      total: tasks.length,
      pending: tasks.filter(t => t.status === 'pending').length,
      executing: tasks.filter(t => t.status === 'executing').length,
      verifying: tasks.filter(t => t.status === 'verifying').length,
      completed: tasks.filter(t => t.status === 'completed').length,
      failed: tasks.filter(t => t.status === 'failed').length
    };
  }
}
```

---

### 3. Configuration Files

#### 3.1 `harness/config/harness.yaml`

```yaml
# Harness main configuration
version: "1.0"

orchestrator:
  max_concurrent_claude: 3
  max_concurrent_codex: 3
  poll_interval_ms: 100
  consensus_threshold: 0.8
  retry_backoff: exponential

paths:
  tasks: "./harness/tasks"
  logs: "./harness/logs"
  metrics: "./harness/metrics"
  context: "./harness/context"
  output: "./harness/output"

logging:
  level: "info"  # debug, info, warn, error
  include_timestamps: true

git_integration:
  enabled: false  # Enable for worktree isolation
  auto_commit: false
  branch_prefix: "harness-"
```

#### 3.2 `harness/config/agents.yaml`

```yaml
# Agent definitions
agents:
  claude-executor:
    type: claude
    role: executor
    command: claude
    args:
      - "-p"
      - "{prompt}"
      - "--allowedTools"
      - "Bash,Read,Edit"
      - "--output-format"
      - "text"
    timeout: 300000  # 5 minutes
    retries: 2
    env:
      ANTHROPIC_API_KEY: "${ANTHROPIC_API_KEY}"

  codex-verifier:
    type: codex
    role: verifier
    command: codex
    args:
      - "exec"
      - "{prompt}"
      - "--allowedTools"
      - "Read,Bash"
    timeout: 180000  # 3 minutes
    retries: 1
    env:
      OPENAI_API_KEY: "${OPENAI_API_KEY}"
```

#### 3.3 `harness/config/security.yaml`

```yaml
# Security configuration
allowed_commands:
  - node
  - npm
  - npx
  - git
  - ls
  - cat
  - mkdir
  - cp
  - mv
  - echo
  - pwd
  - cd
  - jest
  - mocha
  - pytest

blocked_commands:
  - "rm -rf"
  - "rm -r /"
  - sudo
  - su
  - curl
  - wget
  - nc
  - netcat
  - chmod 777
  - "> /dev"

filesystem:
  allowed_paths:
    - "./src"
    - "./tests"
    - "./harness/context"
    - "./harness/output"
    - "./node_modules"
  blocked_paths:
    - "/etc"
    - "/usr"
    - "/var"
    - "~/.ssh"
    - "~/.aws"
    - "~/.config"
    - ".git/config"
    - ".env"

api_limits:
  openai_rpm: 60
  anthropic_rpm: 60

audit:
  enabled: true
  log_path: "./harness/logs/audit.log"
  include_outputs: false
  sanitize_secrets: true
```

---

### 4. CLI Scripts

#### 4.1 `harness/bin/run.sh`

```bash
#!/bin/bash
set -e

# Harness entry point
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
HARNESS_ROOT="$(dirname "$SCRIPT_DIR")"

# Check dependencies
command -v node >/dev/null 2>&1 || { echo "Node.js required but not found"; exit 1; }
command -v claude >/dev/null 2>&1 || { echo "Claude CLI required but not found"; exit 1; }
command -v codex >/dev/null 2>&1 || { echo "Codex CLI required but not found"; exit 1; }

# Check API keys
[ -z "$ANTHROPIC_API_KEY" ] && { echo "ANTHROPIC_API_KEY not set"; exit 1; }
[ -z "$OPENAI_API_KEY" ] && { echo "OPENAI_API_KEY not set"; exit 1; }

# Parse arguments
TASK_FILE=""
while [[ $# -gt 0 ]]; do
  case $1 in
    --task)
      TASK_FILE="$2"
      shift 2
      ;;
    --help)
      echo "Usage: run.sh --task <task.yaml>"
      exit 0
      ;;
    *)
      echo "Unknown option: $1"
      exit 1
      ;;
  esac
done

# Start orchestrator
cd "$HARNESS_ROOT/.."
node harness/src/orchestrator.js ${TASK_FILE:+--task "$TASK_FILE"}
```

#### 4.2 `harness/bin/status.sh`

```bash
#!/bin/bash
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
HARNESS_ROOT="$(dirname "$SCRIPT_DIR")"

cd "$HARNESS_ROOT/.."

echo "=== Harness Status ==="
echo ""

# Task counts
if [ -f "harness/tasks/queue.json" ]; then
  PENDING=$(jq length harness/tasks/queue.json 2>/dev/null || echo 0)
else
  PENDING=0
fi

if [ -f "harness/tasks/in-progress.json" ]; then
  IN_PROGRESS=$(jq length harness/tasks/in-progress.json 2>/dev/null || echo 0)
else
  IN_PROGRESS=0
fi

if [ -f "harness/tasks/completed.json" ]; then
  COMPLETED=$(jq length harness/tasks/completed.json 2>/dev/null || echo 0)
else
  COMPLETED=0
fi

if [ -f "harness/tasks/failed.json" ]; then
  FAILED=$(jq length harness/tasks/failed.json 2>/dev/null || echo 0)
else
  FAILED=0
fi

echo "Tasks: $PENDING pending | $IN_PROGRESS running | $COMPLETED completed | $FAILED failed"
echo ""

# Recent activity
echo "=== Recent Logs ==="
if [ -f "harness/logs/orchestrator.log" ]; then
  tail -5 harness/logs/orchestrator.log
else
  echo "(no logs yet)"
fi
```

#### 4.3 `harness/bin/init.sh`

```bash
#!/bin/bash
set -e

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
HARNESS_ROOT="$(dirname "$SCRIPT_DIR")"

cd "$HARNESS_ROOT/.."

echo "Initializing harness..."

# Create directory structure
mkdir -p harness/tasks
mkdir -p harness/logs
mkdir -p harness/metrics
mkdir -p harness/context
mkdir -p harness/output

# Initialize empty JSON files
echo "[]" > harness/tasks/queue.json
echo "[]" > harness/tasks/in-progress.json
echo "[]" > harness/tasks/completed.json
echo "[]" > harness/tasks/failed.json
echo "[]" > harness/metrics/timing.json
echo "[]" > harness/metrics/consensus.json

# Install dependencies if package.json exists
if [ -f "package.json" ]; then
  npm install
fi

echo "Harness initialized successfully!"
echo "Run: ./harness/bin/run.sh --task <task.yaml>"
```

---

### 5. Context Synchronization Protocol

Based on research findings, implement **Shared Filesystem + Orchestrator Injection**:

#### 5.1 Task Context Structure

```
harness/context/task-{id}/
├── spec.txt              # Task specification (copied from YAML)
├── context/              # Input files for the task
│   ├── file1.js
│   └── file2.js
├── shared-state.json     # Global state snapshot
└── workdir/              # Claude's working directory
    └── (agent operates here)

harness/output/task-{id}/
├── (files created/modified by Claude)
├── verification.json     # Codex verifier output
└── summary.md            # Orchestrator-generated summary
```

#### 5.2 Context Flow

```
1. Orchestrator creates task-{id}/ directory
2. Copies task spec to spec.txt
3. Copies relevant context files to context/
4. Creates workdir/ for Claude to operate in
5. Snapshots current shared-state.json

6. Claude executes in workdir/
7. Claude may read from context/, writes to workdir/

8. Orchestrator copies results to output/task-{id}/
9. Creates diff/summary if applicable

10. Verifier reads from output/task-{id}/
11. Verifier writes verification.json

12. Orchestrator updates shared-state.json with results
```

---

### 6. Success Metrics

Implement measurement collection in `harness/metrics/`:

| Metric | Target | Measurement Point |
|--------|--------|-------------------|
| Spawn Latency | < 2,000ms | Time from spawn() to first stdout |
| Concurrent Agents | ≥ 3 each | Active process count |
| Verification Latency | < 3,000ms | Verifier process duration |
| Task Success Rate | > 95% | completed / (completed + failed) |
| Consensus Score | > 0.8 avg | Verifier scores |
| Harness Overhead | < 500ms | Time spent in orchestrator logic |

---

### 7. Error Handling

#### 7.1 Retry Logic

```javascript
// In orchestrator.js
handleFailure(task, reason) {
  if (reason === 'executor_crash' || reason === 'verification_failed') {
    if (task.attempts < this.config.agents['claude-executor'].retries) {
      console.log(`[Harness] Retrying ${task.id} (attempt ${task.attempts + 1})`);
      this.taskQueue.requeueForRetry(task.id, reason);
    } else {
      console.log(`[Harness] ${task.id} failed after ${task.attempts} attempts`);
      this.taskQueue.markFailed(task.id, { reason });
    }
  } else if (reason === 'security_violation') {
    // No retry for security violations
    console.error(`[Harness] ${task.id} terminated due to security violation`);
    this.taskQueue.markFailed(task.id, { reason });
  }
}
```

#### 7.2 Graceful Shutdown

```javascript
// In orchestrator.js
setupSignalHandlers() {
  process.on('SIGINT', () => this.shutdown());
  process.on('SIGTERM', () => this.shutdown());
  process.on('uncaughtException', (err) => {
    console.error('[Harness] Uncaught exception:', err);
    this.shutdown();
    process.exit(1);
  });
}

async shutdown() {
  console.log('[Harness] Shutting down...');
  this.running = false;

  // Kill all child processes
  const allAgents = [...this.activeAgents.claude, ...this.activeAgents.codex];
  for (const agent of allAgents) {
    try {
      agent.proc.kill('SIGTERM');
    } catch {}
  }

  // Wait briefly for cleanup
  await new Promise(r => setTimeout(r, 1000));

  console.log('[Harness] Shutdown complete');
}
```

---

### 8. Example Task Specification

#### `examples/simple-task.yaml`

```yaml
id: auth-feature
name: Implement JWT Authentication
priority: high
dependencies: []

spec: |
  Implement JWT-based user authentication with:
  1. Login endpoint: POST /api/auth/login
     - Accept { email, password }
     - Return { token, refreshToken, expiresIn }
  2. Token validation middleware
     - Verify JWT signature
     - Attach user to request
  3. Refresh token endpoint: POST /api/auth/refresh
     - Accept { refreshToken }
     - Return new { token, expiresIn }

contextFiles:
  - src/routes/index.js
  - src/config/auth.js

expectedOutputs:
  - src/routes/auth.js
  - src/middleware/jwt.js
  - tests/auth.test.js

verificationCriteria:
  - All tests pass when running `npm test`
  - No hardcoded secrets or keys
  - Proper error handling for invalid credentials
  - Token expiration is configurable
  - Follows project coding conventions
```

---

### 9. Development Phases

#### Phase 1: Core Infrastructure (Days 1-3)
- [ ] Set up project structure
- [ ] Implement TaskQueue
- [ ] Implement Config loader
- [ ] Implement Progress store
- [ ] Create CLI scripts (init, run, status)

#### Phase 2: Agent Integration (Days 4-6)
- [ ] Implement Executor (Claude spawner)
- [ ] Implement Verifier (Codex spawner)
- [ ] Test CLI invocations independently
- [ ] Capture and parse outputs

#### Phase 3: Orchestration (Days 7-9)
- [ ] Implement Orchestrator main loop
- [ ] Implement concurrency control
- [ ] Implement dependency handling
- [ ] Implement Aggregator

#### Phase 4: Security & Polish (Days 10-12)
- [ ] Implement Security module
- [ ] Add audit logging
- [ ] Add metrics collection
- [ ] Error handling and retries

#### Phase 5: Testing & Documentation (Days 13-15)
- [ ] End-to-end testing with example tasks
- [ ] Performance benchmarking
- [ ] Write README and usage docs
- [ ] Create demo video/walkthrough

---

### 10. Acceptance Criteria

The implementation is complete when:

1. **Functional:**
   - [ ] `./harness/bin/init.sh` sets up the harness
   - [ ] `./harness/bin/run.sh --task examples/simple-task.yaml` executes successfully
   - [ ] Claude executor spawns and produces output
   - [ ] Codex verifier spawns and produces verification report
   - [ ] Tasks can be retried on failure
   - [ ] `./harness/bin/status.sh` shows accurate progress

2. **Performance:**
   - [ ] Spawn latency < 2,000ms (measured)
   - [ ] Runs 3+ concurrent agents without issues
   - [ ] Harness overhead < 500ms

3. **Security:**
   - [ ] Blocked commands are prevented
   - [ ] Audit log captures all actions
   - [ ] No secrets in logs

4. **Observability:**
   - [ ] All agent outputs logged
   - [ ] Metrics collected (timing, consensus)
   - [ ] Clear error messages on failures

---

## Appendix A: Research Sources

- [Codex CLI Registry (tessl.io)](https://tessl.io/registry/tessl/npm-openai--codex/0.39.0)
- [VS Code Background Agents](https://code.visualstudio.com/docs/copilot/agents/background-agents)
- [OpenAI Pro Plan Limits](https://community.openai.com/t/pro-plan-hit-5-hour-limit-twice-in-2h-and-1-5h-and-nearly-exhausted-weekly-cap-in-1-day-after-today-s-update/1364782)
- [Anthropic Effective Harnesses](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [Claude Code Headless Mode](https://code.claude.com/docs/en/headless)
- [Claude Code Changelog](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)

---

## Appendix B: ADR Summary

**Decision:** Harness-Orchestrated Dual-Agent System

**Context:** Need to orchestrate Codex and Claude agents for collaborative coding with verification.

**Choice:** Option D (Hybrid CLI + File Communication)

**Consequences:**
- ✅ Maximum use of CLI capabilities (file access, code execution)
- ✅ Full observability via filesystem
- ✅ Minimal external dependencies
- ✅ Easy debugging and replay
- ⚠️ Some process overhead vs API-only
- ⚠️ File I/O could bottleneck at scale

**Mitigations:**
- Use separate worktrees for parallel tasks
- Monitor resource usage
- Can migrate to queue-based if scaling needed

---

*Implementation PRP Generated: 2025-12-29*
*Based on: GPT-5 Pro Research Findings (Q1-Q8)*
*For: Claw Dev Team*
*BMAD Framework Version: 6.0.0-alpha.21*
