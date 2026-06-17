# cp-workflow-template

Centralized GitHub Actions workflows for cp-\* repositories.

All reusable workflows are called via `uses: CoolBitX-Technology/cp-workflow-template/.github/workflows/<file>@master`.

---

## Reusable Workflows

### `assign_reviewers.yml`

Assigns individual reviewers to PRs. Reviewer list is maintained here — other repos only need a caller workflow.

```yaml
jobs:
  assign:
    uses: CoolBitX-Technology/cp-workflow-template/.github/workflows/assign_reviewers.yml@master
```

---

### `build_and_test_image.yml`

Builds a Docker image and runs tests inside the container.

```yaml
jobs:
  build:
    uses: CoolBitX-Technology/cp-workflow-template/.github/workflows/build_and_test_image.yml@master
    secrets:
      PROD_CP_GCP_PROJECT_ID: ${{ secrets.PROD_CP_GCP_PROJECT_ID }}
      DEV_CP_GCP_PROJECT_ID: ${{ secrets.DEV_CP_GCP_PROJECT_ID }}
      DEVOPS_READ_PACKAGE_TOKEN: ${{ secrets.DEVOPS_READ_PACKAGE_TOKEN }}
```

---

### `unit_test_than_build_image.yml`

Runs unit tests with Node.js, then builds a Docker image and uploads it as an artifact. Prefer this over `build_and_test_image.yml` for repos with a proper unit test setup.

Inputs:
- `node-version` (optional, default: `22.16.0`)

```yaml
jobs:
  build:
    uses: CoolBitX-Technology/cp-workflow-template/.github/workflows/unit_test_than_build_image.yml@master
    with:
      node-version: '22.16.0'
    secrets:
      PROD_CP_GCP_PROJECT_ID: ${{ secrets.PROD_CP_GCP_PROJECT_ID }}
      DEV_CP_GCP_PROJECT_ID: ${{ secrets.DEV_CP_GCP_PROJECT_ID }}
      DEVOPS_READ_PACKAGE_TOKEN: ${{ secrets.DEVOPS_READ_PACKAGE_TOKEN }}
```

---

### `push_and_rotate_image.yml`

Pushes the built image to GCR and rotates it in the cluster. Requires artifact from `build_and_test_image.yml` or `unit_test_than_build_image.yml`.

```yaml
jobs:
  push:
    needs: build
    uses: CoolBitX-Technology/cp-workflow-template/.github/workflows/push_and_rotate_image.yml@master
    secrets:
      PROD_CP_GCP_PROJECT_ID: ${{ secrets.PROD_CP_GCP_PROJECT_ID }}
      DEV_CP_GCP_PROJECT_ID: ${{ secrets.DEV_CP_GCP_PROJECT_ID }}
      PROD_CP_GCP_SVC_TERRAFORM: ${{ secrets.PROD_CP_GCP_SVC_TERRAFORM }}
      DEV_CP_GCP_SVC_TERRAFORM: ${{ secrets.DEV_CP_GCP_SVC_TERRAFORM }}
      PROD_CP_GCP_SVC_GITHUB_RUNNER_KEY_FILE: ${{ secrets.PROD_CP_GCP_SVC_GITHUB_RUNNER_KEY_FILE }}
      DEV_CP_GCP_SVC_GITHUB_RUNNER_KEY_FILE: ${{ secrets.DEV_CP_GCP_SVC_GITHUB_RUNNER_KEY_FILE }}
```

---

### `plan_terraform.yml` / `apply_terraform.yml`

Plan and apply Terraform changes. `apply_terraform.yml` typically runs after both `push_and_rotate_image` and `plan_terraform` succeed.

