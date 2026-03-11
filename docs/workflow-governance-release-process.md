# workflow 治理发布与升级规范

本文定义 `quantman888/workflow-reusable` 与 `quantman888/.github` 的职责边界，以及中央 workflow 升级时的标准发布闭环。

## 1. 设计目标

- 组织级 workflow 能长期复用，而不是每仓复制一份后各自漂移
- 顶层长期凭据统一收敛为 GitHub App：`GH_APP_ID` + `GH_APP_PRIVATE_KEY`
- private 仓库可以自动纳入治理，不依赖硬编码仓库名单
- workflow 升级有固定发布顺序、固定回滚路径、固定验收办法

## 2. 仓库职责边界

| 仓库 | 角色 | 负责内容 | 不负责内容 |
| --- | --- | --- | --- |
| `quantman888/workflow-reusable` | 控制面 / 实现层 | reusable workflows、runner probe、GitHub App token minting、secret sync、批量升级 PR、managed PR hardening | 新仓模板入口、组织接入说明 |
| `quantman888/.github` | 模板入口 / 组织文档层 | `workflow-templates/`、新仓接入清单、升级规范、组织审计报告、blessed SHA 入口 pin | 运行时控制逻辑、业务仓实际执行逻辑、managed PR hardening 语义 |
| 业务仓 | 调用方 / 落地层 | 仓库级测试步骤、变量/密钥映射、业务分支模型 | 中央 reusable 实现、组织模板维护 |

一句话：

- `.github` 负责“入口与规则”
- `workflow-reusable` 负责“执行与控制面”
- 业务仓负责“少量仓库特化参数”

## 3. 固定治理原则

- 业务仓对中央 reusable 的 `uses:` 引用必须 pin 到受控 SHA 或受控 `v*` tag。
- `.github` 模板仓只保存“已经验证通过”的 blessed SHA。
- 不允许用长期 PAT 作为组织级通用方案。
- private 仓 `GH_APP_*` 通过控制面 workflow 自动同步，不手工维护仓库列表。
- 升级 caller 仓引用时，优先用 `reusable-workflow-update-pr` 开 PR，而不是直接推默认分支。
- 模板仓只放 bootstrap 模板，不复制 control plane 实现。
- managed PR hardening 仍由 `workflow-reusable` 实现；`.github` 模板层只负责入口与 blessed pin。

## 4. 发布对象

中央治理升级时，实际有三层需要按顺序发布：

1. `workflow-reusable`：先发布控制面实现
2. `quantman888/.github`：再把组织模板 pin 更新到新 SHA
3. caller 仓：最后由 `reusable-workflow-update-pr` 开 PR 升级各仓引用

这三层必须串行推进，不能跳步骤。

其中第二层只更新入口模板与文档，不在模板仓复写控制面 hardening 逻辑。

## 5. 标准升级流程

### 5.1 第一步：先改 `workflow-reusable`

在 `quantman888/workflow-reusable` 中完成 reusable workflow 或控制面的改动，例如：

- `docker-publish.reusable.yml`
- `docker-promote.reusable.yml`
- `branch-sync-pr.reusable.yml`
- `fork-sync.reusable.yml`
- `reusable-workflow-update-pr.reusable.yml`
- `runner-fallback.reusable.yml`
- `github-app-secret-sync.controlplane.yml`

要求：

- 先在控制面仓完成自测与仓内策略校验
- 修改接口时走硬切，不保留旧字段兼容层
- 生成新的受控 SHA；如需要对外语义化发布，可额外打 `v*` tag

### 5.2 第二步：选定 blessed ref

组织内部推荐：

- `.github` 模板仓统一 pin 到完整 40 位 SHA
- 对外或跨阶段发布可额外维护 `v*` tag

不要把模板仓改成 `@main`，也不要让 caller 仓引用浮动分支。

### 5.3 第三步：更新 `quantman888/.github`

在 `quantman888/.github` 中做两件事：

1. 把 `workflow-templates/*.yml` 中的中央 `uses:` pin 改到新的 blessed SHA
2. 如模板接入步骤或变量要求有变化，同步更新接入文档，但不把 control plane 的 hardening 实现复制到模板层

这样做的意义是：

- 今后新建仓库，天然从最新 blessed SHA 起步
- 模板入口与 control plane 版本始终保持一致

### 5.4 第四步：批量升级现有 caller 仓

已接入治理的 caller 仓统一通过以下流程升级：

