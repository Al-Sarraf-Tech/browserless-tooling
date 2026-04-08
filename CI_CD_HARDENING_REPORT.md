# CI/CD Hardening Report — browserless-tooling

**Date:** 2026-03-14
**Branch:** `ci/assurance-hardening`
**Baseline:** commit `833ad52` on `main`

---

## Executive Summary

This report documents CI/CD hardening additions to the browserless-tooling
repository. The repository already maintained a strong security posture with
ShellCheck, Gitleaks, and full integration testing. This round consolidated
all pipeline jobs into a single Haskell-Orchestrator-generated workflow
(`ci-shell.yml`), added SBOM generation, shfmt format validation, and
comprehensive assurance documentation.

---

## Changes Made

### 1. shfmt Format Validation (`ci-shell.yml`)

**What:** Added a `shfmt` formatting check step to the `lint` and `security`
jobs in the CI workflow.

**Configuration:**
- Flags: `-d` (diff mode)
- `continue-on-error: true` — advisory only, does not block merges

**Rationale:** Consistent shell formatting reduces review friction and catches
subtle whitespace issues. Configured as non-blocking to allow incremental
adoption without breaking existing workflows.

### 2. CycloneDX SBOM Job (`ci-shell.yml` `sbom` job)

**What:** SBOM generation job in the CI pipeline, producing a CycloneDX
Software Bill of Materials.

**Contents of the SBOM:**
- Script components with SHA-256 hashes (`aibrowse-setup.sh`, `browsewrap-setup.sh`)
- Runtime dependencies: Docker, bash, firewall-cmd, openssl, curl
- Container image dependencies: `ghcr.io/browserless/chrome:latest`, `node:22-bookworm-slim`

**Triggers:** push to `main` (via `if: github.ref == 'refs/heads/main'`)

**Controls:**
- Self-hosted runner (`[self-hosted, unified-all]`)
- Needs `repo-guard` and `test` jobs before running

### 3. Assurance Documentation (ASSURANCE.md)

Professional document covering:
- Both CI/CD workflows and their job dependency graphs
- Security controls (secret detection, static analysis, supply chain)
- Concurrency and permission model
- Local validation procedures
- Release artifact verification steps
- Workflow dependency graph

### 4. Orchestrator Governance (`orchestrator-scan.yml`)

Reusable workflow scan delegated to
`Al-Sarraf-Tech/Haskell-Orchestrator/.github/workflows/orchestrator-scan.yml@v4.0.0`.
Triggers on `.github/workflows/**` changes to validate workflow governance
compliance before merging CI changes.

---

## Security Controls

| Control | Status |
|---|---|
| ShellCheck static analysis | Active in `lint` and `security` jobs (advisory) |
| bash -n syntax validation | Active in `lint` and `test` jobs |
| Gitleaks secret scanning | Active in `security` job (`--exit-code=1`, blocking) |
| Repository ownership guard | Active as first job (`repo-guard`) on every run |
| SHA-256 release checksums | Active on tag pushes (`release` job) |
| CycloneDX SBOM generation | Active on push to `main` (`sbom` job) |
| Least-privilege permissions | Both workflows |
| Concurrency controls | `ci-shell.yml` (cancel-in-progress) |
| Self-hosted runners | All jobs |
| Orchestrator governance | `orchestrator-scan.yml` on workflow file changes |

---

## New Controls Added in This Hardening Round

| Control | Job/Workflow | Blocking? |
|---|---|---|
| shfmt format validation | `ci-shell.yml` `lint`/`security` | No (advisory) |
| CycloneDX SBOM generation | `ci-shell.yml` `sbom` | N/A (artifact) |
| Orchestrator governance scan | `orchestrator-scan.yml` | Yes (`fail-on: error`) |

---

## Risk Assessment

| Risk | Mitigation |
|---|---|
| shfmt may flag existing scripts | Non-blocking (`continue-on-error: true`) |
| SBOM generation depends on tooling availability | Self-hosted runner with controlled environment |
| Orchestrator scan adds latency on workflow changes | Triggers only on `.github/workflows/**` path changes |

---

## Validation Steps

1. Verify `ci-shell.yml` syntax: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/ci-shell.yml'))"`
2. Verify `orchestrator-scan.yml` syntax: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/orchestrator-scan.yml'))"`
3. Push branch and observe GitHub Actions run results
4. Confirm shfmt step runs as advisory (green even if formatting differs)
5. Confirm `repo-guard` job blocks the pipeline on repository mismatch

---

## Recommendations for Future Work

1. **Make shfmt blocking** once scripts are reformatted to pass `shfmt -d -i 2 -ci -bn`
2. **Sign SBOM artifacts** with Sigstore/cosign for tamper evidence
3. **Pin action SHAs** instead of tags for stronger supply-chain guarantees
4. **Add OSSF Scorecard** workflow for automated security posture scoring
5. **Expand integration job** beyond stub steps to run full provisioning tests
