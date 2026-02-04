# OpenClaw RL Environment

**Version**: 3.1 (Scope: Environment Only)
**Goal**: Build a high-quality RL environment for tool-calling agents using OpenClaw's existing infrastructure. Provide tasks, tools, and reliable feedback signals for external training systems.

---

## Executive Summary

**Scope**: We are building the **RL environment**, not the training system.

This means:

- ✅ Episode management (reset, step, observe)
- ✅ Task definitions with verifiable success criteria
- ✅ Reward evaluation and feedback signals
- ✅ State management (initialization, reset, checkpointing)
- ✅ Trajectory logging (for external analysis)
- ❌ **NOT** building: Policy networks, RL algorithms, training loops

**Why**: The training system will be handled externally. We provide a clean, reliable environment that external trainers can interface with.

**Architecture**: OpenClaw plugin that exposes a Gym-compatible API for environment interaction.

---

## Simplified Architecture

```
┌─────────────────────────────────────────────────────┐
│  External Training System                           │
│  (Built by others - NOT our scope)                  │
│  - Policy networks, RL algorithms, etc.             │
└─────────────────┬───────────────────────────────────┘
                  │
                  │ Gym API (HTTP or gRPC)
                  │ - reset(task_id) → observation
                  │ - step(action) → obs, reward, done, info
                  │
┌─────────────────▼───────────────────────────────────┐
│  OpenClaw RL Environment Plugin                     │
│  (Our scope - what we're building)                  │
│                                                     │
│   ┌────────────────────────────────────────────┐    │
│   │  Episode Controller                        │    │
│   │  - Initialize episodes from tasks          │    │
│   │  - Execute actions via OpenClaw tools      │    │
│   │  - Track progress, timeouts                │    │
│   │  - Determine episode termination           │    │
│   └────────────────────────────────────────────┘    │
│                                                     │
│   ┌────────────────────────────────────────────┐    │
│   │  State Manager                             │    │
│   │  - Initialize environment per task         │    │
│   │  - Reset between episodes                  │    │
│   │  - Browser profile management              │    │
│   │  - File system snapshots                   │    │
│   │  - Database seeding                        │    │
│   └────────────────────────────────────────────┘    │
│                                                     │
│   ┌────────────────────────────────────────────┐    │
│   │  Task System                               │    │
│   │  - Load task definitions (YAML/JSON)       │    │
│   │  - Validate task specs                     │    │
│   │  - Initialize task-specific state          │    │
│   └────────────────────────────────────────────┘    │
│                                                     │
│   ┌────────────────────────────────────────────┐    │
│   │  Reward Evaluator                          │    │
│   │  - Verify success conditions               │    │
│   │  - Compute rewards                         │    │
│   │  - Provide failure diagnostics             │    │
│   └────────────────────────────────────────────┘    │
│                                                     │
│   ┌────────────────────────────────────────────┐    │
│   │  Trajectory Logger                         │    │
│   │  - Record (observation, action, reward)    │    │
│   │  - Store to JSONL/Parquet                  │    │
│   │  - For offline analysis/training           │    │
│   └────────────────────────────────────────────┘    │
│                                                     │
│   ┌────────────────────────────────────────────┐    │
│   │  API Server                                │    │
│   │  - Gym-compatible HTTP endpoints           │    │
│   │  - /reset, /step, /observe, /info          │    │
│   └────────────────────────────────────────────┘    │
└─────────────────┬───────────────────────────────────┘
                  │ Plugin hooks
┌─────────────────▼────────────────────────────────────┐
│  OpenClaw Core (Existing, Unchanged)                 │
│  - Multi-agent runtime                               │
│  - 50+ tools (browser, files, messaging)             │
│  - Session management                                │
│  - Tool execution                                    │
└──────────────────────────────────────────────────────┘
```

**Key Insight**: We're building the "game" that agents play, not the "player" that learns to play it.

---

## Implementation: Single Repository Approach

**Decision**: Implement as an **OpenClaw plugin** (TypeScript)

**Why this works now**:

