# ORACLE — Claude Code Reference

This file is the authoritative guide for working in this repository with Claude Code. Read it before making changes.

---

## What This Project Is

ORACLE is an agentic framework built on the Anthropic Claude API. It is not a chatbot wrapper. It is a coordination layer for multi-agent workflows: planning, execution, memory management, evaluation, and retry logic — all composable and independently replaceable.

The codebase is intentionally layered. Changes to one layer should not bleed into another. Understand the layer boundaries before making changes.

---

## Repository Layout

```
oracle/
├── core/          # The coordination engine — session, planner, executor, evaluator
├── memory/        # Memory tier implementations (working, episodic, semantic)
├── agents/        # Subagent definitions (inheriting BaseAgent)
├── tools/         # Tool implementations and the tool registry
├── cli/           # CLI entry points
├── config/        # Default YAML configs (not secrets)
└── tests/
    ├── unit/      # Pure unit tests, no I/O, no API calls
    └── integration/ # Tests that may hit real or mocked Anthropic API
```

All public interfaces are in `__init__.py` at each package level. If it isn't exported there, treat it as internal.

---

## Core Architecture

### Session Loop

`OracleSession.run(goal)` is the top-level entry point. It drives this loop:

1. **Planner** converts the goal into a `TaskGraph` — a DAG of `Task` objects with typed inputs/outputs and eval criteria.
2. **Executor** runs ready tasks (those whose dependencies are satisfied) in parallel up to `max_parallel_tasks`.
3. **Evaluator** checks each completed task's output against its `EvalCriterion`. Pass → mark done. Fail → retry or trigger replanning.
4. **Memory** is consulted before and updated after every task: working memory for in-context state, episodic for session history, semantic for cross-session retrieval.
5. Loop until all tasks pass or max retries is exceeded.

### Layer Contracts

- `core/` depends on `memory/` and knows about `agents/` and `tools/` only through their registries. No direct imports of specific agents or tools in `core/`.
- `agents/` depend on `core/` (for `BaseAgent`) and `tools/` (via the registry). They must not import from `memory/` directly — memory access goes through `ExecutionContext`.
- `memory/` has no dependencies on `core/`, `agents/`, or `tools/`.
- `tools/` has no dependencies on any other ORACLE package.

Violating these rules creates circular imports and tight coupling. Don't do it.

---

## Models

ORACLE uses different Claude models for different roles. Never hardcode a model string outside `config/defaults.yaml` and `OracleSession.__init__`.

| Role | Default Model | Rationale |
|---|---|---|
| Planner | `claude-opus-4-7` | Complex decomposition needs maximum capability |
| Executor subagents | `claude-sonnet-4-6` | Balanced speed/capability for task work |
| Evaluator | `claude-sonnet-4-6` | Judgment calls need reliability, not just speed |
| Critic/reviewer | `claude-sonnet-4-6` | Same |
| Simple tool calls | `claude-haiku-4-5-20251001` | Cheap, fast, structured output |

When adding a new model role, add it to `config/defaults.yaml` and document it in this table.

### Anthropic SDK Usage

- Always use the `anthropic` Python SDK, not raw HTTP.
- Use `client.messages.create` with `stream=True` for any response that might be long.
- Enable prompt caching (`cache_control: {"type": "ephemeral"}`) on system prompts and large static context blocks. This is not optional — cache misses on long system prompts are expensive.
- Use `max_tokens` conservatively per model tier: Opus tasks get more headroom than Haiku tasks.
- Extended thinking (`thinking: {"type": "enabled", "budget_tokens": N}`) is enabled in the Planner by default. Do not enable it elsewhere without measuring whether it improves output quality.

---

## Memory System

The three memory tiers are not interchangeable. Use the right one:

- **Working memory** (`memory/working.py`): holds the current task's inputs, intermediate state, and tool call results. Scoped to a single task execution. Never persisted.
- **Episodic memory** (`memory/episodic.py`): records what happened during the current session — task completions, agent outputs, errors, decisions. Used by the Planner for replanning. Backed by SQLite (default) or Redis.
- **Semantic memory** (`memory/semantic.py`): embedding-indexed store for facts, documents, and summaries that should persist across sessions. Backed by Chroma (default) or pgvector. Retrieved by similarity search, not exact lookup.

When a task completes, the Executor writes a summary to episodic memory and optionally indexes key outputs in semantic memory. This is done via `ExecutionContext.record_completion()` — do not write to memory stores directly from agent code.

