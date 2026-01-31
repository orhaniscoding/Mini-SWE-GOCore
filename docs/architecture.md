# Architecture Guide

## Overview

**Mini-SWE-GOCore** is a specialized fork of [mini-swe-agent](https://github.com/swe-agent/mini-swe-agent) designed as a **headless, portable execution engine** for AI software engineering agents. This document explains the internal architecture, directory structure, proxy integration, and execution flow.

> **Credits:** Built upon the excellent work of the Princeton NLP & Stanford NLP groups.

---

## Proxy Architecture

Mini-SWE-GOCore is designed to work with a local proxy (e.g., **AntigravityManager**) for centralized API management.

### Traffic Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            REQUEST FLOW                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   1. User runs: uv run mini --model anthropic/claude-opus-4-5-thinking   │
│                                                                          │
│   2. Mini-SWE-GOCore parses "anthropic/claude-opus-4-5-thinking"         │
│      ├─ provider = "anthropic"                                           │
│      └─ model = "claude-opus-4-5-thinking"                               │
│                                                                          │
│   3. LiteLLM checks ANTHROPIC_API_KEY                                    │
│      └─ Finds "sk-dummy" → validation passes                             │
│                                                                          │
│   4. Request sent to MINI_API_BASE (http://127.0.0.1:8045)               │
│      ├─ Headers: Anthropic format (x-api-key, anthropic-version)         │
│      └─ Body: Anthropic Messages API format                              │
│                                                                          │
│   5. Proxy receives request                                              │
│      ├─ Identifies provider from headers/path                            │
│      ├─ Injects real ANTHROPIC_API_KEY                                   │
│      └─ Forwards to api.anthropic.com                                    │
│                                                                          │
│   6. Response flows back through proxy to agent                          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Provider Selection Logic

The `_parse_provider_model` function in `litellm_model.py` handles explicit provider selection:

```python
def _parse_provider_model(self, model_name: str) -> tuple[str, Optional[str]]:
    """Parse 'provider/model' format and return (model, provider).

    Examples:
        'anthropic/claude-thinking' -> ('claude-thinking', 'anthropic')
        'openai/gpt-4' -> ('gpt-4', 'openai')
        'vertex_ai/gemini' -> ('gemini', 'vertex_ai')
        'gpt-4o' -> ('gpt-4o', None)  # LiteLLM auto-detects
    """
    if "/" in model_name:
        provider, model_real_name = model_name.split("/", 1)
        return model_real_name, provider
    return model_name, None
```

When a provider is explicitly specified:
- `custom_llm_provider` is set in LiteLLM
- Correct headers and request format are used
- Proxy receives properly formatted requests

---

## The `.miniswe` Standard

All agent-related files are contained within a single directory: `.miniswe/`

This directory is created **inside your project root** (the `--workdir`), ensuring complete isolation from system-wide configurations.

### Directory Layout

```
{workdir}/
└── .miniswe/                    # Agent Home Directory
    ├── .gitignore               # Auto-generated, ignores data/ and logs/
    ├── configs/                 # Configuration files (VERSION CONTROLLED)
    │   ├── .env                 # Environment variables (API keys)
    │   ├── live.yaml            # Production profile
    │   ├── dev.yaml             # Development profile (optional)
    │   └── custom.yaml          # Custom profiles (optional)
    ├── data/                    # Runtime data (IGNORED by git)
    │   ├── last_run.traj.json   # Most recent run trajectory
    │   └── task_history.txt     # Command history for interactive mode
    └── logs/                    # Debug logs (IGNORED by git)
        └── debug.log            # Detailed debug output
```

---

## Directory Purposes

### `configs/` - Configuration Profiles

**Git Status:** ✅ Version Controlled

This directory contains YAML configuration profiles that define how the agent behaves.

| File | Purpose |
|------|---------|
| `.env` | Environment variables (API keys, secrets) |
| `live.yaml` | Auto-generated default profile |
| `*.yaml` | Custom profiles loaded via `--profile` |

**Why Version Control?**
- Share agent configurations across team members
- Track changes to agent behavior over time
- Ensure reproducible runs across environments

### `data/` - Runtime Data

**Git Status:** ❌ Ignored

This directory contains data generated during agent execution.

| File | Purpose |
|------|---------|
| `last_run.traj.json` | Complete trajectory of the most recent run |
| `task_history.txt` | History of tasks entered in interactive mode |

**Why Ignored?**
- Contains potentially sensitive code snippets
- Large files that bloat repositories
- Not needed for reproducibility (configs are sufficient)

### `logs/` - Debug Output

**Git Status:** ❌ Ignored

This directory contains debug logs for troubleshooting.

| File | Purpose |
|------|---------|
| `debug.log` | Detailed execution logs with timestamps |

**Why Ignored?**
- Debug information is ephemeral
- Can contain sensitive data
- Only needed for local troubleshooting

---

## Isolation Guarantee

Mini-SWE-GOCore **NEVER** touches:

| Path | Status |
|------|--------|
| `~/.config/` | ❌ Never accessed |
| `~/.local/` | ❌ Never accessed |
| `%AppData%` | ❌ Never accessed |
| `%LocalAppData%` | ❌ Never accessed |
| `/etc/` | ❌ Never accessed |
| System temp directories | ❌ Never used for persistence |

**Everything** is contained within `{workdir}/.miniswe/`.

### Portability Benefits

1. **Copy a project = Copy the agent state**
   ```bash
   cp -r my-project /backup/
   # Agent configs, history, everything is preserved
   ```

2. **Delete agent state = Clean slate**
   ```bash
   rm -rf .miniswe/
   # Next run will auto-initialize fresh
   ```

3. **Multiple projects = Isolated agents**
   ```bash
   project-a/.miniswe/  # Agent A's state
   project-b/.miniswe/  # Agent B's state (completely separate)
   ```

---

## Path Resolution

Mini-SWE-GOCore uses these functions to resolve paths:

| Function | Returns | Example |
|----------|---------|---------|
| `get_workdir()` | Project root | `/home/user/my-project` |
| `get_agent_dir()` | Agent home | `/home/user/my-project/.miniswe` |
| `get_config_dir()` | Configs | `/home/user/my-project/.miniswe/configs` |
| `get_data_dir()` | Data | `/home/user/my-project/.miniswe/data` |
| `get_logs_dir()` | Logs | `/home/user/my-project/.miniswe/logs` |

### Resolution Priority

1. `MINI_WORKDIR` environment variable (if set)
2. `--workdir` CLI argument (if provided)
3. Current working directory (default)

---

## Execution Flow

When you run `mini --headless --profile live --task "Fix bug"`, the following sequence occurs:

```
┌─────────────────────────────────────────────────────────────────┐
│                        STARTUP PHASE                            │
├─────────────────────────────────────────────────────────────────┤
│ 1. Parse CLI arguments                                          │
│    └─ workdir, headless, profile, task, etc.                    │
│                                                                 │
│ 2. Set working directory                                        │
│    └─ set_workdir(workdir)                                      │
│                                                                 │
│ 3. Set headless mode                                            │
│    └─ set_headless(True) if --headless                          │
│                                                                 │
│ 4. Auto-initialize .miniswe/                                    │
│    └─ initialize_agent_dir()                                    │
│    └─ Creates configs/, data/, logs/ if missing                 │
│    └─ Generates live.yaml if missing                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     CONFIGURATION PHASE                         │
├─────────────────────────────────────────────────────────────────┤
│ 5. Resolve configuration                                        │
│    └─ Priority: --config > --profile > default                  │
│                                                                 │
│ 6. Load YAML configuration                                      │
│    └─ yaml.safe_load(config_path.read_text())                   │
│                                                                 │
│ 7. Apply CLI overrides                                          │
│    └─ --yolo → agent.mode = "yolo"                              │
│    └─ --cost-limit → agent.cost_limit = value                   │
│    └─ --model → model.model_name = value                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    INITIALIZATION PHASE                         │
├─────────────────────────────────────────────────────────────────┤
│ 8. Create Model instance                                        │
│    └─ get_model(model_name, config["model"])                    │
│                                                                 │
│ 9. Create Environment instance                                  │
│    └─ LocalEnvironment(**config["environment"])                 │
│                                                                 │
│ 10. Select Agent class                                          │
│     └─ HeadlessAgent (if --headless)                            │
│     └─ InteractiveAgent (otherwise)                             │
│                                                                 │
│ 11. Create Agent instance                                       │
│     └─ agent = AgentClass(model, env, **config["agent"])        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      EXECUTION PHASE                            │
├─────────────────────────────────────────────────────────────────┤
│ 12. Run agent loop                                              │
│     └─ agent.run(task)                                          │
│                                                                 │
│ 13. Agent loop iterations:                                      │
│     ┌─────────────────────────────────────────────┐             │
│     │ a. Query model with current context         │             │
│     │ b. Parse response for commands              │             │
│     │ c. Execute commands in environment          │             │
│     │ d. Emit JSON events (headless)              │             │
│     │ e. Check termination conditions             │             │
│     │ f. Repeat until done or limit reached       │             │
│     └─────────────────────────────────────────────┘             │
│                                                                 │
│ 14. Return (exit_status, result)                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      CLEANUP PHASE                              │
├─────────────────────────────────────────────────────────────────┤
│ 15. Save trajectory                                             │
│     └─ save_traj(agent, output_path, ...)                       │
│     └─ Written to .miniswe/data/last_run.traj.json              │
│                                                                 │
│ 16. Exit with status                                            │
│     └─ Exit code 0 on success                                   │
│     └─ Exit code 1 on error                                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Agent Classes

### HeadlessAgent

Used when `--headless` flag is provided.

**Characteristics:**
- No TUI, no interactive prompts
- All output is JSON Lines to stdout
- Designed for subprocess integration
- Machine-parseable events

**Output Stream:**
```
stdout → JSON Lines (events)
stderr → Error messages (if any)
```

### InteractiveAgent

Used in default (non-headless) mode.

**Characteristics:**
- Rich TUI with colors and formatting
- Interactive task input with history
- Confirmation prompts before actions
- Human-readable output

---

## Error Handling

### Graceful Degradation

If configuration loading fails:
1. Profile not found → Use SAFE_MODE_CONFIG defaults
2. Invalid YAML → Exit with error JSON
3. Missing API key → Model will fail at query time

### Trajectory Always Saved

Even on exceptions, the trajectory is saved:
```python
try:
    exit_status, result = agent.run(task)
except Exception as e:
    exit_status, result = type(e).__name__, str(e)
    extra_info = {"traceback": traceback.format_exc()}
finally:
    save_traj(agent, output, exit_status=exit_status, ...)  # Always runs
```

---

## Threading Model

Mini-SWE-GOCore is **single-threaded** by design:

- One agent instance per process
- Synchronous API calls
- Sequential command execution

For parallel task execution, spawn multiple processes:
```bash
# Run multiple agents in parallel (from orchestrator)
mini --headless --workdir ./project-a --task "Task A" &
mini --headless --workdir ./project-b --task "Task B" &
wait
```
