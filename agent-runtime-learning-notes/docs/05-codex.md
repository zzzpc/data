# 05. Codex

## 一句话定位

开源 Codex 是以 Rust 为主的生产级编码 Agent 客户端与 Runtime。它围绕 Thread、Turn、Item、
Submission、Event、Tool Router、审批、沙箱、上下文压缩和多 Agent 构建，并通过 CLI、TUI、
App Server、MCP Server 等不同外壳暴露同一个核心。

官方仓库：[openai/codex](https://github.com/openai/codex)  
官方文档：[OpenAI Codex 文档](https://developers.openai.com/codex/)

> 开源仓库展示的是客户端、协议和执行 Runtime，不包含托管模型服务端的全部实现。

## 1. 仓库层次

| 层次 | 代表模块 | 职责 |
| --- | --- | --- |
| Core | `codex-rs/core` | Session、Turn、模型循环、Tool、上下文、审批 |
| Protocol | `codex-rs/protocol` | Submission、Event、Item 和配置类型 |
| App Server | `codex-rs/app-server` | 面向桌面/IDE 等客户端的 JSON-RPC 服务 |
| CLI/TUI/Exec | `codex-rs/cli`、`tui`、`exec` | 不同交互与非交互入口 |
| Sandbox | `linux-sandbox`、`sandboxing`、Windows 相关 crate | 跨平台执行隔离 |
| MCP | `mcp-server`、`codex-mcp` | 对外提供能力、连接外部工具 |
| Multi-agent | `core/src/agent/control` | 子 Agent 的创建、输入、等待、关闭和配额 |

复杂度来自产品边界，而不是把一个 ReAct 循环写得更花哨。

## 2. 双通道 Session 内核

[`Codex` struct](https://github.com/openai/codex/blob/cfead68e5d3984b247cf0758e3e53b19165de848/codex-rs/core/src/session/mod.rs#L389)
持有：

- `tx_sub`：客户端向 Runtime 提交操作。
- `rx_event`：Runtime 向客户端发出事件。
- `agent_status`：观察当前 Agent 状态。
- `session`：共享 Session 状态与服务。
- `session_loop_termination`：等待后台循环结束。

Spawn 时创建有界 Submission channel 和无界 Event channel，然后启动后台
[`submission_loop`](https://github.com/openai/codex/blob/cfead68e5d3984b247cf0758e3e53b19165de848/codex-rs/core/src/session/handlers.rs#L702)。

```text
UI / CLI / App Server
      | Submission(Op)
      v
 Session dispatch loop
      | start/steer/approve/interrupt
      v
 Turn + Tool Runtime
      | Event
      v
 UI / CLI / App Server
```

这种命令/事件分离使模型运行时仍能接收中断、审批回复、用户追加输入和动态 Tool 回复。

## 3. Submission Loop 是控制平面

`submission_loop` 根据 `Op` 分发：

- 用户输入或新 Turn。
- Interrupt、Shutdown。
- Shell/Patch approval。
- 用户问题与权限请求的回复。
- MCP 刷新与 Elicitation。
- Compact、Rollback、Thread settings。
- Review、实时会话和跨 Agent 通信。

它更像内核的系统调用/控制消息分发器。真正的模型反馈循环在 `run_turn()`。

## 4. Turn 内部循环

源码入口：

- [`run_turn`](https://github.com/openai/codex/blob/cfead68e5d3984b247cf0758e3e53b19165de848/codex-rs/core/src/session/turn.rs#L142)
- `run_sampling_request` 及响应 Item 处理

Turn 开始时：

1. 创建 turn-scoped model client session。
2. 必要时做采样前 compaction。
3. 捕获本轮 `TurnContext` / `StepContext`。
4. 注入指令、Skills、Plugins 和连接器上下文。
5. 记录输入与上下文更新。

然后进入 step loop：

1. 合并运行中收到的 pending user input。
2. 从 History 构造模型可见的 `ResponseItem`。
3. 构建本 step 可见的 Tool specs。
4. 发起流式采样。
5. 处理 assistant message、reasoning 和 tool call Item。
6. 执行 Tool 并记录输出。
7. 若模型或 pending input 需要 follow-up，则继续采样。
8. 接近 token 上限时在 Turn 中间自动压缩。
9. 没有 follow-up 时结束 Turn。

相比简单 `while`，这里每个 step 都重新捕获环境、权限和工具视图，以避免运行中的配置变化
与实际执行脱节。

## 5. Thread、Turn、Item

App Server 将执行暴露为稳定协议：

| 概念 | 含义 |
| --- | --- |
| Thread | 可持久、可恢复或 Fork 的长期会话 |
| Turn | 一次用户目标及其所有模型/Tool 循环 |
| Item | 用户消息、Agent 消息、Tool call、Tool output、diff 等原子记录 |

客户端订阅 Item 生命周期和增量，而不是解析终端文本。这样桌面端、IDE 和自动化调用可以共享
Runtime，同时以不同方式展示相同执行事实。

App Server 协议入口：
[`codex-rs/app-server/README.md`](https://github.com/openai/codex/blob/cfead68e5d3984b247cf0758e3e53b19165de848/codex-rs/app-server/README.md)。

## 6. Tool Router 与 Registry

[`ToolRouter`](https://github.com/openai/codex/blob/cfead68e5d3984b247cf0758e3e53b19165de848/codex-rs/core/src/tools/router.rs#L35)
同时保存：

- `model_visible_specs`：本 step 告诉模型哪些 Tool 可用。
- `ToolRegistry`：实际可分发的 Runtime handler。

[`ToolRegistry`](https://github.com/openai/codex/blob/cfead68e5d3984b247cf0758e3e53b19165de848/codex-rs/core/src/tools/registry.rs#L322)
按名称注册运行时，处理重复注册、并行能力、取消语义和 dispatch。

这个分离非常重要：

```text
模型看见 Tool != Tool 一定可以无条件执行
```

模型可见性、运行时存在性、权限检查和执行环境是四个不同问题。

Tool 来源包括内建 Shell/Patch、MCP、动态 Tool、扩展 Tool、搜索发现 Tool 和多 Agent 协作 Tool。

## 7. 审批、策略与沙箱

Codex 将安全控制放在模型与执行环境之间：

```text
model tool call
  -> parse and normalize
  -> execution policy
  -> approval requirement
  -> sandbox selection
  -> execute
  -> result/event
```

核心概念：

- 文件系统权限与网络权限分开表示。
- Workspace-write、read-only、unrestricted 等策略决定默认保护域。
- Approval policy 决定何时可以向用户申请提权。
- Prefix/rule 可以对重复命令形成窄范围预批准。
- Shell 与 Patch 有各自审批事件。
- 沙箱失败不自动等于允许无沙箱重试；是否升级由策略和用户决定。

跨平台实现不同，但目标一致：Linux、macOS、Windows 都应把模型生成的命令放在明确的保护域中。

## 8. Context、Compaction 与 Rollout

Codex 不只保存聊天文本，还记录：

- Thread history 与 Response Items。
- TurnContext/StepContext。
- World/environment state。
- Tool outputs、diff 和 token usage。
- Rollout/session metadata。

Compaction 可以在 Turn 前或 Turn 中触发。压缩后仍要保留工具调用配对、关键指令、环境事实和
当前任务状态，否则模型得到的“摘要”无法继续执行。

Context manager 的难点不是删掉旧消息，而是维持协议合法性和任务连续性。

## 9. 多 Agent

`core/src/agent/control` 管理：

- `spawn_agent`
- `send_input`
- `wait`
- `resume_agent`
- `close_agent`
- Thread 数量与执行容量
- 父子历史继承和 Fork

子 Agent 是独立 Thread/Session，而不只是一个普通 Python 函数。这样可以单独调度、取消、
限制并发、保存历史并发出事件；代价是必须处理父子权限、上下文复制、资源配额和生命周期。

## 10. 可扩展能力

| 机制 | 合适用途 |
| --- | --- |
| `AGENTS.md` | 仓库内持久指令 |
| Skills | 可复用任务流程与资料 |
| Plugins | 打包 Skills、工具、Hook、MCP 与资产 |
| MCP | 连接外部数据和动作 |
| Hooks | 在工具/命令生命周期执行约束 |
| Dynamic tools | 由宿主在 Thread 启动时注入能力 |

这些扩展最终都会影响某一步的上下文、Tool specs、策略或事件，但不需要重写核心 Turn 循环。

## 11. 最值得学习的设计

- Submission/Event 双通道将控制平面与输出流分开。
- Thread/Turn/Item 构成稳定、可服务化的协议层。
- Tool 的模型可见描述与运行时 Registry 分离。
- 每个 Step 捕获一致的上下文和权限快照。
- 审批与沙箱是执行路径的一部分，不是 Prompt 提醒。
- Compaction、steering、cancel 和 pending input 都进入 Turn 状态机。
- 子 Agent 作为受调度的独立 Session。
- Core 与 TUI/App Server/MCP 等外壳分离。

## 12. 建议源码阅读顺序

1. `codex-rs/core/src/session/mod.rs`
2. `codex-rs/core/src/session/handlers.rs`
3. `codex-rs/core/src/session/turn.rs`
4. `codex-rs/core/src/tools/router.rs`
5. `codex-rs/core/src/tools/registry.rs`
6. `codex-rs/core/src/tools/orchestrator.rs` 与 `sandboxing.rs`
7. `codex-rs/core/src/context_manager/` 与 `compact*.rs`
8. `codex-rs/core/src/agent/control/`
9. `codex-rs/app-server/README.md`

## 13. 动手练习

1. 画出 `UserInput Submission -> Turn -> Tool call -> Event` 的真实调用链。
2. 增加一个只读 Tool，分别定位 spec 构建、registry 注册和 handler。
3. 跟踪一个需要审批的 Shell call，记录每个状态转换。
4. 在 Tool 后注入 pending user input，观察下一次采样如何合并。
5. 跟踪一次自动 compaction 前后的 Items，检查工具调用配对是否保留。

