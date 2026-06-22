# 第2章 ReAct 主循环——Agent 的心脏

> 源码位置：`src/agentscope/agent/_agent.py`（全项目最大文件，~2567 行）。这一章是整个框架的心脏，所有其他机制（记忆、权限、压缩）都挂在这个循环上。

## 一、概念引入

**ReAct = Reasoning + Acting**。核心思想：让模型"想一想，做一步，再想一想"。区别于一次性 prompt-completion，ReAct 是个循环：

```
循环直到完成或超过上限:
    1. Reasoning: 把当前上下文喂给模型，模型返回"下一步要做什么"
    2. 如果模型说"我要调工具" → Acting: 执行工具，把结果塞回上下文
    3. 如果模型说"我做完了"(纯文本回答) → 结束
```

听起来简单，但工程上有无数细节：循环什么时候停？多个工具调用怎么并发？工具要等人确认怎么办？超过最大轮次怎么办？流式输出怎么变成事件？AgentScope 把这些都收敛到一个 `_reply_impl` 方法里。

## 二、源码地图

```
agent/
├── _agent.py      Agent 类 + _reply_impl（主循环）+ 工具执行 + 上下文压缩
├── _config.py     ReActConfig / ContextConfig / ModelConfig
└── _utils.py      _ToolCallBatch（批次数据结构）
```

入口：`Agent.reply()`（`_agent.py:215`）或 `Agent.reply_stream()`（`_agent.py:190`），两者都委托给 `_reply`（`:496`，中间件包装）→ `_reply_impl`（`:541`，核心循环）。

## 三、经典问题

1. ReAct 循环的"想-做"是怎么组织的？一共有几个退出条件？
2. 每轮怎么决定是"想"还是"做"？模型输出怎么分类？
3. 多个工具调用是串行还是并行？怎么判断能不能并行？
4. 工具需要人确认时，循环怎么"暂停"和"恢复"？
5. 超过最大轮次会怎样？

## 四、源码剖析

### 问题 1：主循环结构

`_reply_impl`（`_agent.py:541-698`）分四步：

```
Step 1  检查 incoming event（_agent.py:568）
        ↓
Step 2  处理 event 或开始新回复（_agent.py:575-588）
        ↓
Step 3  while cur_iter < max_iters:（_agent.py:594-669）
          ├─ _check_next_action() → reasoning / acting / exit
          ├─ reasoning: compress + 调模型，无工具调用则结束
          └─ acting: 分批执行工具，遇 Require*Event 则暂停
        ↓
Step 4  超过 max_iters → ExceedMaxItersEvent + 结束（_agent.py:674-698）
```

🔑 **四个退出条件**（任一触发即 return）：

1. **模型给了最终回答**（reasoning 输出无 `ToolCallBlock`）→ `_agent.py:613-619`
2. **需要等待外部输入**（工具 ASKING/SUBMITTED）→ `_agent.py:666`
3. **`_check_next_action` 返回 exit**（有等待中的工具但无可执行的）→ `_agent.py:599-601`
4. **超过 `max_iters`**（默认 20，`_config.py:126`）→ `_agent.py:674-698`

### 问题 2：每轮怎么决策——`_check_next_action`

`_check_next_action`（`_agent.py:2266-2360`）扫描上一条 assistant 消息里**还没有对应 tool_result 的 ToolCallBlock**，按 state 分类：

| state | 含义 | 归类 |
|-------|------|------|
| `PENDING`/`ALLOWED` | 可执行 | executable |
| `ASKING`/`SUBMITTED` | 等待人/外部 | awaiting |

决策表（`_agent.py:2334-2360`）：

- 有 executable → `("acting", None)`
- 有 awaiting 但无 executable → `("exit", AssistantMsg("我在等 N 个确认..."))`
- 都没有 → `("reasoning", None)`（该调模型了）

💡 这个设计很妙：**决策完全基于上一条消息里的工具调用状态**，而不是靠额外标志位。因为状态机（第1章）已经把"进展到哪"编码进了 `ToolCallBlock.state`，循环只需读状态即可。这也是为什么 agent 可以跨进程重启续接——状态都在 context 里。

### 问题 3：Reasoning 步骤

`_reasoning_impl`（`_agent.py:754-880`）：

