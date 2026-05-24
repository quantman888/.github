# workflow 治理发布与升级规范

本文定义 `quantman888/.github` 单仓模式下的 workflow 治理边界，以及中央 workflow 升级时的标准发布闭环。

## 1. 设计目标

- 组织级 workflow 能长期复用，而不是每仓复制一份后各自漂移
- 顶层长期凭据统一收敛为 GitHub App：`GH_APP_ID` + `GH_APP_PRIVATE_KEY`
- private 仓库可以自动纳入治理，不依赖硬编码仓库名单
- workflow 升级有固定发布顺序、固定回滚路径、固定验收办法
- 组织模板入口与运行时控制面统一维护在 `quantman888/.github`

## 2. 单仓职责边界

| 路径 | 角色 | 负责内容 | 不负责内容 |
| --- | --- | --- | --- |
| `workflow-templates/` | 模板入口 / 新仓 bootstrap | 组织级 workflow templates、blessed ref 入口 pin | 运行时控制逻辑、managed PR hardening 语义 |
| `.github/workflows/` | 控制面 / 实现层 | reusable workflows、runner probe、GitHub App token minting、secret sync、批量升级 PR、managed PR hardening | 仓库级业务测试、业务分支模型 |
| `.github/actions/` | shared action 层 | 可复用 composite actions | caller 仓业务逻辑 |
| `docs/` | 治理文档层 | 新仓接入、升级规范、组织审计报告 | 控制面执行逻辑 |
| 业务仓 | 调用方 / 落地层 | 仓库级测试步骤、变量/密钥映射、业务分支模型 | 中央 reusable 实现、组织模板维护 |

一句话：

- `workflow-templates/` 负责“入口”
- `.github/workflows/` 与 `.github/actions/` 负责“执行与控制面”
- `docs/` 负责“规则”
- 业务仓负责“少量仓库特化参数”

## 3. 固定治理原则

- 业务仓对中央 reusable 的 `uses:` 引用必须 pin 到受控 SHA 或受控 `v*` tag。
- 不允许用长期 PAT 作为组织级通用方案。
- private 仓 `GH_APP_*` 通过控制面 workflow 自动同步，不手工维护仓库列表。
- 升级 caller 仓引用时，优先用 `reusable-workflow-update-pr` 开 PR，而不是直接推默认分支。
- 模板只放 bootstrap 入口和 blessed ref，不复制 control plane 实现。
- managed PR hardening 由 `.github/workflows/` 内的 reusable 实现保证。

## 4. 发布对象

中央治理升级时，实际有两层需要按顺序发布：

1. `quantman888/.github`：先发布控制面实现、模板入口和文档
2. caller 仓：再由 `reusable-workflow-update-pr` 开 PR 升级各仓引用

硬切后的默认发布 ref 是 `v1`。发布顺序必须保证：

1. 先合并本仓硬切提交
2. 再把 `v1` tag 创建或更新到该硬切提交
3. 确认 `git ls-remote --tags origin refs/tags/v1` 能查到该 tag
4. 最后再让新模板或 caller 仓引用 `quantman888/.github/.github/workflows/...@v1`

如组织内部更偏好完整 SHA，发布提交落地后应把模板中的 `@v1` 统一替换为该 40 位 SHA。

不要把模板或 caller 仓改成 `@main` / `@master`。

## 5. 标准升级流程

### 5.1 第一步：修改 `.github` 控制面

在 `quantman888/.github` 中完成 reusable workflow、composite action、模板或文档改动，例如：

- `.github/workflows/docker-publish.reusable.yml`
- `.github/workflows/docker-promote.reusable.yml`
- `.github/workflows/branch-sync-pr.reusable.yml`
- `.github/workflows/fork-sync.reusable.yml`
- `.github/workflows/reusable-workflow-update-pr.reusable.yml`
- `.github/workflows/runner-fallback.reusable.yml`
- `.github/workflows/github-app-secret-sync.controlplane.yml`
- `.github/actions/resolve-github-app-bot/action.yml`

要求：

- 先在本仓完成自测与策略校验
- 修改接口时走硬切，不保留旧字段兼容层
- 生成新的受控 SHA；如需要对外语义化发布，可额外打 `v*` tag

### 5.2 第二步：选定 blessed ref

组织内部推荐：

- 模板与 internal cross-repo action 引用统一 pin 到完整 40 位 SHA
- 对外或跨阶段发布可额外维护 `v*` tag

