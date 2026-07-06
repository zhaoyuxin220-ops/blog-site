# Hermes Agent v0.16 Kanban Swarm 功能深度解析：面向 AI 开发者的自动化工作流

## 一、背景：为什么 Kanban Swarm 值得关注

在 AI 辅助开发场景里，最常见的瓶颈已经不是“能不能生成代码”，而是**多任务编排、跨会话协作、结果可追踪**。Hermes Agent 在 v0.16 中把 Kanban、Dispatcher、Worker 生命周期拧在了一起，形成了一个可落地的**多智能体工作队列**，而不是一个概念性的 demo。

这篇文章面向已经会用 Hermes 做单任务的开发者，解释 Kanban Swarm 的实际结构、调度约束、工作区约定，以及它和 `delegate_task` / `cronjob` 的边界。

---

## 二、核心架构：三个关键概念

### 2.1 Board（看板）

- 看板是持久化的协作边界，底层是 SQLite。
- 同一 Board 内的 Task 共享调度器、共享命名空间。
- Board 通过环境变量 `HERMES_KANBAN_BOARD` / `HERMES_KANBAN_DB` 绑定，Worker 不会意外跨板操作。

### 2.2 Task（任务卡）

一张 Task 至少包含：

- `id`：唯一标识
- `title` / `body`：可读规格
- `assignee`：分配给哪个 profile 执行
- `status`：`ready` / `running` / `blocked` / `done`
- `parents` / `children`：依赖图
- `workspace_kind` + `workspace_path`：执行目录约定
- `goal_mode` / `goal_max_turns`：是否启用目标循环
- `max_runtime_seconds`：硬超时

**关键设计点**：Dispatcher 对 `assignee` 做静默校验——如果 assignee 不是一个真实 profile，卡片会永远停在 `ready`，不会报错。这是生产环境最常见的坑。

### 2.3 Worker Context（运行上下文）

Worker 启动时，系统会注入：

- 当前 Task 的完整状态
- `HERMES_KANBAN_TASK`
- `HERMES_KANBAN_WORKSPACE`
- 一个预格式化的 `worker_context` 字符串，可直接用于 reasoning

这意味着 Worker 不需要自己去查 Board，它拿到手的已经是“ ground truth”。

---

## 三、生命周期：从 Ready 到 Done

```
ready → claimed → running → (complete | block)
```

### 3.1 Dispatch

Dispatcher 默认运行在 Gateway 内（`kanban.dispatch_in_gateway: true`）。它会：

1. 扫描 `ready` 且 `parents` 全部 `done` 的卡片
2. 按 `assignee` + `priority` 选择下一个任务
3. 原子 claim，防止重复调度
4. Spawn 对应 profile 的 Hermes 进程

### 3.2 Heartbeat

长时间任务必须定期调用 `kanban_heartbeat(note=...)`，否则：

- 超过 `kanban.dispatch_stale_timeout_seconds`（默认 4 小时）无心跳
- Dispatcher  reclaim 任务，重新放回 `ready`
- 不扣 failure counter，但当前进度丢失

**建议**：超过 1 小时的任务至少每小时心跳一次。

### 3.3 Complete / Block

完成后调用 `kanban_complete(summary=..., metadata=...)`：

- `summary`：给人看的 1-3 句总结
- `metadata`：机器可读事实，如 `changed_files`、`tests_run`、`decisions`
- `artifacts`：交付文件绝对路径，Gateway 会上传给订阅者
- `created_cards`：本 run 内创建的子任务 ID，必须真实存在

如果需要人工介入，调用 `kanban_block(reason=...)`，`kind` 可选：

- `dependency`：等上游任务，自动恢复
- `needs_input`：需要人工决策
- `capability`：硬性阻断（缺凭据、无权限）
- `transient`：临时故障，可能自愈

---

## 四、工作区约定：Scratch / Dir / Worktree

`workspace_kind` 决定了 Worker 的文件操作边界：

| kind | 行为 |
|------|------|
| `scratch` |  fresh tmp dir，默认值 |
| `dir` | 指定共享目录，需要 `workspace_path` |
| `worktree` |  git worktree，用于项目关联任务 |

**Worktree 的特殊性**：

- 分支名通常是 `<project-slug>/<task-id>`
- 主仓库在 workspace 的上两级
- 需要先 `git worktree add`，再 `cd` 进去