1. `yield ModelCallStartEvent`（`:777`）
2. `kwargs = await self._prepare_model_input()`（`:784`）—— 组装 `[SystemMsg, (summary), *context]` + 工具 schema（详见第4章）
3. `res = await self._call_model(**kwargs)`（`:787`）—— 带**重试**和**fallback 模型**（`:2056-2072`），以及 `on_model_call` 中间件链
4. **流式处理**（`:800-821`）：如果返回的是 async generator，逐 chunk 消费；保留最后一个 `is_last` 的 chunk 作为 `completed_response`，其余通过 `_convert_chat_response_to_event`（`:2395`）转成事件 yield 出去
5. **关闭未结束的 block**（`:823-843`）—— 确保每个 `TextBlockStartEvent` 都有配对的 `EndEvent`
6. `yield ModelCallEndEvent`（含 token 用量，`:846`）
7. `_save_to_context(...)`（`:856`）—— 把模型输出存进 context（详见第4章）
8. **判断是否最终回答**（`:862-879`）：如果输出里没有 `ToolCallBlock` → 构造最终 `AssistantMsg` → 循环上层 yield `ReplyEndEvent` 并 return

🔑 **流式的关键**：模型输出是"边生成边来"的，agent 用 `block_ids` 字典（`:792-797`）跟踪每个 text/thinking/tool/data block 的 id，在 chunk 之间正确发出 Start/Delta/End 事件。前端靠这些事件逐 token 渲染。

### 问题 4：Acting 步骤——批次与并发

`_batch_tool_calls`（`_agent.py:1101-1138`）把可执行的工具调用分组：

```python
for tool_call in executable_calls:
    tool = await self.toolkit.get_tool(tool_call.name)
    if tool is None or tool.is_concurrency_safe:
        放进 "concurrent" 批次
    else:
        放进 "sequential" 批次
```

💡 **并发判定只看 `is_concurrency_safe`，不看 `is_read_only`**。这两个属性职责不同：
- `is_concurrency_safe` → 决定**能否和其他工具同时跑**（批次划分）
- `is_read_only` → 决定**权限引擎是否自动放行**（第8章）

⚠️ 常见误解：以为"只读=可并发"。实际上 `is_read_only` 的工具不一定 `is_concurrency_safe`，反之亦然。看 `tool/_base.py:97-120` 的默认值。

**串行执行**（`_execute_sequential_tool_calls`，`:1140`）：按顺序逐个跑，一旦遇到 `RequireUserConfirmEvent`/`RequireExternalExecutionEvent` 就 break，跳过后续。

**并发执行**（`_execute_concurrent_tool_calls`，`:1187`）：
- 用 `asyncio.gather(*tasks, return_exceptions=True)`（`:1242`）
- ⚠️ `return_exceptions=True` 很关键：某个工具失败不会取消其他工具，失败作为值返回
- 全部完成后，收集所有 exception，若有则 `raise ExceptionGroup(...)`（`:1266-1273`）
- 事件通过共享 `asyncio.Queue` 汇聚（`:1233`），sentinel 标记结束（`:1233-1236`）

🔑 这个设计保证：**并发批次里所有工具都跑完才返回**（sentinel 在 gather 之后放入），不会因为一个失败而丢失其他工具的事件流。

### 问题 5：单个工具的生命周期——`_execute_tool_call`

`_execute_tool_call`（`_agent.py:1292-1541`）是单工具执行的完整流程，五步：

```
Step 1  输入校验（:1331-1366）
        tool = toolkit.check_tool_available(...)
        parsed_input = _json_loads_with_repair(tool_call.input, schema)  # JSON 修复解析
        jsonschema.validate(...)
        失败 → _handle_error_tool_call(state=ERROR) → return

Step 2  权限检查（:1371-1381）
        若 state 已是 ALLOWED（用户确认过）→ 直接放行，跳过检查
        否则 decision = engine.check_permission(tool, parsed_input)

Step 3  按 decision 分支（:1388-1541）
        ├─ ASK/PASSTHROUGH: state=ASKING → yield RequireUserConfirmEvent → return（暂停）
        ├─ DENY:           _handle_error_tool_call(state=DENIED) → return
        └─ ALLOW:
             yield ToolResultStartEvent
             ├─ 外部工具: state=SUBMITTED → yield RequireExternalExecutionEvent → return（暂停）
             └─ 本地工具: _acting(tool_call)
                  ├─ ToolResponse(最终): 截断/offload → 存context → state=FINISHED → yield ToolResultEndEvent
                  └─ ToolChunk(中间): 转成 delta 事件流式 yield

Step 4-5 截断大结果 + offload（见第4章）
```

💡 **权限已允许的工具跳过重复检查**（`:1371-1376`）：用户确认过的工具，state 已是 `ALLOWED`，下次执行不再走 `check_permission`。这是 HITL 续接的效率优化。

### 问题 6：HITL 暂停与恢复

**暂停**（`_agent.py:1388-1404`）：
1. 把 `tool_call.state` 置为 `ASKING`（必须在 yield 之前！`:1394-1397`）
2. 附上 `decision.suggested_rules`（`:1399`）——给前端的"建议授权规则"
3. `yield RequireUserConfirmEvent(tool_calls=[tool_call])`
4. `return`

