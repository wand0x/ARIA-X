# ARIA-X deployment guide

ARIA-X is a single-process, standard-library-first application: one Python interpreter, one SQLite file, and an optional
Docker daemon. There is no database service, message queue, Redis, or frontend build step.

> Before deploying, read the authorization and least-privilege checklist in [SECURITY.md](SECURITY.md).

## 1. Requirements

| Component | Requirement |
| --- | --- |
| Python | CPython **≥ 3.11** (3.12 recommended); standard library only |
| Operating system | macOS / Linux; Windows is not verified (path and signal semantics may differ) |
| Disk | `data/` (SQLite + events) + `runs/` (reports); ≥ 1 GB free recommended |
| Optional | Docker / Podman (only when `sandbox.backend="docker"`) |
| Network | A real engagement needs network access to the target and the model API; `aria-x demo` is fully offline |

Using `python3.12` directly is recommended (or set `ARIA_X_PYTHON`):

```bash
python3.12 --version
```

Discovery order in `scripts/_common.sh`: `$ARIA_X_PYTHON` → `python3.12` → `python3.11` →
a `python3` that satisfies ≥3.11.

## 2. Get and start

```bash
cd "/path/to/ARIA-X"
export PYTHONPATH="$PWD/src"

python -m aria_x doctor              # environment self-check
python -m aria_x demo                # offline end-to-end verification
python -m aria_x init                # write configs/aria-x.toml
python -m aria_x serve --open        # Web console
```

ARIA-X does not need to be installed as a package. If you want the `aria-x` command available, create a shell alias yourself:

```bash
alias aria_x='PYTHONPATH="/path/to/ARIA-X/src" python3.12 -m aria_x'
```

(`pyproject.toml` provides the entry point `aria-x = "aria_x.cli:main"`, but the docs do not depend on `pip install`.)

## 3. Configuration

### 3.1 Load order

Defaults < config file < environment variables < explicit overrides.

Config-file search order:

1. `-c/--config <path>` (given explicitly on the command line);
2. `$ARIA_X_CONFIG`;
3. `configs/aria-x.toml`, `aria-x.toml`, `configs/aria-x.json`, `aria-x.json` (current directory).

TOML requires Python 3.11+; JSON always works. **Relative paths (`data_dir`, `runs_dir`,
`llm.scenario`) are resolved against the current working directory**, so run from the repository root.

### 3.2 Environment variables

| Variable | Purpose |
| --- | --- |
| `ARIA_X_CONFIG` | Config file path |
| `ARIA_X_API_KEY` | Model API key (takes precedence over the config file) |
| `ARIA_X_PROVIDER` | `openai` / `anthropic` / `ollama` / `mock` / `scenario` |
| `ARIA_X_BASE_URL` | OpenAI-compatible endpoint |
| `ARIA_X_MODEL` | Model name |
| `ARIA_X_DATA_DIR` | Data directory (SQLite, events) |
| `ARIA_X_AUTH_TOKEN` | Web API Bearer token |
| `ARIA_X_LOG_LEVEL` | `INFO` / `DEBUG`, and so on |
| `ARIA_X_SANDBOX` | `local` / `docker` / `mock` |
| `ARIA_X_APPROVAL` | `auto` / `ask` / `deny` |

### 3.3 Minimal configuration example

```toml
name = "Staging review"
data_dir = "data"
runs_dir = "runs"

[scope]
authorized = true
targets = ["*.staging.example.test", "10.20.0.0/24"]
deny = ["prod.staging.example.test"]
allow_private = true
allow_loopback = true
max_rps = 3.0

[llm]
provider = "openai"
base_url = "https://api.deepseek.com/v1"
model = "deepseek-chat"
temperature = 0.2
max_tokens = 4096

[budget]
max_seconds = 1800
max_llm_calls = 160
max_tokens = 1200000
max_tool_calls = 400
max_concurrency = 2
max_total_intents = 40

[sandbox]
backend = "docker"
image = "aria-x-kali:latest"
network = "bridge"
timeout = 120

[safety]
approval = "auto"
block_destructive = true
redact_secrets = true

[server]
host = "127.0.0.1"
port = 8787
auth_token = ""

[engine]
language = "zh"
bootstrap = true
max_idle_rounds = 2
```

