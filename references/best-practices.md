# 落地检查清单与常见坑

> 本文件只收**协议正文里没有的操作经验**。协议规则本身见 `SKILL.md` 与各 references——不在这里复述。

## 落地检查清单

- [ ] 确定 GitHub Repository 作为任务总线
- [ ] 建 14 个标签（11 状态 + 3 来源），命令见 `setup.md`
- [ ] 放 Issue / PR / Comment 三个模板到 `.github/`
- [ ] 配置各节点 Session：prompt + 触发频率（起步用定时单次 pass）
- [ ] 确认 Reviewer 与 Executor 是独立的 **Agent 实例**（GitHub 账号可共用）
- [ ] 明确 `指定执行者` = 编排（`assignee`，保留 `ready`）≠ 认领（claim + `in_progress`）
- [ ] 设定打回升级阈值（同一 PR 打回 N 次 → `needs-lead`）
- [ ] 为合并建立新鲜度闸门（记录 main SHA / PR head SHA / mergeable）
- [ ] 为 shared invariant 建立反向依赖传播矩阵与累计 inherited gates
- [ ] 确定流水线拓扑（线性 / DAG / 环形 / 扇入扇出，见 `rationale.md`）
- [ ] 用 GitHub Action 做死任务回收（**不要用 agent**——agent 也会死，而没人回收回收者）
- [ ] 放 5 个 Issue 跑一轮，看它在哪里断

## 常见坑（协议外的部分）

1. **Issue 太粗糙或太大** → 实施侧无法直接动手，反复 comment 确认，退回同步模式；或一个 Issue 做三天，阻塞整条流水线。**拆到 1-2 小时粒度，验收标准必须可判定。**
2. **没写退出条件** → 实施侧过度设计。**Issue 里明确「只做 xxx，不做 yyy」（非目标 / 禁止事项）。**
3. **队列流速失控** → 什么都接，全员堵在 `in_progress`。**Planner 控制流出速度，同时 `in_progress` 不超过实施节点数。**
4. **阻塞无人处理** → 卡住后没人管，Issue 一直躺着。**需要有人定期扫 `blocked` / `needs-human` 队列。**
5. **Reviewer 成瓶颈** → PR 全堆在 `needs-review`，Executor 空转。**并行 review，或按优先级分批；持续瓶颈就任命第二个 Reviewer 实例。**
6. **人在 review 每个 PR** → 人重新变成瓶颈，多智能体的吞吐收益归零。**人 review 这套系统而不是单个产出**：看吞吐、打回率、`blocked` 堆积、升级频次。指标异常时改 Issue 模板和节点 prompt。
7. **CI 全局坏掉还在产 PR** → Executor 持续生产注定过不了的 PR。**给节点加前置检查：main 的 CI 是红的就停下报告。**
8. **用共享上下文的 agent team 替代总线** → 上下文窗口成为规模天花板，且审核独立性丧失。**team 只能作为单个节点的内部实现**（如 Researcher 内部定位问题），对总线仍只暴露一个结论 comment。**团队在节点内，协议在节点间。**
