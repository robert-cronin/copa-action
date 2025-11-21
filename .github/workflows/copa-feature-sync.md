---
name: "Copa Feature Sync"
on:
  schedule:
    - cron: "0 6 * * *"
  workflow_dispatch:
  pull_request:
    types: [labeled]
    names: ["test"]
permissions:
  contents: read
  actions: read
  issues: read
  pull-requests: read
  discussions: read
runs-on: ubuntu-latest
timeout-minutes: 30
description: "Continuously align copa-action with the latest Copacetic CLI release."
roles: [admin, maintainer, write]
safe-outputs:
  create-issue:
    title-prefix: "[copa-sync] "
    labels: [automation, needs-triage]
    max: 2
  create-pull-request:
    title-prefix: "[copa-sync] "
    labels: [automation, dependencies]
    draft: true
  add-comment:
    max: 1
  noop:
tracker-id: copa-sync-2024
github-token: ${{ secrets.GITHUB_TOKEN }}
engine:
  id: copilot
  model: gpt-5.1-codex
network: defaults
tools:
  github:
    toolsets: [default, discussions]
  edit:
  bash:
    - "gh release view:*"
    - "gh release list:*"
    - "gh api:*"
    - "gh repo view:*"
    - "gh aw compile"
    - "git status"
    - "git diff"
    - "bats test/test.bats"
    - "sudo apt-get update"
    - "sudo apt-get install -y bats"
  web-fetch:
  agentic-workflows: true
env:
  COPA_REPO: project-copacetic/copacetic
  COPA_STATE_FILE: .github/agents/copa-sync-state.json
  STATE_TEMPLATE: '{"last_release_tag":"","last_checked":"","notes":[]}'
strict: true
---

# Copa Feature Sync

## Mission
Keep this action feature-complete with the most recent Copacetic CLI release so users can adopt new patching capabilities without waiting for manual updates.

## Operating Constraints
- Only use public information or files within this repository and `${{ env.COPA_REPO }}`.
- Prefer code changes + pull requests when you can confidently update the action. Use issues when manual follow-up across other repos is required.
- Avoid duplicating upstream release artifacts; link to them instead.
- If tooling is missing, report it through `safe-outputs.missing-tool` automatically injected by the platform.

## Workflow

### 1. Determine Latest Copacetic Release
1. Query the GitHub Releases API for `${{ env.COPA_REPO }}` and capture:
   - `tag_name`
   - `html_url`
   - `body`
   - `target_commitish`
2. Compare the tag to `${{ env.COPA_STATE_FILE }}`:
   - If the file is absent, create it using `${{ env.STATE_TEMPLATE }}` and treat the latest release as new.
   - If the stored `last_release_tag` matches the latest tag, emit a noop summary via `safe-outputs` and stop.

### 2. Extract Feature Changes
1. Parse the release body in `${{ env.COPA_REPO }}` for:
   - New CLI flags, env vars, subcommands.
   - Behavioral changes that impact patch workflows (timeouts, retries, buildkit behavior, SBOM/output formats, etc.).
2. Produce a short table mapping each feature to the files in this repo that should be audited (e.g., `action.yaml`, `README.md`, `test/test.bats`).

### 3. Evaluate copa-action Parity
For every new or changed feature:
1. Inspect `action.yaml` inputs/outputs and runtime script to verify parity.
2. Update code and docs when gaps are small or well-understood.
   - Example fixes: add a new input, wire it into the docker run command, mirror the flag in tests, update README tables.
   - Keep comments concise and only where logic is non-trivial.
3. When updates span other integrations (Helm chart, operator, Terraform, etc.) or need product decisions, capture them for humans via an issue.

### 4. Implementation Requirements
- When you modify behavior, update the README inputs table and any relevant examples.
- Ensure new inputs have defaults, validation, and are threaded into the container invocation.
- Extend `test/test.bats` (or add new tests) so the change is exercised. Provide deterministic fixtures.
- Add or update `.github/agents/copa-sync-state.json` to record:
  ```json
  {
    "last_release_tag": "vX.Y.Z",
    "last_checked": "2025-11-21T12:00:00Z",
    "notes": [
      {
        "feature": "retry-backoff",
        "status": "implemented",
        "files": ["action.yaml", "README.md"]
      }
    ]
  }
  ```
- Run `bats test/test.bats` after edits. If dependencies (like `bats`) need installation, script it explicitly (e.g., `sudo apt-get update && sudo apt-get install -y bats`). Include the command/output summary in the final report.

### 5. Reporting & Outputs
- **Pull Request**: When code or docs change, prefer a draft PR with a changelog-style description of:
  - release tag processed
  - implemented features (checklist)
  - remaining gaps / required follow-ups
- **Issues**: Use when
  - a feature needs coordination across other repos
  - manual validation is required
  - tooling is missing in this repo
  Tag issues with severity context (critical/high/normal) inside the body, even though labels are preset.
- **Comments / Noop**: If no action is necessary because the repo already matches the release, emit a brief status comment via `safe-outputs.add-comment` or rely on `safe-outputs.noop` with the release tag and evidence links.

### 7. Exit Criteria
- Latest release is documented in `${{ env.COPA_STATE_FILE }}`.
- Tests covering new functionality are updated (or gap recorded with rationale).
- Either a PR or issue conveys next steps. No silent exits.

### 8. Fallbacks
- If GitHub API is unavailable, retry with exponential backoff (sleep 10s, 30s, 60s). After three failures, file an issue noting the outage and skip edits.
- If feature detection is ambiguous, default to creating an issue that links to the specific release section and outlines open questions.

### Reference Links
- Copacetic CLI repo: https://github.com/${{ env.COPA_REPO }}
- Action repo context: https://github.com/project-copacetic/copa-action
- Documentation hub: https://project-copacetic.github.io/copacetic/
