# Integration Guide

## Overview

This guide is for developers building GUI applications (like GOrchestrator) that integrate with Mini-SWE-GOCore. It covers the JSON event schema, subprocess management, and error handling.

---

## Headless Mode

In headless mode (`--headless`), Mini-SWE-GOCore outputs structured JSON Lines to stdout. Each line is a complete JSON object representing an event.

```bash
mini --headless --profile live --task "Fix the bug" > output.jsonl
```

**Output Streams:**
| Stream | Content |
|--------|---------|
| `stdout` | JSON Lines (events) |
| `stderr` | Critical errors only |

---

## JSON Event Schema

### Base Event Structure

Every event follows this structure:

```typescript
interface BaseEvent {
  type: string;           // Event type identifier
  timestamp?: string;     // ISO 8601 timestamp (if enabled)
  data: object;           // Event-specific payload
}
```

### Event Types

| Type | Description | When Emitted |
|------|-------------|--------------|
| `start` | Agent started | Once at beginning |
| `thought` | Model reasoning | After each model response |
| `command` | Command to execute | Before command execution |
| `output` | Command result | After command execution |
| `error` | Error occurred | On any error |
| `cost` | Cost update | After each model call |
| `finish` | Agent completed | Once at end |

---

## Event Definitions

### `start` Event

Emitted once when the agent begins execution.