这是 Hermes 支持**并行代码修改不冲突**的核心机制。

---

## 五、Fan-out / Fan-in：Orchestrator 模式

Orchestrator 的任务不是执行，而是**路由**：

1. 读取上游 handoff（`parent-task` 的 summary + metadata）
2. 用 `kanban_create` 创建多个子任务，显式指定 `assignee` 和 `parents=[...]`
3. 子任务全部 `done` 后自动 promoted 到 `ready`
4. Orchestrator 自己的任务 `kanban_complete` 结束

**规则**：

- Orchestrator 可以再 delegat，但当前用户配置 `max_spawn_depth=1`，所以实际上会被静默降级为 leaf
- 不要自己执行子任务，不要 scope creep
- 每个子任务必须有真实 assignee，否则永远不 dispatch

---

## 六、Kanban Swarm  vs 其他并发原语

| 场景 | 推荐工具 | 原因 |
|------|----------|------|
| 快速并行子任务（分钟级） | `delegate_task` | 开销小，无需持久化 |
| 需要跨会话/跨进程持久化 | `kanban_create` | Board 是 durable store |
| 定时触发 | `cronjob` | 调度器原生支持 |
| 长时间后台运行 | `terminal(background=True)` | 防止主进程退出丢失 |

**核心区别**：`delegate_task` 是进程内并发，Kanban 是跨进程/跨 profile 的 durable queue。

---

## 七、关键约束与坑点

### 7.1 禁止伪造结果

如果你调用了 web hook、文件上传、HTTP POST 等有副作用的操作，必须验证真实结果（URL 可达、文件存在、HTTP status），不能凭“看起来成功了”就 complete。

### 7.2 不要发明 Card ID

`created_cards` 只能填真实 `kanban_create` 返回的 ID，内核会校验， phantom ID 会直接拒绝 complete。

### 7.3 Review 流程

如果产出需要人工 review（大多数代码任务），正确流程是：

1. `kanban_comment` 贴出 `changed_files` / diff 路径
2. `kanban_block(reason="review-required: <one-line summary>")`
3.  reviewer  approve 后 unblock，worker 继续或 complete

不要直接 complete 一个还没被 review 的改动，这不诚实。

### 7.4 跨 Profile 写保护

默认情况下，Worker 只能写自己 profile 的 `skills/`、`plugins/`、`cron/`、`memories/`。要写其他 profile，必须显式传 `cross_profile=True`，且最好有用户明确授权。

---

## 八、实战建议：如何组织一个 Swarm

### 8.1 分解任务

把一个大目标拆成**独立、可验证、有明确 assignee** 的子任务。每个子任务：

- 标题是动作导向的
- body 包含完整 spec + acceptance criteria
- 指定具体的 profile（不要留空）

### 8.2 表达依赖

用 `parents=[...]` 而不是 prose。依赖图让 Dispatcher 自动处理 fan-in，不需要人工去“等 A 做完再启动 B”。

### 8.3 心跳策略

- 短任务（< 30 分钟）：不需要心跳
- 中等任务（30m - 2h）：每 30 分钟一次
- 长任务（> 2h）：每小时至少一次，包含进度 note

### 8.4 交付物

- 真正需要的文件放 `artifacts`
- 中间产物、scratch 文件不要放
- 如果交付物在 workspace 内，用绝对路径

---

## 九、总结

Kanban Swarm 不是“多个 Agent 同时跑”这么简单。它是一个**有状态、有边界、有调度协议**的协作系统：

- **Durable**：Board  survives 进程重启
- **Isolated**：Task / Profile / Tenant 三层隔离
- **Verifiable**：Heartbeat、Complete、Block 都是可观测的状态机
- **Resumable**： reclaimed 任务不扣失败，只是重新排队

对于 AI 开发者来说，真正要掌握的不是 API 调用，而是**如何设计可分解、可验证、可恢复的任务图**——这和写好的异步代码是一样的思维模型。

---

## 参考

- Hermes Agent 官方文档：https://hermes-agent.nousresearch.com/docs
- Hermes Agent GitHub：https://github.com/NousResearch/hermes-agent
- 本文基于 Hermes Agent v0.16 的 `hermes-agent` skill 编写
