# 交付与审核闭环：Issue → PR → Review

Executor 交付 PR，Reviewer 独立审，上下文靠工件（Issue ↔ PR 双向关联）传递而非共享 session。

> **节点每轮的操作规则（扫什么、做什么、禁止什么）见 `SKILL.md` 对应角色章节。**
> 本文件只写那些「不需要每轮读、但判不清楚时要用」的细则。

## 1. PR 的硬性要求

- **body 必须写 `Closes #<issue>`**（或 `Refs #<issue>`，视是否终结该 Issue）。这一行不是礼貌，是**寻址**——Reviewer 靠它反查任务契约，缺了它 PR 就是一堆无从判定对错的 diff。
- **分支命名**：`<node>/<issue-number>-<slug>`，例 `executor-a/128-fix-retry-backoff`。从分支名就能看出「谁、为哪个 Issue、做什么」，扫队列时不用打开就能分流。
- **收尾两件事缺一不可**：开 PR + 回 Issue comment `✅ 已提交 PR #N` 并把标签从 `in_progress` 换成 `needs-review`。只开 PR 不换标签，Reviewer 的 loop 扫不到，任务静默死掉。

## 2. 审核独立性

**唯一的硬红线：不能审、不能合并自己写的代码。**

它约束的是**代码作者**（继承了实现的盲区），不是**规格作者**——把它外推成「定规格的人不能审」会让「1 个 lead + N 个 Executor」这种最常见配置直接死锁，没人能合并任何 PR。

### 同一 GitHub 账号下的审核记录

用户可以用同一个 GitHub 账号控制多个 Agent。是否独立按**被任命的 Agent 实例**判断，不按 GitHub 登录名判断；同一实例仍不能审核或合并自己的代码。

GitHub 拒绝同账号的原生 approval/request-changes 时，PR 评论 + Issue 标签就是正式记录：

```markdown
审核主体：Reviewer-A
实现主体：Executor-B
👍 审核通过，可以合并
```

原生 GitHub review 在可用时可以叠加使用，但不是协议的必要条件。

## 3. 审核增量 diff 的寻址规范（rebase 后必须改口径）

打回后 Executor 追加修复，Reviewer 要审「**这次修了什么**」，不是重审整个 PR。报增量范围时有个**必踩的坑**：

| 场景 | 错误写法 | 正确写法 |
|------|---------|---------|
| Executor 在旧 head 上**追加 commit**（未 rebase） | — | `旧head..新head` ✅ |
| Executor **rebase 到新 main 后**再追加 commit | `旧head..新head` ❌ | `rebase后的原commit..新head` ✅ |

**原因**：rebase 会重写历史，**旧 head 不再是新 head 的祖先**。此时 `旧head..新head` 变成「两个分叉点的对称差」，会把 main 上**别人的合并**全算进来——文件数虚高、看着像夹带无关改动，实际上一次都没碰过。

**落地规则**

1. **Executor 报增量时**：先 `git log --oneline` 确认父 commit 是什么，用 **`父commit..新head`**；不要凭记忆写旧 head。
2. **Reviewer 见范围异常时**：不要直接判「夹带」，先自己跑 `git diff --name-status <可疑范围>` 与 `git show --stat <head>` 对照——**以 `git show --stat HEAD` 为准**，它是唯一不受 rebase 影响的口径。
3. **Executor 若明确 rebase 过**，应在 PR 评论里直接给出父 commit 哈希，省掉 Reviewer 一次猜。

| 命令 | 含义 | 适合判「本次修复」 |
|------|------|------------------|
| `git show --stat HEAD` | 仅该 commit | ✅ 最准 |
| `父commit..HEAD` | 本次修复（含该 commit） | ✅ 准 |
| `main...HEAD` | 整条分支（原功能 + 修复） | ❌ 文件集合可能相同但行数含原功能 |
| `旧head..新head`（rebase 后） | 混入 main 演进 | ❌ 完全错 |

> 实测教训（2026-09-23）：rebase 后误报 `ac220b3..10e9a4c`，显示 10 文件；真实修复增量是 `dc2a256..10e9a4c` 的 **5 文件 / +30 −7**。同一轮 Reviewer 也把文件数数成了 7（实际 5）⇒ **双方都要以 `git show --stat` 为准，不要各自口算范围。**

## 4. 打回环与升级

```
B: PR #256 ──→ C: review
                 ├── 通过 → Issue approved → Integrator 合并 + main 验证 → Issue done
                 └── 打回 → Issue 回 in_progress
                              └→ B 的 loop 扫到「我的 PR 被打回」→ 改 → 重新请审
```

B 不需要"被通知"。它的 loop 本来就在扫「assignee 是我 且 `in_progress`」的 Issue，被打回的任务自然回到它的视野里。这就是总线相对于消息传递的好处：**状态本身就是通知。**

**必须设升级阈值：同一个 PR 被打回 2 次后打 `needs-lead`，交总负责人裁决。** 没有这个阈值，B 和 C 会在「我觉得可以了 / 我觉得还不行」之间无限对打，烧掉大量 token 且不收敛。两次之后说明分歧在标准本身，而标准是总负责人的职责。

## 5. 集成前的新鲜度与最终收尾

Integrator 处理 `approved`，而不是 Reviewer 通过后直接关闭 Issue。合并前必须过新鲜度闸门：

1. 拉取 `main`，记录当前 `main` SHA 和 PR head SHA。
2. 通过 GitHub CLI/API 查询 PR 的 `baseRefOid`、`headRefOid`、`mergeable`、`mergeStateStatus` 和 CI；只有 `mergeable = MERGEABLE` 且状态为 clean 才能继续。
3. 若 `main` 在 PR 创建或审核后前进，要求 Executor 更新分支；在更新后的分支或合并结果上运行受影响验证。
4. 无法读取 GitHub API 状态时不得宣称 PR 可安全合并；仅有 SSH refs 只能用于观察提交，**不能代替 mergeable/CI 检查**。

合并后必须重新拉取 `main`，确认 PR 已 merged，再在 `main` 上执行规定验证；在 Issue 留下合并前 `main` SHA、PR head SHA、合并后的 main commit SHA、验证命令和结果。最后移除活动标签并添加 `done`。**只有这一步完成，交付才真正结束。**

## 6. 传播门禁的位置

依赖传播的完整协议在 `dependency-propagation.md`，本文件不复述。两边各自要执行的检查点：

- **Reviewer 批准前**：源 shared-invariant PR 的影响面已可靠枚举、每个目标都有矩阵行且无 `unknown`；下游 PR 的对应行必须为 `synced` 或获批 `not-applicable`，且 head 已含上游修复并重跑累计 gate。
- **Integrator 合并前**：再执行一遍同一检查。**即使下游 PR 的 CI 为绿，也不得绕过传播门禁。**

## 7. PR 模板

见 `../assets/pr-template.md`。要点只有三个：关联哪个 Issue、验收标准逐条自检、明确说了什么没做（防止 Reviewer 把未做项当遗漏）。
