# cloud-mail 上游同步与 Cloudflare 自动部署 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为 `ZTTYBB/cloud-mail` 增加每日上游同步，并在同步产生变更时把该提交部署到现有 Cloudflare Worker `cloud-mail`。

**Architecture:** `sync-upstream.yml` 使用 GitHub Actions 定时获取 `maillab/cloud-mail:main`，将仓库工作树切换为上游快照，同时临时保存并恢复本仓库的两个自动化工作流。同步提交推送后，同一个工作流通过 `workflow_call` 调用 `deploy-cloudflare.yml`，并传入刚生成的提交 SHA，确保部署内容与同步内容一致。

**Tech Stack:** GitHub Actions, Git, pnpm, Wrangler, Cloudflare Workers/D1/KV, GitHub repository Secrets/Variables

---

## 文件结构

- Create: `.github/workflows/sync-upstream.yml` — 每日/手动上游同步、变更检测、同步提交推送，以及对部署工作流的调用。
- Modify: `.github/workflows/deploy-cloudflare.yml` — 保留现有 push 和手动部署行为，新增 workflow_call 和可选检出 ref。
- Create: `docs/superpowers/plans/2026-08-31-cloud-mail-upstream-sync.md` — 本次实施计划；后续上游快照同步时允许被清理，不参与运行时逻辑。
- Existing reference: `docs/superpowers/specs/2026-08-31-cloud-mail-upstream-sync-design.md` — 已批准设计；本次同步工作流不把该文档加入永久保留清单。

### Task 1: Make the Cloudflare deployment workflow reusable

**Files:**
- Modify: `.github/workflows/deploy-cloudflare.yml`

- [ ] **Step 1: Add the reusable-workflow trigger and optional ref input**

Replace only the current trigger block at the top of `.github/workflows/deploy-cloudflare.yml`:

```yaml
on:
  push:
    branches: [ main ]
    paths:
      - "mail-worker/**"
      - "mail-vue/**"
  workflow_dispatch:
```

with this exact block:

```yaml
on:
  push:
    branches: [ main ]
    paths:
      - "mail-worker/**"
      - "mail-vue/**"
  workflow_dispatch:
    inputs:
      ref:
        description: "要部署的分支、标签或提交 SHA；留空则使用当前事件 ref"
        required: false
        type: string
  workflow_call:
    inputs:
      ref:
        description: "要部署的分支、标签或提交 SHA；留空则使用调用方 ref"
        required: false
        type: string
```

- [ ] **Step 2: Make checkout use the requested ref**

In the existing checkout step, keep `uses: actions/checkout@v4` and add the following `with` block:

```yaml
      - name: 🚚 检出代码仓库 / Checkout repository
        uses: actions/checkout@v4
        with:
          ref: ${{ inputs.ref || github.ref }}
```

Do not change the existing environment variables, build commands, Wrangler commands, database setup, deployment command, initialization check, or workflow-run cleanup step.

- [ ] **Step 3: Check the reusable interface statically**

Confirm that the workflow contains exactly one top-level `on` mapping with `push`, `workflow_dispatch`, and `workflow_call`; `workflow_call.inputs.ref.type` is `string); and the checkout expression is `${{ inputs.ref || github.ref }}`. Confirm that no Cloudflare value or secret was added to the file.

Expected result: the deployment workflow remains valid for push/manual runs and can be called by another workflow with a commit SHA.

### Task 2: Add the upstream synchronization workflow

**Files:**
- Create: `.github/workflows/sync-upstream.yml`

- [ ] **Step 1: Add the complete synchronization workflow**

Create `.github/workflows/sync-upstream.yml` with exactly this content:

```yaml
name: 🔄 Sync upstream and deploy

on:
  schedule:
    - cron: "0 2 * * *"
  workflow_dispatch:

concurrency:
  group: sync-upstream-main
  cancel-in-progress: false

permissions:
  contents: write

