# AGENTS.md - Agentic Coding Guidelines

This file provides build/lint/test commands and code style guidelines for agents
working in the OpenEnv repository.

## Build & Development Commands

```bash
# Install dependencies
uv sync --all-extras

# Run full test suite (excludes browser/websearch/dipg envs)
PYTHONPATH=src:envs uv run pytest tests/ -v --tb=short

# Run single test file
PYTHONPATH=src:envs uv run pytest tests/envs/test_echo_environment.py -v

# Run single test
PYTHONPATH=src:envs uv run pytest tests/envs/test_echo_environment.py::test_name -v

# Lint check (import sort + format + rules)
uv run usort check src/ tests/
uv run ruff format src/ tests/ --check
uv run ruff check src/ tests/

# Auto-format code
uv run usort format src/ tests/
uv run ruff format src/ tests/

# Run via hooks
bash .claude/hooks/lint.sh          # ruff format check
bash .claude/hooks/test.sh          # pytest (excludes special envs)
bash .claude/hooks/check-debug.sh   # Find print, breakpoint, TODO
```

## Code Style Guidelines

### Imports
- Use `usort` for import sorting in src/ and tests/ only
- For envs/ directory: use `ruff format` only (usort and pyfmt disagree on import ordering in try/except blocks)
- Configuration: `first_party_detection = false` in pyproject.toml
- All non-stdlib imports (including openenv.*) go to third_party bucket
- No blank-line separators between import groups
- Use `TYPE_CHECKING` guard for type-only imports

### Formatting
- Line length: 88 characters (ruff default)
- Use ruff for formatting: `uv run ruff format src/ tests/`
- Lint rules: E, F, W (see pyproject.toml for exceptions)
- Disable E402 (module level import not at top) for pytest.importorskip patterns

### Types
- Use Pydantic models for all wire types (Action, Observation, State)
- Use generics for type safety: `EnvClient[Action, Observation, State]`
- Enable `arbitrary_types_allowed = True` for numpy/torch types
- Use `Field()` for validation constraints

### Naming Conventions
- Classes: PascalCase (e.g., `EchoEnvironment`, `EnvClient`)
- Functions/variables: snake_case (e.g., `step()`, `reset()`)
- Constants: SCREAMING_SNAKE_CASE
- Files: snake_case (e.g., `env_client.py`, `my_environment.py`)

### Error Handling
- Return error info in observations, don't raise exceptions
- Use `done=True` with error observation for fatal errors
- Reserve exceptions for truly exceptional cases (server crashes)

```python
def step(self, action: MyAction) -> MyObservation:
    try:
        result = self._execute(action)
        return MyObservation(result=result, error=None, done=False)
    except InvalidAction as e:
        return MyObservation(result="", error=str(e), done=False)
    except FatalError as e:
        return MyObservation(result="", error=str(e), done=True)
```

### Environment Structure
Follow this canonical pattern:
```
my_env/
├── __init__.py          # Export Action, Observation, Client
├── models.py            # Action, Observation, State (Pydantic)
├── client.py            # EnvClient subclass
├── openenv.yaml         # Environment manifest
├── pyproject.toml       # Dependencies
└── server/
    ├── my_environment.py  # Environment subclass
    ├── app.py             # create_app() with HTTPEnvServer
    ├── requirements.txt   # Docker dependencies
    └── Dockerfile
```
Use `openenv init <name>` to scaffold this structure.

### Client-Server Separation
- Clients must NEVER import from server/ directory
- Use Pydantic models for serialization across wire boundary

### Reward Computation
- Rewards are computed inside the environment
- Domain knowledge encapsulated in environment, not external

## Testing Patterns

### Test Organization
- Unit tests: `tests/` mirroring `src/` structure
- Integration tests: `tests/` with `_integration` suffix
- Environment tests: `tests/envs/`

### Common Patterns
```python
# Fixtures
@pytest.fixture
def echo_env():
    return EchoEnvironment()

# Async tests
@pytest.mark.asyncio
async def test_async_client():
    async with create_client() as client:
        result = await client.step(action)

# Parametrized tests
@pytest.mark.parametrize("input,expected", [("a", "A"), ("b", "B")])
def test_transform(input, expected):
    assert transform(input) == expected
```

## Key Invariants

- **No agent reset**: Simulation controls only exposed to training orchestration
- **Dual API**: WebSocket for infrastructure (Gym-like), MCP for agents
- **Rewards inside env**: Domain knowledge encapsulated in environment
- **Client-server separation**: Clients never import from server/ directory

## Common Development Patterns

### Async Client Usage
```python
import asyncio
from openenv import create_client

async def main():
    async with create_client("my_env") as client:
        obs = await client.reset()
        obs = await client.step(MyAction(...))

asyncio.run(main())
```

### Synchronous Client Usage
```python
from openenv import create_sync_client

with create_sync_client("my_env") as client:
    obs = client.reset()
    obs = client.step(MyAction(...))
```

### Debugging Tests
- Use `-v` flag for verbose output
- Use `--tb=short` for concise tracebacks
- Use `pytest -x` to stop at first failure
- Use `pytest -k "pattern"` to run tests matching pattern
- Exclude slow envs: `--ignore=tests/envs/test_browsergym_environment.py --ignore=tests/envs/test_dipg_environment.py --ignore=tests/envs/test_websearch_environment.py`

### Type Checking
- Use `from __future__ import annotations` for forward references
- Prefer `Field()` over `Field(default=...)` for required fields with constraints
- Use `model_rebuild()` when needed for forward references in Pydantic models