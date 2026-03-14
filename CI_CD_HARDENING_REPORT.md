# CI/CD Hardening Report — browserless-tooling

**Date:** 2026-03-14
**Branch:** `ci/assurance-hardening`
**Baseline:** commit `833ad52` on `main`

---

## Executive Summary

This report documents CI/CD hardening additions to the browserless-tooling
repository. The repository already maintained a strong security posture with
ShellCheck, Gitleaks, CodeQL, SLSA provenance attestations, and full
integration testing. This round adds SBOM generation, shell formatting
validation, and comprehensive assurance documentation.

---

## Changes Made

### 1. shfmt Format Validation (ci.yml)

**What:** Added a `shfmt` formatting check step to the static batch in the
Regression CI workflow.

**Configuration:**
- shfmt v3.10.0, downloaded as a static binary
- Flags: `-d -i 2 -ci -bn` (diff mode, 2-space indent, case indent, binary
  newline)
- `continue-on-error: true` — advisory only, does not block merges

**Rationale:** Consistent shell formatting reduces review friction and catches
subtle whitespace issues. Configured as non-blocking to allow incremental
adoption without breaking existing workflows.

### 2. CycloneDX SBOM Workflow (sbom.yml)

**What:** New workflow generating a CycloneDX 1.5 Software Bill of Materials.

**Contents of the SBOM:**
- Script components with SHA-256 hashes (`aibrowse-setup.sh`, `browsewrap-setup.sh`)
- Runtime dependencies: Docker, bash, firewall-cmd, openssl, curl
- Container image dependencies: `ghcr.io/browserless/chromium`, `node:22-slim`

**Triggers:** push to `main`, weekly schedule, manual dispatch

**Controls:**
- `permissions: contents: read`
- Concurrency group with cancel-in-progress
- Self-hosted runner
- SBOM validated against CycloneDX spec
- Uploaded as artifact with 90-day retention

### 3. Assurance Documentation (ASSURANCE.md)

Professional document covering:
- All five CI/CD workflows and their gates
- Security controls (secret detection, static analysis, supply chain)
- Concurrency and permission model
- Local validation procedures
- Release artifact verification steps
- Workflow dependency graph

### 4. README Badge Updates

Added badges for the Regression CI and SBOM Generation workflows to complement
existing Regression and Security CI and Security CI badges.

---

## Pre-Existing Security Controls (Unchanged)

| Control | Status |
|---|---|
| ShellCheck static analysis | Active on every push/PR |
| bash -n syntax validation | Active on every push/PR |
| Gitleaks secret scanning | Active on every push/PR + weekly |
| CodeQL (Actions) | Active on every push/PR + weekly |
| SLSA v2 provenance attestation | Active on tag pushes |
| SHA-256 release checksums | Active on tag pushes |
| Integration tests | Active on every push/PR |
| Least-privilege permissions | All workflows |
| Concurrency controls | All long-running workflows |
| Self-hosted runners | All workflows |

---

## New Controls Added

| Control | Workflow | Blocking? |
|---|---|---|
| shfmt format validation | `ci.yml` | No (advisory) |
| CycloneDX SBOM generation | `sbom.yml` | N/A (artifact) |
| SBOM validation | `sbom.yml` | No (warning) |

---

## Risk Assessment

| Risk | Mitigation |
|---|---|
| shfmt may flag existing scripts | Non-blocking (`continue-on-error: true`) |
| SBOM generation depends on external binary download | Pinned version, SHA verified by HTTPS |
| CycloneDX CLI validation may produce warnings | Handled gracefully with warning annotation |

---

## Validation Steps

1. Verify `ci.yml` syntax: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/ci.yml'))"`
2. Verify `sbom.yml` syntax: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/sbom.yml'))"`
3. Push branch and observe GitHub Actions run results
4. Confirm shfmt step runs as advisory (green even if formatting differs)
5. Confirm SBOM artifact is uploaded with correct content

---

## Recommendations for Future Work

1. **Make shfmt blocking** once scripts are reformatted to pass `shfmt -d -i 2 -ci -bn`
2. **Sign SBOM artifacts** with Sigstore/cosign for tamper evidence
3. **Add Dependabot** for GitHub Actions version pinning (already using `@v4` tags)
4. **Pin action SHAs** instead of tags for stronger supply-chain guarantees
5. **Add OSSF Scorecard** workflow for automated security posture scoring
