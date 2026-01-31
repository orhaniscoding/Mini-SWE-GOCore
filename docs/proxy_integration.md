# Proxy Integration Guide

This guide explains how to use Mini-SWE-GOCore with **Antigravity Manager** or other LLM proxy servers for centralized API management.

---

## Why Use a Proxy?

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         WITHOUT PROXY                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────┐     API Key #1     ┌─────────────┐                           │
│  │ Agent 1  │ ──────────────────▶│  Anthropic  │                           │
│  └──────────┘                    └─────────────┘                           │
│                                                                             │
│  ┌──────────┐     API Key #2     ┌─────────────┐                           │
│  │ Agent 2  │ ──────────────────▶│   OpenAI    │  ❌ No cost visibility    │
│  └──────────┘                    └─────────────┘  ❌ Keys scattered         │
│                                                   ❌ No rate limiting       │
│  ┌──────────┐     API Key #3     ┌─────────────┐  ❌ Hard to rotate keys   │
│  │ Agent 3  │ ──────────────────▶│   Azure     │                           │
│  └──────────┘                    └─────────────┘                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                          WITH PROXY                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────┐                                                               │
│  │ Agent 1  │ ─┐                                                            │
│  └──────────┘  │                                                            │
│                │              ┌──────────────────┐     ┌─────────────┐     │
│  ┌──────────┐  ├─ Proxy Key ─▶│    Antigravity   │────▶│  Anthropic  │     │
│  │ Agent 2  │ ─┤              │      Manager     │────▶│   OpenAI    │     │
│  └──────────┘  │              │                  │────▶│   Azure     │     │
│                │              │  ✅ Cost tracking │     └─────────────┘     │
│  ┌──────────┐  │              │  ✅ Single key   │                          │
│  │ Agent 3  │ ─┘              │  ✅ Rate limiting│                          │
│  └──────────┘                 │  ✅ Easy rotation│                          │
│                               └──────────────────┘                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Benefits

| Benefit | Description |
|---------|-------------|
| **Cost Tracking** | Real-time visibility into token usage and costs per project/agent |
| **Centralized Keys** | Single place to manage all provider API keys |
| **Rate Limiting** | Prevent runaway agents from burning through budget |
| **Key Rotation** | Rotate provider keys without touching agent configs |
| **Audit Logging** | Complete log of all LLM requests for compliance |
| **Model Routing** | Route requests to different providers based on rules |

---

## Antigravity Manager Setup

### Step 1: Install Antigravity Manager

```bash
# Clone and set up Antigravity Manager
git clone https://github.com/anthropics/antigravity-manager.git
cd antigravity-manager
uv sync

# Configure your provider keys
cp .env.example .env
# Edit .env with your actual API keys
```

### Step 2: Start the Proxy

```bash
# Start on default port 8045
uv run antigravity serve

# Or specify a port
uv run antigravity serve --port 8080
```

The proxy is now running at `http://127.0.0.1:8045`

### Step 3: Configure Mini-SWE-GOCore

#### Option A: Environment Variables

```powershell
# Windows PowerShell
$env:MINI_API_BASE = "http://127.0.0.1:8045"
$env:MINI_API_KEY = "your-antigravity-key"  # If proxy requires auth

# IMPORTANT: Bypass local LiteLLM validation
$env:ANTHROPIC_API_KEY = "sk-dummy"
$env:OPENAI_API_KEY = "sk-dummy"

# Run agent
uv run mini --headless --model anthropic/claude-sonnet-4-20250514 --task "Your task"
```

```bash
# macOS / Linux
export MINI_API_BASE="http://127.0.0.1:8045"
export MINI_API_KEY="your-antigravity-key"
export ANTHROPIC_API_KEY="sk-dummy"
export OPENAI_API_KEY="sk-dummy"

uv run mini --headless --model anthropic/claude-sonnet-4-20250514 --task "Your task"
```

#### Option B: .env File

Create `.miniswe/configs/.env`:

```env
# Proxy Configuration
MINI_API_BASE=http://127.0.0.1:8045
MINI_API_KEY=your-antigravity-key

# Bypass keys (proxy handles real auth)
ANTHROPIC_API_KEY=sk-dummy
OPENAI_API_KEY=sk-dummy
```

#### Option C: YAML Profile

Edit `.miniswe/configs/live.yaml`:

```yaml
model:
  model_class: litellm
  model_name: "anthropic/claude-sonnet-4-20250514"
  api_base: "http://127.0.0.1:8045"
  api_key: "your-antigravity-key"
  cost_tracking: "ignore_errors"
```

---

## Understanding the Bypass Pattern