- ✅ No language mismatch (no need for Python ML libraries)
- ✅ Direct access to OpenClaw tools and runtime
- ✅ Simple integration via plugin system
- ✅ All environment code stays in one place

**Location**: `openclaw/extensions/rl-environment/`

**No separate repository needed** since we're not building training infrastructure.

---

## Core Components (What We're Building)

### 1. Episode Controller

**Purpose**: Manage the episode lifecycle (initialize, step, terminate)

**Responsibilities**:

- Start new episodes from task definitions
- Execute agent actions through OpenClaw tool system
- Build observations after each action
- Track episode state (step count, time elapsed, resources used)
- Determine termination (success, failure, timeout, max steps)

**API Endpoints**:

```
POST /rl/reset
  Request: { task_id: string }
  Response: { observation: {...}, info: {...} }

POST /rl/step
  Request: { action: { tool: string, params: {...} } }
  Response: { observation: {...}, reward: number, done: boolean, info: {...} }

GET /rl/observe
  Response: { observation: {...} }

GET /rl/info
  Response: { step: number, max_steps: number, elapsed_ms: number, ... }
```

**Observation Structure**:

```yaml
observation:
  task_description: "Reserve a study room at Saratoga Library for Feb 15 at 2pm"
  tool_results: [...] # Recent tool outputs
  available_tools: ["browser", "file", "message", ...]
  metadata:
    step: 5
    remaining_steps: 45
    time_budget_ms: 240000
```

---

### 2. State Manager

**Purpose**: Initialize and reset environment state between episodes

**Initialization** (per task):

- Set up browser profiles (login sessions, cookies)
- Seed databases (SQLite, MongoDB, CSV files)
- Create initial files/directories
- Configure mock accounts for messaging apps

**Reset Strategies** (tiered for performance):

- **Level 1** (< 1s): Clear agent session, reset in-memory state
- **Level 2** (< 5s): Restore browser profile, reset filesystem overlay, reload DB
- **Level 3** (< 30s): Full container restart from snapshot

**State Snapshot Format**:

```yaml
snapshot:
  browser:
    profile: saratoga-library-logged-in
    cookies: [...]
    localStorage: { ... }
  filesystem:
    base_dir: /tmp/rl-env-12345
    overlay: overlayfs
  database:
    seed_file: data/seeds/library-catalog.sql
```

---

### 3. Task System

**Purpose**: Define, load, and validate tasks

**Task Definition Format**:

```yaml
id: library-room-booking
name: Reserve library study room
description: "Reserve a study room at Saratoga Library for Feb 15 at 2pm"

category: browser-multi-step
difficulty: 3

# Environment initialization
environment:
  browser:
    logged_in: true
    url: https://sccld.org
    profile: saratoga-library
  files: []
  databases: []

# Success criteria (all must pass)
success_conditions:
  - type: browser_url
    pattern: "**/confirmation*"

  - type: browser_element
    selector: ".success-message"
    text_contains: "reservation confirmed"

  - type: browser_element
    selector: ".room-details"
    text_contains: "February 15"

# Rewards
rewards:
  success: 1.0
  failure: -0.5
  step_penalty: -0.01
  efficiency_bonus:
    threshold_steps: 15
    bonus: 0.2

# Constraints
constraints:
  max_steps: 50
  timeout_ms: 300000
  tools_allowed: ["browser"]
```

**Task Categories**:

- Browser automation (navigation, form filling, information extraction)
- File operations (create, read, transform, search)
- Messaging (send, monitor, respond)
- Multi-tool composition (coordinating across tools)

**Task Dataset Structure**:

```
tasks/
├── browser/
│   ├── simple/           # Single-page actions
│   ├── navigation/       # Multi-page workflows
│   └── forms/            # Complex form filling
├── files/
│   ├── simple/           # Basic CRUD
│   ├── processing/       # Parse, transform
│   └── search/           # Find, filter
├── messaging/
│   ├── simple/           # Send messages
│   └── monitoring/       # Watch, respond
└── multi-tool/
    └── workflows/        # Cross-tool tasks
```

