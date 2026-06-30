# 02. smolagents

## 一句话定位

Hugging Face `smolagents` 仍然强调轻量，但把最小循环提升为可复用框架：它有显式步骤记忆、
计划步骤、标准工具调用、Python 代码动作、多种执行器以及可作为 Tool 调用的托管 Agent。

官方仓库：[huggingface/smolagents](https://github.com/huggingface/smolagents)  
官方文档：[Hugging Face smolagents 文档](https://huggingface.co/docs/smolagents/)

## 1. 两种主要 Agent

| 类型 | 模型产生什么 | Runtime 如何执行 |
| --- | --- | --- |
| `ToolCallingAgent` | JSON/原生 function calling | 解析后逐个或并行调用 Tool |
| `CodeAgent` | Python 代码 | 在 Python executor 中解释执行 |

`ToolCallingAgent` 更像传统系统调用接口。`CodeAgent` 让模型用代码组合多个工具、变量和控制流，
表达能力更强，也扩大了执行与隔离风险。

## 2. 基类与主循环

源码入口：

- [`MultiStepAgent.run`](https://github.com/huggingface/smolagents/blob/526069c1ead958b36d9fd09a6b1ef37f68ed6ade/src/smolagents/agents.py#L436)
- [`MultiStepAgent._run_stream`](https://github.com/huggingface/smolagents/blob/526069c1ead958b36d9fd09a6b1ef37f68ed6ade/src/smolagents/agents.py#L540)

`run()` 重置状态、写入 `TaskStep`，把变量和工具交给执行器，然后消费 `_run_stream()`。
后者按 `max_steps` 循环：

1. 必要时生成 `PlanningStep`。
2. 创建当前 `ActionStep`。
3. 调用子类 `_step_stream()`。
4. 记录模型输出、工具调用、Observation、错误和 token。
5. 检查是否得到 final answer。
6. 达到最大步数时生成兜底最终答案。

流式接口不是附加 UI 功能，而是主执行路径；非流式 `run()` 只是把事件全部消费完。

## 3. Memory 是步骤日志

[`memory.py`](https://github.com/huggingface/smolagents/blob/526069c1ead958b36d9fd09a6b1ef37f68ed6ade/src/smolagents/memory.py#L40)
定义了：

- `SystemPromptStep`
- `TaskStep`
- `PlanningStep`
- `ActionStep`
- `FinalAnswerStep`

`ActionStep` 同时保存模型输入、模型输出、tool calls、代码、Observation、错误、token 和时间。
每类 Step 可以通过 `to_messages()` 再投影为模型消息。

这比单纯保存消息更清晰：

```text
运行事实（Step） -> 面向模型的投影（Messages）
```

未来做摘要、审计、UI 展示或评测时，不必反向猜测一条 message 代表什么。

## 4. ToolCallingAgent

[`ToolCallingAgent._step_stream`](https://github.com/huggingface/smolagents/blob/526069c1ead958b36d9fd09a6b1ef37f68ed6ade/src/smolagents/agents.py#L1276)
实现标准 ReAct：

1. Memory 转成输入消息。
2. 将 Tool 和 managed agent schema 交给模型。
3. 接收原生 tool calls，必要时从文本补解析。
4. 执行 tool calls 并产生 `ToolOutput`。
5. 特殊的 final-answer tool 结束运行。

它支持并行工具线程，但并行调用是否安全仍取决于工具是否共享可变状态。

## 5. CodeAgent 与执行器

[`CodeAgent`](https://github.com/huggingface/smolagents/blob/526069c1ead958b36d9fd09a6b1ef37f68ed6ade/src/smolagents/agents.py#L1505)
让模型生成 Python 代码。内置执行器类型包括：

- `local`
- `docker`
- `e2b`
- `modal`
- `blaxel`

本地执行器限制可导入模块，但“限制 import”不等同于强安全沙箱。处理不可信输入或有价值凭据时，
优先使用容器或远程隔离执行器，并最小化注入的 Tool 和变量。

代码动作的价值：

- 可以在一个动作里组合多个 Tool。
- 中间变量可持续存在。
- 循环、条件、聚合和数据处理更自然。
- 模型不必为每个细小操作重新采样。

代价是代码解析、解释器状态、超时、资源限制和安全边界都更复杂。

## 6. 托管 Agent

Managed agent 被包装成一个 Tool，主 Agent 可以像调用函数一样委派任务。这是一种简单而实用的
多 Agent 模式：

```text
Manager Agent
  -> call researcher(task)
  -> receive result as Tool Observation
  -> continue its own loop
```

它容易理解，但子 Agent 的进度、取消、权限继承和共享上下文仍需要上层设计。

## 7. 最值得学习的设计

### 运行记录与模型消息分离

Step 是内部事实，Message 是模型输入视图。这个边界值得在自研框架中保留。

### 计划不是主循环

Planning 是可配置的周期性 Step，而不是把所有执行强制塞进预先生成的 DAG。这样仍保留 ReAct
对新 Observation 的即时调整能力。

### 执行器是独立边界

`CodeAgent` 不直接等于本地 `exec`；执行器可以替换为 Docker 或远程服务。

## 8. 局限与适用场景

适合：

- 数据分析、检索、轻量自动化。
- 需要模型用 Python 组合工具的任务。
- 教学和快速构建自定义 Tool/Agent。
- 比较 CodeAct 与 function calling。

生产使用仍需额外设计身份、凭据、网络策略、持久化、租户隔离、审批与服务调度。

## 9. 建议源码阅读顺序

1. `src/smolagents/agents.py` 中的 `MultiStepAgent`
2. `src/smolagents/memory.py`
3. `ToolCallingAgent`
4. `CodeAgent`
5. `local_python_executor.py`
6. `models.py` 与 Tool 基类
7. managed agents 和远程执行器

## 10. 动手练习

1. 用同一组 Tool 分别运行 `ToolCallingAgent` 和 `CodeAgent`，比较步数与 token。
2. 写一个只允许纯函数的 Tool registry，验证并行调用。
3. 将 `ActionStep` 持久化为 JSONL，并实现中断后只读回放。
4. 将一个 Agent 包装为 Tool，记录父子调用树和各自预算。

