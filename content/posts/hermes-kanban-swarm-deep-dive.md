---
title: "深入 Hermes Agent v0.18 Kanban Swarm：多 Agent 协作的持久化任务编排引擎"
date: "2026-07-07"
tags: ["Hermes", "Kanban", "多Agent", "架构", "源码分析"]
author: "WalkerSun999"
---

> 当 `delegate_task` 不够用的时候 —— 一场从"函数调用"到"工作队列"的架构跃迁

---

如果你用过 Hermes Agent 的 `delegate_task`，你一定熟悉这个模式：主 Agent 发起一个子任务，等待结果返回，然后继续。这就像一个函数调用 —— fork，等待 join，拿到返回值。

但当你的 Agent 协作场景变成这样呢？

- 三个 researcher 并行调研不同维度，一个 analyst 汇总分析，一个 writer 生成最终报告
- 一个 coding agent 修改代码，另一个 review agent 检查质量，不满意就退回重来
- 凌晨 2 点的定时任务需要跨越 3 个 Agent 完成，中间任何一个挂了都能从断点续跑
- 人类用户在任务中途插入评论："这个假设不对，换个方向" —— Agent 需要读到这里并调整

这些场景有一个共同点：**它们不是一次性的 RPC 调用，而是持久化的、多角色的、有状态的协作流程。**

这就是 Hermes v0.18 引入 **Kanban Swarm** 的动机。而它用了一个非常巧妙的方式来实现 —— **不引入第二个调度器，而是在现有 Kanban 内核上写一个薄薄的拓扑层**。

---

## 一、源码导读：kanban_swarm.py 的 278 行

打开 `hermes_cli/kanban_swarm.py`，第一段 docstring 就说清楚了设计哲学：

```python
"""
This module intentionally does not introduce a second scheduler. It writes a
small task graph into the existing Kanban kernel:

    planning root (completed immediately)
        ├─ parallel specialist workers (ready)
        └─ verifier (todo until all workers done)
             └─ synthesizer (todo until verifier done)
"""
```

整个 Swarm 只有 **278 行 Python**。它做的事情非常简单：**把一张任务图写进已有的 Kanban 数据库，然后让 Kanban 的 Dispatcher 来执行**。

### 拓扑结构

Swarm 创建的任务图是固定的五层：

```
Root (立即完成, 变成共享 Blackboard)
 ├── Worker 1 (ready, 并行执行)
 ├── Worker 2 (ready, 并行执行)
 ├── Worker N (ready, 并行执行)
 └── Verifier (等待所有 Worker 完成)
      └── Synthesizer (等待 Verifier 通过)
```

这个拓扑通过 `task_links` 表实现依赖管理。Verifier 的 `parents` 是所有 Worker ID，Synthesizer 的 `parents` 是 Verifier ID。Kanban 的 Dispatcher 在每次 tick 时会检查依赖：只有所有 parents 都 done 了，子任务才会从 `todo` 提升到 `ready`。

### Blackboard：低科技的状态共享

多 Agent 协作最难的问题是什么？**状态共享。**

Swarm 的方案堪称"务实的优雅"：**把结构化 JSON 藏在 Root 任务的 Comment 里**。

```python
BLACKBOARD_PREFIX = "[swarm:blackboard] "
```

Worker 完成时写入 Blackboard，Verifier 读取所有 Blackboard 更新来做 Gate 判断，Synthesizer 读取 Blackboard 来合成最终产出。这一切都在已有的 `task_comments` 表里 —— Dashboard 能看到，通知系统能触发，slash command 能查询。**零额外基础设施。**

`latest_blackboard()` 函数将 Root 任务上所有带 `[swarm:blackboard]` 前缀的 Comment 合并成一个字典，后来的覆盖先来的，并记录每个 key 的 `_authors` 用于追溯：

```python
def latest_blackboard(conn, root_id):
    merged = {}
    authors = {}
    for comment in kb.list_comments(conn, root_id):
        if not comment.body.startswith(BLACKBOARD_PREFIX):
            continue
        payload = json.loads(comment.body[len(BLACKBOARD_PREFIX):])
        key = payload.get("key")
        merged[key] = payload.get("value")
        authors[key] = comment.author
    merged["_authors"] = authors
    return merged
```

---

## 二、Kanban vs delegate_task：一场范式转移

理解 Kanban Swarm 的最佳方式是把它和 `delegate_task` 放在一起对比：

