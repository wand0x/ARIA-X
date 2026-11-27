# ARIA-X Security Model and Legitimate Use

## 0. Authorized Use (Read This Section First)

ARIA-X is an **offensive security testing tool**. Within the scope you configure, it actively sends network requests, executes commands, and tries to discover and verify vulnerabilities.

**You may use ARIA-X only on the following systems:**

- Systems that you own and control;
- Systems that your organization has explicitly authorized you to test;
- Systems covered by written authorization (a letter of authorization, a penetration-testing contract, or a bug-bounty program scope).

**Unauthorized access to a computer system is a criminal offense in most jurisdictions**, even if you only "take a look" or "cause no damage". "Visible on the public internet" is not "allowed to test"; "scanning is not intrusion" is wrong; "the customer said it was okay verbally" is usually not enough to constitute authorization.

Before using ARIA-X, confirm:

1. You have written authorization, and the authorized scope (domains, IPs, ports, time window, prohibited actions) is explicit;
2. You understand the target may be a production system, and testing may cause an outage or data damage;
3. You know the laws of your jurisdiction on vulnerability research, data protection, and computer misuse;
4. You and the system owner have agreed on a point of contact, stop conditions, and an escalation path.

**The authors and contributors are not responsible for any unauthorized or abusive use.** ARIA-X is provided "AS IS" under AGPL-3.0, with no warranty of any kind.

## 1. How Authorization Is Enforced in Code

### 1.1 Startup Gate

`AriaXConfig.require_authorization()` runs before every engagement starts:

```toml
[scope]
authorized = true          # must be explicitly true
targets = ["127.0.0.1"]    # must be non-empty
```

If either is missing, it raises `ConfigError` and does not create a run. A project scope that explicitly sets `authorized=false` is also rejected. The CLI `aria-x run` and `Engagement.run/start_background` both go through the same check.

### 1.2 Target-Level Scope

`src/aria_x/tools/scope.py` supports:

- Exact hosts: `127.0.0.1`, `app.example.test`;
- Wildcard domains: `*.staging.example.test` (the name itself and every subdomain);
- CIDR: `10.10.0.0/24`;
- Host plus port: `db.example.test:5432`;
- URL prefixes: `https://app.example.test:8443/api`;
- `deny` rules take precedence over `allow`;
- `allow_private` / `allow_loopback` control private networks and loopback;
- `max_rps` rate-limits each host.

Network tools (`http_request`, `dns_lookup`, `port_scan`, `tls_probe`, `web_probe`) call `require_url` / `require_host` **before the first packet is sent**. A violation becomes a tool failure plus an audit record, not a silent allow.

> **Limitation**: Tools inside `terminal_exec` (such as `nmap`) can only be checked heuristically against literal URLs or IPs in the command. Indirect addressing, targets inside scripts, DNS rebinding, and similar cases cannot be fully stopped at this layer. The real isolation boundary is a **Docker sandbox + network policy + human review**, not a scope regex.


### 1.3 Approval Gate for High-Risk Tools

`safety.approval` is enforced inside `ToolRegistry.execute` and applies to tools whose `risk` is `high` or `critical` (currently `terminal_exec`, `file_write`, and `port_scan`):

| Mode | Behavior |
| --- | --- |
| `auto` (default) | Execute immediately, still subject to scope, the workdir jail, and command policy |
| `ask` | Write an approval record, emit an `approval.requested` event, and **block waiting** for a human approval (at most 300 seconds; timeout or cancellation counts as denial); on denial the tool returns failure |
| `deny` | Deny immediately, write an audit record, and do not execute |

An approval does not widen scope and does not bypass command policy. If no approver is attached (for example, a library caller builds its own `ToolContext`), `ask` mode fails closed instead of allowing silently. Decisions can be made in the Web console or via `POST /api/approvals/<id>`, and every decision is written to the audit table.

Production guidance: use `approval = "ask"` against real targets, and keep a human confirmation at least for `terminal_exec` and `file_write`; demos and automated tests may use `auto`.

> **Limitation**: Approval is a human in the loop, not technical isolation. The approver may be tired, mis-click, or allow an action without understanding the context. A high-risk engagement should also use a Docker sandbox, network segmentation, and a time window.

