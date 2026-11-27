# ARIA-X Architecture

This document describes modules, data flows, and interfaces that are **already implemented**. It is an engineering document, not a roadmap; anything not yet implemented is collected in [ROADMAP.md](ROADMAP.md).

- Runtime: CPython ≥ 3.11 (development environment 3.12), standard library only.
- Entry points: `python -m aria_x` (CLI), `aria_x.server:serve` (HTTP), `aria_x.engine:Engagement` (library).
- State: SQLite (WAL) + an append-only JSONL event stream + report files in the run directory.

## 1. Layers

```
┌──────────────────────────────────────────────────────────────────┐
│ CLI (cli.py)               Web console (web/) + HTTP API (server/)│
├──────────────────────────────────────────────────────────────────┤
│ Engagement (engine/engagement.py)                                 │
│   └─ Orchestrator (engine/orchestrator.py)                        │
│        ├─ TaskScheduler (engine/scheduler.py)                     │
│        ├─ Budget (engine/budget.py)                               │
│        ├─ agents: planner / bootstrap / workers / specialists /   │
│        │          reporter (agents/)                              │
│        ├─ ToolRegistry + 27 tools (tools/)                        │
│        ├─ ModelRouter + providers (llm/)                          │
│        └─ report renderer (report/)                               │
├──────────────────────────────────────────────────────────────────┤
│ Store (store/sqlite.py)   EventBus (events.py)   Memory (memory/) │
└──────────────────────────────────────────────────────────────────┘
```

### Module Responsibilities

| Path | Responsibility |
| --- | --- |
| `src/aria_x/__init__.py` | Version, license identifier, tagline |
| `src/aria_x/cli.py` | Argument parsing, config assembly, `init/demo/run/serve/doctor/tools/projects/graph/report/export` |
| `src/aria_x/config.py` | Dataclass config tree, TOML/JSON loading, environment-variable overrides, validation, authorization gate |
| `src/aria_x/models.py` | Project / Run / Node / Edge / Asset / Todo / Trace / Task / Evidence / Message / AuditEntry / Usage |
| `src/aria_x/store/` | SQLite schema, migrations, CRUD, graph queries, approvals, audit, export |
| `src/aria_x/events.py` | Thread-safe EventBus, SSE replay buffer, JSONL sink, event-type constants |
| `src/aria_x/llm/` | `ChatMessage` / `LLMResponse` / provider protocol, OpenAI-compatible, Anthropic, Mock, Scenario, JSON-contract validation, ModelRouter |
| `src/aria_x/tools/` | ToolSpec/Result/Registry/Context, Scope, Sandbox, terminal/file/net/knowledge tools, PTY session management |
| `src/aria_x/agents/` | Agent runtime, role prompts, planner/worker/bootstrap/specialist/reporter specifications |
| `src/aria_x/engine/` | Budget, Scheduler, Orchestrator, Engagement |
| `src/aria_x/memory/retrieval.py` | Zero-dependency TF-IDF index and retrieval (nodes and traces) |
| `src/aria_x/report/render.py` | Markdown / self-contained HTML / findings JSON |
| `src/aria_x/server/` | `ThreadingHTTPServer`, routing, SSE, static assets, optional Bearer authentication |
| `src/aria_x/web/` | Console frontend with no framework and no CDN |
| `src/aria_x/demo.py` | `DemoPolicy` for the offline demo, and `DemoTarget` lifecycle management |
| `examples/demo_target.py` | A separately runnable, deliberately vulnerable local target |

## 2. Lifecycle of One Engagement

`Engagement.run(project_id)` (blocking) and `start_background()` (a thread) follow the same path:

1. **Authorization gate**: `AriaXConfig.require_authorization()` requires `scope.authorized=true` and a non-empty `scope.targets`. A project scope that explicitly sets `authorized=false` is also rejected.
2. **Create the Run row** (status=`running`) and write a budget snapshot.
3. **`Orchestrator._prepare`**: Initialize `Budget`, `ModelRouter`, `Scope`, `Sandbox`, `ToolRegistry`, `TaskScheduler`, and the run workdir, and ensure the single `goal` node exists.
4. **bootstrap (optional)**: One best-effort attempt (when `engine.bootstrap=true`), producing facts and findings. If it proves the goal directly, the run may finish immediately.
5. **Main loop `_loop`**:
   - `_reason_step`: the planner reads the blackboard snapshot and returns `{action: intents|complete|noop}`;
   - `_create_intents`: accept only bounded, non-duplicate intents, and sync the planner's todo chain;
   - `_dispatch`: hand runnable intents to `TaskScheduler`. Concurrency is limited by `budget.max_concurrency` and by todo dependencies;
   - `_explore`: one worker per intent. `Agent.run` makes tool calls until it returns the JSON contract;
   - `_persist_outcome`: write facts, findings, assets, evidence, and traces, and update intent and todo status;
   - budget, wall time, idle rounds, and graph-change detection decide whether to continue or finish.
6. **`_finalize`**: Update the Run (status / usage / error), render the report, emit `run.finished`, and write the budget snapshot onto the Run.

Run statuses: `running`, `completed`, `budget`, `cancelled`, `failed`. `budget` means the budget was exhausted but the graph and the report are still consistent. `failed` means an unexpected error; the error is recorded on the event stream and on the run row.

## 3. Data Model

SQLite tables (`src/aria_x/store/schema.py`):

| Table | Contents |
| --- | --- |
| `projects` | Project, goal, scope/config JSON, status |
| `runs` | One run: status, budget/usage JSON, error, start and end time |
| `nodes` | Blackboard nodes: kind ∈ goal/intent/fact/finding/hint/note/guide; status, parent, meta JSON, seq |
| `edges` | Directed relations: spawns / yields / derived_from / proves / supports / depends_on / related / contradicts |
| `assets` | Shared asset graph: root_domain/subdomain/host/ip/service/endpoint/app/credential/artifact, parent_id |
| `anchors` | Many-to-many anchors between nodes and assets |
| `todos` | Cross-wake plan chain: position, status, depends_on JSON, intent_id |
| `tasks` | Scheduler records: bootstrap/reason/explore/specialist/report/consolidate, attempts, error, meta |
| `traces` | Execution traces: agent, seq, kind, tool_name/args/result, content |
| `evidence` | Evidence for a finding: kind, path, sha256, excerpt, meta |
| `messages` | Human/machine messages (a hint also writes a message) |
| `audit` | Audit: actor, action, decision, target, detail |
| `approvals` | Approval requests and decisions |
| `kv` | Key-value store partitioned by scope (memory and metadata) |
| `meta` | Schema version and similar metadata |

What the two graphs mean:

- The **exploration graph** is `nodes` + `edges`. It records how we reasoned: goal → intent → fact/finding.
- The **asset graph** is `assets` + `anchors`. It records what the target actually has: domain → host → service → endpoint or artifact.
- `anchors` connect the two sides, so a report can pivot findings by asset instead of treating reasoning state as facts about the target.

## 4. Model Layer and Providers

`ModelRouter.chat(role, messages, tools=None, ...)` is the only entry point for model calls:

- Roles: `planner` / `worker` / `specialist` / `reporter` (plus bootstrap inference inside the provider);
- Before each call, `budget.check_llm()`; after the call, `budget.record_llm()`, and an `llm.call` event is emitted;
- `router.json(...)` parses and validates the JSON contract, and retries at most `max_retries` times on failure;
- `llm.role_models` can assign a different model to each role.

Built-in providers:

| provider | Behavior |
| --- | --- |
| `openai` | OpenAI-compatible `/chat/completions` (DeepSeek, vLLM, Ollama, and similar); native tool calling |
| `anthropic` | Anthropic Messages API; tool-use conversion |
| `ollama` | Reuses the OpenAI-compatible adapter; default `http://127.0.0.1:11434/v1` |
| `mock` | Replays a list or a policy function; for tests |
| `scenario` | Deterministic replay from a JSON script or per-role queues; for the offline demo and regression tests |