**Target**: 50-100 tasks for initial release, expandable to 200+

---

### 4. Reward Evaluator

**Purpose**: Verify task completion and compute rewards

**Success Validators**:

1. **File Validator**
   - Check file existence
   - Verify content (regex match, exact match)
   - Validate file size, permissions

2. **Browser Validator**
   - Check URL pattern
   - Verify DOM element state (text, attributes, visibility)
   - Validate screenshot against template

3. **Database Validator**
   - Execute SQL query
   - Compare result to expected
   - Check row count, values

4. **Message Validator**
   - Verify message sent to correct recipient
   - Check message content (keywords, regex)
   - Validate delivery timestamp

5. **Custom Validator**
   - Execute sandboxed JavaScript function
   - Return boolean (success/failure)

**Reward Computation**:

```
total_reward = base_reward + efficiency_bonus - step_penalty - tool_costs

where:
  base_reward = +1.0 if success, -0.5 if failure
  efficiency_bonus = +0.2 if steps < threshold
  step_penalty = -0.01 × num_steps
  tool_costs = sum of per-tool costs (optional)
```

**Failure Diagnostics**:

- Which success condition failed
- Expected vs. actual state
- Helpful hints for debugging
- Screenshots/logs at failure point

---

### 5. Trajectory Logger

**Purpose**: Record episode data for offline analysis and training

**Trajectory Format**:

```json
{
  "episode_id": "ep_12345",
  "task_id": "library-room-booking",
  "success": true,
  "total_reward": 0.85,
  "steps": [
    {
      "step": 1,
      "observation": {...},
      "action": {
        "tool": "browser",
        "params": {"action": "open", "url": "..."}
      },
      "reward": -0.01,
      "done": false
    },
    // ... more steps ...
  ],
  "metadata": {
    "duration_ms": 45000,
    "tools_used": ["browser"],
    "timestamp": "2026-02-01T10:30:00Z"
  }
}
```

**Storage**:

- Format: JSONL (one episode per line)
- Location: Configurable (local disk, S3, etc.)
- Compression: Optional gzip
- Retention: Configurable (keep N most recent, or by date)

**Use Cases**:

- Offline RL training (external system loads trajectories)
- Analysis (success rates, common failures)
- Debugging (replay failed episodes)
- Human evaluation (review agent behavior)

---

### 6. API Server

**Purpose**: Expose Gym-compatible HTTP API for external trainers

**Endpoints**:

```
POST /rl/reset
  Initialize new episode

POST /rl/step
  Execute action, get feedback

GET /rl/observe
  Get current observation without stepping

GET /rl/info
  Get episode metadata

GET /rl/tasks
  List available tasks

GET /rl/tasks/{id}
  Get task definition

GET /rl/trajectories
  Query logged trajectories (for offline training)
```

**Authentication**: API key or JWT (configurable)

**Rate Limiting**: Configurable per-client limits

**Multi-Client Support**: Handle multiple concurrent training runs

---

## Data Requirements

### 1. Task Dataset

**Initial Target**: 50-100 tasks covering:

- 40% Browser automation
- 30% File operations
- 15% Messaging
- 15% Multi-tool workflows

**Task Difficulty Distribution**:

- 30% Difficulty 1-2 (simple, single-step or few-step)
- 50% Difficulty 3 (moderate complexity, multi-step)
- 20% Difficulty 4-5 (complex, multi-tool, error handling)

**Task Creation**:

- Manual curation for core tasks
- Templates for common patterns
- Procedural generation for variations

### 2. Environment State Seeds

**Browser Profiles**:

- Pre-logged-in accounts (library, email, etc.)
- Cookies and localStorage pre-populated
- Bookmarks, history (if relevant)

**Database Seeds**:

- SQLite databases with sample data
- MongoDB collections
- CSV/JSON data files

**File System Seeds**:

- Sample documents (PDFs, text files)
- Code repositories (for git tasks)
- Configuration files

### 3. Mock Services

