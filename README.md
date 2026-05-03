# catchai security scan — GitHub Action

Run the [catchai](https://github.com/MihailMihaylov97/catchai) multi-layer
vulnerability scanner on every push or pull request. Findings appear in
the repository's **Security** tab (via SARIF upload to GitHub Code
Scanning) and as a summary comment on each PR.

| Layer | What it covers |
|------:|----------------|
| L1 | Dependency CVEs (Trivy + Grype + OSV.dev), supply-chain risk |
| L2 | Static analysis — SAST patterns, framework-aware sources |
| L3 | Hardcoded secrets (entropy + regex + context) |
| L4 | Config / IaC — Dockerfile, Terraform, k8s, CI/CD |
| L5 | Inter-procedural taint analysis (call graph + flow) |
| L6 | Business-logic / authorization model (entry-point coverage) |
| L7 | LLM semantic review — files mode (per file) or flows mode (per attack-surface, with cross-flow chain detection) |

## Quick start

Create `.github/workflows/security.yml`:

```yaml
name: security

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read
  security-events: write   # for SARIF upload to Code Scanning
  pull-requests: write     # for the PR summary comment

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: MihailMihaylov97/catchai-action@v1
```

That's it. Push the workflow, open a PR. The Action will:

1. Install catchai (single curl).
2. Run a deterministic scan (L1-L6, no LLM calls).
3. Upload SARIF to the Security tab.
4. Post a summary comment on the PR.
5. Fail the workflow step if any high or critical findings exist.

## With Layer 7 (LLM semantic review)

L7 catches multi-step exploits and cross-function reasoning gaps that
L1-L6 cannot — at the cost of LLM API spend.

```yaml
- uses: MihailMihaylov97/catchai-action@v1
  with:
    semantic: 'true'
    semantic-mode: 'flows'             # 'flows' or 'files'
    semantic-top-n: '20'
    semantic-chain-review: 'true'      # cross-flow exploit chain detection
    anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

`anthropic-api-key` must be set as a repository or organisation secret.
The Action reads it from `${{ secrets.ANTHROPIC_API_KEY }}` and passes
it via the `ANTHROPIC_API_KEY` env var to `catchai`. Never logged.

Per-scan LLM cost depends on repo size and `semantic-top-n`. Typical
range: $0.20-$2 per scan on a medium-sized repo. Bound it via the
`cost_budget_tokens` knob in a vendored `prompts.yaml` if you need a
hard cap.

## All inputs

| Input | Default | What it does |
|---|---|---|
| `path` | `.` | Directory to scan |
| `semantic` | `false` | Enable Layer 7 LLM review |
| `semantic-mode` | `files` | `files` (per-file deep review) or `flows` (per-entry-point) |
| `semantic-top-n` | `20` | Max files / flows the LLM stage reviews |
| `semantic-chain-review` | `false` | Cross-flow exploit chain detection (flows mode only) |
| `semantic-verbose` | `false` | Print one progress line per LLM call |
| `anthropic-api-key` | — | Required when `semantic=true`. Pass via secret. |
| `upload-sarif` | `true` | Upload findings to GitHub Code Scanning |
| `fail-on-severity` | `high` | Threshold to fail the step (`critical|high|medium|low|info|none`) |
| `comment-on-pr` | `true` | Post / update a summary comment on the PR conversation |

## All outputs

The Action exposes scan totals as step outputs so downstream jobs can
react to them:

| Output | Example |
|---|---|
| `total-findings` | `42` |
| `critical-findings` | `3` |
| `high-findings` | `7` |
| `sarif-path` | `/.../reports/repo/scan_2026-05-03.sarif` |
| `json-path` | `/.../reports/repo/scan_2026-05-03.json` |

```yaml
- id: scan
  uses: MihailMihaylov97/catchai-action@v1

- name: React to high-severity findings
  if: ${{ steps.scan.outputs.high-findings != '0' }}
  run: |
    echo "Posting incident to PagerDuty..."
```

## Permissions reference

| Permission | Why |
|---|---|
| `contents: read` | Standard checkout. Always needed. |
| `security-events: write` | SARIF upload to Code Scanning. Skip with `upload-sarif: false`. |
| `pull-requests: write` | Posting / updating the PR summary comment. Skip with `comment-on-pr: false`. |

## Code Scanning on private repos

Uploading SARIF to GitHub Code Scanning on a **private** repo requires
GitHub Advanced Security. If your org doesn't have GHAS, set
`upload-sarif: false` — the scan still runs, the report is uploaded as
a workflow artifact (`catchai-report`), and the PR comment shows
finding counts.

## How it works (under the hood)

The Action is a thin composite wrapper. It:

1. Installs the catchai binary by curling
   `https://raw.githubusercontent.com/MihailMihaylov97/catchai-dist/main/install.sh`,
   which downloads the platform-appropriate pre-built wheel.
2. Runs `catchai scan ...` with arguments derived from the inputs.
3. Parses the canonical scan JSON output for severity counts.
4. Uploads the SARIF to GitHub Code Scanning.
5. Posts / updates a PR summary comment using a marker line so a fresh
   push to the PR branch updates the existing comment instead of
   stacking a new one.

The Action source (this `action.yml`) is permissively licensed (see
[`LICENSE`](LICENSE)). The catchai binary itself is governed by its
own terms — see [`LICENSE_NOTE.md`](LICENSE_NOTE.md).

## Versioning

Pin to a major (`@v1`) for automatic minor/patch updates with a stable
Action interface. Pin to a specific tag (`@v1.0.0`) if you want
hash-equivalent reproducibility.

## Issues

Action plumbing bugs (workflow YAML, SARIF upload, PR comment shape)
go in this repo's [issues](https://github.com/MihailMihaylov97/catchai-action/issues).

Scan-result quality issues (missed finding, false positive, wrong
severity) belong in the upstream
[catchai repo](https://github.com/MihailMihaylov97/catchai/issues).