## 4. Model integration

### 4.1 OpenAI-compatible (DeepSeek / vLLM / LM Studio / Ollama)

```toml
[llm]
provider = "openai"
base_url = "https://api.deepseek.com/v1"
model = "deepseek-chat"
```

```bash
export ARIA_X_API_KEY="sk-..."
python -m aria_x doctor --llm
```

Ollama can be used directly with `provider="ollama"` (default `http://127.0.0.1:11434/v1`) or with
`provider="openai"` plus a custom `base_url`.

### 4.2 Anthropic

```toml
[llm]
provider = "anthropic"
base_url = "https://api.anthropic.com"
model = "claude-sonnet-4-20250514"
```

```bash
export ARIA_X_API_KEY="sk-ant-..."
```

### 4.3 Per-role models

```toml
[llm.role_models]
planner = "deepseek-reasoner"
worker = "deepseek-chat"
reporter = "deepseek-chat"
```

### 4.4 Offline providers

- `mock`: replays a list or a policy function; mainly for tests;
- `scenario`: `llm.scenario = "examples/scenario_demo.json"` replays a per-role queue,
  for documentation examples and regression tests. In parallel the queue is consumed concurrently; set `max_concurrency=1` for a deterministic scenario.

## 5. Web console and API

```bash
export ARIA_X_AUTH_TOKEN="$(python3 -c 'import secrets; print(secrets.token_hex(16))')"
python -m aria_x serve --host 127.0.0.1 --port 8787
```

- By default it binds only to `127.0.0.1`. For remote access, **prefer** an SSH tunnel:
  `ssh -L 8787:127.0.0.1:8787 user@host`;
- or put it behind a reverse proxy with TLS (Caddy/Nginx) and set `ARIA_X_AUTH_TOKEN`;
- ARIA-X **does not ship TLS**. Listening on the public internet sends the token and target data in cleartext;
- the SSE endpoint `GET /api/projects/<id>/events` accepts only header authentication (`Authorization: Bearer $ARIA_X_AUTH_TOKEN` or `X-ARIA-X-Token`), and no longer accepts a `?token=` query parameter;
- health: `curl -H "Authorization: Bearer $ARIA_X_AUTH_TOKEN" http://127.0.0.1:8787/api/health`.

## 6. Docker

### 6.1 Build and run

```bash
docker build -f docker/Dockerfile -t aria-x:0.3.0 .
mkdir -p data runs
docker run --rm -p 127.0.0.1:8787:8787 \
  --user "$(id -u):$(id -g)" \
  -e ARIA_X_AUTH_TOKEN="$(openssl rand -hex 16)" \
  -v "$PWD/data:/app/data" -v "$PWD/runs:/app/runs" \
  aria-x:0.3.0
```

The image contains only ARIA-X itself (the standard library) and **does not include** scanning tools. By default the image runs as non-root UID 10001;
a bind-mounted host directory is usually owned by your UID, so the example uses `--user "$(id -u):$(id -g)"` to make the container process
match the host directory owner (otherwise writes to `data/` may be denied). The container writes only the mounted `data/`, `runs/`,
and `/tmp`.

### 6.2 Compose

```bash
export ARIA_X_AUTH_TOKEN="$(openssl rand -hex 16)"
export ARIA_X_API_KEY="sk-..."
export ARIA_X_UID="$(id -u)" ARIA_X_GID="$(id -g)"   # match the mounted data dir owner
docker compose -f docker/docker-compose.yml up --build
```

`docker/docker-compose.yml` requires `ARIA_X_AUTH_TOKEN` to be set explicitly (`docker compose` exits with an error if it is unset or empty, and no longer uses a fixed default password). It also provides a Kali container on the `sandbox` profile as a reference; that container does not start by default
and does not share a network with the ARIA-X container.

### 6.3 Run ARIA-X and a Docker sandbox inside a container

