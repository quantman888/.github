# 新仓库接入清单

本文定义 `quantman888` 组织中新仓库接入 GitHub Actions 治理基线的标准步骤。

目标：

- 新仓可以直接复用组织模板，而不是复制别的仓库现成 workflow
- 顶层长期凭据统一收敛到 GitHub App：`GH_APP_ID` + `GH_APP_PRIVATE_KEY`
- 后续升级通过中央 control plane 驱动，不引入 PAT、硬编码仓库名单或漂移分叉

## 1. 先判断仓库类型

| 仓库类型 | 使用场景 | 必选模板 | 可选模板 | 不应直接套用 |
| --- | --- | --- | --- | --- |
| Docker 服务仓 | 仓库负责镜像构建/发布/发布版 promote | `workflow-ref-policy.yml`、`reusable-workflow-update-pr.yml`、`docker-publish-governed.yml`、`docker-promote-governed.yml` | `branch-sync-main-to-docker-pr.yml`（若存在长期 `docker` 分支） | `fork-sync-upstream-main.yml` |
| Fork 维护仓 | 需要周期性把上游仓库同步到本仓默认分支 | `fork-sync-upstream-main.yml` | `workflow-ref-policy.yml`、`reusable-workflow-update-pr.yml` | Docker 模板（除非该仓也负责镜像） |
| Branch-split 仓 | 默认分支负责同步，`docker` 分支负责构建/定制 | `branch-sync-main-to-docker-pr.yml`、`workflow-ref-policy.yml`、`reusable-workflow-update-pr.yml` | Docker 模板、fork sync 模板 | 无 |
| 定制治理仓 | 例如 `cortex` 这类自身已经有专门控制面和非 Docker 主流程 | 无固定模板基线，先按 control plane 设计 | 可局部借用 `workflow-ref-policy.yml` | 不要机械套用 Docker 基线 |

## 2. Docker 服务仓接入步骤

1. 在目标仓库的 GitHub Actions UI 中，从 `quantman888` 组织模板选择：
   - `workflow-ref-policy.yml`
   - `reusable-workflow-update-pr.yml`
   - `docker-publish-governed.yml`
   - `docker-promote-governed.yml`
2. 按仓库真实情况替换 `docker-publish-governed.yml` 和 `docker-promote-governed.yml` 里的测试步骤。
3. 配置仓库变量与密钥。
4. 先跑一次 `workflow-ref-policy`，确保所有 `uses:` 引用已被 pin。
5. 再跑 Docker publish 的 PR dry-run / main push / manual run。
6. 最后用一次手动 promote 或 release event 验证 release 通道。

## 3. Fork 仓接入步骤

1. 新增 `fork-sync-upstream-main.yml`。
2. 将 `upstream_repo`、`upstream_branch` 改成真实上游。
3. 如仓库还会继续引用中央 reusable，补 `workflow-ref-policy.yml`。
4. 如希望以后统一升级 pin，补 `reusable-workflow-update-pr.yml`。
   该 workflow 默认会一次升级当前仓 `.github/workflows/` 下所有指向 `quantman888/workflow-reusable` 的 central refs。
5. 手动触发一次 `fork-sync-upstream-main`，确认 GitHub App 写回成功。

## 4. Branch-split 仓接入步骤

1. 确认仓库确实存在长期 `docker` 分支。
2. 新增 `branch-sync-main-to-docker-pr.yml`。
3. 把 `workflow-ref-policy.yml` 与 `reusable-workflow-update-pr.yml` 一起接入。
   其中 updater workflow 默认会一次升级仓内所有 central reusable refs，不需要按单个 path 分开维护。
4. 如果 `docker` 分支还负责镜像发布，再追加 Docker publish/promote 模板。
5. 手动触发一次 branch sync，确认能自动开 PR 或复用已有 PR。

## 5. 必备变量与密钥

### 5.1 所有中央治理仓通用

