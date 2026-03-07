# quantman888 组织 workflow 基线审计报告（2026-03-07）

审计时间：2026-03-07  
审计范围：`quantman888` 组织下非 archived 仓库 9 个

- `cortex`
- `dagster`
- `.github`
- `infra`
- `ksrpc`
- `mcp-didatodolist`
- `mcphub-gateway`
- `platform`
- `workflow-reusable`

## 1. 审计口径

本次审计只判断“这些仓库是否已经具备与组织模板相匹配的基础 workflow 基线”，不要求所有仓库都硬套同一套模板。

基线按仓库类型区分：

| 基线类型 | 最低建议配置 |
| --- | --- |
| Docker 服务仓 | `workflow-ref-policy` / `reusable-workflow-update-pr` / `docker-publish` / `docker-promote` |
| Fork 仓 | `fork-sync-upstream-main`，必要时补 `workflow-ref-policy` 与 `reusable-workflow-update-pr` |
| Branch-split 仓 | `branch-sync-main-to-docker-pr` / `workflow-ref-policy` / `reusable-workflow-update-pr` |
| 定制治理仓 | 不机械套模板，但应说明为何不适用并保留自己的验证闭环 |
| 基础设施仓 | `.github` 与 `workflow-reusable` 自身不按 caller 模板基线考核 |

## 2. 总览结论

| 结论分类 | 仓库 |
| --- | --- |
| 已完整覆盖适用基线 | `mcphub-gateway`、`mcp-didatodolist` |
| 已纳入 branch-sync / fork-sync / policy / updater 组织治理闭环 | `ksrpc` |
| 已纳入非 Docker caller 的 policy / updater / runner-fallback 治理基线 | `infra` |
| 定制治理仓，模板部分适用 | `cortex` |
| 当前无 workflow，暂待业务类型明确 | `platform`、`dagster` |
| 组织基础设施仓，不按 caller 基线考核 | `.github`、`workflow-reusable` |

## 3. 仓库逐项审计

### 3.1 `mcphub-gateway`

类型：Docker 服务仓  
当前 workflow：

- `docker-promote.yml`
- `docker-publish.yml`
- `reusable-workflow-update-pr.yml`
- `workflow-ref-policy.yml`

结论：已完整覆盖 Docker 基线。

说明：

- 已采用中央 `runner-fallback`、`docker-publish.reusable`、`docker-promote.reusable`、`reusable-workflow-update-pr.reusable`
- 已具备策略校验、升级 PR、publish、promote 四件套
- 当前是组织内最适合作为 Docker 基线 canary 的样本仓之一

已验证通过的真实 run：

- `workflow-ref-policy`：run `22796139078`
- `Docker Publish`：run `22796139086`

### 3.2 `mcp-didatodolist`

类型：Docker 服务仓  
当前 workflow：

- `docker-promote.yml`
- `docker-publish.yml`
- `reusable-workflow-update-pr.yml`
- `workflow-policy-check.yml`

结论：已完整覆盖 Docker 基线。

说明：

- `workflow-policy-check.yml` 在职责上等价于组织模板里的 `workflow-ref-policy.yml`
- 已具备策略校验、升级 PR、publish、promote 四件套
- 命名仍有历史差异，但治理能力已经齐备

已验证通过的真实 run：

- `workflow-policy-check`：run `22796138802`
- `Docker Publish`：run `22796138809`

### 3.3 `ksrpc`

类型：Branch-split + Docker + fork-sync 复合仓  
默认分支：`ops/sync`  
当前 workflow：

- `docker-publish.yml`
- `docker-release.yml`
- `main-to-docker-pr.yml`
- `sync-upstream.yml`

结论：已纳入组织治理闭环，但 Docker 发布/发布版 promote 仍保留仓内实现。

说明：

- `sync-upstream.yml` 已通过中央 `fork-sync.reusable.yml` 做 upstream sync
- `main-to-docker-pr.yml` 已通过中央 `branch-sync-pr.reusable.yml` 做 `main -> docker` 开 PR
- `sync-upstream.yml` 与 `main-to-docker-pr.yml` 已统一升级到当前 blessed SHA
- 已补 `workflow-ref-policy.yml`
- 已补 `reusable-workflow-update-pr.yml`
- `reusable-workflow-update-pr.yml` 默认可一次升级仓内全部 central reusable refs，适配 `ksrpc` 这种多 path 调用仓
- `docker-publish.yml` / `docker-release.yml` 仍保留较多仓库内自定义逻辑，没有强行改造成模板化 Docker 基线，这是有意保留的仓库特化

后续建议：

