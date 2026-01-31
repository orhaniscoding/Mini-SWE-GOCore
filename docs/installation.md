# Installation Guide

This guide covers the complete installation process for Mini-SWE-GOCore on Windows, macOS, and Linux.

---

## Prerequisites

### Required

| Requirement | Version | Purpose |
|-------------|---------|---------|
| Python | 3.10+ | Runtime |
| uv | Latest | Package management |
| Git | Any | Repository cloning |

### Recommended

| Tool | Purpose |
|------|---------|
| [Antigravity Manager](https://github.com/anthropics/antigravity-manager) | Local LLM proxy for cost tracking |
| PowerShell 7+ (Windows) | Better environment variable handling |

---

## Step 1: Install uv

[uv](https://github.com/astral-sh/uv) is a fast Python package manager that handles virtual environments automatically.

### Windows (PowerShell)

```powershell
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"
```

### macOS / Linux

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### Verify Installation

```bash
uv --version
# Output: uv 0.x.x
```

---

## Step 2: Clone the Repository

```bash
git clone https://github.com/anthropics/mini-swe-gocore.git
cd mini-swe-gocore
```

---

## Step 3: Install Dependencies

uv automatically creates a virtual environment and installs all dependencies:

```bash
uv sync
```

This will:
1. Create `.venv/` in the project directory
2. Install all dependencies from `pyproject.toml`
3. Install the package in editable mode

### Optional: Development Dependencies

For development work (testing, linting):

```bash
uv sync --extra dev
```

---

## Step 4: Verify Installation

### Check CLI

```bash
uv run mini --help
```

Expected output:
```
Usage: mini [OPTIONS]

Options:
  -p, --profile TEXT      Profile name
  -w, --workdir PATH      Working directory
      --headless          JSON Lines output
  -t, --task TEXT         Task description
  ...
```

### Check Version

```bash
uv run python -c "import minisweagent; print(minisweagent.__version__)"
# Output: 2.0.0
```

### Quick Test (No API Key Required)

```bash
# This will fail at API call, but confirms installation works
uv run mini --headless --task "Test installation" 2>&1 | head -5
```

---

## Step 5: First Run Initialization

On first run, Mini-SWE-GOCore creates its runtime directory:

```bash
uv run mini --help
```

This creates:
```
.miniswe/
├── configs/
│   ├── live.yaml      # Default profile
│   └── .env           # Environment variables (create manually)
├── data/              # Session data
└── logs/              # Execution logs
```

---

## Configuration

### Option A: Environment Variables (Quick Start)

```powershell
# Windows PowerShell
$env:MINI_API_BASE = "http://127.0.0.1:8045"
$env:ANTHROPIC_API_KEY = "sk-dummy"  # Bypass for proxy

uv run mini --headless --model anthropic/claude-sonnet-4-20250514 --task "Hello world"
```

```bash
# macOS / Linux
export MINI_API_BASE="http://127.0.0.1:8045"
export ANTHROPIC_API_KEY="sk-dummy"

uv run mini --headless --model anthropic/claude-sonnet-4-20250514 --task "Hello world"
```

### Option B: .env File (Persistent)

Create `.miniswe/configs/.env`:

```env
# Proxy Configuration
MINI_API_BASE=http://127.0.0.1:8045
MINI_API_KEY=your-proxy-key

# Provider Keys (use sk-dummy if using proxy)
ANTHROPIC_API_KEY=sk-dummy
OPENAI_API_KEY=sk-dummy
```

### Option C: YAML Profile

Edit `.miniswe/configs/live.yaml`:

```yaml
model:
  model_class: litellm
  model_name: "anthropic/claude-sonnet-4-20250514"
  cost_tracking: "ignore_errors"
  model_kwargs:
    temperature: 0.0
    max_tokens: 4096

agent:
  mode: "yolo"
  step_limit: 50
  cost_limit: 5.0
```

---

## Platform-Specific Notes

### Windows

1. **Use PowerShell**, not Command Prompt
2. **Long paths**: Enable long path support if you encounter path issues:
   ```powershell
   # Run as Administrator
   New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" -Name "LongPathsEnabled" -Value 1 -PropertyType DWORD -Force
   ```

### macOS

1. **Xcode Command Line Tools** may be required:
   ```bash
   xcode-select --install
   ```

### Linux

1. **Build essentials** for some dependencies:
   ```bash
   # Debian/Ubuntu
   sudo apt-get install build-essential python3-dev

   # RHEL/CentOS
   sudo yum groupinstall "Development Tools"
   ```

---

## Troubleshooting

### "uv: command not found"

Restart your terminal or add uv to PATH:
```bash
# Check installation location
which uv || where uv

# Add to PATH if needed
export PATH="$HOME/.cargo/bin:$PATH"
```

### "Python 3.10+ required"

Install Python 3.10 or later:
```bash
# Using uv (recommended)
uv python install 3.11

# Or system package manager
# macOS: brew install python@3.11
# Ubuntu: sudo apt install python3.11
```

### "ModuleNotFoundError: No module named 'minisweagent'"

Ensure you're using `uv run`:
```bash
# Correct
uv run mini --help

# Incorrect (won't find the package)
python -m minisweagent
```

### SSL Certificate Errors

```bash
# Update certificates
pip install --upgrade certifi

# Or set environment variable
export SSL_CERT_FILE=$(python -c "import certifi; print(certifi.where())")
```

---

## Uninstallation

```bash
# Remove virtual environment
rm -rf .venv/

# Remove runtime directory
rm -rf .miniswe/

# Remove the repository
cd ..
rm -rf mini-swe-gocore/
```

---

## Next Steps

1. [Configuration Reference](configuration.md) - Detailed config options
2. [Proxy Integration](proxy_integration.md) - Set up Antigravity Manager
3. [GOrchestrator Relationship](relationship_with_gorchestrator.md) - Use with orchestration
