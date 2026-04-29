# 教程导航

本目录收录了基于 OpenCode 源码整理的 Agent 工程学习教程。教程重点是梳理 OpenCode 的核心架构、关键概念和工程实现路径，适合搭配 OpenCode 源码一起阅读。

## 阅读路径

建议先阅读 [学习规划](00-learning-plan.md)，再按专题进入各系统：

| 顺序 | 主题 | 文档 | 重点 |
| --- | --- | --- | --- |
| 00 | 学习规划 | [00-learning-plan.md](00-learning-plan.md) | 整体架构、学习阶段、关键文件速查 |
| 01 | System Prompt | [tutorials/01-system-prompt.md](tutorials/01-system-prompt.md) | Prompt 构建、Provider 适配、环境注入 |
| 02 | Permission | [tutorials/02-permission-system.md](tutorials/02-permission-system.md) | 权限规则、审核流程、BashArity |
| 03 | Agent | [tutorials/03-agent-system.md](tutorials/03-agent-system.md) | 内置 Agent、配置系统、权限集成 |
| 04 | Tool | [tutorials/04-tool-system.md](tutorials/04-tool-system.md) | 工具定义、注册机制、执行流程 |
| 05 | Iteration | [tutorials/05-iterative-info-gathering.md](tutorials/05-iterative-info-gathering.md) | 迭代信息收集、Tool Chaining、SubAgent |
| 06 | Skill | [tutorials/06-skill-system.md](tutorials/06-skill-system.md) | Skill 定义、发现机制、工具集成 |
| 07 | Session | [tutorials/07-session-system.md](tutorials/07-session-system.md) | 会话管理、消息流转、上下文压缩 |
| 08 | Provider | [tutorials/08-provider-system.md](tutorials/08-provider-system.md) | 多模型适配、SDK 初始化、成本计算 |

## 核心架构

```text
Clients:  TUI/CLI   IDE 插件/Web   桌面 App
              │          │            │
              └─────> OpenCode Server (Hono: HTTP/OpenAPI/SSE/WebSocket)
                            │
                         Global Bus
                            │
                    ┌──── Session Engine ────┐
                    │                        │
          Config Loader              Storage & Snapshots
     (project/user/env)              (messages/diffs/share)
                    │                        │
                    ├─> Agent Registry (build / plan / general)
                    ├─> System Prompt Builder
                    └─ Tool Registry <──────────┐
                            │                   │
                            │ tool-calls        │ session updates
                            ▼                   │
                   LLM Orchestrator ──> Provider Adapter ──> Models
                            │
                            ├─> PermissionNext (allow/ask/deny) ──> Global Bus
                            ├─> Worktree/File IO + Shell/Pty + LSP
                            ├─> MCP Servers & Plugins
                            └─> Session Engine (write parts/results)
```

## 组件速览

| 组件 | 职责 | 关键文件 |
| --- | --- | --- |
| CLI/TUI | 用户入口，命令解析 | `cli/index.ts`, `cli/cmd/run.ts` |
| Config | 多层配置管理 | `config/config.ts` |
| Session | 会话管理、核心执行循环 | `session/processor.ts`, `session/index.ts` |
| Agent | 智能体定义和配置 | `agent/agent.ts`, `agent/index.ts` |
| Tool | 工具定义、注册和执行 | `tool/tool.ts`, `tool/registry.ts` |
| Permission | 权限控制 | `permission/next.ts` |
| Provider | LLM 提供商集成 | `provider/provider.ts` |
| Skill | 技能管理 | `skill/skill.ts`, `skill/index.ts` |
| Event Bus | 事件通信 | `bus/index.ts`, `bus/global.ts` |

## 学习建议

- 先建立整体图景，再进入单个系统的代码细节。
- 阅读教程时同步打开 OpenCode 源码，对照关键路径和类型定义。
- 如果只关心 Agent 工程实践，优先阅读 System Prompt、Permission、Tool、Session 四篇。
