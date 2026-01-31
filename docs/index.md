# Mini-SWE-GOCore Documentation

## What is Mini-SWE-GOCore?

**Mini-SWE-GOCore** is a specialized fork of [mini-swe-agent](https://github.com/swe-agent/mini-swe-agent) designed as a **proxy-optimized worker agent** for AI software engineering tasks.

> **Credits:** This project builds upon the excellent work of the **Princeton NLP** & **Stanford NLP** groups who created mini-swe-agent.

---

## Primary Role: Worker for GOrchestrator

Mini-SWE-GOCore is designed to be the **"Muscle"** controlled by [GOrchestrator](https://github.com/lbjlaq/GOrchestrator) (the **"Brain"**):

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ORCHESTRATION PATTERN                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   GOrchestrator (Brain)                                             │
│   ├── Task decomposition                                            │
│   ├── Agent lifecycle management                                    │
│   └── Result aggregation                                            │
│            │                                                        │
│            │ spawns as subprocess                                   │
│            ▼                                                        │
│   Mini-SWE-GOCore (Muscle)                                          │
│   ├── Executes single focused task                                  │
│   ├── Outputs JSON Lines to stdout                                  │
│   └── Routes API calls through proxy                                │
│            │                                                        │
│            │ MINI_API_BASE                                          │
│            ▼                                                        │
│   Antigravity Manager (Proxy)                                       │
│   ├── Centralized API key management                                │
│   ├── Cost tracking per agent/project                               │
│   └── Routes to actual LLM providers                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

While it can be used **standalone** via CLI, it is **optimized** for headless operation under orchestrator control.

---

## Key Differentiators

| Feature | Original mini-swe-agent | Mini-SWE-GOCore |
|---------|------------------------|-----------------|
| **API Routing** | Direct to providers | Via local proxy (Antigravity Manager) |
| **Provider Selection** | LiteLLM auto-detect | Explicit `provider/model` format |
| **Auth Pattern** | Standard API keys | Dummy key bypass + proxy routing |
| **Primary Mode** | Interactive TUI | Headless (JSON Lines) |
| **Use Case** | General | Orchestration systems |

---

## The Provider/Model Syntax

Mini-SWE-GOCore introduces **explicit provider selection** via `provider/model` format:

```bash
# Anthropic protocol (Claude models)
--model anthropic/claude-opus-4-5-thinking
--model anthropic/claude-sonnet-4-20250514

# OpenAI protocol (GPT models)
--model openai/gpt-4o
--model openai/gpt-4-turbo

# Vertex AI protocol (Gemini models)
--model vertex_ai/gemini-pro
```

This ensures your proxy receives requests with the **correct headers and format** for each provider.

---

## Quick Start with Proxy

```powershell
# 1. Set proxy URL (Antigravity Manager)
$env:MINI_API_BASE = "http://127.0.0.1:8045"

# 2. Set dummy key to bypass local validation
$env:ANTHROPIC_API_KEY = "sk-dummy"

# 3. Run with explicit provider
uv run mini --headless --model anthropic/claude-sonnet-4-20250514 --task "Your task"
```

---

## The Bypass Pattern Explained

**Why do we need a "dummy" API key?**

LiteLLM performs **local validation** before sending requests. Without provider-specific keys, it throws `AuthenticationError` before the request ever reaches your proxy.

**The Solution:**

1. Set `ANTHROPIC_API_KEY=sk-dummy` → Passes local validation
2. Set `MINI_API_BASE` → Routes traffic to proxy
3. Proxy uses **real** API keys stored securely on the server

This keeps real credentials on the proxy server, never exposed on client machines.

---

## Documentation Index

| Document | Description |
|----------|-------------|
| [Installation Guide](installation.md) | Setup with uv, environment configuration |
| [Configuration Reference](configuration.md) | YAML config, environment variables, profiles |
| [Proxy Integration](proxy_integration.md) | Antigravity Manager setup, bypass pattern |
| [Architecture Guide](architecture.md) | Directory structure, execution flow |
| [Integration Guide](integration.md) | JSON schema, subprocess examples for GOrchestrator |
| [GOrchestrator Relationship](relationship_with_gorchestrator.md) | Brain/Muscle pattern |

---

## Environment Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `MINI_API_BASE` | Proxy URL | `http://127.0.0.1:8045` |
| `MINI_API_KEY` | Proxy authentication | `sk-proxy-xxx` |
| `MINI_API_TIMEOUT` | Request timeout (seconds) | `120` |
| `ANTHROPIC_API_KEY` | Bypass key (dummy) | `sk-dummy` |
| `OPENAI_API_KEY` | Bypass key (dummy) | `sk-dummy` |

---

## Links

- **Original Project:** [mini-swe-agent](https://github.com/swe-agent/mini-swe-agent) (Princeton NLP & Stanford NLP)
- **Orchestrator:** [GOrchestrator](https://github.com/lbjlaq/GOrchestrator)
- **Proxy:** [Antigravity Manager](https://github.com/lbjlaq/Antigravity-Manager)
- **Full SWE-agent:** [princeton-nlp/SWE-agent](https://github.com/princeton-nlp/SWE-agent)
