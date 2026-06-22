# 第7章 多 agent 调度

> 源码位置：`src/agentscope/app/_tools/`（Team 工具）、`src/agentscope/app/_manager/`（调度器/唤醒/取消）、`src/agentscope/app/middleware/_tool_offload_middleware.py`（后台任务）、`src/agentscope/app/message_bus/_base.py`（总线原语）。这是用户最关心的主题之一：**多个 agent 怎么协作、谁来唤醒谁**。

## 一、概念引入

单 agent 是"一个人想-做循环"。多 agent 要解决：

- **通信**：A 怎么把消息送给 B？
- **唤醒**：B 闲着，怎么知道该起来干活？
- **并发控制**：同一个 B 不能被两个进程同时跑（状态会冲突）。
- **跨进程**：B 可能在另一台机器上，怎么找到它？
- **长任务**：B 要跑 10 分钟，A 不能干等。
- **定时**：每天 9 点让 agent 检查邮件。

AgentScope 的核心设计思想是一句话：

🔑 **所有跨 agent 的动作，都收敛到同一个"两原语"投递模式：`inbox_push` + `enqueue_wakeup`。** 无论团队消息、子 agent 首次任务、后台工具完成、cron 触发，全走这条路。这让多 agent 系统的一致性和跨进程能力有统一保证。

## 二、源码地图

```
app/
├── storage/_model/_team.py        TeamData / TeamRecord
├── _tools/
│   ├── _team_create.py            创建团队（自己变 leader）
│   ├── _agent_create.py           创建 worker（继承 leader 配置）
│   ├── _team_say.py               发消息（inbox+wakeup）
│   └── _team_tool_base.py         团队工具基类（前置校验）
├── _service/_toolkit.py           按角色挂工具（leader vs worker）
├── _manager/
│   ├── _scheduler/_scheduler_manager.py   cron 调度
│   ├── _wakeup_dispatcher.py              唤醒分发
│   ├── _cancel_dispatcher.py              取消分发
│   ├── _background_task_manager.py        后台任务
│   └── _chat_run_registry.py              进程内运行注册
├── middleware/
│   ├── _inbox_middleware.py       drain inbox（第5章已讲）
│   └── _tool_offload_middleware.py  慢工具转后台
└── message_bus/_base.py           inbox/wakeup/lock 原语
```

## 三、经典问题

1. 多 agent 是什么拓扑？能网状通信吗？
2. leader 创建一个 worker 时，worker 的模型/权限/工作区从哪来？
3. A 给 B 发消息，B 什么时候、怎么收到？为什么不是同步调用？
4. 怎么保证同一个 session 不会被两个进程同时跑？
5. 一个工具要跑 10 分钟，agent 怎么不阻塞？
6. 怎么让 agent 定时执行？定时任务跨重启怎么办？
7. 怎么取消一个正在跑的 agent？

## 四、源码剖析

### 问题 1：拓扑——星型 leader-worker

AgentScope 的多 agent 是**星型（hub-spoke）**，不是网状：

- **leader**：用户直接对话的 agent，能创建团队、创建 worker、给 worker 派活。
- **worker**：被 leader 创建的子 agent，只能给 leader 汇报，**不能**互相通信。

证据在工具挂载（`_service/_toolkit.py:144-168`）：
- worker（`agent_record.source == "team"`）只挂 `TeamSay(role="worker")`（`:158`）。
- 其他人挂全套：`TeamCreate` / `AgentCreate` / `TeamSay(role="leader")` / `TeamDelete`（`:160-168`）。

`AgentCreate` 的描述（`_agent_create.py:164`）甚至明确**劝阻** worker 间直接通信和"集成者"角色，以保持拓扑简单。

💡 为什么选星型？因为网状多 agent 容易陷入"互相调用死循环"，且难以归因。星型让 leader 是唯一协调者，worker 是执行者，职责清晰。`TeamData`（`_team.py:8`）只存 `member_ids`，没有 worker 间关系。

