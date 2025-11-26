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
  COPA_MIN_VERSION: "v0.6.0"
  BATS_VERSION: "1.11.1"
  STATE_TEMPLATE: '{"releases":{},"last_checked":"","min_version":"v0.6.0"}'
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

- Run release intelligence (next section) every time so you know which Copacetic releases exist and what each introduced.
- Process releases sequentially starting from `${{ env.COPA_MIN_VERSION }}` (v0.6.0). Identify the oldest incomplete release (one with `pending` features or no discussion yet).
- Construct a discussion title using the pattern `Copa Feature Sync - <tag>` (e.g., `Copa Feature Sync - v0.6.0`).
- Search discussions for that title (ignore closed threads). If none exist for the target release, run **Phase 1** only and exit after creating the plan. If a discussion exists, proceed to **Phase 2**.
- Always keep `${{ env.COPA_STATE_FILE }}` updated with a `releases` object keyed by tag, each containing `discussion_url`, `last_checked` (ISO8601), `previous_tag` (for diff reference), and a `features` array where each item has `feature`, `description`, `introduced_in`, `files`, and `status` (`pending`, `in-progress`, `done`, `blocked`).

## Release Intelligence (runs every phase)

- Fetch all releases from `${{ env.COPA_REPO }}` starting from `${{ env.COPA_MIN_VERSION }}` (v0.6.0) and capture `tag_name`, `html_url`, `body`, `published_at`, and `target_commitish` for each.
- Bootstrap `${{ env.COPA_STATE_FILE }}` from `${{ env.STATE_TEMPLATE }}` if it does not exist yet.
- Process releases in ascending order (oldest first, starting with v0.6.0). Find the first release where `releases[tag_name]` either does not exist or has `pending` features remaining.
- If all releases from v0.6.0 onward are complete (all features `done` or `blocked`), emit a noop message citing the latest tag and exit.
- Clone (or fetch) `${{ env.COPA_REPO }}` locally if not present. Check out the exact `tag_name` being processed. Reuse the same checkout directory across runs when possible.

### Differential Feature Detection

When analyzing a release, identify only the **net-new features introduced in that specific version**:

1. **For v0.6.0 (baseline)**: Compare against the action's current state. Any CLI flags, environment variables, or behaviors in v0.6.0 that the action does not yet support are features for v0.6.0's backlog.
2. **For subsequent releases (v0.7.0+)**: Compare against the previous release's source tree using `git diff <prev_tag>..<current_tag>`. Only changes that appear in this diff belong to the current release's backlog.
3. **Never duplicate features**: If feature X exists in both v0.6.0 and v0.7.0, it belongs only to v0.6.0's backlog. Feature Y introduced in v0.7.0 belongs only to v0.7.0.
4. **Parse release notes for hints**: Release notes typically highlight what's new in each version; use them to validate differential detection.

Add or update feature entries under `releases[tag_name].features` so each includes:
- `feature`: Short unique name
- `description`: What changed, with links to upstream commits/docs
- `introduced_in`: The tag where this feature first appeared (for auditing)
- `files`: Target files in this repo (`action.yaml`, `README.md`, `test/test.bats`, etc.)
- `status`: `pending`, `in-progress`, `done`, or `blocked`

## Phase 1 – Planning Discussion (per release)

Run this phase only when a discussion titled `Copa Feature Sync - <tag>` does not already exist for the latest release. Once the discussion is created, stop so humans can review.

- Summarize repository testing/documentation posture and highlight the release deltas discovered for this specific version.
- Present a backlog table covering every `pending` feature from `releases[tag_name].features`, including impact, target files, and open questions.
- Create a discussion body that includes clearly labeled sections: **Release Overview** (tag, date, link), **Plan**, **Execution Checklist**, **Questions / Risks**, **How to Control this Workflow**, and **What Happens Next**. Reuse the control guidance from the Daily Test Improver workflow (commands such as `gh aw disable ...`, `gh aw enable ...`, `gh aw run ... --repeat <n>`, `gh aw logs ...`).
- Use `safe-outputs.create-discussion` to publish the discussion with title `Copa Feature Sync - <tag>`, then store the resulting URL inside `${{ env.COPA_STATE_FILE }}` under `releases[tag_name].discussion_url`.

## Backlog & State Management

- Treat `${{ env.COPA_STATE_FILE }}` as the single source of truth. The `releases` object is keyed by tag name, with each release containing its own `discussion_url`, `last_checked`, `previous_tag`, and `features` array.
- Each feature entry includes `introduced_in` to record which release first added it—this prevents duplication across releases.
- The state file lives in `${{ env.COPA_STATE_FILE }}` and is committed to the repo with blank defaults so the workflow can run immediately; keep it checked in after every update.
- Ensure every feature item appears both in the state file under its release and in that release's discussion checklist so humans can track progress easily.
- When status changes, update the state file and leave a comment on the release-specific discussion so reviewers have context.
- When all features for a release reach `done` or `blocked`, consider the release complete and note this in the discussion.

## Phase 2 – Feature Delivery Loop (single feature per run)

Only execute this phase when the planning discussion for the latest release already exists.

- Read the state file under `releases[tag_name].features` and the corresponding discussion checklist to find the highest-priority feature whose status is neither `done` nor `blocked`.
- Mark that feature as `in-progress` inside `${{ env.COPA_STATE_FILE }}` before editing code.
- Implement exclusively that feature. Update `action.yaml`, docs, and tests as required.
- Validate your change (see Implementation Requirements) and capture evidence such as command output or links to release notes.
- Decide which artifact to produce:
  - Draft PR when code or docs change. Include the targeted feature item, the release tag, commands executed, validation performed, and remaining TODOs.
  - Issue when manual intervention or other repos block progress. Describe severity in the body even though labels are pre-set.
- Post a concise comment to the release-specific planning discussion covering the release tag processed, the feature tackled (and whether it is `done`, `blocked`, or `needs-review`), plus links to PRs/issues and test output.
- Update the state file entry under `releases[tag_name].features` to `done` (or `blocked`) with any relevant notes, commit all changes, and exit without touching additional features.

## Implementation Requirements

- Reflect every behavior change in `README.md` tables/examples and `test/test.bats` (or new tests) with deterministic fixtures.
- Thread new inputs through the Docker invocation in `action.yaml` and ensure defaults are honored.
- Record the commands you ran (especially tests) inside PR descriptions for reproducibility.
- Keep `.github/agents/copa-sync-state.json` valid JSON and check it in with accompanying code changes.
- Run `bats test/test.bats` after edits. If tests fail, fix them, or raise an issue explaining why they could not pass.

## Reporting & Outputs

- **Discussion**: Phase 1 uses `safe-outputs.create-discussion` to create a release-specific discussion (titled `Copa Feature Sync - <tag>`); Phase 2 uses `safe-outputs.add-comment` to provide run summaries and feature status updates on that discussion.
- **Pull Requests**: Draft PRs must map to exactly one feature from one release and include sections for release tag, files touched, validation proof, remaining gaps, and next steps.
- **Issues**: Capture upstream blockers or cross-repo follow-ups, referencing the release-specific discussion and feature entry.
- **Noop**: When no action is required, emit a noop message that cites the release tag and confirms all features are complete or that no new releases exist.

## Exit Criteria

- `${{ env.COPA_STATE_FILE }}` reflects the latest release under `releases[tag_name]` with updated timestamp and feature statuses.
- A discussion titled `Copa Feature Sync - <tag>` exists for the latest release and contains either the initial checklist (Phase 1) or a new comment from this run (Phase 2).
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