### 1.4 Sandbox Backend Boundaries (`local` and `docker`)

- **`local` backend (host privileges, no OS-level filesystem isolation)**: When `sandbox.backend = "local"`, `terminal_exec` and interactive `session_*` commands run with **the privileges of the current operating-system user**, and child processes keep the `HOME` environment variable. The work-directory check only constrains `cwd` at command start (and the path jail of `file_*` tools). `CommandPolicy` only blocks known destructive commands and **does not provide operating-system-level filesystem isolation** — a shell command the model runs can still, in principle, read any file the current user is allowed to access (such as `~/.ssh`, `~/.aws`, or browser data).
- **Isolation guidance**: When you need real filesystem and process isolation, use `sandbox.backend = "docker"` (non-root, `network = "none"`, and do not mount `/var/run/docker.sock`), or run under a dedicated low-privilege account or in an isolated virtual machine.

## 2. Threat Model

### 2.1 What We Defend Against

| Threat | Mitigation |
| --- | --- |
| Accidentally testing an unauthorized target | Startup gate + scope + deny + loopback and private networks allowed by default; the docs repeat this |
| The model expanding scope on its own | The planner alone owns intents; scope is enforced at the tool layer; toolsets are narrowed by role |
| Destructive commands | `CommandPolicy` blacklist (`rm -rf /`, `mkfs`, `dd of=/dev/*`, fork bomb, shutdown, `curl|bash`, `history -c`, and similar); high-risk tools are also controlled by the approval gate |
| Filesystem escape | `file_*` tools are strictly limited to the engagement workdir (path traversal and symlink escape are rejected); the terminal only constrains the starting `cwd`; the `local` backend does not provide OS-level filesystem isolation (strong isolation requires `docker`) |
| Secrets leaking into logs or reports | `util.redact` runs before events, traces, and reports are written; `public_dict()` uses an explicit public-field allowlist and masks secrets in full; `save()` / `export_public()` export only a redacted snapshot that cannot be loaded; a real local save requires an explicit `save_private()` (`0o600`); SSE authenticates only with request headers; HTTP logs are normalized by route template and strip sensitive query parameters and raw exception bodies |
| Remote abuse of the Web console | Binds only to `127.0.0.1` by default; optional Bearer token; a reverse proxy plus TLS is recommended |
| Runaway budget or an infinite loop | Time, LLM calls, tokens, tool calls, total intent count, concurrency, and specialist depth are all bounded |
| Leftover processes | `Engagement.close()` cancels first, then joins (10s cap); the demo target handles SIGTERM |
| Fabricated reports | Reports are rendered from graph data; the model writes only summary text; a finding must be stored |

### 2.2 What We Do Not Defend Against

- **A malicious or poisoned model service**: Content returned by the provider is treated as untrusted input, but ARIA-X will not execute anything the model supplies other than shell; tool arguments are still constrained by schema, scope, and policy. If the model service itself is compromised, it can see the context sent to it (including target data).
- **The target countering the tester**: A malicious target can return prompt injection aimed at the model. ARIA-X treats tool output as data, not instructions, but it has no dedicated prompt-injection classifier; a human should supervise high-risk scenarios.
- **Supply chain**: ARIA-X has no third-party Python dependencies, but it still depends on CPython, the operating system, and optional Docker images. Verify base images and the toolchain yourself.
- **Physical and host-layer attacks**: ARIA-X is not a product that prevents sandbox escape; running it on an untrusted host is itself unsafe.

## 3. Least-Privilege Deployment Checklist

- [ ] List only authorized assets in `scope.targets`; use `deny` to exclude shared infrastructure;
- [ ] Set `scope.authorized=true` only inside this authorization window;
- [ ] `safety.approval="ask"`, covering at least `terminal_exec` / `file_write` / `port_scan`;
- [ ] `sandbox.backend="docker"`, pin the image version, and set `network` as needed (default `none`);
- [ ] Do not use `--privileged`, do not mount `/var/run/docker.sock`, and do not mount the host root;
- [ ] `server.host="127.0.0.1"`; reach it remotely through an SSH tunnel or a reverse proxy with TLS;
- [ ] Set `ARIA_X_AUTH_TOKEN` to a long random value; do not use a default;
- [ ] Keep `ARIA_X_API_KEY` only in the environment, not in the repository; rotate it regularly;
- [ ] Restrict `data/` and `runs/` to the user that runs the process; reports may contain target data;
- [ ] Before a run, `python -m aria_x doctor --llm`, and confirm configuration and connectivity;
- [ ] During a run, watch events and budget through `aria-x serve`; on an anomaly, `POST /api/runs/<id>/stop`;
- [ ] Afterward, review `runs/<run_id>/` and `data/aria-x.db`, and retain or destroy them as the contract requires.

