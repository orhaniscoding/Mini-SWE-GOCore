# Configuration Reference

## Overview

**Mini-SWE-GOCore** uses YAML configuration files stored in `.miniswe/configs/`. This document provides a complete reference for all configuration options, with special focus on **proxy configuration** for use with AntigravityManager.

> **Note:** This is a specialized fork of [mini-swe-agent](https://github.com/swe-agent/mini-swe-agent) designed for local proxy routing.

---

## Configuration Loading

### Priority Order

1. `--config /path/to/file.yaml` (direct path, highest priority)
2. `--profile NAME` (loads `.miniswe/configs/NAME.yaml`)
3. `default` profile (`.miniswe/configs/default.yaml`)
4. Built-in fallback (SAFE_MODE_CONFIG)

### Auto-Generated Default

On first run, `.miniswe/configs/live.yaml` is automatically created with sensible defaults.

---

## Complete `live.yaml` Reference

```yaml
# =============================================================================
# Mini-SWE-GOCore Configuration
# =============================================================================
# Profile: live
# Generated: Auto-created on first run
# =============================================================================

# -----------------------------------------------------------------------------
# MODEL CONFIGURATION
# -----------------------------------------------------------------------------
model:
  # Model class determines which backend to use
  # Options: "litellm" (recommended), "openai", "anthropic"
  model_class: litellm

  # Model identifier (passed to the backend)
  # Examples:
  #   - "gpt-4o"           (OpenAI)
  #   - "gpt-4-turbo"      (OpenAI)
  #   - "claude-3-opus"    (Anthropic via LiteLLM)
  #   - "ollama/llama2"    (Local Ollama)
  model_name: "gpt-4o"

  # Cost tracking behavior
  # Options:
  #   - "strict"        Error if cost cannot be calculated
  #   - "ignore_errors" Continue even if cost tracking fails (recommended)
  #   - "disabled"      No cost tracking
  cost_tracking: "ignore_errors"

  # Model-specific parameters (passed directly to the API)
  model_kwargs:
    temperature: 0.0      # 0.0 = deterministic, 1.0 = creative
    max_tokens: 4096      # Maximum response length

  # API configuration (optional, can also use environment variables)
  # api_base: "http://127.0.0.1:8045"  # Custom endpoint
  # api_key: "sk-..."                   # API key (prefer .env file)

# -----------------------------------------------------------------------------
# ENVIRONMENT CONFIGURATION
# -----------------------------------------------------------------------------
environment:
  # Working directory for command execution (relative to workdir)
  cwd: "."

  # Command execution timeout in seconds
  timeout: 120

  # Shell to use for command execution (optional)
  # shell: "/bin/bash"

# -----------------------------------------------------------------------------
# AGENT CONFIGURATION
# -----------------------------------------------------------------------------
agent:
  # Execution mode
  # Options:
  #   - "confirm"  Ask for confirmation before executing commands (interactive)
  #   - "yolo"     Execute commands without confirmation (headless)
  mode: "confirm"

  # Maximum number of agent steps before forced termination
  step_limit: 50

  # Maximum cost in USD before forced termination (0 = unlimited)
  cost_limit: 5.0

  # Whether to ask for confirmation before exiting (interactive only)
  confirm_exit: true

# -----------------------------------------------------------------------------
# HEADLESS MODE CONFIGURATION
# -----------------------------------------------------------------------------
headless:
  # Include ISO 8601 timestamps in JSON events
  include_timestamps: true

  # Maximum characters for command output in events
  # Longer outputs are truncated with "[truncated]" marker
  max_output_length: 10000
```

---

## Configuration Sections

### `model` Section

Controls the language model backend and parameters.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `model_class` | string | `"litellm"` | Backend: `litellm`, `openai`, `anthropic` |
| `model_name` | string | `"gpt-4o"` | Model identifier |
| `cost_tracking` | string | `"ignore_errors"` | Cost tracking mode |
| `model_kwargs` | object | `{}` | Parameters passed to API |
| `api_base` | string | (env) | Custom API endpoint |
| `api_key` | string | (env) | API key |

#### Model Names by Provider

**OpenAI (via LiteLLM):**
```yaml
model_name: "gpt-4o"
model_name: "gpt-4-turbo"
model_name: "gpt-3.5-turbo"
```

**Anthropic (via LiteLLM):**
```yaml
model_name: "claude-3-opus-20240229"
model_name: "claude-3-sonnet-20240229"
model_name: "claude-3-haiku-20240307"
```

**Local Ollama:**
```yaml
model_name: "ollama/llama2"
model_name: "ollama/codellama"
model_name: "ollama/mistral"
api_base: "http://localhost:11434"
```

**Custom Proxy (Antigravity, LiteLLM Proxy):**
```yaml
model_name: "gpt-4o"  # or any model your proxy supports
api_base: "http://127.0.0.1:8045"
api_key: "your-proxy-key"
```

### `environment` Section

Controls command execution environment.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `cwd` | string | `"."` | Working directory for commands |
| `timeout` | integer | `120` | Command timeout in seconds |
| `shell` | string | (system) | Shell executable path |

### `agent` Section

Controls agent behavior and limits.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `mode` | string | `"confirm"` | `confirm` or `yolo` |
| `step_limit` | integer | `50` | Max agent iterations |
| `cost_limit` | float | `5.0` | Max cost in USD (0 = unlimited) |
| `confirm_exit` | boolean | `true` | Confirm before exit (interactive) |

### `headless` Section

Controls headless mode output.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `include_timestamps` | boolean | `true` | Add timestamps to events |
| `max_output_length` | integer | `10000` | Max chars per output event |

---

## Environment Variables

Environment variables can be set in `.miniswe/configs/.env` or in the shell.

### Core Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `MINI_WORKDIR` | Override working directory | `/path/to/project` |
| `MINI_HEADLESS` | Force headless mode | `true`, `false` |

### API Configuration

| Variable | Description | Example |
|----------|-------------|---------|
| `MINI_API_BASE` | Custom API endpoint | `http://127.0.0.1:8045` |
| `MINI_API_KEY` | API key for custom endpoint | `sk-...` |
| `MINI_API_TIMEOUT` | Request timeout (seconds) | `60` |

### Provider API Keys

| Variable | Description |
|----------|-------------|
| `OPENAI_API_KEY` | OpenAI API key |
| `ANTHROPIC_API_KEY` | Anthropic API key |
| `AZURE_API_KEY` | Azure OpenAI key |
| `AZURE_API_BASE` | Azure endpoint |

### `.env` File Example

```env
# .miniswe/configs/.env

# Provider API Keys
OPENAI_API_KEY=sk-proj-xxxxxxxxxxxxxxxxxxxx
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxxxxxxxxx

# Custom Proxy (overrides provider keys)
# MINI_API_BASE=http://127.0.0.1:8045
# MINI_API_KEY=proxy-secret-key

# Behavior
# MINI_HEADLESS=true
```

---

## Proxy Configuration

### Scenario 1: Antigravity Proxy

```yaml
# .miniswe/configs/live.yaml
model:
  model_class: litellm
  model_name: "gpt-4o"  # Model name your proxy understands
  api_base: "http://127.0.0.1:8045"
  api_key: "antigravity-key"  # If required
```

Or via environment:
```bash
MINI_API_BASE=http://127.0.0.1:8045 mini --headless --task "Fix bug"
```

### Scenario 2: LiteLLM Proxy Server

```yaml
model:
  model_class: litellm
  model_name: "gpt-4o"
  api_base: "http://localhost:4000"
```

### Scenario 3: Local Ollama

```yaml
model:
  model_class: litellm
  model_name: "ollama/codellama"
  api_base: "http://localhost:11434"
  model_kwargs:
    temperature: 0.1
```

### Scenario 4: Azure OpenAI

```yaml
model:
  model_class: litellm
  model_name: "azure/my-deployment-name"
  api_base: "https://my-resource.openai.azure.com"
  api_key: "${AZURE_API_KEY}"  # Reference env var
  model_kwargs:
    api_version: "2024-02-15-preview"
```

---

## CLI Overrides

CLI arguments override configuration file values:

| CLI Argument | Overrides |
|--------------|-----------|
| `--model NAME` | `model.model_name` |
| `--yolo` | `agent.mode = "yolo"` |
| `--cost-limit N` | `agent.cost_limit = N` |

**Example:**
```bash
# Config says model_name: "gpt-4o", but we override:
mini --profile live --model "claude-3-opus" --yolo --task "Fix bug"
```

---

## Multiple Profiles

Create different profiles for different scenarios:

### Development Profile

```yaml
# .miniswe/configs/dev.yaml
model:
  model_name: "gpt-3.5-turbo"  # Cheaper for testing
  cost_tracking: "disabled"

agent:
  mode: "confirm"
  step_limit: 10
  cost_limit: 0.50
```

### Production Profile

```yaml
# .miniswe/configs/prod.yaml
model:
  model_name: "gpt-4o"
  cost_tracking: "strict"

agent:
  mode: "yolo"
  step_limit: 100
  cost_limit: 10.0
```

### Local Testing Profile

```yaml
# .miniswe/configs/local.yaml
model:
  model_name: "ollama/codellama"
  api_base: "http://localhost:11434"
  cost_tracking: "disabled"

agent:
  mode: "yolo"
  step_limit: 20
  cost_limit: 0  # Unlimited (local is free)
```

**Usage:**
```bash
mini --profile dev --task "Quick test"
mini --profile prod --task "Important fix"
mini --profile local --task "Offline development"
```

---

## Validation

Mini-SWE-GOCore performs minimal validation on configuration:

| Check | Behavior |
|-------|----------|
| Missing file | Fall back to defaults |
| Invalid YAML | Exit with error |
| Unknown keys | Ignored (forward compatibility) |
| Invalid types | May cause runtime errors |

**Tip:** Test your configuration with a simple task:
```bash
mini --profile myprofile --task "Say hello" --yolo
```
