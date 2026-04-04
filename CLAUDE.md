# CLAUDE.md — Deep Agents Monorepo

This file provides guidance for AI assistants working in this repository.
See `AGENTS.md` for the authoritative, human-maintained development guidelines.
This file summarises the codebase structure and key conventions.

---

## Repository overview

**Deep Agents** is a batteries-included Python agent harness built on LangGraph.
It ships planning (`write_todos`), filesystem tools, shell execution, sub-agent delegation,
and smart context management out of the box.

- **PyPI package:** `deepagents` (v0.5.0)
- **Default model:** Claude Sonnet 4.6 (`claude-sonnet-4-6`)
- **Docs:** https://docs.langchain.com/oss/python/deepagents/overview
- **License:** MIT

---

## Monorepo layout

```
/
├── libs/
│   ├── deepagents/        # Core SDK (Python 3.11+)
│   ├── cli/               # Textual TUI terminal application (Python 3.11+)
│   ├── acp/               # Agent Context Protocol integration (Python 3.14+)
│   ├── harbor/            # Evaluation / benchmark framework (Python 3.12+)
│   └── partners/          # Optional sandbox integrations
│       ├── daytona/
│       ├── modal/
│       ├── runloop/
│       └── quickjs/
├── examples/              # Reference implementations
├── .github/workflows/     # CI/CD (18 workflows)
├── AGENTS.md              # Authoritative human-maintained dev guidelines ← read this
├── Makefile               # Root-level tasks (lock, lint, format)
└── action.yml             # GitHub Actions action definition
```

Each package under `libs/` has its own `pyproject.toml`, `uv.lock`, and `Makefile`.
All packages are independently versioned.

---

## Tech stack

| Layer | Technology |
|-------|-----------|
| Language | Python 3.11+ (acp: 3.12+) |
| Package manager | `uv` (replaces pip/poetry) |
| Agent framework | LangGraph + LangChain |
| Default LLM | Anthropic Claude (via `langchain-anthropic`) |
| TUI (CLI only) | Textual 8+ |
| Linter/formatter | Ruff |
| Type checker | `ty` |
| Test runner | pytest + pytest-asyncio (auto mode) + pytest-xdist |
| MCP support | `langchain-mcp-adapters` |

---

## Core API

```python
from deepagents import create_deep_agent

agent = create_deep_agent(
    model="anthropic:claude-3-5-sonnet-latest",   # or any LangChain BaseChatModel
    tools=[my_tool],                        # additional tools
    system_prompt="You are a ...",
    middleware=[],                          # AgentMiddleware instances
    subagents=[],                           # SubAgent TypedDicts
    checkpointer=None,                      # LangGraph Checkpointer
    backend=None,                           # BackendProtocol / factory
    debug=False,
)
# Returns CompiledStateGraph — invoke like any LangGraph graph
result = agent.invoke({"messages": [{"role": "user", "content": "..."}]})
```

**Public exports** (`deepagents/__init__.py`):
- `create_deep_agent`
- `SubAgent`, `CompiledSubAgent`, `SubAgentMiddleware`
- `AsyncSubAgent`, `AsyncSubAgentJob`, `AsyncSubAgentMiddleware`
- `FilesystemMiddleware`, `MemoryMiddleware`

**Critical:** Preserve function signatures. Use keyword-only args for new parameters.
Check `__init__.py` before touching any public symbol.

---

## Built-in tools (always injected)

| Tool | Purpose |
|------|---------|
| `write_todos` | Task planning and progress tracking |
| `read_file` | Read with line numbers and truncation |
| `write_file` | Create / overwrite files |
| `edit_file` | String-based in-place editing |
| `ls` | Directory listing |
| `glob` | Pattern-based file search |
| `grep` | Content search |
| `execute` | Shell command (sandbox-dependent) |
| `task` | Delegate to a sub-agent |

---

## Development commands

Run all commands from within the relevant package directory (e.g., `libs/deepagents/`).

```bash
# Tests
make test                  # Unit tests with coverage (pytest -n auto)
make integration_test      # Integration tests (requires network)
make evals                 # LangSmith evaluations
make test_watch            # Watch mode
make update-snapshots      # Refresh smoke test snapshots

# Code quality
make lint                  # ruff check + ruff format --diff + ty type check
make format                # ruff format + ruff check --fix
make type                  # ty static type checker only
make check_imports         # Verify import structure

# Root-level (all packages)
make lock                  # Update all uv.lock files
make lock-check            # Verify all lockfiles are current
make lint                  # Lint all packages
make format                # Format all packages
```

