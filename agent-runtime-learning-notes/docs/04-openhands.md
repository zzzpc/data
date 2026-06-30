# 04. OpenHands SDK

## 一句话定位

OpenHands SDK 面向真实软件工程任务。它把 Agent、Conversation、Event、Workspace、Tool 和
Agent Server 分开，并使用持久事件日志承载执行历史。核心不再只是一个循环，而是一套可恢复、
可远程运行、可服务化的编码 Agent Runtime。

官方仓库：[OpenHands/software-agent-sdk](https://github.com/OpenHands/software-agent-sdk)  
官方文档：[OpenHands SDK 文档](https://docs.openhands.dev/sdk/)

## 1. 主要层次

| 层次 | 作用 |
| --- | --- |
| `Agent` | 构造模型输入、采样、把响应转成 Action |
| `Conversation` | 驱动循环、维护状态、暂停/确认/预算 |
| `Event` / `EventLog` | 记录 Message、Action、Observation、错误和状态变化 |
| `Workspace` | 文件、命令、Git 与运行环境抽象 |
| `Tool` | 将模型调用转成类型化 Action 并执行 |
| `Agent Server` | 远程会话、租约、API、持久化和运行管理 |

`Conversation` 才是调度器，`Agent.step()` 是一次决策；这比把所有责任放进 Agent 类更接近
操作系统的进程与调度器分工。

## 2. Conversation 主循环

源码入口：

- [`LocalConversation.run`](https://github.com/OpenHands/software-agent-sdk/blob/1c230d909eaa0fe9679d49540cb887c05f268add/openhands-sdk/openhands/sdk/conversation/impl/local_conversation.py#L1488)
- [`Agent.step`](https://github.com/OpenHands/software-agent-sdk/blob/1c230d909eaa0fe9679d49540cb887c05f268add/openhands-sdk/openhands/sdk/agent/agent.py#L578)

`LocalConversation.run()`：

1. 初始化 Agent、取消令牌和执行状态。
2. 检查 paused、stuck、finished。
3. 处理人工确认状态。
4. 在状态锁内调用 `agent.step()`。
5. 检查预算、最大迭代数和终止状态。
6. 重复，直到暂停、等待确认、卡住、失败或完成。

它专门处理并发用户消息：Agent 刚设为 Finished 时，如果新消息进入，可将状态恢复为 Idle，
下一轮继续处理，避免消息丢失。

## 3. Agent.step 做什么

`Agent.step()` 的主要路径：

1. 如果上轮有待确认 Action，先执行，不再采样。
2. 从 Conversation 的增量 View 构造模型消息。
3. 必要时先触发上下文 condensation。
4. 调 LLM，并传入 Tool definitions。
5. 按响应类型分发：
   - tool calls
   - 普通 content
   - 只有 reasoning 或空响应
6. Tool calls 变成 Action events，执行后产生 Observation events。

异步 `astep()` 可把阻塞 Tool 放入线程执行器，并用 `asyncio.gather` 调度并行调用。

## 4. 事件溯源

[`EventLog`](https://github.com/OpenHands/software-agent-sdk/blob/1c230d909eaa0fe9679d49540cb887c05f268add/openhands-sdk/openhands/sdk/conversation/event_store.py#L25)
是带锁的持久追加日志。每个 Event 有 ID 和顺序索引，并以 JSON 持久化。

典型链路：

```text
MessageEvent(user)
  -> ActionEvent(agent, tool_call_id)
  -> ObservationEvent(environment, tool_call_id)
  -> MessageEvent(agent)
```

事件日志的价值：

- 进程重启后恢复 Conversation。
- UI 增量订阅和回放。
- 审计某个工具为何执行。
- 用 `tool_call_id` 配对 Action 与 Observation。
- 从事件派生当前 State/View，而不是只保存最终快照。

代价是必须处理并发追加、幂等、事件版本迁移、损坏日志和派生视图重建。

## 5. Conversation State

执行状态包括 Idle、Running、Paused、Waiting for Confirmation、Finished、Stuck、Error 等。

状态机比“最后一条消息是不是 final”更厚，因为真实系统要支持：

- 暂停与恢复。
- 用户确认。
- 卡死检测。
- 最大预算和迭代限制。
- 并发消息。
- 取消。
- Hook 拒绝停止或阻止消息。

这就是从 Agent Demo 走向 Runtime 的关键变化。

## 6. Workspace 与隔离

[`LocalWorkspace`](https://github.com/OpenHands/software-agent-sdk/blob/1c230d909eaa0fe9679d49540cb887c05f268add/openhands-sdk/openhands/sdk/workspace/local.py#L17)
直接访问宿主文件和 Shell，适合开发与测试。仓库还提供 Docker/远程 Workspace，将命令执行、
文件操作和 Git 状态放到可替换边界后面。

Workspace 不等于 Tool：

- Tool 描述模型能请求什么。
- Workspace 决定动作实际在哪里运行。
- 安全策略决定动作是否可以运行。

即使 Tool schema 很窄，若最终拼成任意 Shell，能力面仍然很大。

## 7. 确认、Hook 与卡死检测

Confirmation mode 把 Action 生成和执行拆成两次 `run()`：

```text
第一次 run: 模型生成 Action -> WaitingForConfirmation
第二次 run: 执行待处理 Action -> Observation -> 继续
```

这要求待处理 Action 必须持久、可识别且不能重复执行。

Stuck detector 从最近事件识别重复 Action/Observation 模式。它属于调度器而不是 Prompt：
模型也许认为自己仍在努力，但 Runtime 可以基于历史模式强制停机。

## 8. Agent Server

Agent Server 把本地对象提升为服务：

- Conversation CRUD 与事件流。
- 远程 Workspace。
- 会话所有权和 Lease。
- 后台执行与取消。
- 配置、Profile、凭据和持久化。

Lease 很像分布式锁：同一 Conversation 不能被多个 Worker 无协调地同时推进。服务化后，
“谁有权运行下一步”与“模型下一步做什么”同样重要。

## 9. 最值得学习的设计

- Agent 决策与 Conversation 调度分离。
- 用 EventLog 保存事实，用 State/View 保存派生状态。
- Action 与 Observation 用 ID 配对。
- Workspace 是独立运行环境边界。
- 人工确认被建模为持久状态。
- 卡死检测、压缩、预算和并发消息进入主循环。
- 本地和远程 Conversation 使用相近接口。

## 10. 建议源码阅读顺序

1. `openhands-sdk/openhands/sdk/conversation/impl/local_conversation.py`
2. `openhands-sdk/openhands/sdk/agent/agent.py`
3. `openhands-sdk/openhands/sdk/event/`
4. `conversation/state.py` 与 `event_store.py`
5. `tool/` 与 `workspace/`
6. `condenser/`、`stuck_detector.py`、Hook
7. `openhands-agent-server/`

## 11. 动手练习

1. 从 EventLog 重建某一时刻的 State。
2. 在 Action 生成后崩溃并恢复，验证不会重复执行。
3. 用同一 Agent 分别运行 LocalWorkspace 和 DockerWorkspace。
4. 给 Conversation 增加 Lease，模拟两个 Worker 竞争推进。

