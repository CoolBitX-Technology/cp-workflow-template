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

## Standalone Workflows

| File | Trigger | Purpose |
|---|---|---|
| `push-action.yml` | `push` / `workflow_dispatch` | Example: composes build → push → terraform → notify |
| `pr-action.yml` | `pull_request` | Example: Docker-based unit test + Slack notification |
| `update_pr_description.yml` | `pull_request` → `master` | Auto-updates PR description |
| `delete-failure-job.yml` | `workflow_dispatch` | Manually delete a failed k8s CronJob by namespace + name |
