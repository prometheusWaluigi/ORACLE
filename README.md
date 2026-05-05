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

## The ORACLE Loop: VIBE · ECHO · SAGE · LENS · MUSE · ATLAS

The canonical implementation of the ORACLE stack is a six-agent recursive loop designed for situations where a single generalist agent fails: detecting drift, adjudicating policy, routing around guardrail failures, and writing every decision back into memory so the system learns instead of forgets.

```
VIBE → ECHO → SAGE → LENS → MUSE → ATLAS
  ↑                                    │
  └────────────── write-back ──────────┘
```

Each node is an independent agent with its own system prompt, tools, temperature, and output schema. The orchestrator is deliberately thin — it routes and enforces schemas; it does not reason. All reasoning lives inside the six agents.

This is the first reference implementation of ORACLE's Eigenvector Agile principle in production form: each loop iteration deepens the baseline ECHO compares against, which sharpens VIBE's detection, which tightens SAGE's citations, which improves MUSE's plays. It compounds.

### The Six Nodes

| Node | Role | Model tier | Temp | Primary output |
|---|---|---|---|---|
| **VIBE** | Live signal ingestion; detects anomalies | Fast (Haiku-class) | 0.3 | `VibeReport` — pulse, drift hypothesis, confidence |
| **ECHO** | Historical baseline; is this novel or known? | Reasoning (Sonnet-class) | 0.1 | `EchoReport` — verdict, prior matches, z-scores |
| **SAGE** | Policy adjudication; the only node with hard-stop authority | Reasoning | 0.0 | `SageVerdict` — GREEN/YELLOW/RED + mandatory citations |
| **LENS** | Pre-snap read; compresses situation to one sentence + action space | Reasoning | 0.2 | `LensFrame` — read, candidate actions, decision clarity |
| **MUSE** | Play synthesis; 2–3 concrete, reversible routing options | Reasoning | 0.7 | `MusePlays` — plays with diffs, simulation results, tradeoffs |
| **ATLAS** | Execution + write-back; the only node that mutates production state | Reasoning | 0.0 | `AtlasResult` — applied config, rollback handle, telemetry markers |

**Why six, not one?** Drift detection requires holding two reference frames simultaneously (current vs. historical) — single agents collapse them. Policy reasoning requires deterministic, cited output; play generation requires divergent synthesis. These are incompatible sampling regimes in the same model call. And the loop itself is the memory: each huddle transcript becomes an ECHO document, creating a learning system rather than a stateless pattern-matcher.

### The Huddle Object

Every agent reads and mutates a single `Huddle` object. This is the only cross-agent contract — individual prompts and tools may evolve, but `Huddle` is the stable API.

```typescript
interface Huddle {
  huddle_id: string;        // ULID; becomes ECHO index for future loops
  opened_at: string;        // ISO-8601 UTC
  trigger: {
    source: "scheduled" | "alert" | "manual" | "upstream_huddle";
    signal_id?: string;
    notes?: string;
  };

  // Filled in order. Missing = that node hasn't run yet.
  vibe?:  VibeReport;
  echo?:  EchoReport;
  sage?:  SageVerdict;
  lens?:  LensFrame;
  muse?:  MusePlays;
  atlas?: AtlasResult;

  status: "open" | "short_circuited" | "resolved" | "escalated";
  short_circuit_reason?: string;
  closed_at?: string;

  node_latencies_ms: Record<NodeName, number>;
  token_usage:       Record<NodeName, { input: number; output: number }>;
}
```

The Huddle maps directly onto ORACLE's memory tiers: working memory holds the live `Huddle` object during a session; episodic memory stores every closed huddle for ECHO to search; semantic memory indexes patterns across the full archive for long-horizon matching.

### Node Contracts

#### VIBE — Vectorized Intuition for Business Embodiment

VIBE is first to see new data and last to see outcomes. It reports anomalies, not judgments. Overconfidence is explicitly penalized in its system prompt — SAGE filters; VIBE detects.