⚠️ **team 成员是 session-scoped**（`_team.py:39-46`）：leader 由 `session_id` 标识（一个用户 agent 可在不同 session 领导多个团队），worker 由 agent_id 标识（worker 是 1:1 agent↔session）。

### 问题 2：worker 的配置继承

`AgentCreate.__call__`（`_agent_create.py:245-527`）创建 worker 时，**继承 leader 的关键配置**（`:460-474`）：

| 配置 | 来源 | 行号 |
|------|------|------|
| `workspace_id` | leader 的 workspace | `:464` |
| `chat_model_config` / `fallback` | leader 的模型配置 | `:466-471` |
| permission_context | 模板 + leader 运行时合并 | `:57-102` |
| system_prompt | 模板渲染（含 team_name/leader_name 占位符） | `:416-422` |

🔑 **worker 共享 leader 的 workspace**——同样的工具、技能、MCP、工作目录。worker **没有自己的凭据**，复用 leader 的模型绑定（凭据在 `get_model` 时按 session 的 model_config 从 storage 解析）。

**权限合并**（`_merge_leader_permissions`，`:57-102`）三个 flag：
- `override_leader_mode=False`（默认）→ worker 继承 leader 的权限模式。
- `extend_leader_working_directories=True` → 合并 leader 的工作目录。
- `extend_leader_permission_rules=True` → leader 的规则追加在模板规则**之后**（模板规则优先，但 worker 不必为 leader 已授权的操作重新询问）。

💡 这套继承让"leader 授权过的，worker 直接能用"，避免每个 worker 都来烦用户。但模板规则优先，保证 worker 的权限边界不被 leader 放大。

worker 创建后，`prompt`（初始任务）作为 `<team-message>` HintBlock 推进 worker inbox + 唤醒（`:489-511`）——**worker 的第一次执行就是一次 wakeup 驱动的、drain inbox 的 run**。

### 问题 3：消息投递——inbox + wakeup 两原语

`TeamSay.__call__`（`_team_say.py:117-329`）的投递循环（`:305-311`）是整个多 agent 系统的心脏：

```python
for sid, aid in recipients:
    await self._message_bus.inbox_push(sid, payload)        # 1. 推消息
    await self._message_bus.enqueue_wakeup(user_id, sid, aid)  # 2. 唤醒
```

🔑 **为什么分两步？**
- `inbox_push` 只是把消息塞进 B 的队列（Redis），**不触发 B 执行**。
- `enqueue_wakeup` 发一个信号，让某个进程的 dispatcher 去拉起 B 的 run。
- 解耦的原因：B 可能正在忙、可能在别的进程、可能不存在了。投递方不该阻塞等待。

**路由按名字，不按 agent_id**（`:197-230`）：因为 worker 在 `<team-message from="leader">` 里只看到 leader 的**名字**，看不到 agent_id。所以 `TeamSay` 要建一个 name→(session_id, agent_id) 目录来解析收件人。

**B 怎么收到**——`InboxMiddleware.on_reasoning`（第5章已讲）：B 每次 reasoning 开头 `inbox_drain`，把队列消息转成 HintBlock 注入 context。所以 B **必须正在跑**才会 drain——这就是为什么需要 wakeup 把 B 拉起来。

### 问题 4：唤醒与并发控制

`enqueue_wakeup`（`message_bus/_base.py:759`）做两件事：
1. `queue_push` 到共享队列 `agentscope:wakeups`（ack-on-read drain 队列，`:88-151`），携带 `{user_id, session_id, agent_id}`。
2. `publish` 一个空信号到 `agentscope:wakeup_signal`（fire-and-forget 广播，`:254-308`）。

`WakeupDispatcher`（`_wakeup_dispatcher.py`）每个进程一个长任务：
- 订阅 `wakeup_signal`（`:105-123`）。每来一个信号 → `_drain_and_dispatch`。
- `_drain_and_dispatch`（`:125-191`）：drain 最多 64 个 wakeup → 对每个：
  - **跳过正在运行的 session**：`session_is_running` 查分布式锁（`:145`）。
  - **孤儿保护**：session 已被删除 → 丢弃，不崩（`:148-171`）。
  - `ChatRunRegistry.spawn(ChatService.run(input_msg=None), session_id)`（`:174-183`）——拉起一次"继续当前状态/drain inbox"的 run。

