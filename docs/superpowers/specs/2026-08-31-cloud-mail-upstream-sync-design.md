# cloud-mail 上游同步与 Cloudflare 自动部署设计

- 日期：2026-08-31
- 仓库：`ZTTYBB/cloud-mail`
- 上游：`maillab/cloud-mail`
- 目标：定时同步上游代码，并在同步产生变更后部署到现有 Cloudflare Worker

## 1. 已验证的现状

- fork 的 `main` 与上游 `main` 当前都指向 `57a4b1fa8445ac4929fea184bea78a539c315465`。
- fork 原有提交已保存在 `backup-before-upstream-sync-20260831`，不会因本方案被删除。
- 当前仓库已有 `.github/workflows/deploy-cloudflare.yml`，使用仓库 Secrets/Variables 中的 Cloudflare 和应用配置。
- 现有部署目标为 Cloudflare Worker `cloud-mail`，公开域名为 `545352.xyz`。
- 直接使用 GitHub Actions 的 `GITHUB_TOKEN` 推送同步结果时，不能依赖另一个 `push` 事件工作流自动启动，因此同步工作流必须显式调用部署工作流。

## 2. 目标与非目标

### 目标

1. 每天北京时间 10:00（UTC 02:00）从 `maillab/cloud-mail:main` 获取最新代码。
2. 支持通过 `workflow_dispatch` 手动执行同步。
3. 上游代码作为项目代码的唯一来源；fork 自己的应用代码提交不保留。
4. 上游有变化时，把同步结果推送到 fork 的 `main`，并部署该次同步提交。
5. 上游无变化时不产生空提交，也不触发部署。
6. 保留本 fork 的两个自动化控制文件，使后续同步不会删除自动化本身。

### 非目标

- 不修改、迁移或清理 Cloudflare 的 Worker、D1、KV、域名和现有数据。
- 不在工作流中写入、输出或替换 Cloudflare Secrets/Variables。
- 不恢复 fork 原有应用代码；原提交仅作为备份保留。
- 不将上游代码改造成长期维护的手工合并分支。

## 3. 方案与选择

### 采用：上游代码快照 + 保留自动化工作流 + 可复用部署

同步工作流先把当前两个自动化文件复制到临时目录，再把 Git 工作树和索引切换为 `upstream/main` 的快照，最后恢复两个自动化文件并提交。这样项目的应用代码、依赖和配置以 upstream 为准，同时不会因为上游没有这些自动化文件而丢失定时同步能力。

同步完成后，同一个工作流以同步提交的 SHA 调用部署工作流。部署工作流同时支持原有的 `push`、手动触发和 `workflow_call`，并通过可选的 `ref` 输入明确检出刚同步的提交。

### 不采用的方案

- 只做 `git reset --hard upstream/main`：会删除 fork 中的同步工作流，下一次同步失效。
- 只依赖同步提交触发 `push` 部署：GitHub 对由 `GITHUB_TOKEN` 产生的事件有触发限制，部署结果不可靠。
- 在同步工作流内复制整套部署命令：会造成两套部署逻辑漂移，后续维护和故障排查成本更高。

## 4. 工作流数据流

```text
定时器 / 手动触发
        |
        v
sync-upstream.yml
  - checkout fork/main
  - fetch maillab/cloud-mail/main
  - 临时保存两个自动化文件
  - 将工作树切换为 upstream 快照
  - 恢复两个自动化文件
  - 有差异才 commit + push
        |
        | changed=true，传递同步提交 SHA
        v
deploy-cloudflare.yml（workflow_call）
  - checkout 同步提交 SHA
  - 使用现有 Secrets/Variables
  - 构建并部署 Worker
  - 初始化现有 D1/KV 绑定
  - 访问现有自定义域名完成初始化检查
```

同步失败时不调用部署；部署失败时同步提交仍可在 `main` 上追溯，GitHub Actions 会报告失败，便于重新运行部署。

## 5. 文件边界

### 新增：`.github/workflows/sync-upstream.yml`

- 触发器：`schedule: 0 2 * * *`、`workflow_dispatch`。
- 权限：仅授予 `contents: write`，用于把同步提交推送到当前仓库。
- 远端：只读获取 `https://github.com/maillab/cloud-mail.git` 的 `main`。
- 变更检测：通过暂存区差异判断是否需要提交；无差异时输出 `changed=false`。
- 部署条件：仅当 `changed=true` 时调用部署工作流。
- 不使用强制推送，避免覆盖同步运行期间出现的远端新提交。

### 修改：`.github/workflows/deploy-cloudflare.yml`

- 增加 `workflow_call` 触发器及可选的 `ref` 输入。
- 检出 `inputs.ref`，未传入时退回当前事件的 `github.ref`。
- 保留现有构建、环境检查、KV/D1 设置、部署和初始化步骤。
- 保留现有 `push` 和 `workflow_dispatch` 行为。

## 6. 凭据与并发边界

- 同步工作流不读取 Cloudflare 凭据。
- 调用部署工作流时使用 `secrets: inherit`，继续使用仓库现有的 Secrets/Variables。
- 同步工作流设置固定并发组，避免两个同步同时重写 `main`。
- 工作流日志不得打印 JWT、API Token、管理员邮箱或其他敏感变量。

## 7. 验证标准

1. 两个 YAML 文件能被 GitHub Actions 接受，且同步工作流能被手动触发。
2. 手动同步时：
   - 上游无新代码则无空提交、无部署；
   - 有新代码则 fork `main` 产生一条同步提交；
   - 除两个保留的自动化文件外，项目代码与上游 `main` 的文件快照一致。
3. 同一运行中的部署任务检出同步提交 SHA，而不是旧的 `github.ref`。
4. GitHub Actions 部署成功，并且 `545352.xyz` 可访问。
5. Cloudflare Worker `cloud-mail` 的现有绑定和数据仍然可用。

## 8. 已知风险

- 上游若改变数据库结构，代码同步成功不代表现有 D1 数据迁移一定兼容；部署工作流本身不自动推测或执行未知迁移。
- 上游若改变部署所需的环境变量，现有 Secrets/Variables 可能不足，Actions 会在环境检查阶段失败。
- 定时任务依赖 GitHub Actions 的计划调度，可能出现分钟级延迟；手动触发可用于立即同步。
- 设计文档和其他非上游文件不在自动化保留清单中，后续同步可能按上游快照将其清理；这不影响两个自动化工作流的运行。