---

## Testing conventions

- Test files live in `libs/<pkg>/tests/unit_tests/` and `tests/integration_tests/`
- File structure must mirror source (`deepagents/graph.py` → `tests/unit_tests/test_graph.py`)
- **Do NOT** add `@pytest.mark.asyncio` — `asyncio_mode = "auto"` is set globally
- Network is blocked in unit tests (pytest-socket); use `tests/integration_tests/` for real I/O
- Avoid mocks — test actual implementation
- Tests must be deterministic (no flaky tests)
- Every feature/bugfix needs unit test coverage
- Smoke tests use snapshot assertions; regenerate with `make update-snapshots`

---

## Code style

### Type hints
All functions must have complete type hints and return types.
Never use `Any` without a specific reason.

### Docstrings (Google-style)
```python
def my_func(arg: str, *, optional: bool = False) -> str:
    """Short description.

    Args:
        arg: Description.
        optional: Description.

    Returns:
        Description.

    Raises:
        ValueError: When arg is invalid.
    """
```
- Types go in signatures, NOT in docstrings
- Use single backticks for inline code — never Sphinx double-backtick ` ``code`` `
- American English spelling

### Ruff linting
- Inline `# noqa: RULE` for one-off suppressions — not `per-file-ignores`
- Reserve `per-file-ignores` for categorical policy (e.g., `"tests/**" = ["D1", "S101"]`)

### Security
- No `eval()`, `exec()`, or `pickle` on user-controlled input
- No bare `except:` — always catch specific exceptions
- Proper resource cleanup (file handles, connections)

---

## Git / PR conventions

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
type(scope): description in lowercase
```

**Types:** `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `ci`, `perf`, `style`

**Scopes:** `sdk`, `cli`, `acp`, `harbor`, `infra`, `deps`, `examples`

Examples:
```
feat(sdk): add memory middleware to create_deep_agent
fix(cli): resolve model initialization in headless mode
chore(deps): bump langchain to 1.5.0
```

PR descriptions must:
- Include a disclaimer noting AI agent involvement
- Explain the "why" of the changes
- Highlight areas requiring careful review

---

## CLI package specifics (`libs/cli/`)

The CLI uses [Textual](https://textual.textualize.io/) for its TUI.

### Key rules
- **Startup performance:** Never import `deepagents`, LangChain, or LangGraph at module level in CLI entry points — defer heavy imports inside functions
- The CLI pins an exact `deepagents==X.Y.Z` version; bump it when SDK changes are needed
- `--help` screen is hand-maintained in `ui.show_help()` — update it alongside argparse
- New user-facing features need a tip added to `_TIPS` in `widgets/welcome.py`
- Slash commands are defined in `SLASH_COMMANDS` in `widgets/autocomplete.py`

### Text rendering in Textual widgets
- Prefer `Content` (`textual.content`) over Rich `Text` for widget rendering
- Use `textual.style.Style` (not Rich's) with `Content`
- Never use f-string interpolation in Rich markup — use `Content.from_markup("...$var..", var=value)` for user-controlled values

---

## Environment variables

```ini
# At least one required
ANTHROPIC_API_KEY=
OPENAI_API_KEY=
GOOGLE_API_KEY=

# Optional: LangSmith tracing
LANGSMITH_API_KEY=
LANGSMITH_PROJECT=
LANGSMITH_TRACING=true

# Test flags
RUN_SANDBOX_TESTS=true    # Enable local subprocess sandbox tests
UV_FROZEN=true            # Enforced automatically by Makefile
```

---

## Package-specific notes

| Package | Python | Notes |
|---------|--------|-------|
| `libs/deepagents` | 3.11+ | Core SDK; public API stability is critical |
| `libs/cli` | 3.11+ | Textual TUI; startup perf sensitive |
| `libs/acp` | **3.14+** | Agent Context Protocol (Zed integration) |
| `libs/harbor` | 3.12+ | Evaluation/benchmark framework |
| `libs/partners/*` | 3.11+ | Sandbox providers (Modal, Daytona, Runloop, QuickJS) |

---

## Additional resources

- Full dev guidelines: `AGENTS.md` (authoritative)
- Documentation source: https://github.com/langchain-ai/docs or `../docs/`
- MCP server config: `.mcp.json`
- Release process: `.github/RELEASING.md`
- Contributing guide: https://docs.langchain.com/oss/python/contributing/overview
