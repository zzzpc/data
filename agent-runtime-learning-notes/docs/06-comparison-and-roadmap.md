# 06. 横向比较与学习路线

## 1. 核心架构比较

| 维度 | mini-swe-agent | smolagents | Strands | OpenHands | Codex |
| --- | --- | --- | --- | --- | --- |
| 核心抽象 | Agent + Model + Env | MultiStepAgent | 异步事件循环 | Conversation + Event | Session + Thread/Turn/Item |
| 动作形式 | Shell command | Tool call / Python | Tool use | 类型化 Action | Response Item / Tool call |
| 状态记录 | 线性 messages/trajectory | Step memory | Messages + state/session | 持久 EventLog | History + rollout + events |
| 执行环境 | 本地/可替换 Env | Python executor | Tool executor/sandbox | Workspace | 跨平台 sandbox/runtime |
| 审批控制 | 很少 | 需自行补充 | Intervention/HITL | Confirmation mode/Hooks | Policy + approval + sandbox |
| 上下文管理 | 完整线性历史 | Step -> messages | Conversation/context manager | View + condenser | Context manager + compaction |
| 并发 | 基本无 | 并行 Tool 线程 | 异步、并发 Tool | 并行 Tool/服务 Worker | 异步 Session/Tool/多 Agent |
| 多 Agent | 无内建核心 | managed agent as Tool | Tool/Graph/Swarm/A2A | sender/远程会话能力 | 独立子 Thread 调度 |
| 服务化 | CLI/评测为主 | 需上层封装 | 可嵌入应用 | Agent Server | App Server/MCP/CLI/TUI |
| 最适合学习 | 最小闭环 | Step 与 CodeAct | 可扩展事件 Runtime | 事件溯源与工作区 | 生产级编码 Agent 内核 |

## 2. 五个主循环的演进

```text
mini-swe-agent
  query -> shell -> observation

smolagents
  task -> planning? -> ActionStep -> tool/code -> FinalAnswerStep

Strands
  stream -> event_loop_cycle -> model events -> tool events -> recurse/stop

OpenHands
  Conversation.run -> Agent.step -> ActionEvent -> ObservationEvent -> state transition

Codex
  Submission -> Session dispatch -> Turn step loop
             -> Tool Router -> policy/approval/sandbox
             -> Item/Event -> client and persistent history
```

复杂框架并没有抛弃最小循环，只是把每个箭头展开成可观察、可恢复、可控制的子系统。

## 3. “Agent OS”对应关系

| 操作系统概念 | Agent Runtime 对应物 |
| --- | --- |
| 用户进程 | 模型驱动的 Agent/子 Agent |
| 系统调用 ABI | Tool schema、MCP、function calling |
| 内核调度器 | Conversation/Session/Turn loop |
| 进程控制块 | Agent state、TurnContext、ConversationState |
| 地址空间/保护域 | Workspace、container、sandbox、writable roots |
| 权限与提权 | policy、approval、intervention |
| 文件系统/日志 | EventLog、trajectory、rollout |
| 内存管理 | context window、compaction、offloading |
| 信号 | interrupt、cancel、pause、steer |
| IPC | Agent as Tool、A2A、inter-agent message |
| 监控与性能计数器 | trace、metrics、token/cost budget |

类比的边界也要记住：模型不是确定性进程，Prompt 不是安全策略，token context 也不是可靠数据库。

## 4. 设计自研 Runtime 时的递进路线

### Level 1：最小可运行闭环

实现 Model、Tool/Environment、消息列表、步数限制和 trajectory。参考 mini-swe-agent。

验收标准：

- 工具错误能反馈给模型。
- 超时会杀掉子进程。
- 每次运行可完整重放输入、动作和输出。

### Level 2：结构化步骤与执行器

增加 Task/Plan/Action/Observation/Final Step，分离内部记录和模型消息，引入 Tool registry 与
可替换执行器。参考 smolagents。

验收标准：

- Step 可以序列化。
- Tool call 有唯一 ID。
- 本地与隔离执行器可以切换。

### Level 3：事件化与扩展点

把同步循环变成异步事件流，增加 Hook、Intervention、Session、Checkpoint、Telemetry。
参考 Strands。

验收标准：

- UI 可以实时订阅。
- checkpoint 恢复不会重复副作用。
- 授权策略无需修改主循环。

### Level 4：事件溯源与服务化

引入持久 EventLog、派生 State/View、Workspace、Conversation Lease 和远程 Agent Server。
参考 OpenHands。

验收标准：

- Worker 崩溃后能恢复。
- 多 Worker 不会同时推进同一会话。
- 事件可迁移、审计和回放。

### Level 5：生产级控制面

引入稳定 Thread/Turn/Item 协议、Submission/Event 通道、动态 Tool Router、审批、跨平台沙箱、
中途 steering、compaction 和多 Agent 配额。参考 Codex。

验收标准：

- UI 与 Runtime 解耦。
- 权限与环境在每个 Step 一致。
- 中断、追加输入和审批不会破坏 Tool call 配对。
- 子 Agent 有独立生命周期和资源上限。

## 5. 推荐的四周源码计划

| 周 | 目标 | 产出 |
| --- | --- | --- |
| 第 1 周 | mini-swe-agent + smolagents | 自己实现 200 行最小 Agent，并加入 Step log |
| 第 2 周 | Strands | 异步事件流、Hook、Checkpoint 原型 |
| 第 3 周 | OpenHands | EventLog + State reducer + Workspace |
| 第 4 周 | Codex | Thread/Turn/Item 协议、审批与 sandbox 设计稿 |

每天只追一条调用链。不要从 README 一路横向浏览所有模块；从一个真实请求进入，跟到模型调用、
工具执行、Observation 写回和停止条件。

## 6. 选型建议

| 需求 | 优先考虑 |
| --- | --- |
| 看懂 Agent 最小原理 | mini-swe-agent |
| 快速做 Python Tool/Code Agent | smolagents |
| 构建通用、可嵌入的 Agent 应用 | Strands |
| 构建完整软件工程 Agent 服务 | OpenHands SDK |
| 学习生产级本地编码 Agent 内核 | Codex |

这些项目不是严格替代关系。最有价值的学习方式是把同一个小任务在五个框架中各实现一次，
比较状态归属、能力边界和失败恢复。

## 7. 最终检查清单

设计或评审一个 Agent Runtime 时，至少检查：

- 主循环和停止条件是否明确。
- 模型输出是否始终按不可信输入处理。
- Tool 是否有类型、ID、超时、取消和幂等语义。
- 权限检查是否位于真正执行之前。
- 本地执行是否有明确隔离声明。
- 对话、控制状态、环境状态和日志是否分离。
- 上下文压缩是否保持 Tool call 配对。
- 崩溃恢复是否可能重复副作用。
- 并发工具是否存在共享状态冲突。
- 子 Agent 是否继承了过多权限或上下文。
- 日志是否足以解释一次动作为何发生。
- 预算是否覆盖模型 token、时间、步骤和外部资源。