🔑 **at-most-one-run 双层保证**：
- **集群级**：`MessageBus.session_run`（`_base.py:480-514`）对 `agentscope:session:lock:{sid}` 加分布式锁，租约 600 秒。两个进程抢同一 session，第二个排队等。
- **进程级**：`ChatRunRegistry.spawn`（`_chat_run_registry.py:72`）若该 session 已有未完成的任务 → 抛 `RuntimeError`，dispatcher 捕获并记日志（`_wakeup_dispatcher.py:184-191`）。

💡 这种"投递即走、dispatcher 拉起、锁防重"的设计，让多 agent 通信**天然异步、跨进程、无中心调度器**。任何进程的 dispatcher 都能拉起任何 idle session。

⚠️ **parked-on-tool-call 保护**（`_chat.py:346-363`）：如果一个 wakeup 驱动的 run（`input_msg=None`）发现 agent 正停在 `ASKING`/`SUBMITTED` 的工具调用上（等人确认/等外部结果），**跳过这次 run**——inbox 继续排队，等用户确认或外部结果来了再自然 drain。避免把一个等人确认的 agent 强行推进。

### 问题 5：慢工具转后台

`ToolOffloadMiddleware`（`_tool_offload_middleware.py`，`on_acting` 在 `:104`）处理**执行慢**（不是结果大）的工具。默认超时 10 秒（`:78`）。

流程（`:104-395`）：
1. 把工具的 `next_handler` 包进一个 asyncio task，输出 drain 进 queue（`:166-209`）。
2. 滚动 deadline 消费 queue。**按时完成** → 正常 yield（`:216-220`）。
3. **超时**（`:222-395`）：running task **不取消**，而是：
   - 起 `_deliver_when_done`（`:238-357`）等它跑完，结果通过 **inbox + wakeup** 送回（同样的两原语！）。
   - 把 task 注册到 `BackgroundTaskManager`（`:358-364`），拿 `task_id`。
   - 给 agent 一个**合成占位结果**（`:386-395`）："工具在后台跑，别轮询"——agent 循环立即解阻塞。

🔑 这就是第4章说的"慢工具 offload"。它和"大结果截断"是两套机制：**慢 → 后台任务 + 唤醒回送；大 → 截断 + 文件 offload**。两者都用"占位 + 异步补真值"的思路，但通道不同。

`BackgroundTaskManager`（`_background_task_manager.py`）双账本：
- 本地 `OrderedDict` 持有 asyncio.Task 引用（`:239`），供 cancel/shutdown。
- 全局 Redis hash `agentscope:bg_tasks:{sid}`（`:857-879`，TTL 24h），让任意进程能发现。

### 问题 6：定时调度

`SchedulerManager`（`_scheduler/_scheduler_manager.py`）封装 APScheduler 的 `AsyncIOScheduler`（`:57`）。

**触发**（`_build_trigger`，`:94-259`）：
- **stateful**：复用固定 session `f"{record.id}_stateful"`，跨触发保持记忆（`:140-185`）。
- **non-stateful**：每次触发新建 session，无记忆（`:186-207`）。
- 权限模式默认 `DONT_ASK`（定时任务没人在场确认）。
- 把 `description` 包成 `<scheduled-task>` HintBlock，**同样走 inbox + wakeup**（`:235-243`）。

💡 触发**不直接调 ChatService**，故意绕道总线——让 cron 触发和团队消息/后台任务走**同一条投递路径**，一致性更强（类 docstring `:24-35`）。

**持久化与恢复**：
- 定时任务存 `ScheduleRecord`（storage，跨重启存活）+ APScheduler 内存 job（每进程，临时）。
- 启动时 `__aenter__`（`:63-82`）从 storage `list_all_schedules` + `restore`（`:344-361`），只重注册 **enabled** 的。