---

## Tool System

Tools are registered with `@tool(name, description)`. The registry auto-generates JSON schemas from Python type hints. All tool parameters must be type-annotated; all tools must have a docstring that describes their behavior.

Tools must be:
- **Deterministic given the same inputs** (or clearly documented as non-deterministic)
- **Side-effect-isolated** — tools that write to disk or make network calls must be clearly named as such
- **Sandboxed by default** — shell tools run in a restricted environment; file tools operate within an allowed path prefix

Never add a tool that can exfiltrate credentials, delete arbitrary files, or make unauthenticated outbound requests.

---

## Agents

Subagents inherit from `BaseAgent` in `agents/base.py`. Key methods:

- `complete(system, user, tools, **kwargs)` — single Claude API call, returns text
- `complete_structured(system, user, schema, **kwargs)` — returns a validated Pydantic model
- `get_memory(query)` — retrieves from semantic memory via `ExecutionContext`
- `emit(key, value)` — writes to working memory

Each agent has a `name` (registry key) and `description` (used by the Executor to decide which agent to dispatch a task to).

When adding a new agent: register it in `agents/__init__.py`, write unit tests for its core methods, and document its specialty and limitations in a docstring on the class.

---

## Testing

```bash
# Unit tests only (fast, no API calls)
pytest tests/unit -v

# Integration tests (requires ANTHROPIC_API_KEY)
pytest tests/integration -v

# Full suite
pytest -v
```

- Unit tests must mock all Anthropic API calls using `unittest.mock` or `pytest-mock`.
- Integration tests may make real API calls but must be tagged `@pytest.mark.integration` and skipped by default in CI unless `RUN_INTEGRATION=1` is set.
- Test file naming: `test_<module>.py` mirrors the module it tests.
- Do not test implementation details — test the public contract of each layer.

---

## Code Style

- Python 3.11+. Use `typing` annotations everywhere — no bare `dict` or `list`.
- Pydantic v2 for all data models. Never use `dataclasses` for models that cross layer boundaries.
- `ruff` for linting and formatting. Run `ruff check . && ruff format .` before committing.
- No `print()` in library code — use the `logging` module. The CLI layer may use `rich` for output.
- Async-first in `core/` and `agents/`. Use `asyncio.gather` for parallel task execution. Sync wrappers live only in the CLI layer.

---

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `ANTHROPIC_API_KEY` | Yes | Anthropic API key |
| `ORACLE_CONFIG` | No | Path to `oracle.yaml` config file |
| `ORACLE_LOG_LEVEL` | No | Log level: `DEBUG`, `INFO`, `WARNING` (default: `INFO`) |
| `ORACLE_MEMORY_DIR` | No | Directory for SQLite/Chroma data (default: `~/.oracle/memory`) |

Never commit secrets. Never read secrets from files in the repo — only from environment variables or a secrets manager.

---

## Adding a New Feature

1. Identify which layer owns the feature. If it spans layers, add a thin interface in `core/` and implementations in the appropriate layer.
2. Write the public interface (type signatures, docstrings) before the implementation.
3. Add unit tests first, then implement until they pass.
4. If the feature touches memory or the session loop, add an integration test.
5. Update this file if the architecture changes.

---

## Common Pitfalls

- **Leaking memory across tasks.** Working memory is task-scoped. If you need data from a prior task, retrieve it from episodic memory via `ExecutionContext`, don't pass it as a mutable global.
- **Hardcoding model names.** Always read from config. Model names change; config keys don't.
- **Skipping eval criteria.** Every task needs one. `llm_judge` with a clear rubric is acceptable when no programmatic criterion exists.
- **Synchronous Anthropic calls in async code.** Use `await client.messages.create(...)` — the SDK is fully async. Blocking in an async context starves the executor's parallel task loop.
- **Over-provisioning Opus.** Use Haiku for structured extraction and simple tool calls. Save Opus for planning and novel reasoning. API costs are real.

---

## Session Management for Claude Code

When working in this repo, Claude Code should:

- Run `ruff check . --fix && ruff format .` after any Python edits.
- Run `pytest tests/unit -v` after any change to `core/`, `memory/`, or `agents/`.
- Never write to `config/defaults.yaml` directly during a session — suggest the change and let the user confirm.
- Treat any file under `tests/` as equal-priority to production code.
- The `cli/` layer is the last thing to change. Get the library right first.
