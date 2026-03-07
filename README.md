# quantman888/.github

`quantman888/.github` 是 `quantman888` 组织的 GitHub 配置仓，不承载业务代码。

它只负责三类事情：

- 提供组织级 GitHub Actions workflow templates，供新仓库从 Actions UI 直接选用
- 维护新仓接入、升级、审计这三类组织级文档入口
- 固化一套长期可维护的 workflow 治理基线，避免各仓库各自复制、各自漂移

它不负责运行时控制逻辑。真正执行 Docker 发布、分支同步、runner 探测、GitHub App 写回、批量更新 PR 的地方仍然是 `quantman888/workflow-reusable`。

## 仓库分工

- `quantman888/.github`
  - 模板入口
  - 新仓 bootstrap
  - 接入说明、升级规范、组织审计
- `quantman888/workflow-reusable`
  - 可复用工作流与控制面实现
  - GitHub App token minting
  - 私有仓 `GH_APP_*` 下发
  - Docker publish/promote
  - fork sync / branch sync PR
  - reusable workflow ref 批量升级 PR

## 文档入口

- [新仓库接入清单](docs/new-repo-onboarding-checklist.md)
- [workflow 治理发布与升级规范](docs/workflow-governance-release-process.md)
- [组织基线审计报告（2026-03-07）](docs/org-workflow-baseline-audit-2026-03-07.md)

## 当前受控 control-plane pin

当前模板统一 pin 到：

- `quantman888/workflow-reusable@120185f677ab0a42d93e1289bcd2002b88f17d89`

规则：

- 不允许把模板改成 `@main` 或 `@master`
- 模板仓只更新“已验证通过”的受控 SHA
- `workflow-reusable` 升级后，通过 caller 仓内的 `reusable-workflow-update-pr` 开 PR 做统一 bump

## 模板目录

- `workflow-templates/docker-publish-governed.yml`
  - Docker 构建/发布入口模板
  - 内置 runner probe
  - 必须替换仓库级测试步骤
- `workflow-templates/docker-promote-governed.yml`
  - 基于 digest 的 release promote 模板
- `workflow-templates/reusable-workflow-update-pr.yml`
  - 升级中央 reusable ref 的 PR 模板
  - 默认一次更新该仓 `.github/workflows/` 下所有指向 `quantman888/workflow-reusable` 的 central refs
- `workflow-templates/workflow-ref-policy.yml`
  - `uses:` 引用 pin 策略模板
- `workflow-templates/branch-sync-main-to-docker-pr.yml`
  - `main -> docker` 的自动同步 PR 模板
- `workflow-templates/fork-sync-upstream-main.yml`
  - fork 仓默认分支同步模板

## 新仓推荐接入顺序

Docker 服务仓：

1. `workflow-ref-policy.yml`
2. `reusable-workflow-update-pr.yml`
3. `docker-publish-governed.yml`
4. `docker-promote-governed.yml`
5. 把模板中的 `TODO replace with repository test command` 换成仓库自己的测试命令

Fork 维护仓：

1. `fork-sync-upstream-main.yml`
2. `workflow-ref-policy.yml`（推荐）
3. `reusable-workflow-update-pr.yml`（推荐）

存在长期 `docker` 分支的 branch-split 仓：

1. `branch-sync-main-to-docker-pr.yml`
2. `workflow-ref-policy.yml`
3. `reusable-workflow-update-pr.yml`
4. 如该仓还负责镜像发布，再补 Docker 模板

## 组织级 secrets / vars 基线

所有接入中央治理的仓库，至少应准备：

- `GH_APP_ID`（secret）
- `GH_APP_PRIVATE_KEY`（secret）
- `RUNNER_PROBE_TOKEN`（secret）
- `RUNNER_SELF_HOSTED_LABELS`（variable，可选）
- `RUNNER_GITHUB_HOSTED_LABEL`（variable，可选）

Docker 仓通常还需要：

- `NEXUS_REGISTRY`（variable）
- `IMAGE_NAME`（variable）
- `NEXUS_USERNAME`（secret）
- `NEXUS_PASSWORD`（secret）
- `DOCKERFILE_PATH`（variable，可选）
- `BUILD_CONTEXT`（variable，可选）
- `TARGET_PLATFORMS`（variable，可选）
- `BUILD_ARGS`（variable，可选）

注意：当前模板统一从 `secrets.GH_APP_ID` 读取 GitHub App ID，不要只配 organization variable 而不配 secret。

## Private 仓库接入规则

private 仓库不能把长期 PAT 当成通用方案。组织基线是：

- 顶层长期凭据只保留 `GH_APP_ID` + `GH_APP_PRIVATE_KEY`
- private 仓库所需的同名 secrets 通过 `quantman888/workflow-reusable` 内的 `.github/workflows/github-app-secret-sync.controlplane.yml` 自动下发
- 不维护硬编码仓库名单，新仓库在后续同步周期内自动纳入

## 当前组织覆盖概览（2026-03-07）

- 已完成 Docker 基线覆盖：`mcphub-gateway`、`mcp-didatodolist`
- 已完成 branch-sync / fork-sync / policy / updater 治理闭环：`ksrpc`
- 已完成非 Docker caller 的 policy / updater / runner-fallback 治理基线：`infra`
- 定制治理仓，不应直接硬套 Docker 基线：`cortex`
- 暂未进入这些模板的适用场景：`platform`、`dagster`
- 控制面/模板基础设施仓：`workflow-reusable`、`.github`

详细结论见审计报告：[`docs/org-workflow-baseline-audit-2026-03-07.md`](docs/org-workflow-baseline-audit-2026-03-07.md)

## 运行规则

- 模板是 bootstrap 入口，不是最终真相来源
- 业务仓一旦接入，实际生效的是该仓自己的 workflow 文件和 `quantman888/workflow-reusable` 的可复用工作流
- 仓库级差异应只保留业务测试、镜像参数、分支模型等必要差异；不要把中央控制逻辑复制回业务仓长期分叉维护