| 名称 | 类型 | 是否必须 | 说明 |
| --- | --- | --- | --- |
| `GH_APP_ID` | secret | 是 | GitHub App ID。当前模板统一从 `secrets.GH_APP_ID` 读取。 |
| `GH_APP_PRIVATE_KEY` | secret | 是 | GitHub App 私钥。 |
| `RUNNER_PROBE_TOKEN` | secret | 建议 | `runner-fallback.reusable.yml` 进行 runner 探测时使用。 |
| `RUNNER_SELF_HOSTED_LABELS` | variable | 否 | self-hosted labels，默认 `self-hosted,linux,x64`。 |
| `RUNNER_GITHUB_HOSTED_LABEL` | variable | 否 | fallback label，默认 `ubuntu-latest`。 |

### 5.2 Docker 仓附加项

| 名称 | 类型 | 是否必须 | 说明 |
| --- | --- | --- | --- |
| `NEXUS_REGISTRY` | variable | 是 | 镜像仓库地址。 |
| `IMAGE_NAME` | variable | 是 | 镜像名。 |
| `NEXUS_USERNAME` | secret | 是 | 镜像仓库登录用户名。 |
| `NEXUS_PASSWORD` | secret | 是 | 镜像仓库登录密码。 |
| `DOCKERFILE_PATH` | variable | 否 | 默认 `Dockerfile`。 |
| `BUILD_CONTEXT` | variable | 否 | 默认 `.`。 |
| `TARGET_PLATFORMS` | variable | 否 | 默认 `linux/amd64`。 |
| `BUILD_ARGS` | variable | 否 | 多行 `KEY=VALUE`。 |

### 5.3 Public / Private 规则

- public 仓可直接使用 organization secrets。
- private 仓应通过 `quantman888/workflow-reusable` 的 `github-app-secret-sync.controlplane.yml` 自动下发 `GH_APP_*`。
- 不要为某个单仓单独发长期 PAT 作为“临时兜底”。

## 6. 必改项

以下内容如果不改，模板仓接入后大概率会直接失败：

- `docker-publish-governed.yml` 中的 `TODO replace with repository test command`
- `docker-promote-governed.yml` 中的 `TODO replace with repository test command`
- fork 模板中的 `upstream-owner/upstream-repo`
- branch-sync 模板里的分支名（若仓库不是默认 `docker` 分支模型）
- Docker 仓的 `NEXUS_*` / `IMAGE_NAME` / `BUILD_CONTEXT` 等变量

## 7. 验收清单

### 7.1 Docker 服务仓

- `workflow-ref-policy` 通过
- PR 场景下 `Docker Publish / PR Dry Run` 通过
- 默认分支 push 或手动触发下 `Docker Publish` 通过
- release 或手动 promote 后 `Docker Promote` 通过
- 镜像标签、digest、发布说明符合仓库预期

### 7.2 Fork / Branch-sync 仓

- `fork-sync-upstream-main` 或 `branch-sync-main-to-docker-pr` 能成功获取 GitHub App token
- 有差异时能正确 fast-forward 或开 PR
- 无差异时不会重复制造噪声 PR
- workflow 里的 `uses:` 引用满足 pin 策略

## 8. 常见误配

- 只配置了 organization variable `GH_APP_ID`，没有配置同名 secret
- private 仓没有先同步 `GH_APP_*`，导致 GitHub App token step 直接失败
- 模板接入后忘了替换 `TODO` 测试步骤
- `uses:` 仍然写成 `@main`
- `reusable-workflow-update-pr.yml` 没有接入，导致以后每次升级都得手工改 SHA
- branch-sync 模板被接到并不存在 `docker` 分支的仓库
- fork-sync 模板里 `upstream_repo` 还保留占位值

## 9. 不建议的做法

- 不要使用长期 PAT 作为组织级通用凭据
- 不要把 `workflow-reusable` 的逻辑复制进每个仓长期维护
- 不要为新仓手工粘贴旧仓 workflow 后长期不回收
- 不要允许 `uses:` 引用漂移到 `@main` 或 `@master`

## 10. 推荐接入闭环

新仓建立后，推荐按下面的闭环执行：

1. 通过 `quantman888/.github` 模板接入初始 workflow。
2. 完成仓库级变量、密钥、测试命令替换。
3. 验收通过后，把该仓纳入日常 `reusable-workflow-update-pr` 与 private secret sync 的治理闭环。
4. 后续中央升级时，不再手工逐仓改 workflow，而是用中央升级流程统一推动。
