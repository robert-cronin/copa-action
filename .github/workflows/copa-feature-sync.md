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
  create-discussion: # needed to create planning discussion
    title-prefix: "${{ github.workflow }}"
    category: "ideas"
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
    target: "*"
    discussion: true
  noop:
tracker-id: copa-sync-2024
github-token: ${{ secrets.GITHUB_TOKEN }}
engine:
  id: copilot
  model: gpt-5.1-codex
network: defaults
steps:
  - name: Setup BATS
    uses: mig4/setup-bats@af9a00deb21b5d795cabfeaa8d9060410377686d # v1.2.0
    with:
      bats-version: ${{ env.BATS_VERSION }}
tools:
  github:
    toolsets: [default, discussions]
  edit:
  bash: [:*]
  web-fetch:
  agentic-workflows: true
env:
  COPA_REPO: project-copacetic/copacetic
  COPA_STATE_FILE: .github/agents/copa-sync-state.json
  COPA_DISCUSSION_TITLE: "Copa Feature Sync - Release Tracker"
  BATS_VERSION: "1.11.1"
  STATE_TEMPLATE: '{"last_release_tag":"","last_checked":"","notes":[]}'
strict: true
---

# Copa Feature Sync

## Mission

Keep this action and related integrations aligned with the latest Copacetic CLI capabilities while ensuring reviewers see one focused feature change per run.

## Operating Constraints

- Only use public information or files within this repository, `${{ env.COPA_REPO }}`, and any data you create.
- Default to read-only actions unless a safe output provides explicit write access.
- Ship or log exactly one feature gap per run; never bundle multiple backlog items together.
- Link to upstream release artifacts or docs instead of copying them verbatim.
- Use `safe-outputs.missing-tool` (auto-enabled) whenever repo tooling is insufficient.

## Phase Selection

- Run release intelligence (next section) every time so you know what the latest Copacetic tag introduced.
- Search discussions for `${{ env.COPA_DISCUSSION_TITLE }}` (ignore closed threads). If none exist, run **Phase 1** only and exit after creating the plan. If a discussion exists, proceed to **Phase 2**.
- Always keep `${{ env.COPA_STATE_FILE }}` updated with the latest `last_release_tag`, ISO8601 `last_checked`, and a `notes` array where each feature has `feature`, `description`, `files`, and `status` (`pending`, `in-progress`, `done`, `blocked`).

## Release Intelligence (runs every phase)

- Fetch the latest release from `${{ env.COPA_REPO }}` and capture `tag_name`, `html_url`, `body`, and `target_commitish`.
- Bootstrap `${{ env.COPA_STATE_FILE }}` from `${{ env.STATE_TEMPLATE }}` if it does not exist yet.
- If `last_release_tag` already matches the newest release and the backlog contains no `pending` work, emit a noop message that cites the confirming evidence and exit.
- Clone (or fetch) `${{ env.COPA_REPO }}` locally if not present and check out the exact `tag_name` so you can diff files, confirm input defaults, and copy canonical docs straight from that release. Reuse the same checkout directory across runs when possible.
- Parse release notes, the checked-out source tree, and documentation for new CLI flags, environment variables, behaviors, and integration-impacting changes. Add or update backlog entries so each feature includes a short name, description with links, target files (`action.yaml`, `README.md`, `test/test.bats`, etc.), and its current status (start with `pending`).

## Phase 1 – Planning Discussion

Run this phase only when `${{ env.COPA_DISCUSSION_TITLE }}` does not already exist. Once the discussion is created, stop so humans can review.

- Summarize repository testing/documentation posture and highlight the release deltas you just discovered.
- Present a backlog table covering every `pending` feature from the state file, including impact, target files, and open questions.
- Create a discussion body that includes clearly labeled sections: **Plan**, **Execution Checklist**, **Questions / Risks**, **How to Control this Workflow**, and **What Happens Next**. Reuse the control guidance from the Daily Test Improver workflow (commands such as `gh aw disable ...`, `gh aw enable ...`, `gh aw run ... --repeat <n>`, `gh aw logs ...`).
- Use `safe-outputs.create-discussion` to publish the discussion, then store the resulting URL inside `${{ env.COPA_STATE_FILE }}` (e.g., under `discussion_url`).