| 维度 | `delegate_task` | Kanban Swarm |
|---|---|---|
| **抽象** | RPC 调用 (fork → join) | 持久化消息队列 + 状态机 |
| **生命周期** | 父进程存活期间 | 跨越进程重启 |
| **子任务身份** | 匿名 subagent | **具名 Profile**，拥有独立记忆和技能 |
| **可恢复性** | 失败即丢弃 | block → unblock → 重新调度 |
| **人类介入** | 不支持 | 随时通过 Comment 介入 |
| **审计追踪** | 上下文压缩后丢失 | SQLite 永久存储 |
| **协作模式** | 层级式 (caller → callee) | **对等式** (任何 Profile 可读写任何 Task) |

一句话总结：**`delegate_task` 是函数调用；Kanban Swarm 是工作队列，每个 handoff 都是一行持久化记录，任何 Agent（或人类）都能看见和编辑。**

更重要的是，两者**可以嵌套使用**：一个 Kanban Worker 在执行过程中可以调用 `delegate_task` 做子任务拆分。

### 为什么 Profile 如此重要？

`delegate_task` 的 subagent 是"匿名"的 —— 它没有记忆，没有技能积累，每次都是从零开始。

而 Kanban Worker 是一个**具名 Profile**。这意味着：

- **持久记忆**：Worker 的 Profile 拥有独立的 `state.db`，跨任务积累知识。一个 `researcher` 执行 50 次调研任务后，它的记忆里会有越来越丰富的领域知识。
- **独立技能**：每个 Profile 可以安装不同的 Skills。`writer` 加载 `blog-writer` 和 `humanizer`，`reviewer` 加载 `requesting-code-review`。
- **独立会话**：每个 Worker 有完整的 Agent Loop，可以使用所有工具（terminal, file, web, browser 等）。

---

## 三、深入 Dispatcher：Gateway 内的调度引擎

Swarm 的任务图写完后，谁来执行？**Dispatcher。**

Dispatcher 不是一个独立进程，它运行在 **Gateway 内部**（`kanban.dispatch_in_gateway: true`，默认开启）。每 N 秒（默认 60），它执行一次 sweep：

1. **Reclaim 过期 Claim**：检查 `running` 状态但 Worker 进程已死的任务，回收 Claim
2. **Promote 就绪任务**：检查 `todo` 状态的任务，如果所有 parent 都 `done`，提升到 `ready`
3. **原子 Claim**：从 `ready` 队列中取任务，原子性地标记为 `running`
4. **Spawn Worker**：启动指定 Profile 的完整 Agent 进程，注入 `HERMES_KANBAN_BOARD` 和 `HERMES_KANBAN_TASK` 环境变量

关键设计：

```python
# 故障保护：连续失败 N 次后自动 Block
# kanban.failure_limit 默认 2
# 防止配置错误的 Profile 无限重试
```

Worker 被启动后，它拿到的是一个**受限的工具集**（`kanban_*` 工具），包括 `kanban_show`、`kanban_complete`、`kanban_block`、`kanban_comment`、`kanban_heartbeat` 等 —— 它通过这些工具与 Board 交互，而不是 shell out 到 CLI。

---

## 四、实战：用 Swarm 写一篇技术博客

理论讲完了，来看一个真实的工作流。假设我们要用 Kanban Swarm 完成一篇技术博客的协作写作：

```bash
# 1. 创建 Board
hermes kanban boards create blog --name "技术博客工作板"

# 2. 启动协作拓扑
hermes kanban swarm \
  --worker researcher:"调研 Kanban Swarm 架构与源码":"web-research" \
  --worker researcher:"调研多 Agent 编排竞品对比":"web-research" \
  --verifier reviewer \
  --synthesizer publisher \
  "撰写 Hermes Agent v0.18 Kanban Swarm 深度技术文章，面向 AI 开发者"
```

这一步创建了以下任务图：

```
Root: "Swarm: 撰写 Hermes Agent v0.18 Kanban Swarm..."
  ├── Worker 1: "调研 Kanban Swarm 架构与源码" → researcher (并行)
  ├── Worker 2: "调研多 Agent 编排竞品对比" → researcher (并行)
  ├── Verifier: "Verify swarm outputs" → reviewer (依赖 workers)
  └── Synthesizer: "Synthesize swarm outputs" → publisher (依赖 verifier)
```

**运行会看到什么？**

1. **毫秒级**：Root 任务被创建并立即 `complete`，拓扑元数据写入 Blackboard
2. **几秒后**：Dispatcher sweep 发现 2 个 Worker 处于 `ready`，原子性 Claim 并分别启动 `researcher` Profile 的 Agent 进程
3. **研究阶段**：两个 research worker 并行工作 —— 一个深入读源码，一个调研竞品。每个完成时写入 Blackboard
4. **Gate 阶段**：两个 Worker 都 `done` 后，Verifier 被 promote → claim → spawn。Verifier 读取所有研究结果，判断是否充分，不充分就 block 并写明缺失内容
5. **合成阶段**：Verifier 通过后，Synthesizer 被激活，读取完整 Blackboard，生成最终文章