```yaml
jobs:
  plan:
    needs: build
    uses: CoolBitX-Technology/cp-workflow-template/.github/workflows/plan_terraform.yml@master
    secrets:
      PROD_CP_GCP_PROJECT_ID: ${{ secrets.PROD_CP_GCP_PROJECT_ID }}
      DEV_CP_GCP_PROJECT_ID: ${{ secrets.DEV_CP_GCP_PROJECT_ID }}
      DEVOPS_ACCESS_REPO_TOKEN: ${{ secrets.DEVOPS_ACCESS_REPO_TOKEN }}
      PROD_CP_GCP_SVC_GITHUB_RUNNER_KEY_FILE: ${{ secrets.PROD_CP_GCP_SVC_GITHUB_RUNNER_KEY_FILE }}
      DEV_CP_GCP_SVC_GITHUB_RUNNER_KEY_FILE: ${{ secrets.DEV_CP_GCP_SVC_GITHUB_RUNNER_KEY_FILE }}

  apply:
    needs: [push, plan]
    uses: CoolBitX-Technology/cp-workflow-template/.github/workflows/apply_terraform.yml@master
    secrets: ...
```

---

### `run_integration_test.yml`

Runs integration tests and notifies a Slack user on completion.

Required inputs: `OWNER_SLACK_ID`, `SERVICE`.

---

### `notify_slack.yml`

Sends a Slack notification to the crypto platform channel.

```yaml
jobs:
  notify:
    needs: apply
    uses: CoolBitX-Technology/cp-workflow-template/.github/workflows/notify_slack.yml@master
    secrets:
      CP_SLACK_CHANNEL_RG_CRYPTO: ${{ secrets.CP_SLACK_CHANNEL_RG_CRYPTO }}
```

---

### `build_and_publish_npm_package.yml`

Builds and publishes a package to the GitHub Packages (private `@coolbitx-technology` registry).

---

### `build_and_publish_official_public_npm_package.yml`

Builds and publishes a package to the public npmjs.org registry under `@coolbitx` scope.

---

### `dependabot_auto_merge.yml`

Analyzes Dependabot PRs and automatically approves and merges patch/minor bumps when CI is green. Skips merge (with an explanatory comment) for major bumps or CI failures.

**How it works:**
1. A script classifies each package bump as patch / minor / major from the PR title or body
2. Claude reads the full Dependabot changelog and greps the codebase to assess risk
3. Claude applies the label, approves, and merges — or leaves a comment explaining why it skipped

**Labels applied (auto-created on first run):**
- `ai-approved` — approved and queued for auto-merge
- `ai-skipped-major` — major bump or high-risk signal; needs manual review
- `ai-skipped-ci-failed` — CI failed; needs manual review

**Required org-level secrets** (set once, apply to all caller repos):
- `ANTHROPIC_API_KEY` — Anthropic API key (standard repo secret, not Dependabot secrets namespace)
- `GH_AUTO_MERGE_TOKEN` — PAT with `pull-requests: write` and `contents: write` scopes

**Required repo setting:** Enable auto-merge in each caller repo under Settings → General → Allow auto-merge.

**Caller workflow** (add `.github/workflows/dependabot_auto_merge.yml` to each repo):

```yaml
name: Dependabot Auto-merge

on:
  workflow_run:
    workflows: ["Push to branches action"]
    types: [completed]
    branches:
      - 'dependabot/**'

jobs:
  auto-merge:
    if: >-
      github.event.workflow_run.event == 'pull_request' &&
      github.event.workflow_run.pull_requests[0].number != 0
    uses: CoolBitX-Technology/cp-workflow-template/.github/workflows/dependabot_auto_merge.yml@master
    with:
      pr_number: ${{ github.event.workflow_run.pull_requests[0].number }}
      ci_conclusion: ${{ github.event.workflow_run.conclusion }}
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      GH_AUTO_MERGE_TOKEN: ${{ secrets.GH_AUTO_MERGE_TOKEN }}
```

---

## Standalone Workflows

| File | Trigger | Purpose |
|---|---|---|
| `push-action.yml` | `push` / `workflow_dispatch` | Example: composes build → push → terraform → notify |
| `pr-action.yml` | `pull_request` | Example: Docker-based unit test + Slack notification |
| `update_pr_description.yml` | `pull_request` → `master` | Auto-updates PR description |
| `delete-failure-job.yml` | `workflow_dispatch` | Manually delete a failed k8s CronJob by namespace + name |
