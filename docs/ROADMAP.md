# ARIA-X roadmap

This file separates three things: **what already works**, **what comes next**, and **what will not be done**.
Anything not yet implemented does not appear in the README "features" list.

## v0.1 (current) — implemented

### Engine and state

- [x] SQLite persistence: projects, runs, nodes, edges, assets, anchors, todos, tasks, traces,
      evidence, messages, audit, approvals, and kv (`store/schema.py`).
- [x] Dual-graph model: exploration graph (nodes/edges) + asset graph (assets/anchors).
- [x] Planner is the sole intent generator; intent dedup/limits; todo dependency scheduling.
- [x] Main loop: bootstrap → reason → dispatch → explore → persist → report.
- [x] Budget: time, LLM calls, tokens, tool calls, total intents, concurrency, and specialist depth.
- [x] Run states `completed/budget/cancelled/failed`; a report is still produced when the budget is exhausted.
- [x] **Mandatory approval gate**: when `safety.approval=ask`, `high`/`critical` tools block until a human approves;
      `deny` rejects them outright; `auto` allows them. The decision is written to the approval table and the audit log.
- [x] Background runs, stop, and status queries; in-process Run registry.
- [x] Event bus + JSONL + SSE replay; secret redaction.
- [x] JSON project export.

### Models

- [x] OpenAI-compatible provider (DeepSeek/vLLM/Ollama and others), with native tool calling.
- [x] Anthropic provider.
- [x] `mock` provider (list/policy).
- [x] `scenario` provider (JSON scripts, role queues, substring matching, loops/inline JSON).
- [x] JSON-contract parsing and retries; per-role models; token-usage estimates.

### Tools

- [x] 27 tools: terminal/files, HTTP/DNS/port/TLS/Web probing, external search, blackboard, assets,
      plans, traces, memory, specialist delegation, goal proof, and scope diagnostics.
- [x] Tool registration / argument validation / cancellation checks / exception boundaries.
- [x] `Scope`: host/wildcard/CIDR/URL prefix/deny/loopback/private networks/rate limit.
- [x] Sandbox: `local`, `docker` (one-shot containers), `mock`; destructive-command policy.
- [x] workdir jail, output truncation, and evidence and trace recording.

### Agent

- [x] planner / bootstrap / worker / specialist (researcher/coder/adviser/memorist) /
      reporter specs and the worker runtime.
- [x] Specialist-delegation depth limit.
- [x] Role prompts and JSON contracts.

### Reports and UI

- [x] `report.md`, self-contained `report.html`, and `findings.json`.
- [x] Zero-dependency TF-IDF memory retrieval.
- [x] HTTP API (projects/graph/traces/audit/reports/runs/stop/hint/approvals/export/SSE).
- [x] Framework-free, CDN-free Web console (ships with the server; screenshots will be added after the UI is finalized).
- [x] Optional Bearer token; binds to `127.0.0.1` by default.

### Engineering and compliance

- [x] Standard library only, zero third-party dependencies; Python 3.11+.
- [x] `unittest` suite + offline E2E (real target + real engine + scripted policy).
- [x] `scripts/bootstrap.sh`, `dev.sh`, and `verify.sh` (no pip, no network).
- [x] Dockerfile / compose / Kali sandbox docs.
- [x] AGPL-3.0 LICENSE, NOTICE, THIRD_PARTY, and the FUSION map.
- [x] Chinese README + English README; architecture/tools/security/deployment docs.

## v0.2 — next (in priority order)

### Security and correctness

- [ ] **Finer approval policy**: approval rules by tool, argument, and target (today there is one policy by risk level),
      plus an approval timeout and automatic denial.
- [ ] **Run recovery**: after a process crash, resume unfinished intents from Run/Task/Trace instead of only starting a new run.
- [ ] **Prompt-injection marking**: mark target responses as untrusted data, add a simple hint for injection signatures,
      and require a second confirmation for high-risk actions.
- [ ] **DNS pinning for scope**: resolve once and pin the IP, to reduce DNS-rebinding risk; today each tool call may resolve again.
- [ ] **Document the per-run network policy**: a matrix of Docker networks combined with scope, and a reference topology for an isolated lab.

### Capabilities

- [ ] **More tools**: directory/parameter discovery (bounded), OpenAPI/GraphQL probing, JWT parsing, and screenshot evidence
      (needs an optional headless browser; the default standard-library path stays unaffected).
- [ ] **Attack-path graph**: visualize fact → intent → finding in the report (today this is prose).
- [ ] **Multi-target projects**: parallelize by asset group within one project; concurrency is currently global at the intent level.
- [ ] **Report PDF**: do not add a heavy library under the zero-dependency constraint; prefer instructions for printing from the browser.
- [ ] **Finer budgets**: quotas by role and tool category; today the dimensions are global.

### Experience

- [ ] Web console screenshots, a GIF, and a demo video;
- [ ] Chinese/English UI switch (report language is currently controlled by `engine.language`);
- [ ] `aria-x import`: restore a read-only project view from a JSON export;
- [ ] Friendlier `doctor` output (dependency graph, ports in use, model latency).

### Engineering

- [ ] CI (GitHub Actions): `unittest` + `verify.sh` + documentation link checks;
- [ ] A coverage threshold and property-based tests of core paths (still using the standard library);
- [ ] Release process: version number, CHANGELOG, container-image signing/SBOM;
- [ ] Performance: pagination and index tuning for `graph_overview` on large graphs (>10k nodes).

## Explicitly out of scope

- **Automated weaponization / large-scale exploitation**: no worm-like propagation, bulk 0day exploitation, persistent backdoors,
  or large-scale credential theft. ARIA-X is for authorized assessment and evidence production.
- **Evasion / anti-forensics**: no stealth, log wiping, EDR bypass, or C2 hiding.
- **Unauthorized target discovery**: no internet-wide asset mapping, random IP scanning, or Shodan-style discovery.
- **Multi-tenant SaaS**: this is a single-machine, single-team tool; there is no public multi-tenant control plane.
- **Built-in vulnerability database / exploit database**: the model may generate payloads, but ARIA-X does not distribute an exploit collection.
- **Replacing a human authorization decision**: `scope.authorized=true` is a person's declaration; the tool will not obtain authorization for you.

## How to contribute

1. Read [CONTRACTS.md](CONTRACTS.md) and [ARCHITECTURE.md](ARCHITECTURE.md) first;
2. One PR does one thing; a new tool must include a schema, a risk label, and tests;
3. Network-related code must check scope before I/O; do not introduce third-party Python dependencies;
4. Documentation changes must match the code; do not put "planned" features into the README features list;
5. License: contributions are licensed under AGPL-3.0; anything that involves upstream code must discuss attribution first.
