# Docker sandbox and Kali tool image

ARIA-X's default sandbox is `local` (commands run directly with the current user's privileges). For a real isolation boundary,
set `sandbox.backend` to `docker` and use a Kali image to carry tools such as `nmap`, `ffuf`, and `nuclei`.

> **Security note**: the Docker sandbox is not a scope boundary. Target authorization is enforced by ARIA-X `Scope` and the network tools;
> Docker only isolates commands, limits resources, and provides a disposable filesystem. Do not mount a host directory,
> `docker.sock`, or the host network into a tool container just for "convenience".

## 1. Prepare the image

```bash
docker pull kalilinux/kali-rolling
# derived image with common tools (pick what you need; the build needs network access)
cat > /tmp/Dockerfile.kali <<'DOCKER'
FROM kalilinux/kali-rolling
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      nmap curl whatweb ffuf gobuster sqlmap nikto dnsutils openssl \
      python3 ca-certificates \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /work
DOCKER
docker build -f /tmp/Dockerfile.kali -t aria-x-kali:latest .
```

If you only need the HTTP/TCP tools that ship with ARIA-X, the stock `kalilinux/kali-rolling` image is enough;
no packages need to be installed.

## 2. Configure ARIA-X

```toml
[sandbox]
backend = "docker"
image = "aria-x-kali:latest"   # or kalilinux/kali-rolling
network = "none"             # change to "bridge" when the target must be reachable
timeout = 120

[scope]
authorized = true
targets = ["127.0.0.1", "*.lab.example.test"]
```

```bash
PYTHONPATH=src python -m aria_x doctor     # checks the Docker daemon and the image
```

## 3. Behavior and limits

- Each `terminal_exec` starts a **one-shot container** (`docker run --rm -i`);
  `/work` is the current engagement working directory; the container is destroyed on exit.
- `network = "none"` means no external network. To scan a lab target, use `bridge`,
  and put that target only on an isolated lab network.
- Commands in the image run as container root (the upstream Kali image default). ARIA-X's outer
  `CommandPolicy` blocks destructive commands such as `rm -rf /`, `mkfs`, `dd of=/dev/sd*`, and fork bombs,
  but that is only a heuristic, not a complete sandbox.
- On timeout, ARIA-X runs `docker kill`; output is truncated to `sandbox.max_output` bytes.
- It does not use `--privileged`, does not mount `/var/run/docker.sock`, and does not mount the host root.

## 4. Run ARIA-X itself in a container

`docker/Dockerfile` contains only ARIA-X itself (standard library, no pip). It **does not include** the Docker CLI,
so `sandbox.backend` inside that container can only be `local` (the container itself is the isolation layer).
If you really need a container that starts further containers, you must install the Docker CLI yourself and mount the host socket —
that hands host root to the process inside the container. ARIA-X does not recommend this and does not support it by default.

## 5. Cleanup

```bash
docker ps -a --filter name=aria-x- --format '{{.ID}}' | xargs -r docker rm -f
docker image rm aria-x-kali:latest
```