`docker/Dockerfile` does not install the Docker CLI, so `sandbox.backend` inside the container should stay `local`
(the container itself is the isolation layer). Starting further tool containers from inside the ARIA-X container means mounting the host
`/var/run/docker.sock` — that hands host root to the process inside the container, which **ARIA-X does not recommend**.
The safer pattern is to run ARIA-X on the host (or a dedicated VM) and give only the tool containers to the Docker sandbox.

The Kali image and `sandbox.backend="docker"` are documented in
[docker/kali-sandbox.md](../docker/kali-sandbox.md).

## 7. Data directory, backup, and restore

```
data/
  aria-x.db            # SQLite (WAL: also -wal/-shm)
  events.jsonl       # append-only event stream
runs/
  <run_id>/report.md|html|findings.json
  demo-<stamp>/<run_id>/...
```

Backup (after writes have stopped):

```bash
python -m aria_x export -o backup-export.json     # logical export (recommended across versions)
sqlite3 data/aria-x.db ".backup 'aria-x-backup.db'" # physical backup (requires the sqlite3 CLI)
```

Restore:

- A logical export is suitable for review and archival. ARIA-X currently has no `import` subcommand, so restoring a running state means keeping
  `data/aria-x.db` itself;
- put the backed-up `aria-x.db` and `runs/` back in place, then verify with `aria-x projects`.

Upgrades: ARIA-X is at 0.x (pre-1.0), and the schema may change. Back up `data/` and `runs/` before upgrading; if startup reports
an incompatible schema, keep the backup and read the release notes.

## 8. Troubleshooting

| Symptom | Check |
| --- | --- |
| `ModuleNotFoundError: aria_x` | `PYTHONPATH=$PWD/src` is not set, or you are not in the repository root |
| `refusing to start: scope.authorized is false` | This is the authorization gate; set true only after you have written authorization |
| `refusing to start: scope.targets is empty` | Configure at least one target |
| `config file not found` | The `-c` path, `ARIA_X_CONFIG`, or `configs/aria-x.toml` in the current directory |
| `llm.api_key is required` | `export ARIA_X_API_KEY=...`, or set `llm.api_key` |
| `address already in use` | Change `server.port` or `--port`; confirm no old process is left |
| Docker sandbox `image not present` | `docker pull <image>`, or build `aria-x-kali` first |
| `docker daemon not running` | Start Docker Desktop / `systemctl start docker` |
| Run status is `budget` | Normal finish; raise `budget.*` or narrow the target |
| Tools keep getting scope-denied | `python -m aria_x tools` and inspect `scope_check`; confirm URL host/port/prefix coverage |
| Web console returns 401 | Set and send `Authorization: Bearer $ARIA_X_AUTH_TOKEN` |
| SSE does not update | The reverse proxy must disable buffering; or connect directly with `Authorization: Bearer $ARIA_X_AUTH_TOKEN` |
| A run is stuck | Inspect tasks at `/api/runs/<id>` and `data/events.jsonl`; `POST /api/runs/<id>/stop` |

## 9. Pre-production checklist

- [ ] `bash scripts/verify.sh` passes;
- [ ] `python -m aria_x doctor --llm` passes;
- [ ] `scope.authorized` and `scope.targets` come from written authorization, and `deny` covers shared infrastructure;
- [ ] `server.host` is `127.0.0.1`, or the service sits behind a TLS reverse proxy;
- [ ] `ARIA_X_AUTH_TOKEN` is set and is not a default value;
- [ ] `sandbox.backend="docker"`, the image is pinned to a version, and `--privileged` is not used;
- [ ] permissions and retention for `data/` and `runs/` match the contract;
- [ ] stop conditions, a contact, and an incident process are agreed;
- [ ] a person reviews sensitive information before reports are distributed.

## 10. Uninstall / cleanup

```bash
# after stopping the process
rm -rf data runs                      # ⚠️ deletes all engagement state and reports
docker ps -a --filter name=aria-x- --format '{{.ID}}' | xargs -r docker rm -f
docker image rm aria-x:0.3.0 aria-x-kali:latest 2>/dev/null || true
```

Confirm the path before deleting, so you do not remove another project by mistake.