当前硬切模板使用 `@v1`，发布时必须在合入后的提交上创建或更新该 tag。`v1` 不存在时，模板里所有跨仓 `uses:` 都会在 GitHub 解析 reusable workflow 前失败。不要让 caller 仓引用浮动分支。

### 5.3 第三步：一次性迁移旧 caller 仓

硬切前已接入治理的 caller 仓通常仍包含旧调用：

```yaml
uses: quantman888/workflow-reusable/.github/workflows/<file>.yml@<old-ref>
```

这些引用不能靠只设置 `reusable_repo` 的普通 ref bump 自动迁到新仓。硬切时必须先给每个存量 caller 仓开一次迁移 PR，至少替换：

```yaml
uses: quantman888/.github/.github/workflows/<file>.yml@v1
```

迁移 PR 应覆盖 caller 仓自己的 `.github/workflows/reusable-workflow-update-pr.yml`，否则后续 updater 仍会调用旧控制面。

新 updater 支持 `source_reusable_repo`，可在 caller 的 updater 已切到 `quantman888/.github` 后，把仓内其余旧引用从 `quantman888/workflow-reusable` 批量迁到 `quantman888/.github`。新模板默认传入：

```yaml
source_reusable_repo: quantman888/workflow-reusable
reusable_repo: quantman888/.github
```

该 PR 合并后，caller 仓才进入正常的统一升级闭环。

### 5.4 第四步：批量升级已迁移 caller 仓

已接入治理的 caller 仓统一通过以下流程升级：

- 业务仓内存在 `reusable-workflow-update-pr.yml`
- 中央发布后，通过 `.github/workflows/release-dispatch-consumers.yml` 向这些仓触发 `reusable-workflow-release` 事件，或手动 dispatch 其 updater workflow
- updater workflow 调用 `quantman888/.github/.github/workflows/reusable-workflow-update-pr.reusable.yml`
- updater 默认一次升级 caller 仓 `.github/workflows/` 下所有指向 `quantman888/.github` 的 refs；仅在需要局部灰度时才显式传单个 `reusable_workflow_path`
- 由 GitHub App bot 开升级 PR
- 仓库在 PR 上执行测试、策略校验、发布 dry-run，验证通过后再合并

这一步的关键点是“开 PR 升级”，不是“直接改默认分支”。

managed PR hardening 的约束也在这一层由 `.github/workflows/` 保证：

- provenance context 使用稳定且单一的名称，避免同一天多次更新产生多张状态语义
- merge-controller 中的 PR title/body 只承担 routing fingerprint 角色，不作为主信任根
- 真正的合并信任根是目标 `head_sha` 上的 success status
- 仍保留同一张 bot PR 持续更新，不引入多 PR 拓扑
- 通用 updater 使用固定分支 `automation/reusable-workflow-update`，不再按 `target_ref` 派生新 bot 分支

### 5.5 第五步：补齐 private 仓密钥同步

如新增了 private caller 仓，或 GitHub App 相关密钥发生轮换：

- 运行 `quantman888/.github/.github/workflows/github-app-secret-sync.controlplane.yml`
- 默认 `target_visibility=private`
- 如需全量补齐，可手动使用 `target_visibility=all`

这样新仓会自动纳入治理，不需要手工维护硬编码名单。

## 6. 推荐的发布节奏

### 6.1 控制面发布节奏

适用：修改了 reusable 实现本身。

- 在 `quantman888/.github` 合并改动
- 形成新的 blessed SHA
- 必要时打 release/tag
- 同步更新模板中的 blessed ref

### 6.2 存量仓升级节奏

适用：现有 caller 仓需要跟进新 ref。

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

如果新 ref 在 caller 仓验证失败，按下面顺序回滚：

1. 暂停后续仓库的 updater dispatch
2. 让失败仓的升级 PR 不合并，必要时关闭 PR
3. 若模板已更新，把模板 pin 回退到上一个 blessed ref
4. 如问题根源在控制面实现，回退 `.github/workflows/` 或 `.github/actions/` 改动并重新产出稳定 ref
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

- 不要把 central reusable 实现复制到每个业务仓长期分叉维护
- 不要使用 `@main` / `@master`
- 不要使用长期 PAT 作为组织级通用 credential
- 不要用硬编码仓库名单管理 private repo 的 GitHub App 密钥同步