### Why `sk-dummy`?

LiteLLM performs local validation of API keys before making requests. When using a proxy, this validation fails because:

1. LiteLLM expects a real Anthropic key format
2. The real key is on the proxy server, not the client
3. Local validation runs before the request reaches the proxy

### The Solution

```
┌───────────────────────────────────────────────────────────────────────────┐
│  Mini-SWE-GOCore Client                                                    │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  1. ANTHROPIC_API_KEY="sk-dummy"                                          │
│     └── Satisfies LiteLLM's local validation check                        │
│                                                                           │
│  2. Request prepared with provider/model format                            │
│     └── anthropic/claude-sonnet-4-20250514                                 │
│                                                                           │
│  3. Request sent to MINI_API_BASE                                          │
│     └── http://127.0.0.1:8045                                             │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────────────────┐
│  Antigravity Manager Proxy                                                 │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  1. Receives request with dummy key                                        │
│     └── Ignores client-provided key                                        │
│                                                                           │
│  2. Parses provider from model name                                        │
│     └── "anthropic/claude-sonnet-4-20250514" → Anthropic provider          │
│                                                                           │
│  3. Injects real API key from proxy config                                 │
│     └── Uses ANTHROPIC_API_KEY from proxy's .env                          │
│                                                                           │
│  4. Forwards request to actual provider                                    │
│     └── https://api.anthropic.com/v1/messages                             │
│                                                                           │
│  5. Logs usage and calculates cost                                         │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

### Security Note

The `sk-dummy` pattern is **safe** because:
- It never leaves your machine as a real credential
- The proxy intercepts requests before they reach the internet
- Real credentials are stored only on the proxy server

---

## Provider/Model Syntax

Mini-SWE-GOCore uses `provider/model` syntax to ensure correct API formatting:

```bash
# Anthropic protocol (Anthropic-specific headers)
--model anthropic/claude-opus-4-5-thinking
--model anthropic/claude-sonnet-4-20250514

# OpenAI protocol
--model openai/gpt-4o
--model openai/gpt-4-turbo

# Azure OpenAI
--model azure/my-deployment-name

# Google Vertex AI
--model vertex_ai/gemini-pro

# Local Ollama
--model ollama/llama2
--model ollama/codellama
```

This syntax tells the proxy:
1. Which API format to use (message structure, headers)
2. Which provider endpoint to route to
3. Which API key to inject

---

## Alternative Proxies

### LiteLLM Proxy Server

```bash
# Start LiteLLM proxy
litellm --model gpt-4o --port 4000

# Configure Mini-SWE-GOCore
export MINI_API_BASE="http://localhost:4000"
```

### OpenAI-Compatible Servers

Any OpenAI-compatible server works:

```yaml
# For vLLM, LocalAI, text-generation-inference, etc.
model:
  model_class: litellm
  model_name: "openai/my-local-model"
  api_base: "http://localhost:8000/v1"
```

---

## Troubleshooting

### "Invalid API Key" Error

**Symptom**: Error during local validation, before request sent

**Solution**: Set bypass keys
```bash
export ANTHROPIC_API_KEY="sk-dummy"
export OPENAI_API_KEY="sk-dummy"
```

### "Connection Refused"

**Symptom**: `ConnectionRefusedError: [Errno 111]`

**Solution**: Ensure proxy is running
```bash
# Check if proxy is running
curl http://127.0.0.1:8045/health

# Start proxy if not running
cd antigravity-manager && uv run antigravity serve
```

### "401 Unauthorized" from Proxy

**Symptom**: Request reaches proxy but is rejected

**Solution**: Check proxy authentication
```bash
# Verify MINI_API_KEY matches proxy config
echo $MINI_API_KEY
```

### "Model Not Found"

**Symptom**: Proxy doesn't recognize model name

**Solution**: Use provider prefix
```bash
# Wrong
--model claude-sonnet-4-20250514

# Correct
--model anthropic/claude-sonnet-4-20250514
```

### Timeout Issues

**Symptom**: Requests timing out

**Solution**: Increase timeout
```bash
export MINI_API_TIMEOUT=300  # 5 minutes
```

---

## Best Practices

1. **Always use provider prefix** for clarity: `anthropic/model-name`
2. **Store proxy URL in .env** to avoid repetition
3. **Use cost limits** to prevent runaway spending: `--cost-limit 5.0`
4. **Check proxy logs** when debugging issues
5. **Rotate proxy keys** periodically for security

---

## Next Steps

- [Configuration Reference](configuration.md) - All configuration options
- [GOrchestrator Relationship](relationship_with_gorchestrator.md) - Multi-agent orchestration