```typescript
interface VibeReport {
  pulse: "steady" | "rising" | "spiking" | "degraded";
  drift_hypothesis: string | null;     // 1 sentence, or null
  anomalous_dimensions: Array<{
    dimension: string;
    observed: number | string;
    baseline_hint: string;             // ECHO will formalize this
  }>;
  confidence: number;                  // 0.0 – 1.0
  recommend_escalate: boolean;
}
```

Tools: `telemetry.query_window`, `telemetry.classifier_scores`, `telemetry.recent_refusals`

#### ECHO — Enterprise Coherence & Historical Optimization

ECHO runs hybrid vector + BM25 search over prior huddle transcripts to determine whether VIBE's anomaly is structurally novel or a known recurrence. Structural similarity matters more than surface similarity.

```typescript
interface EchoReport {
  verdict: "novel" | "recurrence" | "drift_trajectory" | "noise";
  closest_prior_huddles: Array<{
    huddle_id: string;
    similarity: number;
    one_line_summary: string;
  }>;
  trajectory_note: string | null;
  baseline_delta: Record<string, {
    observed: number;
    baseline_p50: number;
    baseline_p95: number;
    z_score: number;
  }>;
  known_pattern_match: string | null;
}
```

Tools: `echo.search_huddles`, `echo.baseline`, `echo.pattern_library.match`

#### SAGE — Systems Architecture Guidance Engine

SAGE is the middle linebacker. It interprets policy — and only policy. No business impact. No convenience weighting. Every verdict must cite at least one policy document; uncited verdicts are rejected by the orchestrator.

```typescript
interface SageVerdict {
  verdict: "GREEN" | "YELLOW" | "RED";
  rationale: string;             // ≤ 3 sentences
  citations: Array<{
    policy_doc_id: string;
    section: string;
    quoted_rule: string;         // exact text, for audit
    relevance: "direct" | "analogous";
  }>;
  violated_control?: string;     // required if RED
  mitigation_available: boolean;
}
```

Tools: `policy.search`, `policy.fetch`, `classifier.run`

RED verdicts trigger a hard-stop and security escalation. The loop does not continue.

#### LENS — Learning Engine for Networked Systems

LENS compresses the full VIBE + ECHO + SAGE picture to a single sentence (`read`) and enumerates the live action space. If `decision_clarity < 0.5`, the orchestrator pauses for human sign-off before MUSE runs.

```typescript
interface LensFrame {
  read: string;                  // 1 sentence; written for a principal engineer with 8 seconds
  candidate_actions: Array<{
    id: string;
    name: string;
    category: "tighten" | "loosen" | "route" | "rollback" | "escalate";
    one_line_tradeoff: string;
  }>;
  blocked_actions: Array<{ action: string; blocker: string; }>;
  decision_clarity: number;      // 0.0 – 1.0; < 0.5 triggers HITL
  dependencies_at_risk: string[];
}
```

Tools: `graph.dependencies`, `router.current_config`, `blast_radius.estimate`

#### MUSE — Memory-Unified Systems Engineering

MUSE generates 2–3 concrete plays, ordered by recommendation. Plays must be reversible. Irreversible plays require explicit human approval. MUSE prefers routing around drift over suppression — suppression hides signal from the next VIBE cycle.

```typescript
interface MusePlays {
  plays: Array<{
    id: string;
    name: string;
    summary: string;
    config_diff: object;         // actual patch for the Semantic Policy Router
    reversibility: "reversible" | "reversible_with_cost" | "irreversible";
    estimated_blast_radius: string;
    simulation_result?: {
      false_positive_delta: number;
      false_negative_delta: number;
      refusal_rate_delta: number;
    };
    requires_human_approval: boolean;
  }>;
  recommended_play_id: string;
  rationale_for_recommendation: string;
}
```

Tools: `router.config_diff`, `simulator.run`, `prior_plays.search`

#### ATLAS — Architectural Translation, Learning & Alignment Systems