### `scenario` Semantics (Implemented)

- The role is inferred from the rendered task header `# TASK (reason|explore|bootstrap|report)`. If that cannot be decided, it falls back to the system prompt (`PLANNER of ARIA-X`, `BOOTSTRAP agent`, `EXPLORER (worker)`, `REPORTING agent`, `<name> specialist`).
- The `roles.<role>` queue is consumed in order. `"loop": true` cycles. With the default `false`, once the queue is exhausted it moves to `responses`, then to `default`.
- `responses[].match` is a case-insensitive substring of the **last user message**. One hit consumes the entry (`loop=true` is the exception). An entry with no `match` is the wildcard fallback.
- An entry may be a string, `{"content": ...}`, `{"json": {...}}`, `{"tool_calls": [{"name","arguments"}]}`, or a bare list of tool-call dicts.
- `llm.scenario` is a path relative to the CWD, and it also accepts inline JSON. A missing file raises `LLMError`.
- Parallel runs consume the same queue concurrently. **When you need determinism, set `max_concurrency` to 1**.

Secrets for a real provider are read only from configuration or environment variables. `AriaXConfig.public_dict()` uses an explicit public-field allowlist, masks the API key and the server token, and strips custom request headers, sandbox environment variables, and credentials embedded in URLs (`save()` / `export_public()` write only a redacted snapshot that cannot be loaded; reloadable local persistence must call `save_private()` explicitly and is written with mode `0o600`). Every event and report passes through `util.redact` before it is written.

## 5. Tool Execution Boundary

```
Agent loop
  → ModelRouter.chat(tools=[...])
  → LLMResponse.tool_calls
  → ToolRegistry.execute(name, args, ctx)
       ├─ ctx.check_cancel()
       ├─ argument schema validation / light coercion
       ├─ _enforce_approval (high/critical + safety.approval)
       ├─ tool.run(ctx, **args)
       │    ├─ Scope.require_* (network tools)
       │    ├─ Sandbox.run (terminal tools)
       │    └─ Store writes + events
       └─ exception → ToolResult.failure (scope/approval/aria_x/unknown)
  → tool result is returned to the model as a role="tool" message
```

- `ToolContext` carries the project, run and task ids, agent, role, store, config, events, scope, sandbox, workdir, budget, delegate, cancel, and request_approval.
- File tools are jailed to the engagement workdir. `file_read` rejects parent-directory traversal and symlink escape.
- Terminal tools pass `CommandPolicy` first (a blacklist of destructive commands), then enter the sandbox. The `local` backend also requires `cwd` to be inside the workdir.
- A scope violation does not kill the engagement. It becomes a tool failure plus an audit record, so the model can see it and correct itself.
- **Approval gate (enforced)**: After argument validation and before the tool runs, `ToolRegistry.execute` calls `_enforce_approval`. `safety.approval="auto"` allows the call. `"deny"` immediately rejects `high` and `critical` tools. `"ask"` uses `ToolContext.request_approval` to create an approval record, emit `approval.requested`, and **block waiting** for a human decision (at most 300 seconds; timeout or cancellation counts as denial; Web console or API). Approval continues the call; denial raises `ApprovalDenied`. The decision is written to the audit table. `port_scan`, `file_write`, and `terminal_exec` are `high` risk, so this gate controls them in `ask` and `deny` mode. Approval does not widen scope: scope is always enforced on its own.

## 6. Events and the Event Stream

Event types (`src/aria_x/events.py`) include `run.started/finished/failed/paused/resumed`, `task.started/finished/failed`, `node.created/updated`, `asset.found`, `finding.created`, `todo.updated`, `trace`, `log`, `llm.call`, `tool.call`, `approval.requested/decided`, `budget`, `report.ready`, `hint`, and `error`.

- `EventBus.emit()` is thread-safe. It keeps a bounded replay buffer (default 200, configurable) and appends a JSON line to `data/events.jsonl` (a write failure does not affect the run).
- Every payload goes through recursive secret redaction.
- SSE: `GET /api/projects/<id>/events?replay=N` first replays the most recent N events (default 100, cap 500), then pushes live events, and sends a heartbeat when idle. The stream thread **does not access SQLite**.