**cron 解析**手动 5 字段（`:295-301`）——因为 APScheduler 的 `CronTrigger.from_crontab` 不支持 `start_date`/`end_date`。

### 问题 7：取消

`CancelDispatcher`（`_cancel_dispatcher.py`）和 WakeupDispatcher 对称，但方向相反——**停止**工作：

订阅两个频道（`:103-178`）：
- `session:cancel`（`:103-118`）→ 取消该 session 的本地 chat run + **所有**它的后台任务（`_cancel_session`，`:124-147`）。
- `task:cancel`（`:153-178`）→ 只取消单个后台任务。

🔑 **广播 + 自选（broadcast-and-self-select）**：取消信号广播给所有进程，但只有"持有该任务的进程"会响应，其他忽略。因为投递方不知道哪个进程持有任务。

`ToolStop` 工具（`_background_task_manager.py:72-209`）三条路：
1. **本地**：task 在本进程且同 session → 直接 `asyncio_task.cancel()`（`:156-175`）。同 session 检查（`:159`）防跨 session 误杀。
2. **远程**：task 在全局注册但别的进程 → `bus.task_publish_cancel` 广播（`:179-198`）。
3. **未找到**（`:201-209`）。

## 五、端到端流程（leader 派活 → worker 汇报）

```
1. leader 调 AgentCreate(prompt="去查X")
   → 建 worker AgentRecord(source="team") + SessionRecord（继承 leader 配置）
   → worker 入 team.member_ids
   → 初始任务 HintBlock → worker inbox + wakeup

2. 某进程 WakeupDispatcher drain 到 wakeup
   → worker session idle → spawn ChatService.run(input_msg=None)
   → 抢分布式锁（at-most-one-run）

3. ChatService 组装 agent（挂 InboxMiddleware + worker 工具 TeamSay(role="worker")）
   → InboxMiddleware.on_reasoning drain inbox → <team-message from="leader"> 进 context
   → worker 开始干活

4. worker 调慢工具（>10s）→ ToolOffloadMiddleware 转后台
   → agent 拿占位结果解阻塞；真结果完成后 inbox+wakeup 回送

5. worker 干完，调 TeamSay(content="X的结果", to=leader名字)
   → 按名字路由 → leader inbox + wakeup

6. leader 的 WakeupDispatcher 拉起 leader run
   → InboxMiddleware drain → worker 汇报进 leader context → leader 继续协调
```

🔑 整条链路**只用了两个原语**：inbox_push + enqueue_wakeup。这是多 agent 系统一致性的根基。

📝 **思考题**：
1. 如果 leader 给 worker 发消息时，worker 正在忙（锁被占），会发生什么？wakeup 会丢吗？（提示：看 `_wakeup_dispatcher.py:145` 的跳过逻辑 + drain 队列的持久性）
2. 为什么定时任务默认 `DONT_ASK` 权限模式？如果用 `DEFAULT` 会怎样？
3. 后台任务跨进程取消靠"广播+自选"。如果持有任务的进程刚好挂了，取消信号会怎样？task 会在全局注册表里残留吗？（看 TTL `_BG_TASKS_TTL_SECS`）
4. 星型拓扑禁止 worker 间通信。如果你想要"worker 互相讨论"，在不改框架的前提下能怎么做？（提示：让 leader 转发）

## 六、本章小结

🔑 五句话：
1. **拓扑是星型 leader-worker**，leader 协调、worker 执行，禁止 worker 间直接通信。
2. **worker 继承 leader 的 workspace/模型/权限**，但权限模板优先、leader 规则补充。
3. **一切跨 agent 动作收敛到 `inbox_push` + `enqueue_wakeup` 两原语**——团队消息、首次任务、后台完成、cron 全走这条路。
4. **at-most-one-run 靠分布式锁（集群）+ ChatRunRegistry（进程）双层**，wakeup 天然异步跨进程。
5. **慢工具转后台 + 定时调度 + 取消**都复用同一总线，无中心调度器。

下一章：[第8章 权限系统](./08-权限系统.md)——看工具调用前的安全闸门。