ATLAS is the offensive line. It executes exactly what MUSE produced — no improvisation. If the approved play cannot be executed exactly as specified, ATLAS aborts. Its output becomes the ECHO corpus entry for this huddle; write it for your future self.

```typescript
interface AtlasResult {
  status: "applied" | "aborted" | "partial";
  applied_play_id: string;
  config_changes: object;
  rollback_handle: string;       // ID for router.rollback()
  telemetry_markers: string[];   // event IDs VIBE will track next cycle
  post_conditions: Array<{ check: string; passed: boolean; }>;
  notes: string;
}
```

Tools: `router.apply_config`, `router.rollback`, `telemetry.emit`, `huddle.archive`

### Orchestration

The orchestrator is a state machine, not a reasoner. Its only jobs: run nodes in order, honor short-circuits, validate schemas, and insert human checkpoints.

```python
def run_huddle(trigger) -> Huddle:
    h = Huddle.open(trigger)

    h.vibe = run_agent(VIBE, h)
    if h.vibe.pulse == "steady" and not h.vibe.recommend_escalate:
        return h.resolve("no_signal")

    h.echo = run_agent(ECHO, h)
    if h.echo.verdict == "noise":
        return h.resolve("echo_filtered_noise")

    h.sage = run_agent(SAGE, h)
    require_citations(h.sage)                     # reject if uncited
    if h.sage.verdict == "RED":
        return h.short_circuit("policy_violation", notify=SECURITY_CHANNEL)
    if h.sage.verdict == "YELLOW" and not h.sage.mitigation_available:
        return h.short_circuit("yellow_no_mitigation", notify=ARCH_REVIEW)
    if h.sage.verdict == "GREEN" and not OPTIMIZE_ON_GREEN:
        return h.resolve("green")

    h.lens = run_agent(LENS, h)
    if h.lens.decision_clarity < 0.5:
        await_human_approval(h, checkpoint="lens_low_clarity")

    h.muse = run_agent(MUSE, h)
    chosen = select_play(h.muse, h.lens)
    if chosen.requires_human_approval or chosen.reversibility == "irreversible":
        await_human_approval(h, checkpoint="muse_human_gate", play=chosen)

    h.atlas = run_agent(ATLAS, h, selected_play=chosen)
    if h.atlas.status == "aborted":
        return h.escalate("atlas_aborted")

    echo_corpus.archive(h)       # write-back: next VIBE cycle sees this
    return h.resolve("applied")
```

### Short-Circuit Matrix

| Condition | Where | Action |
|---|---|---|
| VIBE `pulse=steady`, no escalate | After VIBE | Close: `no_signal` |
| ECHO `verdict=noise` | After ECHO | Close: `echo_filtered_noise` |
| SAGE `verdict=RED` | After SAGE | Short-circuit; notify security; open incident |
| SAGE `verdict=YELLOW`, no mitigation | After SAGE | Short-circuit; escalate to arch review |
| SAGE `verdict=GREEN` (default mode) | After SAGE | Close; optional optimization pass |
| LENS `decision_clarity < 0.5` | Before MUSE | **HITL checkpoint** (mandatory) |
| MUSE play is irreversible or flagged | Before ATLAS | **HITL checkpoint** (mandatory) |
| ATLAS `status=aborted` | After ATLAS | Escalate; no automatic retry |

### Human-in-the-Loop Checkpoints

Two checkpoints are mandatory and non-configurable:

- **`lens_low_clarity`** — If LENS returns `decision_clarity < 0.5`, the huddle pauses until a principal engineer approves continuation. Prevents MUSE from confidently generating plays for situations the system doesn't understand.
- **`muse_human_gate`** — Any play marked `irreversible` or `requires_human_approval` must receive explicit sign-off before ATLAS executes. ATLAS will refuse to apply without the signed approval token.

### Observability & KPIs

Every huddle emits OpenTelemetry-compatible events: `huddle.opened`, `node.started`, `node.completed`, `node.schema_violation`, `huddle.short_circuit`, `huddle.hitl_checkpoint`, `huddle.resolved`.