jobs:
  sync:
    name: 🔄 Sync from upstream
    runs-on: ubuntu-latest
    outputs:
      changed: ${{ steps.sync.outputs.changed }}
      commit_sha: ${{ steps.sync.outputs.commit_sha }}
    steps:
      - name: 🚚 检出 fork / Checkout fork
        uses: actions/checkout@v4
        with:
          ref: main
          fetch-depth: 0

      - name: 🔄 同步上游代码 / Sync upstream tree
        id: sync
        shell: bash
        run: |
          set -euo pipefail

          preserve_dir="$(mktemp -d)"
          trap 'rm -rf -- "$preserve_dir"' EXIT

          mkdir -p "$preserve_dir/.github/workflows"
          cp .github/workflows/deploy-cloudflare.yml "$preserve_dir/.github/workflows/deploy-cloudflare.yml"
          cp .github/workflows/sync-upstream.yml "$preserve_dir/.github/workflows/sync-upstream.yml"

          git remote add upstream https://github.com/maillab/cloud-mail.git
          git fetch --no-tags upstream main
          upstream_sha="$(git rev-parse upstream/main)"

          git read-tree -mu upstream/main
          mkdir -p .github/workflows
          cp "$preserve_dir/.github/workflows/deploy-cloudflare.yml" ".github/workflows/deploy-cloudflare.yml"
          cp "$preserve_dir/.github/workflows/sync-upstream.yml" ".github/workflows/sync-upstream.yml"
          git add --all

          if git diff --cached --quiet; then
            echo "changed=false" >> "$GITHUB_OUTPUT"
            echo "commit_sha=$(git rev-parse HEAD)" >> "$GITHUB_OUTPUT"
            echo "upstream_sha=$upstream_sha" >> "$GITHUB_OUTPUT"
            echo "✅ Already synchronized with upstream."
            exit 0
          fi

          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git commit -m "chore: sync from maillab/cloud-mail"
          git push origin HEAD:main

          commit_sha="$(git rev-parse HEAD)"
          echo "changed=true" >> "$GITHUB_OUTPUT"
          echo "commit_sha=$commit_sha" >> "$GITHUB_OUTPUT"
          echo "upstream_sha=$upstream_sha" >> "$GITHUB_OUTPUT"
          echo "✅ Synced upstream at $upstream_sha."

      - name: ✅ 输出同步结果 / Report sync result
        if: always()
        run: |
          echo "changed=${{ steps.sync.outputs.changed }}"
          echo "commit_sha=${{ steps.sync.outputs.commit_sha }}"
          echo "upstream_sha=${{ steps.sync.outputs.upstream_sha }}"

  deploy:
    name: 🚀 Deploy synchronized commit
    needs: sync
    if: needs.sync.outputs.changed == 'true'
    uses: ./.github/workflows/deploy-cloudflare.yml
    with:
      ref: ${{ needs.sync.outputs.commit_sha }}
    secrets: inherit
```

- [ ] **Step 2: Check synchronization safety**

Confirm the workflow has `schedule.cron: "0 2 * * *"`, `workflow_dispatch`, `permissions.contents: write`, and a non-canceling concurrency group. Confirm that the only preserved files are:

```text
.github/workflows/deploy-cloudflare.yml
.github/workflows/sync-upstream.yml
```

Confirm that the code uses `git read-tree -mu upstream/main`, stages with `git add --all`, skips an empty commit when `git diff --cached --quiet` succeeds, and uses `git push origin HEAD:main` without force push.

Expected result: the app tree comes from upstream; the automation control files survive future syncs; a failed push stops before deployment; and an unchanged upstream does not deploy.

### Task 3: Commit the workflow changes

**Files:**
- Modify: `.github/workflows/deploy-cloudflare.yml`
- Create: `.github/workflows/sync-upstream.yml`

- [ ] **Step 1: Push both workflow files as one commit**

Use one GitHub commit on `ZTTYBB/cloud-mail:main` with message:

```text
ci: sync upstream and deploy to Cloudflare
```

The commit must contain only the two workflow changes. Do not modify Cloudflare resources or repository Secrets/Variables.

- [ ] **Step 2: Re-read the committed files**

Read both files from `refs/heads/main` and verify:

1. `sync-upstream.yml` references `maillab/cloud-mail.git`, the `02:00 UTC` schedule, `workflow_dispatch`, and the reusable deployment call.
2. `deploy-cloudflare.yml` contains `workflow_call` and checks out `${{ inputs.ref || github.ref }}`.
3. The current `main` commit contains both files and the workflow file contents were not truncated.

Expected result: both files are present at the same commit and the committed text matches Tasks 1 and 2.

### Task 4: Run the workflow and verify the resulting deployment

**Files:**
- No repository file changes.

- [ ] **Step 1: Manually start `sync-upstream.yml`**

Use the GitHub Actions workflow page for `ZTTYBB/cloud-mail` to run `Sync upstream and deploy` on `main`. Leave the optional deployment ref empty because this is a synchronization test.

Expected result: one sync run is created.

- [ ] **Step 2: Verify the sync result**

Wait for the run to finish and inspect its jobs.

For the current baseline where fork `main` already matches upstream except for the design/plan documents, the expected behavior is:

- the sync job succeeds;
- it removes non-upstream documentation files and preserves the two workflow files;
- it creates one synchronization commit because the design/plan documents are not in upstream;
- the deploy job is called with that synchronization commit SHA;
- the called deployment job completes successfully.

If the run reports no change instead, verify that `main` already contains only upstream files plus the two workflows and record the no-op result.

- [ ] **Step 3: Verify GitHub and Cloudflare state**

Confirm all of the following:

- `ZTTYBB/cloud-mail:main` points to the synchronization commit reported by `sync-upstream.yml`.
- The upstream application tree at `maillab/cloud-mail:main` matches fork `main` except for the two intentionally preserved workflow files.
- The deploy job checked out the reported synchronization SHA.
- The Cloudflare deployment job succeeded using existing repository configuration.
- `https://545352.xyz` returns a successful application response.
- No Cloudflare Worker, D1, KV, domain, secret, or variable was deleted or recreated.

Expected result: future upstream changes at 02:00 UTC are automatically synchronized and deployed; manual runs can reproduce the same path.

### Self-review checklist

- [ ] Every requirement in `docs/superpowers/specs/2026-08-31-cloud-mail-upstream-sync-design.md` is covered by Tasks 1–4.
- [ ] No placeholder such as `TBD`, `TODO`, or “implement later” exists in this plan.
- [ ] The `changed` and `commit_sha` output names match between the sync job and deploy job.
- [ ] The deployment ref passed by the caller is the same SHA that the sync job pushed.
- [ ] The plan does not require a Cloudflare credential or a force push.
