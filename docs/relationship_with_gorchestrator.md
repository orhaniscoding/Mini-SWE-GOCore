# Relationship with GOrchestrator

This document explains how Mini-SWE-GOCore fits into the larger GOrchestrator ecosystem and how it functions as the execution engine for multi-agent workflows.

---

## The Brain and Muscle Analogy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        GORCHESTRATOR ECOSYSTEM                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         GOrchestrator                                │   │
│  │                          (The Brain)                                 │   │
│  │                                                                     │   │
│  │  • Task decomposition and planning                                   │   │
│  │  • Agent spawning and lifecycle management                           │   │
│  │  • Result aggregation and conflict resolution                        │   │
│  │  • Cost monitoring across all workers                                │   │
│  │  • Parallel execution coordination                                   │   │
│  │                                                                     │   │
│  └──────────────────────────────┬──────────────────────────────────────┘   │
│                                 │                                           │
│                    spawns multiple instances                                │
│                                 │                                           │
│         ┌───────────────────────┼───────────────────────┐                  │
│         ▼                       ▼                       ▼                  │
│  ┌─────────────┐        ┌─────────────┐        ┌─────────────┐            │
│  │ GOCore #1   │        │ GOCore #2   │        │ GOCore #3   │            │
│  │ (Muscle)    │        │ (Muscle)    │        │ (Muscle)    │            │
│  │             │        │             │        │             │            │
│  │ Task: Fix   │        │ Task: Add   │        │ Task: Write │            │
│  │ login bug   │        │ validation  │        │ unit tests  │            │
│  └──────┬──────┘        └──────┬──────┘        └──────┬──────┘            │
│         │                      │                      │                    │
│         └──────────────────────┴──────────────────────┘                    │
│                                │                                           │
│                         All route through                                   │
│                                ▼                                           │
│                    ┌───────────────────────┐                               │
│                    │   Antigravity Proxy   │                               │
│                    │   (Cost Tracking)     │                               │
│                    └───────────────────────┘                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Role Separation

### GOrchestrator (The Brain)

**Purpose**: High-level orchestration and decision-making

| Responsibility | Description |
|----------------|-------------|
| Task Planning | Breaks complex tasks into subtasks |
| Agent Management | Spawns, monitors, and terminates workers |
| Resource Allocation | Assigns agents to tasks based on availability |
| Result Synthesis | Combines outputs from multiple workers |
| Error Recovery | Handles worker failures and retries |
| Cost Control | Enforces budgets across all workers |

### Mini-SWE-GOCore (The Muscle)

**Purpose**: Focused task execution

| Responsibility | Description |
|----------------|-------------|
| Code Understanding | Reads and analyzes source code |
| Code Modification | Makes targeted changes to files |
| Command Execution | Runs shell commands safely |
| LLM Interaction | Communicates with language models |
| JSON Output | Streams structured events to stdout |

---

## Communication Protocol

GOrchestrator communicates with Mini-SWE-GOCore via **subprocess I/O**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SUBPROCESS PROTOCOL                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  GOrchestrator                              Mini-SWE-GOCore                  │
│  ─────────────                              ────────────────                  │
│                                                                             │
│  1. spawn process ─────────────────────────▶ process starts                 │
│     command: uv run mini --headless                                         │
│              --model anthropic/claude-sonnet                                 │
│              --task "Fix login bug"                                          │
│                                                                             │
│  2. read stdout ◀─────────────────────────── emit JSON events               │
│                                                                             │
│     {"type": "start", "data": {...}}                                        │
│     {"type": "step", "data": {"step": 1, ...}}                              │
│     {"type": "thought", "data": {"content": "..."}}                         │
│     {"type": "step", "data": {"step": 2, ...}}                              │
│     ...                                                                     │
│     {"type": "finish", "data": {"status": "success", ...}}                  │
│     {"type": "cost", "data": {"total": 0.05, ...}}                          │
│                                                                             │
│  3. collect result ◀────────────────────── process exits                    │
│     exit code: 0 = success, 1 = error                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### JSON Event Types

| Event Type | Description | Key Fields |
|------------|-------------|------------|
| `start` | Agent started working | `task`, `model`, `workdir` |
| `step` | Completed one action | `step`, `action`, `output`, `cost_so_far` |
| `thought` | Agent's reasoning | `content` |
| `log` | Informational message | `level`, `content` |
| `finish` | Agent completed | `status`, `result` |
| `error` | Agent failed | `error`, `type`, `traceback` |
| `cost` | Final cost summary | `total`, `calls` |

---

## Spawning Pattern

### From GOrchestrator (Python)

