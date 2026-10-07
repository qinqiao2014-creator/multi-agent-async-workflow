# Multi-Agent Async Workflow

多 Agent 异步协同工作流：用 GitHub Issues 作为任务总线，让不同能力的模型按拓扑结构异步协作，持续流转任务直至闭环。

> 主要贡献者：群友zzszmyf

![Architecture Overview](assets/architecture-overview.svg)

## 核心模式

这不是 prompt 路由，也不是 session 桥连——是一个**基于 Issue 的多节点异步协作协议**。

**关键洞察：Issue 在这里不是 bug tracker，是跨 session 的持久化消息队列。**

这个架构把 Agent 理解为模型的承载工具：真正干活的是背后的模型。每个节点是一个独立 Session，背后跑着不同能力的模型。它们不需要预先编排顺序，也不需要中心调度器，只需要无限循环做一件事：

```
看队列 → 认领 → 处理 → 流转状态 → 再看
```

### 三条不可破的约束

1. **状态即通知** —— 下游不靠被通知，它扫的标签状态本身就是通知。只干活不换标签 = 任务静默死亡。
2. **不能审、不能合并自己写的代码** —— 唯一的硬红线。约束的是**代码作者**，不是规格作者。
3. **上下文靠工件传递** —— Issue 写 why / what / 验收标准，PR 写 how / diff。没写下来的等于不存在。

## Issue 状态机

![Issue State Machine](assets/issue-state-machine.svg)

标签全集与流转规则见 `references/issue-protocol.md`；节点角色与各自动作见 `SKILL.md`。

## 目录

```
multi-agent-workflow/
├── SKILL.md                       # agent 加载的协议正文：节点怎么干活
├── RED_SKILL.md                   # 对外发布文案（给人看，非技术文档）
├── assets/
│   ├── *.svg                      # 架构图 / 状态机 / 拓扑
│   ├── issue-template.md          # Issue 模板（Planner 填）
│   ├── pr-template.md             # PR 模板（Executor 填）
│   └── comment-protocol.md        # 各节点 comment 话术
└── references/
    ├── setup.md                   # 从零搭总线
    ├── issue-protocol.md          # 标签体系与状态机
    ├── pr-review-protocol.md      # 交付与审核闭环
    ├── dependency-propagation.md  # 依赖传播与累计回归门禁
    ├── rationale.md               # 设计理由与适用边界
    └── best-practices.md          # 落地检查清单与常见坑
```

## 快速开始

**当 skill 用**：仓库放进 skill 目录 → agent 加载 `SKILL.md` → 告诉它「你是这条流水线的 Executor，总线是 `<repo>`」→ 它自己做开工前检查，然后扫队列干活。

**搭一条流水线**：按 `references/setup.md` 走 —— 建标签 → 放模板 → 编排节点 → 放 5 个 Issue 跑一轮看它在哪断。

## 适用边界

适合：需求可拆成独立任务、能写出可判定的验收标准、允许异步轮询延迟。

不适合：需要实时来回的任务（轮询 + 工件化开销比直接对话慢）、一次性小改动（写 Issue 的成本高于改代码）、写不出可判定验收标准的需求（整套系统的正确性都压在验收标准上）。

设计理由与逐项权衡见 `references/rationale.md`。

## License

MIT