## 7. HTTP API

`aria_x.server.create_server(config, engagement, bus, store)` returns a `ThreadingHTTPServer` that is bound but not started. `serve(config, ...)` starts it. The default is `127.0.0.1:8787`. If `server.auth_token` (or `ARIA_X_AUTH_TOKEN`) is set, every API except `/api/health` (including the SSE event stream) requires an `Authorization: Bearer <token>` header or an `X-ARIA-X-Token` header. The console subscribes to SSE with a streaming `fetch` that sends the auth header, and no longer passes a long-lived token on the `?token=` query string.

| Method | Path | Description |
| --- | --- | --- |
| GET | `/` | Console HTML |
| GET | `/static/*` | Static assets (JS/CSS) |
| GET | `/api/health` | `{ok, version, time}` |
| GET | `/api/config` | Public config + tool catalog + scope summary |
| GET/POST | `/api/projects` | List / create projects |
| GET | `/api/projects/<id>` | Project + counts + runs + findings + assets + todos |
| GET | `/api/projects/<id>/graph` | Graph overview |
| GET | `/api/projects/<id>/traces` | Traces (`task_id`, `limit`) |
| GET | `/api/projects/<id>/audit` | Audit |
| GET | `/api/projects/<id>/report` | Most recent report, as Markdown |
| POST | `/api/projects/<id>/runs` | Background run `{goal?}` → `{run_id}` |
| POST | `/api/projects/<id>/hints` | Inject a human hint (also writes a message) |
| GET | `/api/projects/<id>/events` | SSE |
| GET | `/api/runs/<id>` | Run status + budget + tasks |
| POST | `/api/runs/<id>/stop` | Request a stop |
| GET | `/api/approvals?project_id=` | Pending approvals |
| POST | `/api/approvals/<id>` | `{approved, reason?}` |
| GET | `/api/report/<run_id>` | HTML report |
| GET | `/api/export/<project_id>` | Full JSON export |

## 8. Storage and Concurrency

- Each `Store` holds one `sqlite3.Connection` (`check_same_thread=False`). Transactions use an explicit `BEGIN/COMMIT`. WAL allows concurrent reads and serializes writes.
- `Engagement` is the thread-safety boundary. `run()` executes on the calling thread. `start_background()` uses one daemon thread per run. `close()` sets the cancel flag first, then joins (cap 10s), then closes the store and event bus it owns.
- `Budget` is not a durable scheduler. After a process crash it does not resume on its own. State and reports remain readable, and the next run creates a new Run.

## 9. Reports

`report/render.write_report(run_dir, export, meta)` produces:

- `report.md`: executive summary, scope and method, findings ordered by severity, attack path, coverage and blind spots;
- `report.html`: a self-contained dark theme, with no external CDN;
- `findings.json`: a structured export of the findings.

`export` comes from `Store.export_project(project_id)` and includes the project, nodes, edges, assets, todos, findings, traces, evidence (grouped by finding id), audit, and runs. A report failure does not fail the run: `Orchestrator._render_report` falls back to the built-in deterministic renderer.

## 10. Key Design Tradeoffs

| Tradeoff | Reason |
| --- | --- |
| One process plus SQLite, not a message queue | Simple to deploy for one person or a small team; state can be copied and audited; distribution is left for later |
| Only the planner generates intents | Stops each worker from expanding scope on its own, and makes "why this step" reviewable afterward |
| Two graphs, not one large table | Reasoning state and target facts mean different things; mixing them pollutes reports and retrieval |
| Enforce scope in the tool layer, not in the prompt | A prompt can be bypassed; the boundary is a code check before network I/O |
| The provider abstraction is only the minimum protocol | No SDK, and it can be tested offline; a new provider only has to implement `chat()` |
| The event stream is separate from SQLite | SSE is a long-lived connection; a slow client must not hold the database lock |
| Budget exhaustion returns `budget` instead of crashing | Evidence already collected must still be able to produce a report |