1. 继续评估是否将 `docker-publish.yml` / `docker-release.yml` 逐步收敛到组织模板或中央 reusable 的标准调用面
2. 保持 `ops/sync` 作为默认分支承载自动化，避免上游同步分支污染

### 3.4 `cortex`

类型：定制治理仓  
当前 workflow：

- `cortex-sync-skills.reusable.yml`
- `cortex-sync-skills.yml`
- `cortex-sync.reusable.yml`
- `cortex-sync.yml`
- `cortex-validate.reusable.yml`
- `cortex-validate.yml`

结论：定制治理仓，不应硬套 Docker 模板基线。

说明：

- `cortex` 的工作流目标不是 Docker 发布，而是 release sync / skills sync / validate 这一套定制控制面
- 已切入 GitHub App 写回治理路线，但不属于标准 Docker caller 模板适用对象
- 可按需要借用组织级 pin/policy 规则，但不建议强行接入 Docker publish/promote 模板

已验证通过的真实 run：

- `cortex-sync-release`：run `22796260768`
- `cortex-sync-skills`：run `22796284463`
- `cortex-validate`：run `22796315564`

### 3.5 `infra`

类型：基础设施仓 / 局部治理接入  
当前 workflow：

- `commit-message-guard.yml`

结论：已纳入非 Docker caller 的治理基线。

说明：

- 当前仍只有 commit message guard 这一个业务 workflow，但已经纳入统一治理闭环
- 已补 `workflow-ref-policy.yml`
- 已补 `reusable-workflow-update-pr.yml`
- `commit-message-guard.yml` 已升级到当前 blessed SHA 的 `runner-fallback.reusable.yml`

后续建议：

1. 现阶段无需补 Docker/fork/branch-sync 模板
2. 如后续引入更多中央 reusable，直接复用现有 updater/policy 基线即可

### 3.6 `platform`

类型：待明确业务类型  
当前 workflow：无

结论：暂不适用当前模板基线。

说明：

- 目前没有现成 workflow，无法判断该仓最终会落到 Docker / fork / branch-split / custom 哪一类
- 应在仓库职责明确后，再按接入清单选择模板

### 3.7 `dagster`

类型：待明确业务类型  
当前 workflow：无

结论：暂不适用当前模板基线。

说明：

- 与 `platform` 类似，目前没有足够事实判断应接入哪类模板
- 等业务形态明确后再接入，不建议预先乱铺模板

### 3.8 `workflow-reusable`

类型：组织控制面仓  
当前 workflow：

- `branch-sync-pr.reusable.yml`
- `docker-promote.reusable.yml`
- `docker-publish.reusable.yml`
- `fork-sync.reusable.yml`
- `github-app-secret-sync.controlplane.yml`
- `release-dispatch-consumers.yml`
- `release-publish.yml`
- `reusable-workflow-update-pr.reusable.yml`
- `runner-fallback.reusable.yml`
- `workflow-reference-policy.yml`

结论：这是中央控制面仓，不按 caller 模板基线考核。

说明：

- 它是 `.github` 模板仓的运行时实现层
- 它必须保持 release / policy / control plane 的可用性与可升级性
- 审计重点不是“是否接入模板”，而是“是否继续作为唯一中央实现来源”

### 3.9 `.github`

类型：组织模板仓  
当前 workflow：无 caller workflow 需求

结论：这是组织模板入口仓，不按 caller 模板基线考核。

说明：

- 它负责 `workflow-templates/` 与组织文档
- 它不应承载业务发布逻辑
- 它当前已提供 Docker、fork sync、branch sync、policy、upgrade PR 五类模板

## 4. 缺口与优先级

### P1：待业务形态明确后再接入

- `platform`
- `dagster`

## 5. 审计结论

截至 2026-03-07，`quantman888` 组织的 workflow 治理状态可以概括为：

- Docker 基线已经在 `mcphub-gateway` 与 `mcp-didatodolist` 形成可复用样板
- `ksrpc` 已完成 branch-sync / fork-sync / policy / updater 的组织级治理闭环，剩余差异主要是仓库特化的 Docker 发布实现
- `cortex` 属于定制治理仓，应保持单独治理路径，不应被误纳入 Docker 模板基线
- `infra` 已进入非 Docker caller 的治理闭环，后续若继续接中央 reusable 不需要再补基线文件
- `.github` 与 `workflow-reusable` 的职责边界已经清晰：前者负责模板入口，后者负责运行时 control plane

下一步最有价值的动作不是继续造新模板，而是根据业务形态决定 `platform` / `dagster` 未来各自该接哪类模板。
