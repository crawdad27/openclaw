# RL Implementation Strategy

**Updated Scope**: Building **environment only**, not training infrastructure.

---

## Scope Clarification

**What we're building**: RL Environment

- Episode management (reset, step, observe)
- Task definitions and loading
- Reward evaluation
- State management
- Trajectory logging

**What we're NOT building**: RL Training

- Policy networks
- RL algorithms (PPO, DPO, etc.)
- Training loops
- PyTorch/TRL/Transformers

**Why**: External training systems will interface with our environment. We provide the "game," they provide the "player."

---

## Decision: Single Repository (OpenClaw Plugin)

### Recommendation

**Implement as OpenClaw plugin** (`extensions/rl-environment/`)

### Rationale

**Language issue resolved**:

- ❌ No need for Python (no ML training code)
- ✅ TypeScript works perfectly for environment logic
- ✅ Direct access to OpenClaw tools and runtime

**Simplicity**:

- ✅ Single codebase
- ✅ No API boundaries within OpenClaw
- ✅ Standard plugin architecture (opt-in)
- ✅ Easy deployment (just enable plugin)

**No downsides**:

- Since we're not doing training, we don't need PyTorch/TRL
- All environment code (episode management, tasks, rewards) fits naturally in TypeScript
- Plugin approach keeps it isolated from OpenClaw core

---

## Architecture

```
┌─────────────────────────────────────────┐
│  External Training System               │
│  (Out of scope - built by others)       │
└────────────────┬────────────────────────┘
                 │
                 │ HTTP Gym API
                 │
┌────────────────▼────────────────────────┐
│  OpenClaw                                │
│                                          │
│  ┌────────────────────────────────┐     │
│  │ extensions/rl-environment/     │     │
│  │                                │     │
│  │ - Episode controller           │     │
│  │ - State manager                │     │
│  │ - Task loader                  │     │
│  │ - Reward evaluator             │     │
│  │ - Trajectory logger            │     │
│  │ - HTTP API server              │     │
│  └───────────┬────────────────────┘     │
│              │                           │
│  ┌───────────▼────────────────────┐     │
│  │ OpenClaw Core (unchanged)      │     │
│  │ - Tools, agents, sessions      │     │
│  └────────────────────────────────┘     │
└──────────────────────────────────────────┘
```

---

## Implementation Plan

### Repository Structure

```
openclaw/
├── extensions/
│   └── rl-environment/
│       ├── src/
│       │   ├── episode/
│       │   │   ├── controller.ts
│       │   │   └── observation.ts
│       │   ├── state/
│       │   │   ├── manager.ts
│       │   │   ├── reset.ts
│       │   │   └── initializers/
│       │   ├── tasks/
│       │   │   ├── loader.ts
│       │   │   ├── schema.ts
│       │   │   └── validators/
│       │   ├── rewards/
│       │   │   ├── evaluator.ts
│       │   │   └── validators/
│       │   ├── trajectory/
│       │   │   └── logger.ts
│       │   └── server/
│       │       ├── api.ts
│       │       └── routes.ts
│       ├── package.json
│       └── README.md
├── tasks/
│   ├── browser/
│   ├── files/
│   ├── messaging/
│   └── multi-tool/
└── ...
```

### Phase 1: Core Environment (4 weeks)

**Deliverables**:

- Episode controller (reset, step, observe)
- Basic state manager (in-memory reset)
- Task loader (YAML parsing)
- Simple reward evaluator (boolean success/failure)
- HTTP API server (reset, step endpoints)
- 5-10 example tasks

### Phase 2: Robustness (4 weeks)

**Deliverables**:

- All validator types (file, browser, DB, message)
- Advanced state reset (browser profiles, filesystem)
- Trajectory logging
- 30-50 tasks across categories
- Comprehensive rewards (efficiency bonuses, step penalties)

### Phase 3: Production (3 weeks)

**Deliverables**:

- API authentication
- Multi-client support
- Docker containerization
- Monitoring/metrics
- Documentation

**Total**: ~11 weeks

---

## Key Decisions

### 1. TypeScript Only

**Why it works**:

- Environment logic doesn't need ML frameworks
- TypeScript is perfect for:
  - Episode management
  - Task validation
  - Reward computation
  - API serving

### 2. Plugin Architecture

**Benefits**:

- Opt-in (enable/disable RL environment)
- No core changes to OpenClaw
- Standard plugin hooks
- Easy to maintain separately

### 3. Gym-Compatible API

**Interface**:

- `POST /rl/reset` - Start episode
- `POST /rl/step` - Execute action
- `GET /rl/observe` - Get observation
- `GET /rl/info` - Episode metadata

**Why HTTP**:

- Language-agnostic (any training system can use it)
- Simple to debug and test
- Can add gRPC later if performance matters

---

## Integration Example

External training system using the environment:

```python
# Python client (built by training team, not us)
import requests

class OpenClawEnv:
    def __init__(self, api_url, api_key):
        self.api_url = api_url
        self.headers = {'Authorization': f'Bearer {api_key}'}

    def reset(self, task_id):
        resp = requests.post(
            f'{self.api_url}/rl/reset',
            json={'task_id': task_id},
            headers=self.headers
        )
        return resp.json()['observation']

    def step(self, action):
        resp = requests.post(
            f'{self.api_url}/rl/step',
            json={'action': action},
            headers=self.headers
        )
        result = resp.json()
        return result['observation'], result['reward'], result['done'], result['info']

# Usage
env = OpenClawEnv('http://localhost:18790', 'api_key_here')
obs = env.reset('library-room-booking')
obs, reward, done, info = env.step({
    'tool': 'browser',
    'params': {'action': 'open', 'url': '...'}
})
```

---

## Comparison: Why This Approach

| Aspect               | Monorepo             | Separate Repos    | Plugin (Our Choice)                    |
| -------------------- | -------------------- | ----------------- | -------------------------------------- |
| **Training code**    | ❌ Would need Python | ✅ Python repo    | ✅ Not our problem                     |
| **Environment code** | ✅ TypeScript        | ⚠️ Need API layer | ✅ TypeScript                          |
| **Complexity**       | ❌ Mixed concerns    | ❌ Two repos      | ✅ Single plugin                       |
| **Integration**      | ✅ Direct            | ❌ HTTP overhead  | ✅ Direct (OpenClaw) + HTTP (external) |
| **Maintenance**      | ❌ High              | ⚠️ Medium         | ✅ Low                                 |

**Verdict**: Plugin approach is simplest since we're only building environment.

---

## Success Criteria

After implementation:

- ✅ 50+ tasks available
- ✅ Environment runs reliably (99.9% uptime)
- ✅ Accurate rewards (< 1% false positives)
- ✅ Fast resets (< 1s typical, < 5s fallback)
- ✅ Clean API for external trainers
- ✅ Good documentation

---

## Next Steps

1. **Define task format** (this week)
   - YAML schema for tasks
   - Success condition types
   - Reward structure

2. **Build episode controller** (weeks 1-2)
   - Basic reset/step/observe
   - Tool execution via OpenClaw
   - Simple observation building

3. **Add task infrastructure** (weeks 3-4)
   - Task loader
   - Validator framework
   - Basic rewards

4. **Iterate** (ongoing)
   - Add more tasks
   - Improve validators
   - Optimize performance
