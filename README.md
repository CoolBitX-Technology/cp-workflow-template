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

### `release_pr.yml`

自動維護一個隨時保持最新的 `dev → master` release PR。只有一個 workflow，由 caller 用兩種事件觸發：

| Caller 觸發 | `create_pr` | 效果 |
|---|---|---|
| schedule / `workflow_dispatch` | `true` | 沒有開著的 PR 就開一個，接著填入 body |
| `push` 到 `dev` | `false` | 只更新開著的 PR 的 body，不會開新 PR |

> **支援的合併方式：** body 是從 `git log` 的 commit 標題抓 PR 編號，支援 **merge commit**（`Merge pull request #N from ...`）和 **squash**（標題結尾為 `(#N)`，例如 dependabot 自動合併的 PR）。**rebase** 合併不會留下 PR 編號，不會出現在 body 裡。
>
> **只保留一個改寫 body 的 workflow：** 這個 workflow 每次都會整份覆蓋 release PR 的 body。採用前請移除 caller repo 裡其他會改寫 release PR body 的 workflow（例如 `update_pr_description.yml`），否則兩者會互相覆蓋。

**行為：**
- `head` 沒有 `base` 缺少的 commit → skip
- 沒有開著的 PR，且 `create_pr: false` → skip
- 沒有開著的 PR，且 `create_pr: true` → 開新 PR，標題為 `release: <head_branch> -> <base_branch> (YYYY-MM-DD)`，日期為台北時間（reviewer 請由 caller repo 的 `assign_reviewers.yml` 在 PR opened 時指定）
- 最後一定會用目前已 merge 的 PR 列表覆蓋 PR body

同一個 repo + branch 組合的 run 會依序執行（`concurrency` 排隊、不取消），排程和 push 同時觸發時不會重複開 PR，較舊的 run 也不會蓋掉較新的 body。GitHub 每個 group 只保留一個等待中的 run，極少數情況下等待中的排程 run 會被後來的 push run 取代，這時 PR 要等下次排程或手動觸發才會開。

**Inputs：**

| Input | 型別 | 預設值 | 說明 |
|---|---|---|---|
| `head_branch` | string | `dev` | 來源 branch |
| `base_branch` | string | `master` | 目標 branch |
| `create_pr` | boolean | `true` | 沒有開著的 PR 時是否開新 PR。caller 傳 `${{ github.event_name != 'push' }}`，push 觸發時就只更新 body |

**必要 secret：** `PERSONAL_ACCESS_TOKEN`。請使用只授權給目標 repo 的 **fine-grained PAT**，權限為 `Pull requests: write` 和 `Contents: read`。不能用 `GITHUB_TOKEN`，因為它開的 PR 不會觸發其他 workflow。

**PR body 格式：**
```markdown
## Included PRs

- #123
- #456

---
*Auto-generated by release_pr*
```

每一行只寫 PR 編號，GitHub 會在畫面上自動展開成「圖示 + PR 標題」，標題也會跟著 PR 改名更新。還沒有任何已合併的 PR 時，列表位置會顯示 `_No merged PRs yet._`。

#### Caller 範例

在 caller repo 新增 **`.github/workflows/release_pr.yml`**：

```yaml
name: Release PR

on:
  workflow_dispatch:
  schedule:
    - cron: '0 2 * * 1'      # 每週一台北時間 10:00（UTC+8）
  push:
    branches:
      - dev

jobs:
  release-pr:
    uses: CoolBitX-Technology/cp-workflow-template/.github/workflows/release_pr.yml@master
    with:
      head_branch: dev
      base_branch: master
      create_pr: ${{ github.event_name != 'push' }}   # 只有排程 / 手動觸發會開 PR
    secrets:
      PERSONAL_ACCESS_TOKEN: ${{ secrets.PERSONAL_ACCESS_TOKEN }}
```

> `schedule` 只會在 caller repo 的 **default branch** 觸發。caller workflow 要先合進 default branch，排程才會開始跑。

---

## Standalone Workflows

| File | Trigger | Purpose |
|---|---|---|
| `push-action.yml` | `push` / `workflow_dispatch` | Example: composes build → push → terraform → notify |
| `pr-action.yml` | `pull_request` | Example: Docker-based unit test + Slack notification |
| `update_pr_description.yml` | `pull_request` → `master` | Auto-updates PR description |
| `delete-failure-job.yml` | `workflow_dispatch` | Manually delete a failed k8s CronJob by namespace + name |