**Purpose**: Avoid external costs and API rate limits

**Components**:

- Local SMTP server (for email tasks)
- Mock Slack/Telegram bots (test workspaces)
- Fake API endpoints (for API call tasks)

---

## Implementation Phases

### Phase 1: Core Environment (4 weeks)

**Deliverables**:

- Episode controller with basic lifecycle
- State manager (Level 1-2 reset)
- Task loader (5-10 example tasks)
- Basic reward evaluator (file and browser validators)
- HTTP API server (reset, step, observe)

**Success Criteria**: Can run single task episodes end-to-end, verify success, provide consistent feedback.

---

### Phase 2: Robustness & Scale (4 weeks)

**Deliverables**:

- Expanded task set (30-50 tasks)
- All validator types (file, browser, DB, message, custom)
- Trajectory logging system
- Advanced state reset (Level 3, container snapshots)
- Task difficulty metadata and filtering

**Success Criteria**: Environment runs reliably across diverse task types, provides accurate rewards, logs complete trajectories.

---

### Phase 3: Production Features (3 weeks)

**Deliverables**:

- Multi-client API support
- Authentication and rate limiting
- Monitoring dashboard (episode stats, success rates)
- Docker containerization
- Comprehensive documentation

**Success Criteria**: Environment ready for external training systems to use at scale.

---

## Technical Decisions

### 1. All TypeScript (No Python)

**Decision**: Implement entire environment in TypeScript within OpenClaw

**Rationale**:

- ✅ No training code means no need for PyTorch/TRL
- ✅ Direct access to OpenClaw internals
- ✅ Single codebase, simple deployment
- ✅ Existing tool/session infrastructure

### 2. Plugin Architecture

**Decision**: Build as OpenClaw plugin, not core modification

**Rationale**:

- ✅ Opt-in (can enable/disable)
- ✅ Standard plugin hooks
- ✅ No impact on production OpenClaw usage
- ✅ Easy to maintain separately

### 3. Task-First Design

**Decision**: Task definitions drive environment behavior

**Rationale**:

- ✅ Easy to add new tasks without code changes
- ✅ Non-developers can contribute tasks
- ✅ Clear separation: tasks (data) vs. environment (logic)

### 4. Tiered Reset

**Decision**: Fast reset (< 1s) for most episodes, slower resets as fallback

**Rationale**:

- ✅ Maximize throughput (more episodes per hour)
- ✅ Safety net for state corruption (full reset available)
- ✅ Configurable per task

### 5. Structured Rewards

**Decision**: Composite rewards (success + efficiency - costs)

**Rationale**:

- ✅ Encourages both task completion and efficiency
- ✅ Penalizes wasteful tool usage
- ✅ Customizable per task

---

## Environment Quality Metrics

### Reliability

