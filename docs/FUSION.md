# ARIA-X Fusion Notes (ARTEX × Cairn × PentAGI)

This document answers three questions:

1. What **concepts** each of the three upstreams contributed;
2. **Which module, class, or function** in ARIA-X those concepts landed in;
3. What we **deliberately did not copy**, and why.

> **Code provenance**: ARIA-X did not copy any upstream source code. The fusion is a reimplementation of data structures, state machines, prompt contracts, and tool boundaries. Upstream is only a reference for concepts and behavior.
>
> **License**: ARTEX and Cairn are AGPL-3.0. PentAGI is MIT. ARIA-X as a whole therefore uses **AGPL-3.0-or-later**. The full text is in [../LICENSE](../LICENSE). Attribution is in [../NOTICE.md](../NOTICE.md).

Upstream versions that were studied:

| Upstream | Version studied | Notes |
| --- | --- | --- |
| ARTEX | 0.3.15 | The original repository `Autumn-27/ARTEX` was deleted by its author; this repository uses it only as a conceptual reference |
| Cairn | 0.2.1 | `oritera/Cairn`, AGPL-3.0 |
| PentAGI | 2.2.0 | `vxcontrol/pentagi`, MIT |

## 1. Overview

```
        Cairn                         ARTEX                         PentAGI
  ┌──────────────┐            ┌──────────────────┐          ┌──────────────────┐
  │ Fact/Intent/ │            │ exploration graph│          │ specialist team  │
  │ Hint blackboard           │ + asset graph    │          │ provider layer   │
  │ reason→explore            │ planner owns     │          │ sandbox terminal │
  │ OODA workers │            │ intents, todo    │          │ layered memory   │
  └──────┬───────┘            │ chain, traces,   │          └────────┬─────────┘
         │                    │ approval, report │                   │
         └────────────┬───────┴────────┬─────────┴───────────┬───────┘
                      ▼                ▼                     ▼
                         ARIA-X engine / store / tools / llm
```

ARIA-X is not the three projects bolted together. It merges their **invariants**:

- From Cairn: **a decision must be traceable to blackboard state**;
- From ARTEX: **only the planner may generate an intent, and evidence must land on both graphs**;
- From PentAGI: **execution must be isolated, and capability must be provided through tool and provider abstractions**.

## 2. ARTEX → ARIA-X

| ARTEX concept | Where it lands in ARIA-X | Difference and tradeoff |
| --- | --- | --- |
| Exploration graph (tasks, intents, observations) | `models.Node` (kind=`goal/intent/fact/finding/hint/note/guide`) + `edges`; `Store.add_node/list_nodes/add_edge/neighbors` | One node table, distinguished by `kind`; Cairn hints live in the same graph |
| Shared asset graph | `models.Asset` + `store.assets` + `store.anchors`; the `add_asset` / `anchor_node` tools | Assets are separate from exploration nodes, so a report can pivot by asset |
| The planner is the only intent generator | `agents/roles.planner_spec`, `agents/workers.run_planner`, `Orchestrator._reason_step/_create_intents` | The worker toolset has no tool that creates an intent; only the planner runs a `reason` task |
| Concurrent exploration by many workers | `engine/scheduler.py`, `Orchestrator._dispatch`, `budget.max_concurrency` | Concurrency is a budget dimension, not an unbounded thread pool; the default is 2 |
| Cross-wake todo chain | `models.Todo`, `Store.add_todo/update_todo/ready_todos`, the `todo_update` tool, `Orchestrator._apply_planner_todos` | The planner maintains todos; a worker may update only the item assigned to it; a todo with unfinished dependencies is not scheduled |
| Cross-worker execution traces | `models.Trace`, `Store.add_trace/search_traces`, the `search_traces` tool | A trace is a structured record (tool/args/result), not a free-text log; retrieval is provided by `memory/retrieval.py` |
| Approval gate | `Store.request_approval/decide_approval/get_approval/list_approvals`, `Orchestrator._approval_for`, `tools/base._enforce_approval`, `safety.approval` | `ToolRegistry.execute` enforces it for `high` / `critical` tools: `auto` allows, `deny` rejects, `ask` creates an approval and blocks until a human decides; the decision is written to the audit log |
| Run plus report | `engine/engagement.Engagement`, `report/render.write_report`, `runs/<run_id>/report.{md,html}` + `findings.json` | The report is rendered from graph data; the model may not invent it freely; the model only writes the summary text |
| Process-level records (terminal calls and similar) | `tools/terminal.py` writes a `Trace`; `file_*` / `http_request` also record the call | Recording is unified in the tool layer, instead of each agent keeping its own notes |

**Not copied**: ARTEX's Go/norma agent kernel, its PostgreSQL schema, its frontend, and its Docker distribution. ARIA-X reimplements state and scheduling with the Python standard library and replaces the data layer with SQLite, so it stays single-machine, offline-capable, and zero-dependency.

## 3. Cairn → ARIA-X

