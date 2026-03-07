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
| 部分等价覆盖，仍建议补齐组织模板基线 | `ksrpc` |
| 定制治理仓，模板部分适用 | `cortex` |
| 暂不适用当前模板基线，但已有局部中央治理接入 | `infra` |
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

结论：部分等价覆盖，但尚未补齐组织模板基线。

说明：

- `sync-upstream.yml` 已通过中央 `fork-sync.reusable.yml` 做 upstream sync
- `main-to-docker-pr.yml` 已通过中央 `branch-sync-pr.reusable.yml` 做 `main -> docker` 开 PR
- `docker-publish.yml` / `docker-release.yml` 仍保留较多仓库内自定义逻辑，并未统一切到组织模板基线
- 当前远端 branch-sync / fork-sync 仍 pin 在较老 SHA（`3a74494444ebf60ec855923fe1f98c111e9a8270`），与模板仓当前 blessed SHA 不一致
- 缺少面向组织治理的 `workflow-ref-policy.yml`
- 缺少面向后续统一升级的 `reusable-workflow-update-pr.yml`

建议动作：

1. 先补 `workflow-ref-policy.yml`
2. 再补 `reusable-workflow-update-pr.yml`
3. 评估是否将 `docker-publish.yml` / `docker-release.yml` 收敛到组织模板或中央 reusable 的标准调用面

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

结论：当前不适用 Docker/fork/branch-sync 模板基线，但已经局部接入中央 runner probe。

说明：

- 当前仅有 commit message guard，不属于本轮模板覆盖目标
- 该 workflow 已通过中央 `runner-fallback.reusable.yml` 接入 runner probe
- 目前 pin 到较老 SHA（`8399681997e0822753ffe04761821784f1ac9a7c`）

建议动作：

1. 现阶段无需补 Docker/fork/branch-sync 模板
2. 如后续继续引入更多中央 reusable，优先补 `workflow-ref-policy.yml`
3. 在下一轮中央升级时，把 runner-fallback pin 统一到新的 blessed SHA

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

### P1：立即补齐

- `ksrpc`
  - 补 `workflow-ref-policy.yml`
  - 补 `reusable-workflow-update-pr.yml`

### P2：下一轮统一 pin 升级时处理

- `ksrpc`
  - 把 branch sync / fork sync 的旧 SHA 升级到 blessed SHA
- `infra`
  - 把 `runner-fallback` 的旧 SHA 升级到 blessed SHA

### P3：待业务形态明确后再接入

- `platform`
- `dagster`

## 5. 审计结论

截至 2026-03-07，`quantman888` 组织的 workflow 治理状态可以概括为：

- Docker 基线已经在 `mcphub-gateway` 与 `mcp-didatodolist` 形成可复用样板
- `ksrpc` 已具备部分等价能力，但还没有完全进入组织级模板与升级闭环
- `cortex` 属于定制治理仓，应保持单独治理路径，不应被误纳入 Docker 模板基线
- `.github` 与 `workflow-reusable` 的职责边界已经清晰：前者负责模板入口，后者负责运行时 control plane

下一步最有价值的动作不是继续造新模板，而是把 `ksrpc` 补齐到组织级升级闭环里。
