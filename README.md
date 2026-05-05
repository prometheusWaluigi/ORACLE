# ORACLE

**Orchestrated Reasoning and Cognitive Learning Engine**

A modular, multi-agent framework built on top of the Anthropic Claude API. ORACLE provides the scaffolding for composing autonomous agents that plan, remember, act, evaluate, and iterate — without requiring you to rebuild that infrastructure from scratch every time.

---

## What It Is

ORACLE is an agentic stack, not a single agent. It treats intelligence as a pipeline of cooperating subsystems:

- **Planner** — breaks goals into ordered, dependency-aware task graphs
- **Executor** — dispatches tasks to specialized subagents or tools
- **Memory** — maintains short-term working context and long-term episodic/semantic stores
- **Evaluator** — scores outputs against goals, triggers retries or re-planning
- **Oracle** — the top-level coordinator that routes between all of the above

Each layer is independently swappable. You can use ORACLE's built-in planner or bring your own. You can use its vector memory layer or swap in a database you already have.

---

## Why ORACLE

Most agent frameworks solve the LLM-call problem. ORACLE solves the *coordination* problem: how do you get a cluster of models to pursue a goal coherently across many turns, tools, and failures?

Key design decisions:

1. **Typed task graphs, not chat loops.** Each task has explicit inputs, outputs, dependencies, and a success criterion. The evaluator checks the criterion; the planner decides what to do next.
2. **Memory is first-class.** Context windows fill up. ORACLE manages what stays in-context, what gets compressed to episodic memory, and what gets retrieved via semantic search.
3. **Failure is a first-class state.** Retries, fallbacks, and re-planning are baked in at the framework level, not left to each agent to implement ad hoc.
4. **Anthropic-native.** Built specifically around Claude's strengths: extended thinking, tool use, prompt caching, and the multi-turn conversation format.

---

## Design Philosophy: Eigenvector Agile

Most agent frameworks are built for derivative thinking. Try something, measure it, pivot to whatever worked. That's the right model when you're a startup groping in the dark for product-market fit — local gradient descent on an unknown landscape.

ORACLE is built for a different problem: you already know what you're trying to do. You're not searching for the hill. You're trying to get taller.

In linear algebra, an **eigenvector** of a matrix is a direction that doesn't change when the transformation is applied — it only scales. Applied to organizational or system design: your eigenvector is the thing you do that gets *stronger* every time you repeat it. Apple deepens design intuition. Amazon deepens logistical efficiency. Derivative optimization finds new hills; eigenvector alignment makes your existing hill a mountain.

ORACLE encodes this distinction at the architectural level.

### Derivative Agile vs. Eigenvector Agile

| | Derivative Agile | Eigenvector Agile |
|---|---|---|
| Model | `∇f(x)` — follow the gradient | `Ax = λx` — amplify your direction |
| Mode | Sprint → Pivot → Sprint | Sprint → Deepen → Sprint → Compound |
| Suits | Startups finding product-market fit | Systems with a known identity to strengthen |
| Memory | Forget fast, stay nimble | Remember everything, compound forever |
| Failure mode | Never converges on identity | Ossifies if eigenvector is mis-identified |

ORACLE is an eigenvector system. Its core direction — **typed coordination + recursive evaluation + compound memory** — is the thing that scales with repetition. Each design decision either deepens that direction or it doesn't belong in the codebase.

### How This Shows Up in the Code

**Compound Memory, Not Clean Slates**

The three-tier memory system (working → episodic → semantic) is a direct implementation of recursive organizational intelligence. Working memory is task-scoped and cheap. Episodic memory captures what happened in this session so the Planner can replan from a position of knowledge, not amnesia. Semantic memory indexes learnings across sessions so the system gets smarter every time it runs. Each tier compounds what the previous tier couldn't afford to hold.

**Deepening, Not Pivoting**

The Planner doesn't just decompose goals — it replans from episodic context when tasks fail. A traditional retry loop forgets why it failed and tries again. ORACLE's replanning reads the episodic record, identifies what specifically went wrong, and constructs a graph that routes around the failure. This is `Sprint → Deepen`, not `Sprint → Pivot`.

**Fractal Execution**

