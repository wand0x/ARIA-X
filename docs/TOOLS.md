# ARIA-X Tool Reference

Tools are the **only** way an agent can change the world. Each tool declares:

- `name` / `description` / JSON Schema parameters;
- `risk`: `low` / `medium` / `high` / `critical`;
- `requires_scope`: whether `Scope` authorization is enforced before I/O;
- `read_only`: whether it is read-only;
- `tags`: a category.

Every call goes through `ToolRegistry.execute(name, args, ctx)`: argument validation → `ctx.check_cancel()` → the tool implementation → exceptions converted into a `ToolResult` (a scope violation, an approval denial, any other AriaXError, and an unexpected exception are handled separately). The result is returned to the model as a `role="tool"` message and written to a trace.

> Risk level drives both the **approval gate** (`high` / `critical` are intercepted under `ask` / `deny`) and policy hints, but it is not a runtime sandbox. Real isolation comes from `sandbox.backend` (Docker) and from `Scope`. No built-in tool currently uses `critical`.

## Summary Table (27)

| Name | Risk | Scope | Required parameters | Purpose |
| --- | --- | --- | --- | --- |
| `terminal_exec` | high | command heuristic | `command` | Run one shell command inside the sandbox |
| `session_open` | high | command heuristic | `command` | Start a persistent interactive session (PTY) |
| `session_send` | high | command heuristic | `session_id` | Send text to the session and collect output |
| `session_read` | medium | — | `session_id` | Read new output from the session |
| `session_close` | low | — | `session_id` | Close the session and reap the process group |
| `session_list` | low | — | — | List the current sessions |
| `file_read` | medium | — | `path` | Read a UTF-8 file inside the workdir |
| `file_write` | high | — | `path`, `content` | Write a file inside the workdir |
| `file_list` | low | — | — | List a directory inside the workdir |
| `http_request` | medium | yes | `url` | One HTTP request |
| `dns_lookup` | low | yes | `host` | An A/AAAA or PTR query |
| `port_scan` | high | yes | `host` | A bounded TCP connect scan |
| `tls_probe` | medium | yes | `host` | TLS handshake and certificate information |
| `web_probe` | medium | yes | `base_url` | Probe paths from a small wordlist |
| `web_search` | low | — | `query` | External research search (not target traffic) |
| `add_fact` | low | — | `title`, `body` | Write a fact to the blackboard |
| `add_finding` | medium | — | `title`, `severity`, `body` | Write a finding, plus evidence, to the blackboard |
| `add_asset` | low | — | `kind`, `value` | Upsert into the asset graph |
| `anchor_node` | low | — | `node_id`, `asset_value` | Anchor a node to an asset |
| `graph_query` | low | — | `view` | Query the graph, findings, assets, intents, todos, or traces |
| `search_traces` | low | — | `query` | Retrieve traces from other workers |
| `memory_store` | low | — | `text` | Write project memory |
| `memory_search` | low | — | `query` | TF-IDF search over project memory |
| `todo_update` | low | — | — | Maintain the todo chain |
| `prove_goal` | medium | — | `summary` | Mark the goal as proved by evidence |
| `delegate_specialist` | low | — | `specialist`, `question` | Delegate a narrow question to a specialist |
| `scope_check` | low | — | `target` | Diagnose a scope decision |

Tools with `requires_scope=yes` call `Scope.require_url` / `require_host` before any network I/O. A violation raises `ScopeViolation`, which the Registry turns into a failure result and an audit record.

## Terminal and Files

### `terminal_exec(command, timeout?, cwd?)`

- Runs `bash -lc <command>` inside the `sandbox`. `cwd` must be a subdirectory of the engagement workdir.
- Passes `CommandPolicy` first. It rejects patterns such as `rm -rf /`, `mkfs`, `dd of=/dev/sd*`, a fork bomb, shutdown or reboot, `curl|bash`, writes to a raw device, and `history -c`.
- Runs a heuristic scope check on literal URLs and IPs in the command. This is not a full parse, and it cannot cover every indirect target.
- Returns `exit_code/stdout/stderr/timed_out/duration_ms`. Output is truncated to `sandbox.max_output`.
- On timeout, the whole process group is killed (local) or `docker kill` is used (docker).
- Records one `Trace` (tool name, arguments, and a result summary).

Example:

```json
{"command": "curl -sS -D - http://127.0.0.1:8000/ -o /dev/null", "timeout": 15}
```

### `file_read(path, max_bytes?)`

- A relative path is resolved against the workdir. Parent-directory traversal, escape via an absolute path, and a symlink that points outside are rejected.
- Only UTF-8 text is accepted (a decode failure returns an error). The implementation truncates at a default cap.
- Suitable for reading evidence files, scan output, and generated payloads.

### `file_write(path, content)`