**中途出错了怎么办？**

如果 research worker 崩溃了，Dispatcher 会检测到 stale claim，回收任务并重新调度。如果连续失败，自动 block（防止无限重试）。人类可以随时通过 `hermes kanban comment <task_id>` 插入指导意见。

---

## 五、设计哲学：薄层胜过重框架

读完源码后，我认为 Kanban Swarm 最值得学习的地方是它的**克制**：

1. **不造新轮子**：Swarm 不引入新调度器、新存储、新协议。它只是在 Kanban 的 `create_task` + `link` + `complete` 之上加了一层拓扑编排
2. **Blackboard 是 Comment**：状态共享直接复用 `task_comments` 表。没有任何新表、新 API、新服务
3. **Profile 是一等公民**：每个 Worker 是独立 Profile，有独立记忆和技能 —— 这天然解决了 Agent 协作中的"上下文隔离"问题
4. **Dispatcher 藏在 Gateway 里**：不额外占用进程，不引入新的守护进程。Gateway 已经在运行了，Dispatcher 只是它的一个定时循环

这些设计决策让整个系统保持**可观测、可调试、可恢复**。你可以用 `hermes kanban list` 看任务状态，用 `hermes kanban show <id>` 看完整上下文，用 `hermes kanban tail <id>` 实时跟踪事件流。**没有黑盒。**

---

## 六、什么时候用 Swarm，什么时候不用？

### ✅ 适合 Swarm 的场景

- **研究-分析-写作管道**：多个平行 researcher → reviewer gate → synthesizer
- **代码审查流水线**：implementer → reviewer → merger
- **定时深度简报**：每天凌晨 6 点启动 Swarm，采集 N 个数据源，汇总成日报
- **需要人类介入的工作流**：Agent 做到一半 block 住，等待人类审批后继续
- **跨 Session 的长期任务**：任务可能需要几天，中间可以重启 Hermes 甚至重启电脑

### ❌ 不适合的场景

- **简单的一次性查询**：`delegate_task` 更快更轻
- **实时对话式协作**：Agent 需要立即响应你的追问
- **单个 Agent 能独立完成的工作**：不要为了用 Swarm 而用 Swarm

---

## 七、源码中的彩蛋

在 `kanban_swarm.py` 的最后，有一个精心设计的 `parse_worker_arg` 函数：

```python
def parse_worker_arg(raw: str) -> SwarmWorkerSpec:
    parts = [p.strip() for p in raw.split(":", 2)]
    if len(parts) < 2:
        raise ValueError("worker must be profile:title or profile:title:skill,skill")
    skills = []
    if len(parts) == 3 and parts[2]:
        skills = [s.strip() for s in parts[2].split(",") if s.strip()]
    return SwarmWorkerSpec(profile=parts[0], title=parts[1],
                           body=parts[1], skills=skills)
```

这个 CLI 参数解析器让用户可以用一行字符串定义 Worker：`researcher:调研源码:web-research,github-code-review`。简单、直观、可读。

而 `create_swarm` 函数的 **幂等性保护** 也值得一提：

```python
existing = latest_blackboard(conn, root).get("topology")
if isinstance(existing, dict):
    # 检测到已有的 Swarm 拓扑，直接返回已有 ID，不重复创建
    return SwarmCreated(root, worker_ids, verifier_id, synthesizer_id)
```

通过 `idempotency_key` 机制，同一个 Swarm 不会因为重复执行 CLI 命令而被创建两次。这在自动化脚本中至关重要。

---

## 结语

Kanban Swarm 不是 Hermes 最耀眼的功能，但它可能是**最有架构参考价值的**。在一个 Agent 框架普遍追求"更大、更多、更复杂"的趋势下，Hermes 团队选择了一条相反的路径：**用 278 行代码，在已有的基础设施上写一个薄层，解决一个真问题。**

对于那些正在构建多 Agent 系统的开发者来说，Kanban Swarm 提供了一个值得深思的范式：**持久化状态 + 对等协作 + 人类可介入 = 可靠的 Agent 工作流。**

而它的 Blackboard 模式 —— 用 Comment 做状态共享 —— 可能是整个设计中最聪明的部分。不需要 Redis，不需要消息队列，不需要 Raft 共识。一个 SQLite 数据库 + 约定好的 JSON 格式，就够了。

---

> 本文通过 Hermes Agent Kanban 流程完成：`researcher` Profile 调研 → `writer` Profile 撰写 → `reviewer` Profile 审核 → `publisher` Profile 发布。全部协作记录持久化在 `kanban.db` 中，可追溯、可审计。