The same structured pattern — goal → task graph → evaluation criterion — applies whether you're running a 3-task workflow or a 300-task research pipeline. The eigenvector (typed coordination) is visible at every scale: in a single agent's `complete_structured()` call, in the Executor's parallel dispatch, in the Planner's DAG construction. If a contribution breaks that self-similarity, it's fighting the framework's identity.

**Evaluation as Memory Formation**

The Evaluator doesn't just pass or fail tasks — when a task passes, `ExecutionContext.record_completion()` writes a structured summary to episodic memory and optionally indexes it in semantic memory. The act of evaluation is the act of remembering. Criteria that can't be checked programmatically use `llm_judge` with an explicit rubric, which itself becomes part of the episodic record. Nothing is thrown away.

### What This Means for Contributors

When you're adding a feature or agent, the right question isn't "is this useful?" It's "does this deepen ORACLE's core direction?" A new memory backend that improves recall quality: yes. A one-off integration that works around the task graph because it's faster to ship: no. The framework gets stronger by going deeper, not wider.

> Find your eigenvector and sprint toward it, not away from it.

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│                    ORACLE Core                  │
│                                                 │
│  ┌──────────┐   ┌──────────┐   ┌────────────┐  │
│  │ Planner  │──▶│ Executor │──▶│ Evaluator  │  │
│  └──────────┘   └──────────┘   └────────────┘  │
│       ▲               │               │        │
│       └───────────────┴───────────────┘        │
│                       │                        │
│              ┌─────────────────┐               │
│              │  Memory Layer   │               │
│              │  ┌───────────┐  │               │
│              │  │ Working   │  │               │
│              │  │ Episodic  │  │               │
│              │  │ Semantic  │  │               │
│              │  └───────────┘  │               │
│              └─────────────────┘               │
└─────────────────────────────────────────────────┘
         │
         ▼
   ┌───────────┐     ┌───────────┐     ┌──────────┐
   │ Subagent  │     │ Subagent  │     │  Tools   │
   │ (Analyst) │     │ (Coder)   │     │ (Shell,  │
   └───────────┘     └───────────┘     │  API...) │
                                       └──────────┘
```

### Layers

| Layer | Responsibility | Key Abstraction |
|---|---|---|
| Oracle | Goal intake, session management, top-level routing | `OracleSession` |
| Planner | Task decomposition, dependency graph, replanning | `TaskGraph` |
| Executor | Subagent dispatch, tool calls, parallelism | `ExecutionContext` |
| Evaluator | Output scoring, success/fail determination | `EvalCriterion` |
| Memory | Context window management, storage backends | `MemoryStore` |

---

## Quick Start

```bash
# Clone and install
git clone https://github.com/prometheuswaluigi/oracle
cd oracle
pip install -e ".[dev]"

# Set your API key
export ANTHROPIC_API_KEY=sk-ant-...

# Run a goal
oracle run "Research the top 5 open-source vector databases and produce a comparison table"
```

Or programmatically:

```python
from oracle import OracleSession

session = OracleSession(model="claude-opus-4-7")
result = session.run(
    goal="Analyze the sentiment of every review in reviews.csv and summarize by product",
    tools=["file_read", "python_exec"],
)
print(result.summary)
```

---

## Core Concepts

### Goals vs. Tasks

A **goal** is what the user wants. A **task** is a unit of work the planner creates to achieve a goal. Goals are high-level and may be ambiguous. Tasks are specific, bounded, and have typed outputs.

```python
# Goal (user-facing)
"Build a REST API for user authentication"

