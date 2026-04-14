# Browserless Tooling

[![Regression CI](https://github.com/Al-Sarraf-Tech/browserless-tooling/actions/workflows/ci-shell.yml/badge.svg)](https://github.com/Al-Sarraf-Tech/browserless-tooling/actions/workflows/ci-shell.yml)
![](https://img.shields.io/badge/release-v1.0.1-0a0a0a?style=flat-square&labelColor=353535)
![](https://img.shields.io/badge/bash-set%20-Eeuo%20pipefail-0a0a0a?style=flat-square&labelColor=353535)
![](https://img.shields.io/badge/security-firewall__aware-0a0a0a?style=flat-square&labelColor=353535)
![](https://img.shields.io/badge/automation-idempotent-0a0a0a?style=flat-square&labelColor=353535)

> CI runs on self-hosted runners governed by the [Haskell Orchestrator](https://github.com/Al-Sarraf-Tech/Haskell-Orchestrator). A `repo-guard` job verifies repository ownership before all other pipeline jobs run.

Hardened provisioning scripts for [Browserless](https://www.browserless.io/) Chromium and an LM Studio MCP wrapper. Security-first defaults, deterministic outputs, idempotent execution.

---

## What It Does

Two scripts form a two-tier stack:

1. **`aibrowse-setup.sh`** provisions a Browserless Chromium Docker container with a randomly selected high-range port, a generated token, persistent volumes, firewall rules, and a smoke-test screenshot.
2. **`browsewrap-setup.sh`** deploys a lightweight Node 22 MCP wrapper container that bridges LM Studio to the Browserless instance and registers itself in `~/mcp.json`.

Both scripts are idempotent: rerunning against an existing instance reuses the stored credentials and reconfigures the stack without generating new secrets or ports.

---

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│  Step 1 — aibrowse-setup.sh <instance>                   │
│                                                          │
│  dependency guard → port collision check (20k–39k range) │
│  → openssl token gen → .env / .compose.env (0640)        │
│  → docker-compose.yml → docker compose up                │
│  → health poll (metrics endpoint, 90 attempts × 2s)      │
│  → smoke-test screenshot (POST /chrome/screenshot)        │
│  → firewalld rule (trusted zone, IPv4 + IPv6)            │
│                                                          │
│  Artifacts: /docker/aibrowse/<instance>/                 │
│    .env  .compose.env  docker-compose.yml                │
│    profiles/  downloads/  logs/  client_examples/        │
└──────────────────────────┬───────────────────────────────┘
                           │ reads .env
┌──────────────────────────▼───────────────────────────────┐
│  Step 2 — browsewrap-setup.sh <instance>                 │
│                                                          │
│  dependency guard → load Browserless secrets             │
│  → port collision check (41k–58k range)                  │
│  → generate Node 22 app (Dockerfile + server.mjs)        │
│  → docker compose build --pull + up                      │
│  → health poll (/healthz, 30 attempts × 2s)              │
│  → update ~/mcp.json (upsert browsewrap-<instance>)      │
│  → firewalld rule (trusted zone)                         │
│                                                          │
│  Artifacts: /docker/browsewrap/<instance>/               │
│    .env  .compose.env  docker-compose.yml                │
│    app/  (Dockerfile + src/server.mjs)  logs/            │
└──────────────────────────────────────────────────────────┘
```

### MCP Wrapper

`server.mjs` is a minimal Node.js HTTP server that exposes:

- `GET /healthz` — returns `{"status":"ok","browserless_endpoint":"...","uptime_seconds":N}`
- All other paths return `404`

On startup the wrapper appends to `logs/wrapper.log`. `~/mcp.json` is updated with an entry keyed by `browsewrap-<instance>` pointing to the wrapper's port and the Browserless token. LM Studio reads this file to discover the connector.

### Security Posture

| Control | Implementation |
|---|---|
| Token generation | `openssl rand -hex 24` at first run; reused on subsequent runs |
| Credential storage | `.env` files at `0640`; never committed, never logged |
| Firewall | `firewall-cmd --permanent --zone=trusted --add-port=<port>/tcp`; skipped gracefully if firewalld absent or inactive |
| Port selection | Random from safe ranges; checked against both `ss` (local processes) and `docker ps` (containers) before assignment |
| Container logging | `json-file` driver, 10 MB max, 5 file rotation |

---

## Prerequisites

- Linux host with Docker (Compose plugin or standalone `docker-compose`)
- `openssl`, `curl`, `ss` (`iproute2`)
- `python3` (for `browsewrap-setup.sh` MCP config update)
- `firewall-cmd` (optional; firewall configuration skipped if absent)
- Root or `sudo` access

---

## Usage

```bash
# Step 1 — provision Browserless Chromium
sudo bash aibrowse-setup.sh <instance-name>

# Verify: metrics endpoint
curl -fsS "http://localhost:<port>/metrics?token=<token>"
# Verify: smoke-test screenshot (>10 KB)
ls -lh /docker/aibrowse/<instance-name>/downloads/smoke-test.png

# Step 2 — deploy MCP wrapper
sudo bash browsewrap-setup.sh <instance-name>

# Verify: wrapper health
curl -fsS "http://localhost:<wrapper-port>/healthz"
# Verify: LM Studio connector registered
cat ~/mcp.json
```

Instance names must be lowercase alphanumeric with optional dashes or underscores (`[a-z0-9][a-z0-9_-]{0,62}`).

**Re-provision an existing instance** (e.g., after a host reboot):

```bash
sudo bash aibrowse-setup.sh <instance-name>   # reuses existing .env
sudo bash browsewrap-setup.sh <instance-name> # reuses existing .env
```

**Playwright CDP example** is scaffolded at:

```
/docker/aibrowse/<instance-name>/client_examples/playwright_connect.py
```

---

## Observability

```bash
# Browserless container status and logs
sudo docker ps --filter name=browserless-<instance>
sudo docker logs browserless-<instance> --tail 50

# MCP wrapper status and logs
sudo docker ps --filter name=browsewrap-<instance>
sudo docker logs browsewrap-<instance> --tail 50
cat /docker/browsewrap/<instance>/logs/wrapper.log

# Smoke-test artifact
ls -lh /docker/aibrowse/<instance>/downloads/smoke-test.png
```

---

## Local Validation

```bash
# Syntax check
bash -n aibrowse-setup.sh browsewrap-setup.sh

# ShellCheck lint
shellcheck -x --exclude=SC2155,SC1091 aibrowse-setup.sh browsewrap-setup.sh

# shfmt format check (advisory)
shfmt -d -i 2 -ci -bn aibrowse-setup.sh browsewrap-setup.sh

# Secret scan
gitleaks detect --source . --no-banner --exit-code=1

# Full integration test (requires root, Docker, firewalld)
INSTANCE="local-test-$(date +%s)"
sudo bash ./aibrowse-setup.sh "${INSTANCE}"
sudo bash ./browsewrap-setup.sh "${INSTANCE}"
# Cleanup
sudo docker rm -f "browserless-${INSTANCE}" "browsewrap-${INSTANCE}"
sudo rm -rf "/docker/aibrowse/${INSTANCE}" "/docker/browsewrap/${INSTANCE}"
```

---

## CI/CD

Governed by the [Haskell Orchestrator](https://github.com/Al-Sarraf-Tech/Haskell-Orchestrator). All jobs run on self-hosted runners.

**`ci-shell.yml`** — triggers on push to `main`, version tags, pull requests, weekly schedule (Monday 04:00 UTC), and manual dispatch.

```
repo-guard
├── lint       bash -n, ShellCheck (advisory), shfmt (advisory)
│   ├── test   bash -n validation
│   │   ├── sbom         CycloneDX SBOM (main branch only)
│   │   └── integration  static analysis stubs
│   └── security   Gitleaks (blocking), ShellCheck (advisory)
│       └── release    SHA-256 checksums + GitHub Release (tag pushes only)
```

**`orchestrator-scan.yml`** — triggers on workflow file changes; delegates to the Haskell Orchestrator reusable workflow (`@v4.0.0`, `fail-on: error`) for governance validation.

See [ASSURANCE.md](ASSURANCE.md) for full security controls, concurrency model, and release artifact verification.

---

## Repository Map

```
.
├── aibrowse-setup.sh         # Browserless Chromium provisioning
├── browsewrap-setup.sh       # LM Studio MCP wrapper deployment
├── ASSURANCE.md              # CI/CD gates, security controls, local validation
├── CI_CD_HARDENING_REPORT.md # Hardening change log and risk assessment
├── LICENSE
└── .github/
    ├── workflows/
    │   ├── ci-shell.yml          # Primary CI pipeline
    │   └── orchestrator-scan.yml # Workflow governance scan
    └── CODEOWNERS
```