**Target**: 99.9% uptime (environment doesn't crash)

**Measures**:

- Error handling for all tool failures
- Graceful degradation (partial success still provides reward)
- Automatic recovery from transient failures

### Consistency

**Target**: Deterministic outcomes for same action sequence

**Measures**:

- Seeded randomness (if any)
- Controlled external dependencies
- State isolation between episodes

### Feedback Quality

**Target**: Reward signal accurately reflects task success

**Measures**:

- False positive rate < 1% (claiming success when task failed)
- False negative rate < 5% (claiming failure when task succeeded)
- Informative failure diagnostics

### Performance

**Target**: Episodes run efficiently

**Measures**:

- Reset time: < 1s (Level 1), < 5s (Level 2), < 30s (Level 3)
- Episode throughput: 50+ episodes/hour (single environment)
- Parallel scaling: 16+ concurrent environments

---

## Integration with OpenClaw

### Plugin Installation

```bash
# Enable RL environment plugin
openclaw config set plugins.rl-environment.enabled true

# Configure task directory
openclaw config set rl.tasksDir ./tasks

# Set trajectory storage
openclaw config set rl.trajectoriesDir ./trajectories

# Start gateway with RL environment
openclaw gateway run
```

### Configuration

```json5
{
  rl: {
    enabled: true,
    tasksDir: "./tasks",
    trajectoriesDir: "./trajectories",
    api: {
      port: 18790,
      auth: "api_key",
      rateLimit: {
        maxRequestsPerMinute: 1000,
      },
    },
    reset: {
      defaultStrategy: "level1",
      fallbackStrategy: "level2",
      fullResetInterval: 100, // Full reset every N episodes
    },
  },
}
```

### Coexistence with Production

**Production mode**: Normal OpenClaw usage (no episodes)
**RL mode**: Episode-based operation

**Toggle**: `OPENCLAW_RL_MODE=true` or via config

**No conflicts**: RL plugin only active when explicitly used

---

## API Usage Example

```typescript
// External training system interacting with environment

// 1. Start new episode
const response1 = await fetch("http://localhost:18790/rl/reset", {
  method: "POST",
  headers: { Authorization: "Bearer API_KEY" },
  body: JSON.stringify({ task_id: "library-room-booking" }),
});
const { observation } = await response1.json();

// 2. Agent decides on action
const action = {
  tool: "browser",
  params: { action: "open", url: "https://sccld.org/locations/saratoga/" },
};

// 3. Execute action
const response2 = await fetch("http://localhost:18790/rl/step", {
  method: "POST",
  headers: { Authorization: "Bearer API_KEY" },
  body: JSON.stringify({ action }),
});
const { observation: obs2, reward, done, info } = await response2.json();

// 4. Repeat until done === true
```

---

## Future Extensions

### 1. Multi-Agent Tasks

Support tasks requiring coordination between multiple agents:

- Task delegation (main agent → specialist agents)
- Parallel execution (agents work simultaneously)
- Verification (one agent checks another's work)

### 2. Visual Observations

Add screenshot-based observations for richer browser tasks:

- Include base64 screenshots in observations
- Enable vision-language models to "see" the browser

### 3. Long-Horizon Tasks

Support complex tasks requiring 50+ steps:

- Hierarchical task decomposition
- Sub-task checkpointing
- Partial credit for progress

### 4. Human-in-the-Loop Evaluation

Interactive task verification:

- Human evaluator reviews episode outcomes
- Override automated reward when ambiguous
- Collect preference data for training

---

## Summary

### What We're Building

✅ **Episode Controller** - Manage task execution lifecycle
✅ **State Manager** - Initialize and reset environment
✅ **Task System** - Load and validate task definitions
✅ **Reward Evaluator** - Verify success and compute rewards
✅ **Trajectory Logger** - Record episodes for analysis
✅ **API Server** - Expose Gym-compatible interface

### What We're NOT Building

❌ Policy networks
❌ RL training algorithms (PPO, DPO, etc.)
❌ Training orchestration
❌ Model checkpointing
❌ Hyperparameter tuning

### Implementation

**Location**: `openclaw/extensions/rl-environment/` (TypeScript plugin)
**Timeline**: ~11 weeks (4 + 4 + 3)
**Deployment**: OpenClaw with RL plugin enabled

### Success Criteria

After implementation:

- ✅ 50+ diverse tasks available
- ✅ < 1% false positive rate on rewards
- ✅ 50+ episodes/hour throughput
- ✅ Stable, deterministic environment behavior
- ✅ Clean API for external training systems

---

## Next Steps

1. **Build credential management foundation** (1 week)
   - Choose credential storage (1Password or env vars)
   - Define account schema
   - Implement getCredential() function
   - Test with one account

2. **Define initial task set** (1 week)
   - Identify 10-15 core tasks
   - Write task definitions (YAML)
   - Validate task feasibility manually

3. **Build core episode controller** (2 weeks)
   - Implement reset/step/observe API
   - Basic state initialization
   - Simple reward computation (boolean success/failure)

4. **Add task infrastructure** (1 week)
   - Task loader
   - Validator framework
   - Trajectory logging

5. **Expand and refine** (ongoing)
   - Add more tasks incrementally
   - Improve validators based on testing
   - Optimize reset performance
   - Document API for external users