```python
import subprocess
import json

def spawn_worker(task: str, model: str = "anthropic/claude-sonnet-4-20250514") -> dict:
    """Spawn a Mini-SWE-GOCore worker and collect results."""

    process = subprocess.Popen(
        [
            "uv", "run", "mini",
            "--headless",
            "--model", model,
            "--task", task,
            "--cost-limit", "5.0",
        ],
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
        env={
            **os.environ,
            "MINI_API_BASE": "http://127.0.0.1:8045",
            "ANTHROPIC_API_KEY": "sk-dummy",
        },
        cwd="/path/to/target/repo",
    )

    events = []
    for line in process.stdout:
        event = json.loads(line.decode())
        events.append(event)

        # Handle events in real-time
        if event["type"] == "step":
            print(f"Step {event['data']['step']}: {event['data']['action'][:50]}...")
        elif event["type"] == "error":
            print(f"Error: {event['data']['error']}")

    process.wait()
    return {
        "exit_code": process.returncode,
        "events": events,
        "result": next((e for e in events if e["type"] == "finish"), None),
        "cost": next((e for e in events if e["type"] == "cost"), None),
    }

# Example usage
result = spawn_worker("Fix the authentication bug in login.py")
print(f"Status: {result['result']['data']['status']}")
print(f"Cost: ${result['cost']['data']['total']:.4f}")
```

### Parallel Execution

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor

async def parallel_workers(tasks: list[str]) -> list[dict]:
    """Run multiple workers in parallel."""

    with ProcessPoolExecutor(max_workers=4) as executor:
        loop = asyncio.get_event_loop()
        futures = [
            loop.run_in_executor(executor, spawn_worker, task)
            for task in tasks
        ]
        return await asyncio.gather(*futures)

# Example: Fix multiple bugs in parallel
tasks = [
    "Fix login validation",
    "Add input sanitization to forms",
    "Write unit tests for auth module",
]

results = asyncio.run(parallel_workers(tasks))
total_cost = sum(r["cost"]["data"]["total"] for r in results)
print(f"All tasks complete. Total cost: ${total_cost:.4f}")
```

---

## Configuration for GOrchestrator

When used with GOrchestrator, Mini-SWE-GOCore should be configured for:

### Headless Operation

```yaml
# .miniswe/configs/orchestrated.yaml
model:
  model_class: litellm
  model_name: "anthropic/claude-sonnet-4-20250514"
  cost_tracking: "ignore_errors"

agent:
  mode: "yolo"           # No confirmation prompts
  step_limit: 100        # Allow more steps for complex tasks
  cost_limit: 10.0       # Per-worker budget
  confirm_exit: false    # Don't wait for user

headless:
  include_timestamps: true
  max_output_length: 50000  # Larger for debugging
```

### Environment Variables

```bash
# Set by GOrchestrator before spawning
MINI_API_BASE=http://127.0.0.1:8045
MINI_API_KEY=orchestrator-key
MINI_WORKDIR=/path/to/target/repo
MINI_HEADLESS=true
ANTHROPIC_API_KEY=sk-dummy
```

---

## Standalone vs Orchestrated Mode

| Aspect | Standalone | With GOrchestrator |
|--------|------------|-------------------|
| **Invocation** | User runs CLI | Spawned as subprocess |
| **Input** | CLI arguments | CLI args + env vars |
| **Output** | JSON to stdout | JSON parsed by orchestrator |
| **Confirmation** | Optional | Always disabled |
| **Cost Control** | Per-session | Aggregated by orchestrator |
| **Parallelism** | Single instance | Multiple parallel instances |
| **Error Handling** | Exit with code | Captured by orchestrator |

---

## Benefits of This Architecture

### Separation of Concerns

- **GOrchestrator** handles complexity: planning, coordination, synthesis
- **GOCore** stays focused: one task, one codebase, deterministic execution

### Scalability

```
                    GOrchestrator
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
       GOCore         GOCore         GOCore
       (Worker 1)     (Worker 2)     (Worker 3)
          │              │              │
          └──────────────┴──────────────┘
                         │
                    Proxy (shared)
```

- Add more workers by spawning more processes
- Workers are isolated - one failure doesn't affect others
- Proxy provides centralized cost visibility

### Reproducibility

- Each worker runs with explicit configuration
- JSON output provides complete audit trail
- Same task + same config = same result

---

## Anti-Patterns

### Don't Do This

```python
# ❌ Don't try to control GOCore interactively
process = subprocess.Popen(["uv", "run", "mini", "--task", "..."])
process.stdin.write(b"yes\n")  # Won't work in headless mode

# ❌ Don't parse stdout as text
output = process.stdout.read().decode()
if "success" in output:  # Fragile!
    ...

# ❌ Don't share state between workers via filesystem
# Each worker should be independent
```

### Do This Instead

```python
# ✅ Use --headless and parse JSON events
process = subprocess.Popen(
    ["uv", "run", "mini", "--headless", "--task", "..."],
    stdout=subprocess.PIPE,
)
for line in process.stdout:
    event = json.loads(line)
    if event["type"] == "finish":
        status = event["data"]["status"]

# ✅ Give each worker its own workdir
# ✅ Aggregate results in the orchestrator
```

---

## Next Steps

- [Installation Guide](installation.md) - Get Mini-SWE-GOCore running
- [Proxy Integration](proxy_integration.md) - Set up centralized proxy
- [Configuration Reference](configuration.md) - All configuration options