## 4. Secret Handling

- `llm.api_key` may come from `ARIA_X_API_KEY`; `server.auth_token` may come from `ARIA_X_AUTH_TOKEN`.
- `util.redact` covers: `Authorization: Bearer/Basic ...`, `api_key/token/secret/password/session=...`, `sk-...`, AWS `AKIA...`, and GitHub `ghp_/gho_/...`.
- The event bus turns redaction on by default (`safety.redact_secrets=true`); JSONL, SSE, reports, and `public_dict()` use the same logic.
- **Redaction is best-effort**: custom secret formats, images, binary content, and secrets the model restates in free text can slip through. Do not treat ARIA-X as a DLP product; a human should review the report of a sensitive engagement before it is distributed.
- Do not put real credentials into example configuration, scenario JSON, or test fixtures.

## 5. Security Design of the Demo Target

`examples/demo_target.py` deliberately contains three weaknesses, but they are constrained to be **harmless**:

1. **Path traversal**: `/files?path=../flag.txt` uses a naive `os.path.join` and will escape `public/`; the handler then allows reads of only two things: files under the `public/` subtree and the generated `flag.txt`. Arbitrary paths such as `/etc/passwd` return 404.
2. **Missing security response headers**: there is no CSP, HSTS, X-Frame-Options, or nosniff — this exists so a scanner has something to find.
3. **Information disclosure at `/debug`**: it returns a fake version and `debug=true`.

Other constraints:

- It binds only to `127.0.0.1` and does not accept external connections;
- Each start creates a one-shot temporary root directory, which is discarded on exit;
- It does not execute commands, write system paths, or make network requests;
- It exits cleanly on SIGTERM and SIGINT;
- `--json` writes one line of startup JSON to stdout; logs go to stderr.

**Do not** expose the demo target to the public internet, and do not put real data in it.

## 6. Vulnerability Disclosure (for ARIA-X Itself)

If you find a security issue in ARIA-X:

1. **Do not** publish reproducible details or an exploit immediately;
2. Report it through a private channel the maintainers provide (the repository SECURITY contact, or the maintainer's email);
3. Include the affected version, reproduction steps, an impact assessment, and a suggested fix;
4. Give the maintainers a reasonable window to fix it (usually 90 days); do not publish during that window;
5. After the fix is released, credit them as both sides agreed.

In scope: scope bypass, sandbox escape, approval bypass, authentication bypass, path traversal, command injection, secret leakage, report XSS, denial of service against SSE or HTTP, and similar issues.

## 7. Data and Privacy

- ARIA-X writes target responses, command output, and evidence summaries to `data/aria-x.db`, `data/events.jsonl`, and `runs/`. That data may contain personal information, credentials, or trade secrets.
- Before a run, confirm that the retention period and storage location comply with the authorization contract and local law (GDPR, personal-information protection laws, and similar).
- To delete one engagement: stop the service, delete the corresponding `runs/<run_id>/`, and use `aria-x export` to back up conclusions you need to keep. `data/aria-x.db` is a shared database; deleting a single project requires SQL (there is no CLI subcommand for it today).
- Do not commit a report that contains real target data to a public repository.

## 8. Compliance Notes

- **AGPL-3.0**: If you offer ARIA-X over a network, you must make the corresponding source available to users (§13). Put a "Source" link in the console footer that points at your repository mirror.
- **Export controls and sanctions**: Some jurisdictions restrict export of penetration-testing tools or cryptographic tools; confirm this yourself.
- **The contract comes first**: Restrictions in the authorization contract (no DoS, no data exfiltration, no social engineering, and similar) override any default behavior of ARIA-X. Tighten with `deny`, `budget`, and `sandbox.network="none"` when you need to.
