# 03. Strands Agents

## 一句话定位

Strands Agents 把 Agent 建模为“模型驱动的事件循环”，并围绕它提供模型 Provider、Tool Provider、
Hook、Intervention、Session、Memory、Checkpoint、Telemetry 和多 Agent 编排。它比前两者更像
一个可扩展 Runtime。

官方仓库：[strands-agents/sdk-python](https://github.com/strands-agents/sdk-python)  
官方文档：[strandsagents.com](https://strandsagents.com/)

> 2026-06-30 的官方仓库已经是 Python/TypeScript monorepo，Python 实现在 `strands-py/`。

## 1. Agent 对象是一组可替换策略

[`Agent.__init__`](https://github.com/strands-agents/sdk-python/blob/712ab53e7f86e6dd73415a084c7c158062fb971c/strands-py/src/strands/agent/agent.py#L138)
接收的关键组件包括：

| 组件 | 作用 |
| --- | --- |
| `model` | 模型 Provider |
| `tools` / `ToolProvider` | 能力集合与生命周期 |
| `conversation_manager` | 对话窗口管理 |
| `context_manager` | 上下文卸载与压缩策略 |
| `session_manager` | 会话级持久化 |
| `memory_manager` | 跨会话记忆 |
| `tool_executor` | 顺序或并发执行工具 |
| `hooks` / `plugins` | 生命周期扩展 |
| `interventions` | 授权、护栏、引导和人工介入 |
| `checkpointing` | 在模型后或工具后暂停/恢复 |
| `sandbox` | 执行隔离能力 |

这体现了 Runtime 的重要设计：主循环稳定，各种策略通过接口注入。

## 2. 同步入口只是异步事件流的外壳

[`Agent.__call__`](https://github.com/strands-agents/sdk-python/blob/712ab53e7f86e6dd73415a084c7c158062fb971c/strands-py/src/strands/agent/agent.py#L697)
把同步调用桥接到 `invoke_async()`；`invoke_async()` 实际消费 `stream_async()`，直到最后一个事件
携带 `AgentResult`。

这说明 Strands 的第一等抽象是事件流，而不是最终字符串：

```text
Start
  -> model stream events
  -> model message
  -> tool events
  -> tool result message
  -> next cycle
  -> stop/result
```

CLI、Web UI、Tracing 和回调都可以订阅同一条执行流。

## 3. 中央事件循环

源码入口：

- [`event_loop_cycle`](https://github.com/strands-agents/sdk-python/blob/712ab53e7f86e6dd73415a084c7c158062fb971c/strands-py/src/strands/event_loop/event_loop.py#L182)
- [`recurse_event_loop`](https://github.com/strands-agents/sdk-python/blob/712ab53e7f86e6dd73415a084c7c158062fb971c/strands-py/src/strands/event_loop/event_loop.py#L398)

单个 cycle 做这些事：

1. 检查 turns/token 预算。
2. 初始化 cycle ID、request state、metrics 和 trace。
3. 调模型并流式产生事件。
4. 读取 stop reason。
5. 若为 `tool_use`，执行工具并追加 Tool Result。
6. 递归进入下一 cycle。
7. 若为 `end_turn`，产生停止事件和最终结果。
8. 在结构化输出模式下，必要时强制模型调用输出 Tool。

这里的“递归”是逻辑上的继续循环。重要的是 request state、metrics 和 structured-output context
在各 cycle 间传递。

## 4. Tool 系统

[`ToolProvider`](https://github.com/strands-agents/sdk-python/blob/712ab53e7f86e6dd73415a084c7c158062fb971c/strands-py/src/strands/tools/tool_provider.py#L11)
不只提供 Tool，还管理 consumer 和清理生命周期。

Tool 来源可以是：

- `@tool` 装饰的 Python 函数。
- 模块或文件中的工具。
- MCP Server 提供的工具。
- Tool Provider 动态加载的集合。
- 另一个 Agent，经 `as_tool()` 包装。

Agent 默认可使用并发 Tool Executor。工具并发必须考虑顺序依赖、共享状态和副作用冲突；Runtime
能并行分发，不代表业务语义允许并行。

## 5. Hook、Plugin 与 Intervention

三者容易混淆：

| 机制 | 主要用途 |
| --- | --- |
| Hook | 在明确生命周期点观察或修改行为 |
| Plugin | 打包初始化逻辑、Hook 和能力扩展 |
| Intervention | 对动作做 Allow/Deny/Guide/Interrupt 等控制 |

Intervention 更接近“策略执行点”。授权和廉价护栏应先运行，昂贵的 LLM 判断放后面；Deny
可以短路，Guide 可以累积反馈。

## 6. 三层记忆

| 层次 | 解决的问题 |
| --- | --- |
| Conversation manager | 当前上下文窗口装什么 |
| Session manager | 对话和 Agent state 如何跨调用恢复 |
| Memory manager | 哪些信息跨会话提取、检索和注入 |

把三层分开很关键：上下文窗口不是数据库，会话持久化也不自动等于有选择的长期记忆。

## 7. Checkpoint 与结构化输出

Checkpoint 可在 `after_model` 或 `after_tools` 停住。恢复时必须知道：

- 停在哪个 cycle。
- 模型动作是否已经生成。
- 工具是否已经执行。
- Tool result 是否已经写回。

这与数据库恢复中的“动作是否已提交”很相似。没有位置标记就容易重复执行有副作用的工具。

结构化输出被实现为特殊 Tool：如果模型自然结束却没有调用它，Runtime 会追加提示并再跑一轮。
这比仅在最终文本上做脆弱 JSON 解析更可控。

## 8. 多 Agent

Strands 提供多种层次：

- Agent as Tool：简单委派。
- Graph：显式节点与边。
- Swarm：Agent 之间动态移交。
- A2A：通过 Agent-to-Agent 协议互操作。

选择原则：

- 固定流程与依赖用 Graph。
- 主 Agent 统一控制用 Agent as Tool。
- 角色动态接力用 Swarm。
- 跨进程、跨组织互操作再考虑 A2A。

## 9. 最值得学习的设计

- 将流式事件作为主路径。
- 将模型、Tool、执行、上下文、Session 和 Memory 分层。
- 用 Hook 和 Intervention 提供内核扩展点。
- 用 stop reason 驱动状态转换，而不是解析自然语言猜测。
- 将 checkpoint 位置纳入协议。
- 内建 OpenTelemetry 风格的 metrics 与 trace。

## 10. 建议源码阅读顺序

1. `strands-py/src/strands/agent/agent.py`
2. `strands-py/src/strands/event_loop/event_loop.py`
3. `strands-py/src/strands/event_loop/streaming.py`
4. `strands-py/src/strands/tools/`
5. `strands-py/src/strands/hooks/` 与 `interventions/`
6. `session/`、`memory/`、`multiagent/`
7. `team/designs/` 中的设计文档

## 11. 动手练习

1. 写一个 Intervention：写文件前要求确认，读文件直接允许。
2. 为一次 Tool call 记录 trace，并关联父 Agent invocation。
3. 在 `after_model` checkpoint 后恢复，验证 Tool 不会提前执行。
4. 将同一个子 Agent 分别接成 Tool 和 Graph 节点，比较状态归属。

