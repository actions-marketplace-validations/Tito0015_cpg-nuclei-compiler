# Upstream nuclei-templates refactoring pipeline

This document describes how to use **cpg-nuclei-compiler** as a CI gate when refactoring templates in a **fork** of [projectdiscovery/nuclei-templates](https://github.com/projectdiscovery/nuclei-templates). It does not open or update upstream pull requests automatically.

## Non-destructive SOP

Some community templates mutate application state (settings, watches, sessions). Before scanning a real deployment:

1. **Snapshot** sensitive state (for example `GET /settings` on Changedetection.io).
2. **Execute** Nuclei only in an isolated lab (Docker fixtures or dedicated test hosts).
3. **Restore** state from the snapshot after the scan (or use in-template teardown).

The vendored template `templates/CVE-2024-51483.yaml` in this repo follows that pattern. A worked upstream example is [nuclei-templates PR #17007](https://github.com/projectdiscovery/nuclei-templates/pull/17007) (CVE-2024-51483).

See also the **Non-destructive Nuclei SOP** section in the root [README](../../README.md).

## Prerequisites

- A fork of `projectdiscovery/nuclei-templates` with your refactored YAML.
- A **lab target URL** (local fixture, docker-compose stack, or CI service)—never a production host.
- GitHub Actions runner: `ubuntu-latest` with Docker available (same as this repo’s dogfood workflow).

## Copy-paste workflow (fork only)

Add this file to your **nuclei-templates fork** as `.github/workflows/cpg-harness-verify.yml` (not to cpg-nuclei-compiler). Adjust `template` and `target` for the rule under review.

```yaml
name: CPG harness verify (template under review)

on:
  pull_request:
    paths:
      - '**.yaml'
      - '**.yml'
  workflow_dispatch:
    inputs:
      template:
        description: Path to template YAML (repo-relative)
        required: true
      target:
        description: Lab target URL
        required: true
        default: 'http://127.0.0.1:5000'

jobs:
  harness:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Optional: start your vulnerable fixture here (docker compose, etc.)
      # - run: docker compose -f lab/docker-compose.yml up -d

      - name: Verify template with CPG Nuclei harness
        uses: Tito0015/cpg-nuclei-compiler@v1
        with:
          template: ${{ github.event.inputs.template || 'http/cves/2024/CVE-2024-51483.yaml' }}
          target: ${{ github.event.inputs.target || 'http://127.0.0.1:5000' }}
```

**Notes:**

- `template` is resolved relative to the **caller** workspace (`github.workspace`), so paths match your fork’s tree.
- Pin `uses: Tito0015/cpg-nuclei-compiler@v1` only after a `v1` tag exists on the action repository.
- For local-only smoke tests in **this** compiler repo, use `uses: ./` as in [`.github/workflows/verify-harness.yml`](../../.github/workflows/verify-harness.yml).

## What this pipeline does not do

- It does **not** replace `nuclei -validate` or ProjectDiscovery’s own template CI.
- It does **not** create or update upstream PRs; maintainers still open PRs manually after local/CI verification.
- It does **not** run against arbitrary internet targets from shared CI without explicit lab setup.

## Related reading

- [NUCLEI_BIBLE.md](./NUCLEI_BIBLE.md) — DSL and metadata conventions in this project.
- [README — Primary Use Case: Refactoring Destructive Templates](../../README.md)
