# Agent Runtime Learning Notes

这是一份面向源码学习的 Agent 框架中文笔记，按复杂度逐层阅读：

1. [mini-swe-agent](docs/01-mini-swe-agent.md)：看清最小闭环。
2. [smolagents](docs/02-smolagents.md)：理解步骤、计划、代码动作和子 Agent。
3. [Strands Agents](docs/03-strands.md)：理解事件循环、Provider、Hook、Session 与多 Agent。
4. [OpenHands SDK](docs/04-openhands.md)：理解软件工程 Agent 的事件溯源、工作区和服务化。
5. [Codex](docs/05-codex.md)：理解生产级编码 Agent 的会话内核、工具路由、权限和沙箱。

建议先读 [Agent Runtime 总图](docs/00-agent-runtime-map.md)，再按顺序读五个框架，最后阅读
[横向比较与学习路线](docs/06-comparison-and-roadmap.md)。

## 为什么按这个顺序

| 阶段 | 核心增量 | 类比操作系统 |
| --- | --- | --- |
| mini-swe-agent | 最小 `model -> action -> observation` 循环 | 单任务监控程序 |
| smolagents | 结构化步骤、计划、代码解释器、托管 Agent | 进程状态与解释器 |
| Strands | 异步事件、Provider、Hook、Session、多 Agent | 可扩展微内核 |
| OpenHands | 持久事件日志、工作区、恢复、Agent Server | 带日志和远程控制的运行时 |
| Codex | Thread/Turn/Item 协议、工具路由、审批、跨平台沙箱 | 面向编码任务的完整 Agent OS |

## 阅读原则

- 先找停止条件，再找主循环。
- 把“模型输出”当作不可信的系统调用请求。
- 区分对话记忆、运行状态、持久化日志和长期记忆。
- 区分工具定义、工具路由、工具执行和执行隔离。
- 观察错误如何变成下一轮模型可见的 Observation。
- 观察并发、取消、预算、压缩、恢复和审计如何进入主循环。

## 快速结论

Agent 的核心不是“调用一次大模型”，而是一个受控反馈系统：

```text
输入/状态
  -> 构造模型上下文
  -> 模型提出动作
  -> 策略与权限检查
  -> 在运行环境中执行
  -> 结果写回状态
  -> 判断继续、暂停、失败或完成
```

模型更像不可信的用户态策略程序；Agent Runtime 更像内核，负责调度、系统调用、权限、
资源限制、隔离、日志和恢复。

## 资料快照

本笔记基于 2026-06-30 拉取的官方仓库源码。固定提交、许可证和链接见
[资料索引](docs/07-sources.md)。框架演进很快，阅读新版本时应先比较主循环和协议对象的变化。

