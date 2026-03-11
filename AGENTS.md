# browserless-tooling AGENTS

## What This Repo Does

This repo contains two Bash provisioning scripts: `aibrowse-setup.sh` for hardened Browserless Chromium instances and `browsewrap-setup.sh` for a Node 22 MCP wrapper layer that integrates with LM Studio.

## Main Entrypoints

- `aibrowse-setup.sh`: Browserless provisioning, token generation, port selection, firewall updates, smoke test.
- `browsewrap-setup.sh`: wrapper deployment, `~/mcp.json` updates, wrapper health checks.
- `README.md`: provisioning flow and validation matrix.

## Commands

- `shellcheck aibrowse-setup.sh browsewrap-setup.sh`
- `bash -n aibrowse-setup.sh`
- `bash -n browsewrap-setup.sh`
- `sudo bash aibrowse-setup.sh <instance>`
- `sudo bash browsewrap-setup.sh <instance>`

## Repo-Specific Constraints

- These scripts are high-impact and require root for real runs.
- Generated infrastructure under `/docker/aibrowse/<instance>` and `/docker/browsewrap/<instance>` is script-owned; do not hand-edit it.
- Do not hardcode ports or tokens; both scripts intentionally allocate and persist them.
- Firewall logic is security-sensitive; treat changes there as risky.
- Keep the smoke-test and `/healthz` verification path intact.

## Agent Notes

- Prefer static validation (`shellcheck`, `bash -n`) unless the task explicitly requires live provisioning.
- Document any uncertainty before suggesting changes to firewall or generated file logic.