- 业务仓内存在 `reusable-workflow-update-pr.yml`
- 中央发布后，通过 `workflow-reusable` 内的 `release-dispatch-consumers.yml` 向这些仓触发 `reusable-workflow-release` 事件，或手动 dispatch 其 updater workflow
- updater workflow 调用 `quantman888/workflow-reusable/.github/workflows/reusable-workflow-update-pr.reusable.yml`
- updater 默认一次升级 caller 仓 `.github/workflows/` 下所有指向 `quantman888/workflow-reusable` 的 refs；仅在需要局部灰度时才显式传单个 `reusable_workflow_path`
- 由 GitHub App bot 开升级 PR
- 仓库在 PR 上执行测试、策略校验、发布 dry-run，验证通过后再合并

这一步的关键点是“开 PR 升级”，不是“直接改默认分支”。

managed PR hardening 的约束也在这一层由 `workflow-reusable` 保证：

- `cortex` 的 provenance context 使用稳定且单一的名称，避免同一天多次更新产生多张状态语义
- merge-controller 中的 PR title/body 只承担 routing fingerprint 角色，不作为主信任根
- 真正的合并信任根是目标 `head_sha` 上的 success status
- 仍保留同一张 bot PR 持续更新，不引入多 PR 拓扑
- 通用 updater 使用固定分支 `automation/reusable-workflow-update`，不再按 `target_ref` 派生新 bot 分支

### 5.5 第五步：补齐 private 仓密钥同步

如新增了 private caller 仓，或 GitHub App 相关密钥发生轮换：

- 运行 `quantman888/workflow-reusable/.github/workflows/github-app-secret-sync.controlplane.yml`
- 默认 `target_visibility=private`
- 如需全量补齐，可手动使用 `target_visibility=all`

这样新仓会自动纳入治理，不需要手工维护硬编码名单。

## 6. 推荐的发布节奏

推荐把升级分成三层节奏：

### 6.1 控制面发布节奏

适用：修改了 reusable 实现本身。

- 在 `workflow-reusable` 合并改动
- 形成新的 blessed SHA
- 必要时打 release/tag

### 6.2 模板发布节奏

适用：新的 blessed SHA 已验证完成。

- 更新 `quantman888/.github` 模板 pin
- 更新模板说明文档，明确模板层只做入口与 blessed pin
- 合并后，新仓全部从新基线起步

### 6.3 存量仓升级节奏

适用：现有 caller 仓需要跟进新 SHA。

- 优先选择 canary 仓先跑
- 验证通过后，再批量 dispatch updater workflow
- 逐仓看 PR 测试结果后合并

## 7. canary 与验收顺序

当前组织里，建议使用以下仓库作为验收样本：

- `mcphub-gateway`
  - Docker 基线完整
  - 已验证通过：
    - `workflow-ref-policy` run `22796139078`
    - `Docker Publish` run `22796139086`
- `mcp-didatodolist`
  - Docker 基线完整
  - 已验证通过：
    - `workflow-policy-check` run `22796138802`
    - `Docker Publish` run `22796138809`
- `cortex`
  - 定制治理样本
  - 已验证通过：
    - `cortex-sync-release` run `22796260768`
    - `cortex-sync-skills` run `22796284463`
    - `cortex-validate` run `22796315564`

推荐顺序：

1. 先过 `mcphub-gateway`
2. 再过 `mcp-didatodolist`
3. 对定制治理改动，再补 `cortex`
4. 最后再批量推到其余适用仓

## 8. 回滚策略

如果新 SHA 在 caller 仓验证失败，按下面顺序回滚：

1. 暂停后续仓库的 updater dispatch
2. 让失败仓的升级 PR 不合并，必要时关闭 PR
3. 若 `.github` 模板已更新，把模板 pin 回退到上一个 blessed SHA
4. 如问题根源在 `workflow-reusable`，回退控制面改动并重新产出稳定 SHA
5. 再按同样流程重新发布

回滚原则：

- 优先回滚引用，不直接修改业务仓默认分支历史
- 不使用强推覆盖业务仓
- 不引入临时 PAT 作为应急方案

## 9. 新仓纳入闭环的方法

新仓库要想从第一天起就自动跟随组织治理，至少满足这三个条件：

1. workflow 通过 `quantman888/.github` 模板创建
2. 仓内存在 `reusable-workflow-update-pr.yml`
3. private 仓能通过 `github-app-secret-sync.controlplane.yml` 自动拿到 `GH_APP_*`

满足后，该仓会自然进入：

- 模板 bootstrap
- control plane 升级
- App secret 同步
- PR 化升级验证

这就是组织级长期治理闭环。

## 10. 禁止事项

- 不要把 `.github` 当成运行时控制逻辑仓
- 不要把 `workflow-reusable` 里的实现复制到每个业务仓长期分叉维护
- 不要使用 `@main` / `@master`
- 不要使用长期 PAT 作为组织级通用 credential
- 不要用硬编码仓库名单管理 private repo 的 GitHub App 密钥同步
