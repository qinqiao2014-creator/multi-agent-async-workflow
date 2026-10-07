# Issue 协议规范

多节点异步协作要求 Issue 本身承载足够信息：**契约在 Issue 上**（why / what / 涉及范围 / 验收标准），**实现在 PR 上**（how / diff / CI / 自检）。

## 标签体系

**11 个协议状态标签**：

| 标签 | 含义 | 自动流转 |
|------|------|----------|
| `backlog` | 待规划 | → `ready` |
| `ready` | 待认领（**认领者可能是人，也可能是 Agent**） | → `in_progress` |
| `in_progress` | 实施中 | → `needs-review` / `blocked` |
| `needs-research` | 需研究 | → `ready` |
| `needs-clarification` | 验收标准不清，退回 Planner | → `ready` |
| `needs-review` | 待审核 | → `approved` / `in_progress` |
| `approved` | 审核通过，待集成负责人合并和 main 验证 | → `done` / `in_progress` |
| `blocked` | 阻塞；传播阻断进入前必须在矩阵记录唯一的阻断前状态 | → 恢复该状态（缺失/非法/过期则 → `needs-lead`）；其他阻塞解除后 → `ready` |
| `needs-lead` | 需总负责人裁决（规格 / 优先级 / 方案分歧） | → `ready` / `in_progress` |
| `needs-human` | 需真人决定（花钱 / 对外承诺 / 法律权限 / 业务方向） | → `ready`（决策后） |
| `done` | 已完成 | 终态 |

**3 个来源标签**：`source:human` / `source:ai` / `human-only` —— 语义与认领规则见 `SKILL.md`「任务来源与执行者」。

### 只在这里写死的几条约束

- **`needs-lead` 与 `needs-human` 不可合并成一个**：前者总负责人自己拍板，后者必须到真人。混用会导致本该一句话解决的分歧堆着等人，或 AI 替真人做了它不该做的决定。
- **打 `needs-review` 的前提**是 Issue 下已有 `✅ 已提交 PR #N` 的 comment，且该 PR body 写了 `Closes #<issue>`。**没有关联 PR 的 `needs-review` 视为无效**，Reviewer 应直接退回 `in_progress`。
- **`done` 只能由集成负责人在 PR 合并且 `main` 验证通过后设置。** GitHub 因 `Closes #N` 自动关闭 Issue 时，也必须补齐标签和验证 comment，不能把自动关闭当作验收完成。
- **`指定执行者` / `审核负责人` / `集成负责人` 记录的是用户任命的 Agent 实例**，不依赖 GitHub 账号是否不同。指定 ≠ 认领（三者正交、指定后保留 `ready`），规则见 `SKILL.md`「指定执行者 = 编排，不是认领」。
- **认领以 Issue 中 append-only 的 claim lifecycle comment 为身份事实源**，assignee / 标签只负责路由。只有最新合法事件为 `active` 且 lease 未过期才表示已认领。完整状态机与回收规则见 `dependency-propagation.md` 第 2 节。
- **inherited gates 必须在当前 artifact 上产生证据。** 旧 commit / 旧构建 / 旧配置的证据标为 `historical`，不能填 `pass`。

## 配套工件

| 工件 | 出处 |
|------|------|
| Issue 模板（Planner 填） | `../assets/issue-template.md` |
| PR 模板（Executor 填） | `../assets/pr-template.md` |
| 各节点 comment 话术 | `../assets/comment-protocol.md` |
| 交付与审核闭环细则 | `pr-review-protocol.md` |
| 认领协议与依赖传播 | `dependency-propagation.md` |