## Backlog & State Management

- Treat `${{ env.COPA_STATE_FILE }}` as the single source of truth. Keep entries sorted so the most recent release features appear first.
- The state file lives in `${{ env.COPA_STATE_FILE }}` and is committed to the repo with blank defaults so the workflow can run immediately; keep it checked in after every update.
- Ensure every backlog item appears both in the state file and in the discussion checklist so humans can track progress easily.
- When status changes, update the state file and leave a comment on the discussion so reviewers have context.

## Phase 2 – Feature Delivery Loop (single feature per run)

Only execute this phase when the planning discussion already exists.

- Read the state file + discussion checklist to find the highest-priority feature whose status is neither `done` nor `blocked`.
- Mark that feature as `in-progress` inside `${{ env.COPA_STATE_FILE }}` before editing code.
- Implement exclusively that feature. Update `action.yaml`, docs, and tests as required.
- Validate your change (see Implementation Requirements) and capture evidence such as command output or links to release notes.
- Decide which artifact to produce:
  - Draft PR when code or docs change. Include the targeted backlog item, commands executed, validation performed, and remaining TODOs.
  - Issue when manual intervention or other repos block progress. Describe severity in the body even though labels are pre-set.
- Post a concise comment to the planning discussion covering the release tag processed, the feature tackled (and whether it is `done`, `blocked`, or `needs-review`), plus links to PRs/issues and test output.
- Update the state file entry to `done` (or `blocked`) with any relevant notes, commit all changes, and exit without touching additional backlog items.

## Implementation Requirements

- Reflect every behavior change in `README.md` tables/examples and `test/test.bats` (or new tests) with deterministic fixtures.
- Thread new inputs through the Docker invocation in `action.yaml` and ensure defaults are honored.
- Record the commands you ran (especially tests) inside PR descriptions for reproducibility.
- Keep `.github/agents/copa-sync-state.json` valid JSON and check it in with accompanying code changes.
- Run `bats test/test.bats` after edits. If tests fail, fix them, or raise an issue explaining why they could not pass.

## Reporting & Outputs

- **Discussion**: Phase 1 uses `safe-outputs.create-discussion`; Phase 2 uses `safe-outputs.add-comment` to provide run summaries and backlog status.
- **Pull Requests**: Draft PRs must map to exactly one backlog item and include sections for release tag, files touched, validation proof, remaining gaps, and next steps.
- **Issues**: Capture upstream blockers or cross-repo follow-ups, referencing the discussion and backlog entry.
- **Noop**: When no action is required, emit a noop message that cites the release tag and state file hash or discussion comment.

## Exit Criteria

- `${{ env.COPA_STATE_FILE }}` reflects the latest release tag, timestamp, and updated backlog statuses.
- The planning discussion exists and now contains either a refreshed checklist or a new comment from this run.
- Exactly one feature moved forward (to `done` or `blocked`) during Phase 2, or Phase 1 completed the discussion setup.
- Tests were executed (or a logged issue explains why they could not run).

## Fallbacks

- If GitHub API calls fail, retry after 10s, 30s, and 60s. After the third failure, raise an issue documenting the outage.
- When backlog interpretation is unclear, add questions to the discussion and exit so humans can respond.
- Use `safe-outputs.missing-tool` whenever additional tooling or permissions are required.

## Reference Links

- Copacetic CLI repo: [project-copacetic/copacetic](https://github.com/project-copacetic/copacetic)
- Action repo context: [project-copacetic/copa-action](https://github.com/project-copacetic/copa-action)
- Documentation hub: [Copacetic docs](https://project-copacetic.github.io/copacetic/)