外层 `_reply_impl` 收到这个事件 → `break_execution=True`（`:646-653`）→ yield 一条 "Waiting for tool calls..." 的 AssistantMsg → return（`:657-666`）。**整个 reply 生成器结束**，控制权交还调用方（前端）。

**恢复**：前端带着 `UserConfirmResultEvent` 再次调 `reply`。
- `_reply_impl` Step 1 识别这是 incoming event（`:568`）
- `_handle_incoming_event`（`:1000-1045`）处理：
  - 用户**批准** → state 置 `ALLOWED`，可能更新 input，把接受的 rules 加进权限引擎（`:1014-1029`）
  - 用户**拒绝** → `_handle_error_tool_call(state=DENIED)`（`:1031-1042`）
- 回到 while 循环，`_check_next_action` 看到 `ALLOWED` 工具 → acting → 执行（权限检查被跳过）

🔑 **暂停/恢复的本质**：agent 不是"阻塞等待"，而是**结束当前 reply、靠 context 里的 state 标记记住进度**。下次 reply 从 state 续接。这让它天然支持跨进程（state 存储后任何进程都能续）。

### 问题 7：超过最大轮次

`max_iters` 默认 20（`_config.py:126`）。循环退出后（`:674-698`）：
1. `yield ExceedMaxItersEvent`（`:674`）
2. log warning（`:678`）
3. `yield ReplyEndEvent`（`:688`）—— ⚠️ 即使超限也要发 `ReplyEndEvent`，否则订阅终端事件的 SSE 客户端会一直挂起
4. `yield AssistantMsg("Executed maximum iterations...")`（`:693`）

⚠️ `reply()`（非流式）返回**最后一条 Msg**，所以超限时调用方拿到的是"Executed maximum iterations..."这条。`reply_stream()` 过滤掉 Msg，流式消费者只看到事件。

## 五、事件流全景

一次完整回复可能 yield 的事件（`event/_event.py:476-504` 定义了 26 种）：

```
ReplyStartEvent
├─ ModelCallStartEvent
│   ├─ TextBlockStart/Delta/EndEvent        (模型文本)
│   ├─ ThinkingBlockStart/Delta/EndEvent    (推理过程)
│   ├─ ToolCallStart/Delta/EndEvent         (模型要调工具)
│   └─ DataBlockStart/Delta/EndEvent        (模型输出图片等)
├─ ModelCallEndEvent                        (含 token 用量)
├─ ToolResultStartEvent
│   ├─ ToolResultTextDelta/DataDeltaEvent   (工具流式输出)
│   └─ ToolResultEndEvent                   (含 SUCCESS/ERROR/DENIED)
├─ [RequireUserConfirmEvent]                (HITL 暂停)
└─ ReplyEndEvent / ExceedMaxItersEvent
```

这些事件既给前端流式渲染（SSE），也折叠进最终 `Msg` 持久化（见第9章）。

## 六、本章小结

```
_reply_impl:
  while cur_iter < max_iters:
    action = _check_next_action()  ← 基于工具调用状态决策
    ├─ reasoning: compress → 调模型 → 流式事件 → 存context → 无工具则结束
    └─ acting:    分批(concurrent/sequential) → 每个工具5步(校验/权限/执行/截断/存)
                  遇 Require*Event → 暂停(靠 state 记忆进度) → 下次 reply 续接
  超限 → ExceedMaxItersEvent
```

🔑 三句话总结：
1. **ReAct 循环靠 `ToolCallState` 状态机驱动**，决策不靠标志位，靠读 context 里工具调用的状态——这让 agent 天然可中断、可恢复、可跨进程。
2. **工具并发只看 `is_concurrency_safe`**，用 `gather(return_exceptions=True)` 保证一个失败不连累其他。
3. **HITL 不是阻塞等待，而是"结束当前 reply + 靠 state 记进度"**，这是整个服务化架构（第7、10章）能跨进程调度的基础。

📝 **思考题**：如果模型在一轮里输出了 3 个工具调用，其中 2 个是只读可并发、1 个会写文件。`_batch_tool_calls` 会分成几批？执行顺序是什么？如果可并发的那个触发 ASK，另一个会跑吗？（提示：看 `_execute_concurrent_tool_calls` 的 break 逻辑 vs 串行的 break 逻辑——并发批次的语义和串行不同。）

下一章：[第3章 模型抽象与格式化](./03-模型与格式化.md)——看 `_call_model` 内部，统一 `Msg` 如何变成各家 API 请求。
