# CLAUDE.md — browserless-tooling

Bash provisioning toolkit for hardened Browserless Chromium + LM Studio MCP wrappers.
Two idempotent setup scripts. No compiled code — pure shell.

---

## Organizational Directive (Claude Only)

> **This directive applies ONLY when Claude Code is in use — it is a standing operational policy, not a suggestion.**
>
> Claude operates in this repository as a structured internal engineering organization: single point of contact, adaptive team complexity (Tier 0–4), mandatory review on all work, batch processing, and parallelization where safe. Full directive: `~/.claude/CLAUDE.md`.

---

## Scripts

| Script | Role | Requires |
|---|---|---|
| `aibrowse-setup.sh` | Boot Browserless Chromium with persistent profiles, firewall rules, smoke-test screenshot | root, Docker, `firewall-cmd` |
| `browsewrap-setup.sh` | Deploy Node 22 MCP wrapper for LM Studio, update `~/mcp.json` | root, running Browserless instance |

---

## Usage

```bash
# Provision Browserless (first)
sudo bash aibrowse-setup.sh <instance-name>

# Verify health
curl -fsS http://localhost:<port>/metrics?token=<token>

# Add MCP wrapper
sudo bash browsewrap-setup.sh <instance-name>

# Verify wrapper
# Check /docker/browsewrap/<instance>/logs/wrapper.log for /healthz output
```

Smoke-test screenshot lands at `/docker/aibrowse/<instance>/downloads/smoke-test.png`
(must be >10 KB to pass).

---

## File Locations After Provisioning

```
/docker/aibrowse/<instance>/    # Browserless data, downloads, logs
/docker/browsewrap/<instance>/  # MCP wrapper app, logs
~/mcp.json                      # Updated by browsewrap-setup.sh with new endpoint
```

---

## Coding Conventions

- `#!/usr/bin/env bash` and `set -Eeuo pipefail` on all scripts.
- All functions are named. No anonymous inline logic blocks.
- Idempotent: re-running a script on an already-provisioned instance must be safe.
- Firewall rules managed via `firewall-cmd --permanent` + `--reload`. Never `iptables`.
- Port selection is randomised at provision time and stored in the instance `.env`.
  Do not hardcode ports.
- No secrets in script source. Tokens are generated at provision time or read from `.env`.
- Validate binary prerequisites with `command -v` before use.

---

## Do Not Change Without Review

- Firewall rules logic — wrong changes can lock out SSH.
- `.env` generation — the MCP wrapper reads these at runtime.
- The smoke-test section — it is the acceptance gate for a successful provision.

---

## Validation

```bash
shellcheck aibrowse-setup.sh browsewrap-setup.sh
# Then do a dry run on a test instance name
sudo bash aibrowse-setup.sh test-01
sudo bash browsewrap-setup.sh test-01
```

---

## Toolchain

| Tool | Path | Version |
|---|---|---|
| bash | `/usr/bin/bash` | system |
| shellcheck | `/usr/bin/shellcheck` | system (dnf) |
| Go | `/go/bin/go` | 1.26.1 — not used by this repo |
| Rust | `/usr/bin/rustc` | 1.93.1 — not used by this repo |
| Python | `/usr/bin/python3` | 3.14.3 — not used by this repo |

This repo is pure Bash. No build step required.


---

## CI/CD Pipeline (Enforced)

This repository's CI/CD pipeline is **generated and managed by the Haskell CI Orchestrator** (`~/git/haskell-ci-orchestrator`). Do not manually edit `.github/workflows/ci.yml` — changes will be overwritten on the next sync.

**Directives:**
- All CI/CD runs through the unified `ci.yml` pipeline (lint → test → security → sbom → docker → integration → release)
- **Never release for macOS or Windows** — linux-only releases, no macOS/Windows runners
- **Never use the Gentoo runner** — all jobs target `[self-hosted, unified-all]`
- **Never touch `haskell-money` or `haskell-ref`** — hard-denied by the orchestrator
- Pipeline changes go through the orchestrator catalog (`CI.Catalog`), not direct YAML edits
- The orchestrator validates, generates, and syncs workflows across all 15 repos