| Cairn concept | Where it lands in ARIA-X | Difference and tradeoff |
| --- | --- | --- |
| Blackboard | nodes, edges, assets, todos, and traces in the `Store`; every planner and worker call is injected with a `BLACKBOARD SNAPSHOT` | The blackboard is persisted, not an in-memory object; it stays readable across runs |
| Fact / Intent / Hint | `fact/intent/hint` in `models.NODE_KINDS`; the `add_fact` tool; the planner produces intents; a human hint arrives through `POST /api/projects/<id>/hints` or `events.HINT` | A finding is a first-class citizen added by ARIA-X (influence from ARTEX); `_consume_hints` marks a hint as read |
| reason → explore OODA | `Orchestrator._reason_step` (planner) → `_explore` (worker) → `_persist_outcome`; the worker prompt states Observe/Orient/Decide/Act | One worker owns only one intent; it may not wander across intents |
| Workers with no fixed role | `agents/roles.worker_spec` generates the system prompt dynamically; the worker toolset is injected by role | ARIA-X still separates the **capability boundaries** of planner, worker, specialist, and reporter, but a worker is not bound to a fixed "persona" |
| State-space search | The graph, plus budget, idle and change detection, and todo dependencies; `Orchestrator._loop` | Cairn's full library of search operators is not implemented; the model proposes a search and the code constrains it |
| Event-driven | `EventBus` + SSE; stable event names in `events.py` | Events serve the UI and are also written to JSONL for audit and replay |
| Observability | The `graph` command, `/api/projects/<id>/graph`, `trace` events, and the report | Observation is part of the product, not leftover debugging |

**Not copied**: Cairn's container build, its external knowledge-base repository, and its binding to a particular model vendor. ARIA-X's `memory/retrieval.py` is zero-dependency TF-IDF, not Cairn's vector-store design.

## 4. PentAGI → ARIA-X

| PentAGI concept | Where it lands in ARIA-X | Difference and tradeoff |
| --- | --- | --- |
| Specialist team | `agents/roles.resolve_specialist/specialist_spec`, `agents/workers.run_specialist`, the `delegate_specialist` tool, `budget.max_specialist_depth` | Four specialists: researcher / coder / adviser / memorist. A specialist may only answer a narrow question and may not expand scope |
| Provider abstraction | `llm/base.py` (`ChatMessage` / `LLMResponse` / `LLMProvider`), `llm/router.py`, `llm/openai.py`, `llm/anthropic.py` | No official SDK; only `urllib`. `mock` and `scenario` are first-class providers too |
| Sandboxed terminal | `tools/sandbox.py` (`LocalSandbox` / `DockerSandbox` / `MockSandbox`), `terminal_exec`, `sandbox.backend` | The default is `local`. `docker` is a one-shot container. `mock` is for offline tests |
| Tool-call loop | `agents/base.Agent.run`: model → tool → observation → model, until the JSON contract or `max_steps` | Every tool call passes the Registry, scope, command policy, and the approval gate (for high-risk tools). Output is truncated, and calls time out |
| Layered memory | `memory/retrieval.py` (TF-IDF over nodes and traces), the `memory_store` / `memory_search` tools, the `kv` table | Memory is per project and is not shared across projects. There is no external vector database |
| Task orchestration / sub-agents | `TaskScheduler` + `agents/workers` + `delegate_specialist` | Sub-agent depth is capped (`max_specialist_depth`), which prevents unbounded recursion |
| Report | `report/render.py` + `REPORTER_SYSTEM` | Report data comes from the graph; the model may not invent it. The HTML is self-contained |

**Not copied**: PentAGI's Go backend, Graphiti/Neo4j/pgvector, the Kali MCP suite, and its Web frontend framework. Retrieval and memory in ARIA-X are deliberately zero-dependency, so it can run with no network and no database server.

## 5. Capabilities Added by the Fusion

These are not a direct copy of any one upstream. They are ARIA-X behavior that the fusion produced:

1. **One node table carries the semantics of all three**: Cairn's fact, intent, and hint coexist with ARTEX's goal and finding. Relationships are expressed with `kind` and `edges`.
2. **Two graphs plus anchors**: the exploration graph and the asset graph are separate, but `anchors` connect them, so a report can roll up by asset.
3. **Budget as a first-class safety property**: time, LLM calls, tokens, tool calls, total intents, concurrency, and specialist depth are all configurable. When the budget is exhausted the run returns status `budget` and still produces a report.
4. **An offline deterministic demo**: `aria_x.demo.DemoPolicy` + `examples/demo_target.py` turns "planner→worker→finding→report" into a repeatable test, instead of a demo that depends on an API key.
5. **A deterministic scenario provider**: `llm/scenario.py` replays a full engagement from a JSON script, so documentation examples and regression tests do not depend on a model.
6. **A single tool boundary**: scope, the sandbox, command policy, approval, audit, and trace recording all happen inside `ToolRegistry.execute` and the tool implementations. An agent cannot bypass them.

## 6. License Consequences (Must Be Followed)

- ARTEX and Cairn are AGPL-3.0. Their **concepts** are not protected by copyright, but any reuse of **code** would put the corresponding part of ARIA-X under the AGPL. This project chooses AGPL-3.0 for the project as a whole, to avoid a dispute about the boundary.
- MIT code from PentAGI may be included in an AGPL project, but its copyright and license notice must be kept (see [../THIRD_PARTY.md](../THIRD_PARTY.md)).
- If you offer ARIA-X as a network service, AGPL-3.0 §13 requires you to give users a way to obtain the source.
- Upstream names and logos are used only to state attribution. They are not used as trademarks to brand ARIA-X.

## 7. Reproducibility When Upstream Is Unavailable

- The original ARTEX repository was deleted by its author. Public mirrors and archives may no longer be stable. ARIA-X builds with no upstream artifact (zero third-party dependencies). Upstream is used only to check concepts and to state attribution.
- Cairn and PentAGI are still public. Refer to their official repositories directly.
- ARIA-X did not copy upstream source, so this repository does not ship upstream code or binaries.
