# 07. 资料索引

## 快照说明

源码于 2026-06-30 从官方 GitHub 仓库拉取。正文中的源码链接固定到以下 commit，避免主分支变化
导致行号失效。

| 项目 | 官方仓库 | 固定提交 | 日期 | 许可证 |
| --- | --- | --- | --- | --- |
| mini-swe-agent | [SWE-agent/mini-swe-agent](https://github.com/SWE-agent/mini-swe-agent) | [`408a133`](https://github.com/SWE-agent/mini-swe-agent/commit/408a133f68c3956937ac80645f64a120c6271fc8) | 2026-06-29 | MIT |
| smolagents | [huggingface/smolagents](https://github.com/huggingface/smolagents) | [`526069c`](https://github.com/huggingface/smolagents/commit/526069c1ead958b36d9fd09a6b1ef37f68ed6ade) | 2026-06-16 | Apache-2.0 |
| Strands Agents | [strands-agents/sdk-python](https://github.com/strands-agents/sdk-python) | [`712ab53`](https://github.com/strands-agents/sdk-python/commit/712ab53e7f86e6dd73415a084c7c158062fb971c) | 2026-06-30 | Apache-2.0 |
| OpenHands SDK | [OpenHands/software-agent-sdk](https://github.com/OpenHands/software-agent-sdk) | [`1c230d9`](https://github.com/OpenHands/software-agent-sdk/commit/1c230d909eaa0fe9679d49540cb887c05f268add) | 2026-06-30 | MIT |
| Codex | [openai/codex](https://github.com/openai/codex) | [`cfead68`](https://github.com/openai/codex/commit/cfead68e5d3984b247cf0758e3e53b19165de848) | 2026-06-29 | Apache-2.0 |

## 官方文档

- mini-swe-agent：[https://mini-swe-agent.com/](https://mini-swe-agent.com/)
- smolagents：[https://huggingface.co/docs/smolagents/](https://huggingface.co/docs/smolagents/)
- Strands Agents：[https://strandsagents.com/](https://strandsagents.com/)
- OpenHands SDK：[https://docs.openhands.dev/sdk/](https://docs.openhands.dev/sdk/)
- Codex：[https://developers.openai.com/codex/](https://developers.openai.com/codex/)

## 各项目优先阅读文件

### mini-swe-agent

- `src/minisweagent/agents/default.py`
- `src/minisweagent/environments/local.py`
- `src/minisweagent/__init__.py`

### smolagents

- `src/smolagents/agents.py`
- `src/smolagents/memory.py`
- `src/smolagents/local_python_executor.py`

### Strands Agents

- `strands-py/src/strands/agent/agent.py`
- `strands-py/src/strands/event_loop/event_loop.py`
- `strands-py/src/strands/tools/`
- `team/designs/`

### OpenHands SDK

- `openhands-sdk/openhands/sdk/conversation/impl/local_conversation.py`
- `openhands-sdk/openhands/sdk/agent/agent.py`
- `openhands-sdk/openhands/sdk/conversation/event_store.py`
- `openhands-sdk/openhands/sdk/workspace/`
- `openhands-agent-server/`

### Codex

- `codex-rs/core/src/session/mod.rs`
- `codex-rs/core/src/session/handlers.rs`
- `codex-rs/core/src/session/turn.rs`
- `codex-rs/core/src/tools/router.rs`
- `codex-rs/core/src/tools/registry.rs`
- `codex-rs/core/src/tools/orchestrator.rs`
- `codex-rs/core/src/agent/control/`
- `codex-rs/app-server/README.md`

## 使用提示

源码优先用于解释“实际如何实现”，官方文档优先用于确认“公开支持的使用方式”。如果两者不同，
先检查是否为实验特性、内部接口、版本差异或尚未发布的主分支代码。

