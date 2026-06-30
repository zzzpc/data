# 01. mini-swe-agent

## 一句话定位

`mini-swe-agent` 是理解编码 Agent 最合适的最小实现之一：核心控制流几乎可以直接读完，
默认只让模型生成 Shell 命令，再把命令输出作为 Observation 送回模型。

官方仓库：[SWE-agent/mini-swe-agent](https://github.com/SWE-agent/mini-swe-agent)  
官方文档：[mini-swe-agent.com](https://mini-swe-agent.com/)

## 1. 核心对象

| 对象 | 职责 |
| --- | --- |
| `Model` Protocol | 格式化消息、调用模型、提取动作、格式化 Observation |
| `Environment` Protocol | 执行动作、提供环境变量、序列化配置 |
| `DefaultAgent` | 维护消息、调用次数、成本并驱动循环 |
| `LocalEnvironment` | 在本机用子进程执行 Shell 命令 |
| trajectory | 保存配置、模型统计、消息和退出结果 |

这里没有复杂的图、事件总线或工作流 DSL。最重要的接口只有：

```python
message = model.query(messages)
outputs = [environment.execute(action) for action in message.actions]
messages += model.format_observation_messages(message, outputs)
```

## 2. 主循环

源码入口：

- [`DefaultAgent.run`](https://github.com/SWE-agent/mini-swe-agent/blob/408a133f68c3956937ac80645f64a120c6271fc8/src/minisweagent/agents/default.py#L88)
- [`DefaultAgent.step/query/execute_actions`](https://github.com/SWE-agent/mini-swe-agent/blob/408a133f68c3956937ac80645f64a120c6271fc8/src/minisweagent/agents/default.py#L124)

`run()` 初始化 system/user 消息，然后重复 `step()`。`step()` 只有一行实质逻辑：

```python
return self.execute_actions(self.query())
```

停止条件不是一个单独的调度器，而是最后一条消息的角色变成 `exit`。退出消息可能来自：

- 模型按约定提交最终结果。
- 步数、费用或墙钟时间超过限制。
- 连续格式错误超过限制。
- 未捕获异常。

这展示了最小 Agent 的本质：循环本身并不聪明，智能来自模型；Runtime 只负责反馈与边界。

## 3. 动作与执行环境

[`LocalEnvironment.execute`](https://github.com/SWE-agent/mini-swe-agent/blob/408a133f68c3956937ac80645f64a120c6271fc8/src/minisweagent/environments/local.py#L24)
读取动作中的 `command`，调用 `_run()`，返回：

```text
output + returncode + exception_info
```

`_run()` 使用 `subprocess.Popen(shell=True)`，合并 stdout/stderr，并在超时时终止进程组。
首行输出为约定的完成标记时，环境抛出 `Submitted`，把后续文本转成最终提交。

关键认识：

- 默认能力面很小，Shell 本身却是一个“超级工具”。
- 动作格式与完成协议由 Prompt、Model adapter 和 Environment 共同约定。
- 超时杀进程组很重要，否则子进程会在 Agent 结束后继续存活。
- 本地环境不是强隔离；运行不可信任务时要换容器或远程环境。

## 4. 状态与可观测性

`DefaultAgent.serialize()` 保存：

- 模型调用次数和费用。
- Agent/Model/Environment 配置。
- 完整消息历史。
- 退出状态和 submission。

这是一条线性 trajectory，优点是容易查看、复现和用于评测；缺点是缺乏事件级并发语义、
增量恢复和复杂分支。

## 5. 最值得学习的设计

### Protocol 加 Duck Typing

[`Model/Environment/Agent Protocol`](https://github.com/SWE-agent/mini-swe-agent/blob/408a133f68c3956937ac80645f64a120c6271fc8/src/minisweagent/__init__.py#L43)
只规定最小结构。替换模型或环境不需要继承庞大的基类。

### 把错误反馈给模型

格式错误和工具错误不一定终止任务；它们可被编码成消息，让模型在下一轮修正。

### 把预算放在模型调用之前

`query()` 在采样前检查 step、cost 和 wall-time，避免已经越界后再调用一次模型。

## 6. 局限与适用场景

适合：

- 学习 Agent 闭环。
- SWE-bench 等可控评测。
- 快速验证 Prompt、模型或执行环境。
- 作为自研 Runtime 的可运行基线。

不应直接当作完整生产平台，因为默认本地 Shell 隔离较弱，持久化和并发模型简单，也没有
完整的审批、身份、租户和分布式调度层。

## 7. 建议源码阅读顺序

1. `src/minisweagent/agents/default.py`
2. `src/minisweagent/__init__.py`
3. `src/minisweagent/environments/local.py`
4. `src/minisweagent/models/`
5. `src/minisweagent/config/`
6. CLI 入口和 trajectory 保存逻辑

## 8. 动手练习

1. 增加一个只读 Environment，只允许 `rg`、`git diff` 和测试命令。
2. 给每次动作增加唯一 ID，并记录开始/结束时间。
3. 把线性消息拆成 `Action` 与 `Observation` 两类事件。
4. 增加一次动作前审批，观察主循环需要新增哪些状态。

完成这些练习后，就自然走到了 `smolagents` 和更完整 Runtime 的设计空间。