# Tasks (planner-generated)
[
  Task(id="t1", name="Design data model", output_type="schema"),
  Task(id="t2", name="Implement endpoints", depends_on=["t1"], output_type="code"),
  Task(id="t3", name="Write tests", depends_on=["t2"], output_type="test_suite"),
  Task(id="t4", name="Document API", depends_on=["t2"], output_type="docs"),
]
```

### Memory Tiers

| Tier | Scope | Backend | Retrieval |
|---|---|---|---|
| Working | Current task | In-process dict | Direct key access |
| Episodic | Current session | SQLite / Redis | Recency-ranked |
| Semantic | Cross-session | pgvector / Chroma | Embedding similarity |

### Evaluation Criteria

Each task specifies how its output is checked before the session moves on:

```python
Task(
    name="Write unit tests",
    eval_criterion=EvalCriterion(
        type="code_passes_linter",
        threshold=1.0,
    )
)
```

Built-in criterion types: `llm_judge`, `code_passes_linter`, `contains_keywords`, `schema_valid`, `human_approval`.

---

## Project Layout

```
oracle/
├── core/
│   ├── session.py         # OracleSession — top-level coordinator
│   ├── planner.py         # TaskGraph construction and replanning
│   ├── executor.py        # Task dispatch, parallelism, tool calls
│   └── evaluator.py       # EvalCriterion implementations
├── memory/
│   ├── working.py         # In-context working memory
│   ├── episodic.py        # Session-scoped episodic store
│   └── semantic.py        # Cross-session vector store
├── agents/
│   ├── base.py            # BaseAgent — inheritable subagent pattern
│   ├── analyst.py         # Research and synthesis specialist
│   ├── coder.py           # Code generation and execution specialist
│   └── critic.py          # Output review and evaluation specialist
├── tools/
│   ├── registry.py        # Tool registration and schema generation
│   ├── shell.py           # Sandboxed shell execution
│   ├── file_io.py         # Filesystem read/write
│   └── web.py             # Web fetch and search
├── cli/
│   └── main.py            # `oracle` CLI entry point
├── config/
│   └── defaults.yaml      # Default model, memory, and eval settings
└── tests/
    ├── unit/
    └── integration/
```

---

## Configuration

`oracle.yaml` at the project root (or `~/.config/oracle/oracle.yaml` for global defaults):

```yaml
model:
  planner: claude-opus-4-7
  executor: claude-sonnet-4-6
  evaluator: claude-haiku-4-5-20251001

memory:
  working_limit_tokens: 8000
  episodic_backend: sqlite
  semantic_backend: chroma
  semantic_top_k: 5

execution:
  max_parallel_tasks: 4
  max_retries: 3
  retry_backoff: exponential

evaluation:
  default_criterion: llm_judge
  llm_judge_model: claude-sonnet-4-6
  pass_threshold: 0.85
```

---

## Extending ORACLE

### Custom Subagent

```python
from oracle.agents.base import BaseAgent
from oracle.core.session import task

class DataEngineerAgent(BaseAgent):
    name = "data_engineer"
    description = "Specializes in data pipelines, ETL, and SQL"

    @task(output_type="pipeline_spec")
    def design_pipeline(self, goal: str, schema: dict) -> dict:
        return self.complete(
            system="You are a senior data engineer...",
            user=f"Design a pipeline for: {goal}\nSchema: {schema}",
        )
```

### Custom Tool

```python
from oracle.tools.registry import tool

@tool(name="query_db", description="Run a read-only SQL query")
def query_db(query: str, connection_string: str) -> list[dict]:
    # implementation
    ...
```

### Custom Eval Criterion

```python
from oracle.core.evaluator import EvalCriterion, register_criterion

@register_criterion("json_valid")
def json_valid(output: str, **kwargs) -> float:
    try:
        json.loads(output)
        return 1.0
    except json.JSONDecodeError:
        return 0.0
```

---

## Roadmap

- [ ] Core session loop with planner, executor, evaluator wiring
- [ ] Working and episodic memory backends
- [ ] Semantic memory with Chroma and pgvector adapters
- [ ] Built-in subagents: analyst, coder, critic
- [ ] CLI: `oracle run`, `oracle session`, `oracle replay`
- [ ] Tool registry with schema auto-generation from type hints
- [ ] Streaming output and partial results
- [ ] Human-in-the-loop pause/resume
- [ ] Tracing and observability (OpenTelemetry)
- [ ] Multi-model routing (use cheaper models for simple subtasks)
- [ ] Persistent sessions with replay

---

## Contributing

ORACLE is in early development. The architecture is stable; the implementation is not.

If you're building something on top of ORACLE or contributing to it, read `CLAUDE.md` — it's the canonical reference for how this codebase is structured and how to work in it.

---

## License

MIT