| Node | KPI | Direction | Target |
|---|---|---|---|
| VIBE | Mean time to drift detection | ↓ | < 10 min |
| ECHO | Pattern match precision | ↑ | > 0.85 |
| SAGE | Citation coverage on YELLOW/RED | ↑ | 100% |
| LENS | Mean `decision_clarity` score | ↑ | > 0.7 |
| MUSE | Play reversibility rate | ↑ | > 95% |
| ATLAS | Applied-play false-positive rate | ↓ | < 5% |
| Loop | Wall time, non-HITL hudles | ↓ | < 3 min |
| Loop | Drift recurrence within 7 days | ↓ | < 10% |

### Reference Walkthrough: PHI Drift

A healthcare LLM deployment's refusal rate on category-7 prompts (PHI-adjacent) climbs silently from 3% to 8% over 72 hours. No single prompt is out-of-distribution; per-request classifiers see nothing. This is the canonical quiet drift — the kind that destroys trust before anyone notices.

1. **VIBE fires** (scheduled 6-hour sweep): category-7 refusal rate 8.1% vs. rolling baseline ~3%. `pulse="rising"`, `confidence=0.74`, `recommend_escalate=true`.

2. **ECHO matches**: finds two prior huddles where refusal rate rose following an upstream safety classifier update. `verdict="drift_trajectory"`, `z_score=3.2`. Pattern name: `post-classifier-update category drift`.

3. **SAGE rules**: pulls deployment AUP and healthcare partner BAA. `verdict="YELLOW"`, `mitigation_available=true`. Cites internal AUP §4.3: refusal-rate drift >2σ requires review within 24 hours. No RED — but a governed response is required.

4. **LENS reads**: `read="Upstream classifier is likely over-triggering on category-7 synonyms introduced in last week's ontology update; two mitigations available without loosening policy."` `decision_clarity=0.78`.

5. **MUSE plays**: three plays. Play A — route synonym cluster through secondary classifier (reversible, low blast radius). Play B — tighten primary threshold by 0.05 (reversible with cost). Play C — rollback ontology update (reversible, higher blast radius). MUSE recommends Play A.

6. **ATLAS executes**: Play A is reversible, within auto-execute bounds. Router config diff applied. Telemetry markers emitted. Full huddle archived to ECHO corpus. Resolution: `applied` in 2m 47s.

7. **The loop learns**: Six hours later, VIBE reads the markers ATLAS emitted and sees refusal rate returning to baseline. ECHO's corpus now holds a concrete example of the pattern. The next occurrence matches immediately and the loop accelerates. This is the recursive attractor.

### Implementation Staging

Don't build it all at once. Each week has a clear exit criterion:

| Week | What to build | Exit criterion |
|---|---|---|
| 1 | Huddle schema, orchestrator skeleton, VIBE + ECHO only (stdout output) | Can detect drift |
| 2 | SAGE + policy corpus + citation validation | Adjudicator cites real rules |
| 3 | LENS + MUSE, humans execute plays manually via Slack | Plays are useful to real engineers |
| 4 | ATLAS + router integration; auto-apply reversible plays only | Loop closes end-to-end |
| 5+ | Expand auto-apply envelope; instrument KPIs; weekly human huddle review | Compounding |

### Known Failure Modes

| Failure | Mitigation |
|---|---|
| Agent output doesn't match schema | Strict validation + one retry with the error appended to system prompt; never silently coerce |
| SAGE cites a policy section that doesn't exist | Post-validate every `(doc_id, section)` pair against the policy store before accepting the verdict; invalid citation = retry |
| ECHO develops recency bias from its own write-backs | Time-weighted decay on similarity scoring |
| MUSE always prefers its own plays over retrieved prior ones | If a prior play matches with similarity > 0.9, MUSE must include it and argue against it explicitly if preferring a new play |
| ATLAS "improves" a MUSE play before applying it | Input hash check: if the play passed to ATLAS doesn't match what MUSE produced, abort |

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
