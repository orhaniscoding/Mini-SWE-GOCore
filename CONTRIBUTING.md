# Contributing to Mini-SWE-GOCore

Thank you for your interest in contributing to Mini-SWE-GOCore! This document provides guidelines for development, testing, and submitting contributions.

---

## Development Setup

### Prerequisites

- Python 3.10+
- [uv](https://github.com/astral-sh/uv) package manager
- Git

### Clone and Install

```bash
# Clone the repository
git clone https://github.com/anthropics/mini-swe-gocore.git
cd mini-swe-gocore

# Install with development dependencies
uv sync --extra dev
```

### Verify Setup

```bash
# Run tests
uv run pytest

# Run linter
uv run ruff check .

# Run type checker (if configured)
uv run mypy src/
```

---

## Project Structure

```
mini-swe-gocore/
├── src/minisweagent/          # Main package
│   ├── __init__.py            # Version, protocols, path utilities
│   ├── agents/                # Agent implementations
│   │   ├── default.py         # Base agent class
│   │   ├── headless.py        # JSON Lines agent
│   │   └── interactive.py     # TUI agent
│   ├── models/                # LLM backends
│   │   ├── litellm_model.py   # LiteLLM integration
│   │   └── anthropic.py       # Direct Anthropic
│   ├── environments/          # Execution environments
│   │   ├── local.py           # Local shell execution
│   │   └── docker.py          # Docker execution
│   ├── config/                # Configuration handling
│   └── run/                   # CLI entry points
├── tests/                     # Test suite
├── docs/                      # Documentation
├── pyproject.toml             # Project metadata
└── README.md
```

---

## Coding Standards

### Style Guide

We use [Ruff](https://github.com/astral-sh/ruff) for linting and formatting.

```bash
# Check for issues
uv run ruff check .

# Auto-fix issues
uv run ruff check --fix .

# Format code
uv run ruff format .
```

### Key Rules

1. **Line length**: 120 characters max
2. **Quotes**: Double quotes for strings
3. **Imports**: Sorted with isort (via ruff)
4. **Type hints**: Encouraged but not required

### Code Example

```python
"""Module docstring explaining purpose."""

from pathlib import Path
from typing import Any

from minisweagent import Model, Environment


class MyAgent:
    """Agent that does something specific.

    Attributes:
        model: The language model backend.
        env: The execution environment.
    """

    def __init__(self, model: Model, env: Environment) -> None:
        self.model = model
        self.env = env

    def run(self, task: str) -> tuple[str, str]:
        """Execute the task and return (status, result)."""
        # Implementation here
        return "success", "Task completed"
```

---

## Testing

### Running Tests

```bash
# Run all tests
uv run pytest

# Run with coverage
uv run pytest --cov=minisweagent

# Run specific test file
uv run pytest tests/test_agents.py

# Run specific test
uv run pytest tests/test_agents.py::test_headless_output

# Verbose output
uv run pytest -v

# Skip slow tests
uv run pytest -k "not slow"
```

### Writing Tests

```python
# tests/test_myfeature.py
import pytest
from minisweagent.agents.headless import HeadlessAgent


class TestHeadlessAgent:
    """Tests for HeadlessAgent."""

    def test_json_output_format(self):
        """Verify JSON events are properly formatted."""
        # Setup
        agent = HeadlessAgent(mock_model, mock_env)

        # Execute
        status, result = agent.run("test task")

        # Assert
        assert status == "success"
        assert "result" in result

    @pytest.mark.slow
    def test_long_running_task(self):
        """Test with actual LLM (requires API key)."""
        # This test is marked slow and can be skipped
        pass
```

### Test Categories

| Marker | Description | Command |
|--------|-------------|---------|
| (none) | Unit tests, fast | `pytest` |
| `@pytest.mark.slow` | Integration tests | `pytest -m slow` |
| `@pytest.mark.asyncio` | Async tests | Handled automatically |

---

## Making Changes

### Branching Strategy

```bash
# Create feature branch
git checkout -b feature/my-feature

# Create bugfix branch
git checkout -b fix/issue-123

# Create docs branch
git checkout -b docs/update-readme
```

### Commit Messages

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
type(scope): description

[optional body]

[optional footer]
```

**Types:**
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation
- `refactor`: Code refactoring
- `test`: Adding tests
- `chore`: Maintenance

**Examples:**

```
feat(agents): add streaming output to HeadlessAgent

fix(proxy): handle timeout in MINI_API_BASE connection

docs: update installation instructions for Windows

refactor(models): extract common LiteLLM configuration
```

### Pull Request Process

1. **Fork** the repository
2. **Create** a feature branch
3. **Make** your changes
4. **Test** thoroughly
5. **Lint** with ruff
6. **Commit** with clear messages
7. **Push** to your fork
8. **Open** a Pull Request

### PR Template

```markdown
## Description
Brief description of changes.

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Documentation
- [ ] Refactoring

## Testing
- [ ] Added/updated tests
- [ ] All tests pass
- [ ] Manual testing performed

## Checklist
- [ ] Code follows style guidelines
- [ ] Self-reviewed code
- [ ] Commented complex code
- [ ] Updated documentation
```

---

## Documentation

### Building Docs

Documentation is in Markdown format in the `docs/` directory.

```bash
# Preview docs locally (if using mkdocs)
uv run mkdocs serve
```

### Documentation Style

- Use clear, concise language
- Include code examples
- Use tables for structured data
- Add diagrams for complex concepts

---

## Release Process

Releases are managed by maintainers:

1. Update version in `src/minisweagent/__init__.py`
2. Update CHANGELOG.md
3. Create git tag: `git tag v2.0.1`
4. Push tag: `git push origin v2.0.1`
5. CI builds and publishes

---

## Getting Help

- **Issues**: Open a [GitHub Issue](https://github.com/anthropics/mini-swe-gocore/issues)
- **Discussions**: Use [GitHub Discussions](https://github.com/anthropics/mini-swe-gocore/discussions)
- **Security**: Email security@anthropic.com for vulnerabilities

---

## Code of Conduct

We expect all contributors to:

- Be respectful and inclusive
- Accept constructive criticism
- Focus on what's best for the community
- Show empathy towards others

---

## License

By contributing, you agree that your contributions will be licensed under the MIT License.

---

## Acknowledgments

This project builds on [mini-swe-agent](https://github.com/swe-agent/mini-swe-agent) by Princeton NLP and Stanford NLP. We're grateful for their foundational work.
