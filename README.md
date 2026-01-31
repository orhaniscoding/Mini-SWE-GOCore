# Mini-SWE-GOCore

[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://python.org)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE.md)
[![Version](https://img.shields.io/badge/version-2.0.0-green.svg)](https://github.com/anthropics/mini-swe-gocore)

**Proxy-Optimized Worker Agent for AI Software Engineering**

Mini-SWE-GOCore is a specialized fork of [mini-swe-agent](https://github.com/swe-agent/mini-swe-agent) designed as the **execution engine** for multi-agent orchestration systems.

---

## Primary Use Cases

| Use Case | Description |
|----------|-------------|
| **Worker for [GOrchestrator](https://github.com/lbjlaq/GOrchestrator)** | Spawned as subprocess by the orchestration platform |
| **Proxy-Routed Agent with [Antigravity Manager](https://github.com/lbjlaq/Antigravity-Manager)** | Centralized cost tracking and API key management |
| **Standalone Headless Agent** | CLI automation, CI/CD pipelines, script integration |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              YOUR ENVIRONMENT                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────┐         ┌─────────────────────┐         ┌──────────┐ │
│  │                  │         │                     │         │          │ │
│  │  GOrchestrator   │────────▶│  Mini-SWE-GOCore    │────────▶│ Antigrav │ │
│  │  (Brain)         │ spawns  │  (Worker)           │ routes  │ Manager  │ │
│  │                  │ as      │                     │ via     │ (Proxy)  │ │
│  └──────────────────┘ subprocess└────────────────────┘ API    └────┬─────┘ │
│         OR                              │                          │       │
│  ┌──────────────────┐                   │                          │       │
│  │  CLI / Scripts   │───────────────────┘                          ▼       │
│  │  (Direct Use)    │                                     ┌────────────────┐│
│  └──────────────────┘                                     │ LLM Providers  ││
│                                                           │ • Anthropic    ││
│                                                           │ • OpenAI       ││
│                                                           │ • Azure        ││
│                                                           │ • Vertex AI    ││
│                                                           └────────────────┘│
└─────────────────────────────────────────────────────────────────────────────┘
```

### Data Flow

```
┌─────────────┐     JSON Lines      ┌──────────────────┐     HTTP/REST     ┌─────────────┐
│             │ ◀──────────────────▶│                  │ ◀───────────────▶│             │
│ Orchestrator│   stdout/stdin      │ Mini-SWE-GOCore  │  MINI_API_BASE   │   Proxy     │
│ or CLI      │                     │                  │  localhost:8045  │             │
└─────────────┘                     └──────────────────┘                  └─────────────┘
```

---

## Key Features

| Feature | Description |
|---------|-------------|
| **Headless Mode** | JSON Lines output for automation - no TUI dependencies |
| **Proxy-First Design** | Route all LLM traffic through local proxy |
| **Provider/Model Syntax** | Explicit `provider/model` format for correct API headers |
| **Portable** | All agent files in `.miniswe/` directory |
| **GOrchestrator Ready** | Designed to be spawned as subprocess |

---

## Quick Start

### Prerequisites

- Python 3.10+
- [uv](https://github.com/astral-sh/uv) package manager
- [Antigravity Manager](https://github.com/lbjlaq/Antigravity-Manager) (recommended)

### Installation

```bash
git clone https://github.com/anthropics/mini-swe-gocore.git
cd mini-swe-gocore
uv sync
```

### Run with Proxy (Recommended)

```powershell
# 1. Set proxy endpoint (Antigravity Manager)
$env:MINI_API_BASE = "http://127.0.0.1:8045"

# 2. Bypass local validation (proxy handles real auth)
$env:ANTHROPIC_API_KEY = "sk-dummy"

# 3. Run with explicit provider/model syntax
uv run mini --headless --model anthropic/claude-sonnet-4-20250514 --task "Your task"
```

### Provider/Model Syntax

```bash
# Anthropic models
--model anthropic/claude-opus-4-5-thinking
--model anthropic/claude-sonnet-4-20250514

# OpenAI models
--model openai/gpt-4o
--model openai/gpt-4-turbo

# Vertex AI models
--model vertex_ai/gemini-pro
```

---

## Documentation

| Document | Description |
|----------|-------------|
| [Installation Guide](docs/installation.md) | Setup instructions |
| [Configuration Reference](docs/configuration.md) | All config options and environment variables |
| [Proxy Integration](docs/proxy_integration.md) | Using with Antigravity Manager |
| [Architecture Guide](docs/architecture.md) | Internal structure and execution flow |
| [Integration Guide](docs/integration.md) | JSON schema for GOrchestrator integration |
| [GOrchestrator Relationship](docs/relationship_with_gorchestrator.md) | Brain/Muscle architecture |

---

## Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `MINI_API_BASE` | Proxy endpoint | `http://127.0.0.1:8045` |
| `MINI_API_KEY` | Proxy authentication | `sk-proxy-xxx` |
| `MINI_API_TIMEOUT` | Request timeout (seconds) | `120` |
| `ANTHROPIC_API_KEY` | Bypass key (use `sk-dummy` with proxy) | `sk-dummy` |
| `OPENAI_API_KEY` | Bypass key | `sk-dummy` |

---

## CLI Reference

```
Usage: mini [OPTIONS]

Options:
  -p, --profile TEXT      Profile name (loads .miniswe/configs/{profile}.yaml)
  -w, --workdir PATH      Working directory [default: .]
      --headless          JSON Lines output, no interactive TUI
  -t, --task TEXT         Task/problem statement
  -m, --model TEXT        Model (e.g., 'anthropic/claude-sonnet-4-20250514')
  -y, --yolo              Run without confirmation prompts
  -l, --cost-limit FLOAT  Cost limit in USD (0 to disable)
  -o, --output PATH       Output trajectory file path
      --help              Show this message and exit
```

---

## Credits & Acknowledgments

This project is a specialized fork built upon the excellent work of:

### Original Authors

**[mini-swe-agent](https://github.com/swe-agent/mini-swe-agent)** by:

- **[Princeton NLP Group](https://nlp.cs.princeton.edu/)** - Princeton University
- **[Stanford NLP Group](https://nlp.stanford.edu/)** - Stanford University

Original authors:
- **Kilian Lieret** ([@klieret](https://github.com/klieret))
- **Carlos E. Jimenez** ([@carlosejimenez](https://github.com/carlosejimenez))

We extend our sincere gratitude to the Princeton NLP and Stanford NLP teams for creating the robust foundation upon which Mini-SWE-GOCore is built.

### Related Projects

- [SWE-agent](https://github.com/princeton-nlp/SWE-agent) - The original full-featured agent
- [SWE-bench](https://github.com/princeton-nlp/SWE-bench) - Benchmark for software engineering agents
- [GOrchestrator](https://github.com/lbjlaq/GOrchestrator) - Multi-agent orchestration platform
- [Antigravity Manager](https://github.com/lbjlaq/Antigravity-Manager) - LLM proxy for cost tracking

---

## License

MIT License - See [LICENSE.md](LICENSE.md)

```
Copyright (c) 2025 Kilian A. Lieret and Carlos E. Jimenez
```

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for development setup and guidelines.