```json
{
  "type": "start",
  "timestamp": "2024-01-15T10:30:00.000Z",
  "data": {
    "task": "Fix the authentication bug in src/auth.py",
    "profile": "live",
    "model": "gpt-4o",
    "workdir": "/home/user/my-project",
    "version": "2.0.0"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `task` | string | The task/problem statement |
| `profile` | string | Configuration profile name |
| `model` | string | Model being used |
| `workdir` | string | Working directory path |
| `version` | string | Mini-SWE-GOCore version |

---

### `thought` Event

Emitted when the model produces reasoning or planning text.

```json
{
  "type": "thought",
  "timestamp": "2024-01-15T10:30:05.123Z",
  "data": {
    "content": "I need to first examine the auth.py file to understand the current implementation. Let me read the file.",
    "step": 1
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `content` | string | Model's reasoning text |
| `step` | integer | Current step number |

---

### `command` Event

Emitted before a command is executed.

```json
{
  "type": "command",
  "timestamp": "2024-01-15T10:30:06.456Z",
  "data": {
    "command": "cat src/auth.py",
    "cwd": "/home/user/my-project",
    "step": 1
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `command` | string | Shell command to execute |
| `cwd` | string | Current working directory |
| `step` | integer | Current step number |

---

### `output` Event

Emitted after a command completes.

```json
{
  "type": "output",
  "timestamp": "2024-01-15T10:30:07.789Z",
  "data": {
    "stdout": "def login(username, password):\n    # TODO: Add validation\n    return True\n",
    "stderr": "",
    "exit_code": 0,
    "truncated": false,
    "step": 1
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `stdout` | string | Standard output |
| `stderr` | string | Standard error |
| `exit_code` | integer | Process exit code |
| `truncated` | boolean | True if output was truncated |
| `step` | integer | Current step number |

**Note:** If `truncated` is `true`, the output exceeded `max_output_length` and was cut off.

---

### `error` Event

Emitted when an error occurs.

```json
{
  "type": "error",
  "timestamp": "2024-01-15T10:30:10.000Z",
  "data": {
    "error": "Command timed out after 120 seconds",
    "error_type": "TimeoutError",
    "step": 3,
    "recoverable": true
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `error` | string | Human-readable error message |
| `error_type` | string | Exception class name |
| `step` | integer | Step where error occurred |
| `recoverable` | boolean | Whether agent can continue |

**Error Types:**
| Type | Meaning |
|------|---------|
| `TimeoutError` | Command exceeded timeout |
| `APIError` | Model API call failed |
| `CostLimitExceeded` | Cost limit reached |
| `StepLimitExceeded` | Step limit reached |
| `ConfigurationError` | Invalid configuration |

---

### `cost` Event

Emitted after each model API call with updated cost information.

```json
{
  "type": "cost",
  "timestamp": "2024-01-15T10:30:08.000Z",
  "data": {
    "step_cost": 0.0045,
    "total_cost": 0.0123,
    "input_tokens": 1500,
    "output_tokens": 200,
    "model": "gpt-4o"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `step_cost` | float | Cost of this API call (USD) |
| `total_cost` | float | Cumulative cost (USD) |
| `input_tokens` | integer | Tokens in prompt |
| `output_tokens` | integer | Tokens in response |
| `model` | string | Model used |

---

### `finish` Event

Emitted once when the agent completes (success or failure).

```json
{
  "type": "finish",
  "timestamp": "2024-01-15T10:35:00.000Z",
  "data": {
    "status": "completed",
    "result": "Fixed the authentication bug by adding input validation to the login function.",
    "steps": 12,
    "total_cost": 0.0456,
    "duration_seconds": 300.5,
    "trajectory_path": "/home/user/my-project/.miniswe/data/last_run.traj.json"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | Final status (see below) |
| `result` | string | Summary or error message |
| `steps` | integer | Total steps executed |
| `total_cost` | float | Total cost in USD |
| `duration_seconds` | float | Total execution time |
| `trajectory_path` | string | Path to saved trajectory |

**Status Values:**
| Status | Meaning |
|--------|---------|
| `completed` | Task finished successfully |
| `cost_limit` | Stopped due to cost limit |
| `step_limit` | Stopped due to step limit |
| `error` | Stopped due to error |
| `interrupted` | User/system interrupted |

---

## Python Integration

### Basic Subprocess Example

```python
import subprocess
import json
from pathlib import Path
from typing import Iterator, Dict, Any

def run_agent(
    task: str,
    workdir: Path,
    profile: str = "live"
) -> Iterator[Dict[str, Any]]:
    """
    Run Mini-SWE-GOCore and stream events.

    Args:
        task: The task/problem statement
        workdir: Project directory
        profile: Configuration profile name

    Yields:
        Event dictionaries as they arrive
    """
    process = subprocess.Popen(
        [
            "uv", "run", "mini",
            "--headless",
            "--profile", profile,
            "--workdir", str(workdir),
            "--task", task,
        ],
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
        text=True,
        bufsize=1,  # Line buffered for real-time output
    )

    # Stream events from stdout
    for line in process.stdout:
        line = line.strip()
        if not line:
            continue

        try:
            event = json.loads(line)
            yield event
        except json.JSONDecodeError:
            # Non-JSON output (shouldn't happen, but handle gracefully)
            yield {
                "type": "raw",
                "data": {"content": line}
            }

    # Wait for process to complete
    process.wait()

    # Check for errors on stderr
    stderr = process.stderr.read()
    if stderr and process.returncode != 0:
        yield {
            "type": "error",
            "data": {
                "error": stderr,
                "error_type": "ProcessError",
                "recoverable": False
            }
        }


# Usage example
def main():
    workdir = Path("./my-project")

    for event in run_agent("Fix the login bug", workdir):
        event_type = event["type"]
        data = event.get("data", {})

        if event_type == "start":
            print(f"Started: {data.get('task')}")

        elif event_type == "thought":
            print(f"Thinking: {data.get('content')[:100]}...")

        elif event_type == "command":
            print(f"Running: {data.get('command')}")

        elif event_type == "output":
            output = data.get("stdout", "")[:200]
            print(f"Output: {output}...")

        elif event_type == "cost":
            print(f"Cost: ${data.get('total_cost', 0):.4f}")

        elif event_type == "error":
            print(f"ERROR: {data.get('error')}")

        elif event_type == "finish":
            status = data.get("status")
            print(f"Finished: {status}")
            if status == "completed":
                print(f"Result: {data.get('result')}")


if __name__ == "__main__":
    main()
```

---

### Async Integration (for GOrchestrator)

```python
import asyncio
import json
from pathlib import Path
from typing import AsyncIterator, Dict, Any, Optional, Callable

class MiniSWEGOCore:
    """
    Async wrapper for Mini-SWE-GOCore integration.

    Designed for GOrchestrator and similar GUI applications.
    """

    def __init__(self, workdir: Path, profile: str = "live"):
        self.workdir = Path(workdir).resolve()
        self.profile = profile
        self.process: Optional[asyncio.subprocess.Process] = None
        self._running = False

    async def start(self, task: str) -> "MiniSWEGOCore":
        """Start the agent with the given task."""
        if self._running:
            raise RuntimeError("Agent already running")

        self.process = await asyncio.create_subprocess_exec(
            "uv", "run", "mini",
            "--headless",
            "--profile", self.profile,
            "--workdir", str(self.workdir),
            "--task", task,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
        )
        self._running = True
        return self

    async def stream_events(self) -> AsyncIterator[Dict[str, Any]]:
        """Yield events as they arrive."""
        if not self.process or not self.process.stdout:
            raise RuntimeError("Agent not started")

        async for line in self.process.stdout:
            line = line.decode().strip()
            if not line:
                continue

            try:
                yield json.loads(line)
            except json.JSONDecodeError:
                yield {"type": "raw", "data": {"content": line}}

    async def run_with_callback(
        self,
        task: str,
        on_event: Callable[[Dict[str, Any]], None]
    ) -> Dict[str, Any]:
        """
        Run agent and call callback for each event.

        Returns the final 'finish' event.
        """
        await self.start(task)

        final_event = None
        async for event in self.stream_events():
            on_event(event)
            if event["type"] == "finish":
                final_event = event

        await self.wait()
        return final_event

    async def stop(self) -> None:
        """Stop the agent gracefully."""
        if self.process and self._running:
            self.process.terminate()
            try:
                await asyncio.wait_for(self.process.wait(), timeout=5.0)
            except asyncio.TimeoutError:
                self.process.kill()
                await self.process.wait()
        self._running = False

    async def wait(self) -> int:
        """Wait for the agent to complete and return exit code."""
        if self.process:
            await self.process.wait()
            self._running = False
            return self.process.returncode
        return -1

    @property
    def trajectory_path(self) -> Path:
        """Get the path to the trajectory file."""
        return self.workdir / ".miniswe" / "data" / "last_run.traj.json"

    def load_trajectory(self) -> Optional[Dict[str, Any]]:
        """Load the trajectory from the last run."""
        if self.trajectory_path.exists():
            return json.loads(self.trajectory_path.read_text())
        return None


# Usage example
async def main():
    agent = MiniSWEGOCore(
        workdir=Path("./my-project"),
        profile="live"
    )

    def handle_event(event: Dict[str, Any]):
        print(f"[{event['type']}] {event.get('data', {})}")

    result = await agent.run_with_callback(
        task="Add unit tests for the auth module",
        on_event=handle_event
    )

    print(f"Final status: {result['data']['status']}")


if __name__ == "__main__":
    asyncio.run(main())
```

---

### Rust Integration

```rust
use std::io::{BufRead, BufReader};
use std::path::Path;
use std::process::{Command, Stdio};
use serde::{Deserialize, Serialize};
use serde_json::Value;

#[derive(Debug, Deserialize)]
struct Event {
    #[serde(rename = "type")]
    event_type: String,
    timestamp: Option<String>,
    data: Value,
}

fn run_agent(
    task: &str,
    workdir: &Path,
    profile: &str,
) -> Result<Vec<Event>, Box<dyn std::error::Error>> {
    let mut child = Command::new("uv")
        .args([
            "run", "mini",
            "--headless",
            "--profile", profile,
            "--workdir", workdir.to_str().unwrap(),
            "--task", task,
        ])
        .stdout(Stdio::piped())
        .stderr(Stdio::piped())
        .spawn()?;

    let stdout = child.stdout.take().unwrap();
    let reader = BufReader::new(stdout);

    let mut events = Vec::new();

    for line in reader.lines() {
        let line = line?;
        if line.is_empty() {
            continue;
        }

        match serde_json::from_str::<Event>(&line) {
            Ok(event) => {
                println!("[{}] {:?}", event.event_type, event.data);
                events.push(event);
            }
            Err(e) => {
                eprintln!("Failed to parse event: {}", e);
            }
        }
    }

    child.wait()?;
    Ok(events)
}

fn main() {
    let events = run_agent(
        "Fix the authentication bug",
        Path::new("./my-project"),
        "live",
    ).expect("Failed to run agent");

    // Find the finish event
    if let Some(finish) = events.iter().find(|e| e.event_type == "finish") {
        println!("Finished: {:?}", finish.data);
    }
}
```

---

## Error Handling

### Detecting Agent Crashes

```python
async def run_with_error_detection(task: str, workdir: Path):
    """Run agent with comprehensive error detection."""

    agent = MiniSWEGOCore(workdir)

    try:
        await agent.start(task)

        finish_event = None
        error_events = []

        async for event in agent.stream_events():
            if event["type"] == "error":
                error_events.append(event)
                if not event["data"].get("recoverable", True):
                    # Non-recoverable error, agent will stop
                    break

            elif event["type"] == "finish":
                finish_event = event

        exit_code = await agent.wait()

        # Analyze result
        if exit_code != 0 and not finish_event:
            # Process crashed without finish event
            raise RuntimeError(f"Agent crashed with exit code {exit_code}")

        if finish_event:
            status = finish_event["data"]["status"]
            if status == "error":
                # Agent finished but with error status
                raise RuntimeError(finish_event["data"]["result"])
            elif status in ("cost_limit", "step_limit"):
                # Agent stopped due to limits
                print(f"Warning: Agent stopped due to {status}")

        return finish_event

    except asyncio.TimeoutError:
        await agent.stop()
        raise RuntimeError("Agent timed out")

    except Exception as e:
        await agent.stop()
        raise
```

### Exit Codes

| Exit Code | Meaning |
|-----------|---------|
| 0 | Success (completed or graceful stop) |
| 1 | Error (configuration, API, or runtime error) |

**Note:** Always check the `finish` event's `status` field for the actual result, not just the exit code.

---

## Best Practices

### 1. Always Handle All Event Types

```python
EVENT_HANDLERS = {
    "start": handle_start,
    "thought": handle_thought,
    "command": handle_command,
    "output": handle_output,
    "error": handle_error,
    "cost": handle_cost,
    "finish": handle_finish,
}

for event in run_agent(task, workdir):
    handler = EVENT_HANDLERS.get(event["type"], handle_unknown)
    handler(event)
```

### 2. Implement Timeouts

```python
async def run_with_timeout(task: str, workdir: Path, timeout: float = 600):
    """Run agent with overall timeout."""
    agent = MiniSWEGOCore(workdir)

    try:
        await asyncio.wait_for(
            agent.run_with_callback(task, on_event=print),
            timeout=timeout
        )
    except asyncio.TimeoutError:
        await agent.stop()
        raise RuntimeError(f"Agent exceeded {timeout}s timeout")
```

### 3. Save Raw Output for Debugging

```python
import datetime

def run_with_logging(task: str, workdir: Path):
    """Run agent and save raw output for debugging."""

    timestamp = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
    log_file = workdir / ".miniswe" / "logs" / f"run_{timestamp}.jsonl"

    with open(log_file, "w") as f:
        for event in run_agent(task, workdir):
            # Save raw event
            f.write(json.dumps(event) + "\n")
            f.flush()

            # Process event
            yield event
```

### 4. Handle Truncated Output

```python
def handle_output(event: Dict[str, Any]):
    """Handle output events, detecting truncation."""
    data = event["data"]

    if data.get("truncated"):
        print("Warning: Output was truncated. Full output in trajectory file.")
        # Optionally read from trajectory after agent finishes

    print(data.get("stdout", ""))
```

---

## Trajectory Files

After each run, the complete trajectory is saved to `.miniswe/data/last_run.traj.json`.

### Trajectory Structure

```json
{
  "task": "Fix the authentication bug",
  "profile": "live",
  "model": "gpt-4o",
  "start_time": "2024-01-15T10:30:00.000Z",
  "end_time": "2024-01-15T10:35:00.000Z",
  "exit_status": "completed",
  "result": "Fixed the bug...",
  "total_cost": 0.0456,
  "steps": 12,
  "messages": [
    {"role": "system", "content": "..."},
    {"role": "user", "content": "..."},
    {"role": "assistant", "content": "..."},
    ...
  ],
  "commands": [
    {"command": "cat src/auth.py", "output": "...", "exit_code": 0},
    ...
  ]
}
```

### Loading Trajectories

```python
def analyze_trajectory(workdir: Path) -> None:
    """Analyze the last run trajectory."""
    traj_path = workdir / ".miniswe" / "data" / "last_run.traj.json"

    if not traj_path.exists():
        print("No trajectory found")
        return

    traj = json.loads(traj_path.read_text())

    print(f"Task: {traj['task']}")
    print(f"Status: {traj['exit_status']}")
    print(f"Steps: {traj['steps']}")
    print(f"Cost: ${traj['total_cost']:.4f}")
    print(f"Duration: {traj['end_time']} - {traj['start_time']}")

    print("\nCommands executed:")
    for cmd in traj.get("commands", []):
        print(f"  $ {cmd['command']} (exit: {cmd['exit_code']})")
```
