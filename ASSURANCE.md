# Assurance Documentation — browserless-tooling

This document describes the CI/CD gates, security scanning, supply-chain
controls, and local validation procedures for the browserless-tooling
repository.

---

## CI/CD Pipeline Overview

The repository maintains five GitHub Actions workflows that collectively
enforce code quality, security posture, and supply-chain integrity.

### 1. Regression CI (`ci.yml`)

**Triggers:** push to `main`, pull requests, manual dispatch

Runs two parallel batches on self-hosted runners:

| Batch | Checks |
|---|---|
| `static` | `bash -n` syntax validation, ShellCheck lint, shfmt format check (advisory) |
| `integration` | Full end-to-end provisioning: Browserless bring-up, metrics endpoint verification, smoke-test screenshot validation (>1 KB), MCP wrapper deployment, `/healthz` assertion |

The shfmt step runs with `continue-on-error: true` and produces warnings
rather than blocking merges, allowing incremental adoption of consistent
formatting.

### 2. Regression and Security (`regression-security.yml`)

**Triggers:** push to all branches (except `gh-pages`), pull requests

Multi-language regression runner with automatic project-type detection.
For this shell-only repository, the language-specific steps are skipped.
Includes a Gitleaks secret scan via the official GitHub Action.

### 3. Security CI (`security.yml`)

**Triggers:** push to `main`, pull requests, weekly schedule (Monday 04:30 UTC), manual dispatch

Two parallel security batches plus a dedicated CodeQL job:

| Batch | Checks |
|---|---|
| `secrets` | Gitleaks v8.24.3 full-history secret detection with redacted output |
| `static` | `bash -n` syntax validation, ShellCheck static analysis |
| `codeql` | GitHub CodeQL analysis for Actions workflow security |

### 4. Release (`release.yml`)

**Triggers:** push of `v*` tags

Builds a release tarball containing the setup scripts, README, and LICENSE.
Generates SHA-256 checksums and attests build provenance using the official
`actions/attest-build-provenance@v2` action (SLSA v2 compatible).

Published releases include:
- `browserless-tooling-<version>.tar.gz`
- `browserless-tooling-<version>.tar.gz.sha256`
- SLSA provenance attestation

### 5. SBOM Generation (`sbom.yml`)

**Triggers:** push to `main`, weekly schedule (Monday 06:00 UTC), manual dispatch

Generates a CycloneDX 1.5 SBOM listing all script components, their SHA-256
hashes, and runtime dependencies (Docker, bash, firewall-cmd, openssl, curl,
container images). The SBOM is validated against the CycloneDX specification
and uploaded as a build artifact with 90-day retention.

---

## Security Controls

### Secret Detection

- **Gitleaks** runs on every push and pull request across the full git history.
- Secrets are redacted in CI output to prevent accidental exposure.
- Weekly scheduled scans catch any secrets introduced between CI runs.

### Static Analysis

- **ShellCheck** enforces shell scripting best practices with targeted
  exclusions (`SC2155` for declare-and-assign, `SC1091` for sourced files).
- **bash -n** validates script syntax without execution.
- **CodeQL** analyzes GitHub Actions workflows for injection vulnerabilities,
  insecure patterns, and misconfigurations.

### Supply Chain

- **SLSA v2 provenance attestations** are generated for every release artifact,
  providing verifiable build provenance.
- **CycloneDX SBOM** documents all components and runtime dependencies.
- **SHA-256 checksums** accompany every release tarball.
- All CI runs on **self-hosted runners** with controlled environments.

### Runtime Security (Scripts)

- Tokens generated with `openssl rand -hex 24` at provision time.
- `.env` files created with `0640` permissions.
- Firewalld rules restrict service exposure to trusted zones.
- No secrets stored in script source code.

---

## Concurrency and Permissions

All workflows follow the principle of least privilege:

| Workflow | Permissions | Concurrency Group |
|---|---|---|
| `ci.yml` | `contents: read` | `browserless-regression-${{ github.ref }}` |
| `regression-security.yml` | `contents: read` | None (runs on all branches) |
| `security.yml` | `contents: read` (+ `security-events: write` for CodeQL) | `browserless-security-${{ github.ref }}` |
| `release.yml` | `contents: write`, `id-token: write`, `attestations: write` | None (tag-triggered only) |
| `sbom.yml` | `contents: read` | `browserless-sbom-${{ github.ref }}` |

All concurrency groups use `cancel-in-progress: true` to avoid queuing
redundant runs.

---

## Local Validation

To reproduce CI checks locally before pushing:

```bash
# Syntax validation
bash -n aibrowse-setup.sh browsewrap-setup.sh

# ShellCheck lint
shellcheck -x --exclude=SC2155,SC1091 aibrowse-setup.sh browsewrap-setup.sh

# shfmt format check (advisory)
shfmt -d -i 2 -ci -bn aibrowse-setup.sh browsewrap-setup.sh

# Secret scan (requires gitleaks installed)
gitleaks detect --source . --redact --verbose

# Full integration test (requires root, Docker, firewalld)
INSTANCE="local-test-$(date +%s)"
sudo bash ./aibrowse-setup.sh "${INSTANCE}"
# Verify: curl metrics endpoint, check smoke-test.png > 1 KB
sudo bash ./browsewrap-setup.sh "${INSTANCE}"
# Verify: /healthz returns {"status":"ok"}
# Cleanup:
sudo docker rm -f "browserless-${INSTANCE}" "browsewrap-${INSTANCE}"
sudo rm -rf "/docker/aibrowse/${INSTANCE}" "/docker/browsewrap/${INSTANCE}"
```

---

## Validating Release Artifacts

After downloading a release:

```bash
# Verify checksum
sha256sum -c browserless-tooling-v*.tar.gz.sha256

# Verify SLSA provenance (requires gh CLI with attestation extension)
gh attestation verify browserless-tooling-v*.tar.gz \
  --repo jalsarraf0/browserless-tooling
```

---

## Workflow Dependency Graph

```text
push/PR to main ──┬── ci.yml (static + integration)
                   ├── regression-security.yml (multi-lang + gitleaks)
                   ├── security.yml (gitleaks + shellcheck + CodeQL)
                   └── sbom.yml (CycloneDX SBOM)

tag push (v*) ──── release.yml (bundle + SHA-256 + SLSA attestation)

weekly schedule ─┬─ security.yml (scheduled secret scan)
                 └─ sbom.yml (scheduled SBOM refresh)
```