- Likewise jailed to the workdir. Parent directories are created automatically.
- Risk is high: the model may write scripts, but it **cannot** write outside the workdir.
- The write is recorded as a trace. It is not executed automatically.

### `file_list(path?)`

- Lists directory entries (name, type, size). The default is the workdir.
- Read-only, and low risk.

## Interactive Sessions (PTY)

`terminal_exec` means "run one command and take the result back". It is not enough for programs that need terminal semantics (`msfconsole`, `ssh`, a database REPL, an `nc` reverse shell). The session tools fill that gap.

| Tool | Description |
| --- | --- |
| `session_open(command, name?, cwd?)` | Start a persistent process on a PTY and return `session_id` plus the initial banner |
| `session_send(session_id, text?, newline?, wait_ms?)` | Send text and collect output within `wait_ms`. Omitting `text` means press Enter |
| `session_read(session_id, wait_ms?)` | Read only new output, for polling something that prints asynchronously |
| `session_close(session_id)` | Send `exit`, then SIGTERM, then SIGKILL. Returns the exit code and the final output |
| `session_list()` | List each session's id, name, liveness, idle seconds, and buffered bytes |

Constraints and guarantees:

- **The same guardrails**: both the `session_open` command and the `session_send` text pass `CommandPolicy` and the scope heuristic. A session is not a back door around those checks.
- **Bounded**: at most 4 sessions per run, a 256 KB output buffer per session, and 8 KB returned per read.
- **Reaped automatically**: a session idle for more than 30 minutes is swept on the next session operation. When the run ends (including an exception, cancellation, or an exhausted budget), the `Orchestrator` calls `close_all()` and does not leave an orphan shell.
- **Docker backend**: a session is started with `docker run -i -t`. The container gets a TTY as well, and close falls back to `docker kill`.
- **Approval**: `session_open` and `session_send` are `high` risk. When `safety.approval="ask"`, a human must approve them.

Typical use:

```text
session_open("msfconsole -q")            -> s_ab12cd34ef
session_send(s_ab12cd34ef, "use auxiliary/scanner/http/http_version")
session_send(s_ab12cd34ef, "set RHOSTS 10.0.0.5")
session_send(s_ab12cd34ef, "run", wait_ms=5000)
session_close(s_ab12cd34ef)
```

## Network (Scope Is Enforced)

Every network tool follows the same order: **scope authorization → per-host rate limit → timeout → truncated output → recorded evidence**. `scope.max_rps` is a per-host token bucket. `port_scan` concurrency is capped at 32.

### `http_request(url, method?, headers?, body?, timeout?, max_bytes?)`

- `http` and `https` only.
- Returns `status`, selected response headers (sensitive headers redacted), a body summary, the length, and the elapsed time.
- Saves request and response evidence, which a finding can cite.
- `method` defaults to GET. POST, PUT, PATCH, DELETE, and similar methods are supported.
- It does not follow a redirect onto an arbitrary scheme. The caller decides whether to request the next URL, and that new URL still needs scope.

Example:

```json
{"url": "http://127.0.0.1:8000/files?path=../flag.txt", "method": "GET"}
```

### `dns_lookup(host, record_type?)`

- `A` and `AAAA` go through `getaddrinfo`. `PTR` uses reverse lookup.
- The result includes the resolved addresses. A failed lookup is a normal result, not an exception.

### `port_scan(host, ports?, timeout?, max_ports?, banners?)`

- TCP connect scan. `ports` accepts `"22,80,443"` or `"1-100"`.
- When omitted, `scope.ports` is used (18 common ports by default).
- `max_ports` caps the total (default 200, and never below the number of configured ports).
- Concurrency is at most 32, and `max_rps` is honored. `banners=true` does a lightweight banner read.
- Returns open ports, an optional service guess, and the elapsed time.

### `tls_probe(host, port?, timeout?)`

- Completes the TLS handshake and reads the certificate.
- Returns subject, issuer, and SAN, the validity period, whether it is self-signed, and the TLS version and cipher suite when those are available.
- A failed handshake is a normal result.

### `web_probe(base_url, paths?, timeout?)`

- Sends GET to a few common paths (`/`, `robots.txt`, `sitemap.xml`, `.well-known/security.txt`, and similar) and returns the status code, length, and Content-Type.
- `paths` can replace the path list. Each request still needs scope authorization and the rate limit.
- It is not a directory brute-forcer. The wordlist is small, so a demo does not turn into noise.

### `web_search(query, limit?, timeout?)`

- Queries the DuckDuckGo HTML endpoint and returns title, url, and snippet.
- **Not target traffic**: it is for retrieving external knowledge, and it does not need scope.
- On failure it degrades gracefully (it returns an error result and does not retry in a burst).

## Blackboard and Assets

### `add_fact(title, body, confidence?, intent_id?, asset_values?)`

- Writes a `kind=fact` node with status=`done`.
- If `intent_id` is given, creates a `yields` edge.
- `asset_values` anchors the fact to assets that already exist (if one does not exist, it is upserted first).

### `add_finding(title, severity, body, asset_values?, intent_id?, cvss?, confidence?, evidence?)`

- `severity ∈ critical/high/medium/low/info`.
- Writes a `kind=finding` node (status=`done`) and writes `evidence` (a string or a structured dict) into the `evidence` table. A file path gets a sha256 when the file exists.
- Automatically creates a `proves` edge from the intent to the finding.
- The report and `findings.json` are both read from here. **Do not** record an unconfirmed guess as a finding.

### `add_asset(kind, value, parent_value?)` / `anchor_node(node_id, asset_value, kind?, note?)`

- Asset kinds: root_domain, subdomain, host, ip, service, endpoint, app, credential, artifact.
- `parent_value` builds the asset hierarchy (for example host → service).
- `anchor_node` creates a many-to-many anchor between a node and an asset, so a report can pivot on it.

### `graph_query(view, query?, limit?)`

- `view`: `overview` / `facts` / `findings` / `assets` / `intents` / `todos` / `traces`.
- Optional substring filter and limit. Read-only.
- Lets the model re-read the blackboard in a long session instead of relying on context memory.

## Plans, Traces, and Memory

### `todo_update(todo_id?, title?, status?, detail?, intent_id?)`

- With no `todo_id`, creates a new todo; otherwise updates the existing one.
- `status`: pending, active, done, blocked, dropped.
- The planner maintains the plan chain. A worker may update only the item assigned to it. A todo whose dependencies are not finished is not scheduled.

### `search_traces(query, limit?)`

- Searches traces from **other tasks** (the current task is excluded), so the same pitfall is not repeated.
- Returns the agent, the tool, an argument summary, a result summary, and the matching snippet.

### `memory_store(text, tags?)` / `memory_search(query, limit?)`

- Memory is persistent text at project scope, for notes such as "what worked, dead ends, environment quirks".
- Retrieval uses zero-dependency TF-IDF (`memory/retrieval.py`) and tokenizes both Chinese and English.
- Do not store credentials here. Events and reports run secret redaction, but do not write a secret in on purpose.

## Collaboration and Control

### `delegate_specialist(specialist, question, context?)`

- `specialist`: `researcher` / `coder` / `adviser` / `memorist`.
- A specialist answers only a narrow question and does not share the main agent's full tool permissions (`toolset()` decides that).
- Bounded by `budget.max_specialist_depth`, which prevents recursive delegation.
- Returns the specialist's text answer. A failure is returned as a tool failure.

### `prove_goal(summary, evidence_ids?)`

- Marks the goal node `proven`, which can serve as a completion signal.
- `evidence_ids` are checked for existence. The Orchestrator still re-checks the completion decision; the model's claim alone is not enough.

### `scope_check(target)`

- A diagnostic tool. It returns `allowed/reason/host/port/ips`.
- It does not make a network request. The model uses it to confirm the boundary before acting.

## Approval and Human Intervention (Enforced)

`safety.approval` is `auto`, `ask`, or `deny`. After argument validation and before the tool runs, `ToolRegistry.execute` calls `_enforce_approval`:

| Mode | `high` / `critical` tools | `low` / `medium` tools |
| --- | --- | --- |
| `auto` (default) | Run immediately (still subject to scope and policy) | Run immediately |
| `ask` | Create an approval record, emit an `approval.requested` event, and **block waiting** for human approval (at most 300 seconds; timeout or cancellation counts as denial). Denial fails the tool | Run immediately |
| `deny` | Deny immediately and write an audit record | Run immediately |

Tools commonly affected: `terminal_exec`, `file_write`, and `port_scan` (all `high`). Approval records are listed with `GET /api/approvals?project_id=` and decided with `POST /api/approvals/<id>` (`{"approved": true|false, "reason": "..."}`). The console has an approval queue.

Even a granted approval does **not** widen scope: target authorization is always enforced by `Scope` on its own. If no approver is attached (for example, library code constructs a `ToolContext` directly), `ask` mode fails closed (the call is denied).

## Conventions for Adding a Tool

When you add a tool:

1. Create a `ToolSpec` in the matching module. State the parameter schema, risk, tags, and `requires_scope`.
2. The function signature is `fn(ctx: ToolContext, kwargs: dict) -> ToolResult`.
3. Call `ctx.scope.require_*` before network I/O.
4. Truncate large output (`truncate`), and redact sensitive fields with `util.redact`.
5. Write state changes to the `Store` and emit an event or an audit record.
6. Add a unit test and run `bash scripts/verify.sh`.
