# Learn Agents from OpenCode

## 项目简介

OpenCode 是一个优秀的开源 Coding Agent, **Agent**、**Tool**、**Permission**、**Skill**、**Session**(Context Engineering) 等Agent开发的核心概念均在其中得到了全面实现。

这使得OpenCode成为了学习Agent开发的绝佳案例。

我也在做Agent开发相关的工作，但是做的远没有OpenCode全面和深入，所以在
本项目旨在通过对 OpenCode 源码的深入分析，帮助开发者理解其核心架构和设计理念，从而掌握 Agent 系统的开发方法。

**得益于现在AI和各种编程工具的强大，我们自己开发Agent的时候，可以把OpenCode的代码发给AI参考，所以本教程致力于梳理OpenCode的核心架构和关键概念，并不会详细深入代码细节，学习的时候需要搭配OpenCode源码一起阅读。**

**本教程主要使用OpenCode/Claude Code/Codex等工具生成，人工review。**

## 这不是从0开始的教程

首先这不是一套“从 0 开始教你写 Agent”的教程，而是一套**基于优秀开源Agent项目的拆解**，你至少具备以下两个条件：

1. 使用任何一款coding agent做过至少一个项目或者工具，哪怕是给自己使用的也行
2. 熟悉任何一门编程语言

这个教程更适合下面这些场景：

* 已经用过 Claude Code / Codex / OpenCode等coding agent，想知道它们**内部是怎么工作的**
* 想自己实现一个 **可控的 Coding Agent 或者其他类型的Agent**
* 对 Agent 的权限 / Tool / Session / Prompt 这些模块有真实工程需求

---

## 教程列表

| 序号 | 教程 | 说明 | 难度 | 人工review | Reviewer | 备注 |
|------|------|------|:----:|:----------:|:--------:|------|
| 00 | [学习规划](00_OPENCODE_LEARNING_PLAN.md) | 完整的架构学习路线图 | - | ✅ | @yexia553 | - |
| 01 | [System Prompt 系统](01_SYSTEM_PROMPT_TUTORIAL.md) | Prompt 构建、Provider 适配、环境注入 | ⭐⭐ | | | |
| 02 | [权限审核系统](02_PERMISSION_SYSTEM_TUTORIAL.md) | 权限规则、请求流程、BashArity | ⭐⭐⭐ | | | |
| 03 | [Agent 系统](03_AGENT_SYSTEM_TUTORIAL.md) | 内置 Agent、配置系统、权限集成 | ⭐⭐ | | | |
| 04 | [Tool 系统](04_TOOL_SYSTEM_TUTORIAL.md) | 工具定义、注册、执行流程 | ⭐⭐⭐ | | | |
| 05 | [迭代信息收集模式](05_ITERATIVE_INFO_GATHERING_TUTORIAL.md) | 核心循环、Tool Chaining、Explore Agent | ⭐⭐⭐⭐ | | | |
| 06 | [Skill 系统](06_SKILL_SYSTEM_TUTORIAL.md) | Skill 定义、发现机制、工具集成 | ⭐⭐ | | | |
| 07 | [Session 系统](07_SESSION_SYSTEM_TUTORIAL.md) | 会话管理、消息流转、上下文压缩 | ⭐⭐⭐ | | | |
| 08 | [Provider 系统](08_PROVIDER_SYSTEM_TUTORIAL.md) | 多模型适配、SDK 初始化、成本计算 | ⭐⭐⭐ | | | |


---

## 核心架构

```
Clients:  TUI/CLI   IDE插件/Web   桌面App
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
                   LLM Orchestrator ──> Provider Adapter ──> Models (Claude/OpenAI/…)
                            │
                            ├─> PermissionNext (allow/ask/deny) ──> Global Bus
                            ├─> Worktree/File IO + Shell/Pty + LSP
                            ├─> MCP Servers & Plugins
                            └─> Session Engine (write parts/results)
```

核心执行流程概述：
- 客户端（TUI/IDE/Web/桌面）通过 SDK 调用本地或远程 Hono Server，事件使用 SSE/WebSocket。
- Server 路由到 Session Engine，加载 Config/Project，创建或恢复 Session，并将事件透传 Global Bus。
- Session Engine 依据 Agent/Permission 构建 System Prompt，调用 LLM，处理工具调用（`ToolRegistry`）并进行权限拦截（`PermissionNext`）。
- 工具执行文件读写、Pty/Bash、LSP、MCP 等操作，结果写回 Session，快照/存储模块维护消息、diff 和分享信息。
- 事件和权限请求通过 Bus 推送给客户端，形成可观察的迭代循环。

---

## 组件职责速览

| 组件 | 职责 | 关键文件 |
|------|------|----------|
| CLI/TUI | 用户入口，命令解析 | `cli/index.ts`, `cli/cmd/run.ts` |
| Config | 多层配置管理 | `config/config.ts` |
| Session | 会话管理、核心执行循环 | `session/processor.ts`, `session/index.ts` |
| Agent | 智能体定义和配置 | `agent/agent.ts`, `agent/index.ts` |
| Tool | 20+ 工具实现 | `tool/tool.ts`, `tool/registry.ts` |
| Permission | 权限控制 | `permission/next.ts` |
| Provider | LLM提供商集成 | `provider/provider.ts` |
| Skill | 技能管理 | `skill/skill.ts`, `skill/index.ts` |
| Event Bus | 事件通信 | `bus/index.ts`, `bus/global.ts` |

---

## 关键概念

### Agent

Agent 是 OpenCode 的执行主体，每个 Agent 包含：
- **名称和描述** - 标识和用途
- **运行模式** - primary / subagent / all
- **System Prompt** - 行为规范
- **权限规则** - 可执行操作限制
- **模型配置** - 使用的 LLM 模型

### Tool

Tool 是 Agent 可调用的外部能力，包括：
- **文件系统** - read, edit, write, glob, grep
- **命令执行** - bash
- **Agent 调用** - task (subagent)
- **信息获取** - webfetch, websearch
- **技能加载** - skill

### Permission

权限系统控制 Agent 的操作范围：
- **allow** - 允许操作
- **deny** - 拒绝操作
- **ask** - 询问用户

### Skill

Skill 是 OpenCode 的专业技能扩展机制：
- **定义格式** - SKILL.md (YAML frontmatter)
- **发现机制** - .claude/skills/ 和 .opencode/skill/
- **权限控制** - skill 权限类型管理
- **内容呈现** - Skill 工具加载和解析
- **用途** - 提供专家级指导和工作流程

### Session

Session 管理对话的生命周期：
- **消息管理** - 用户/助手消息
- **上下文压缩** - 长对话优化
- **会话分叉** - 实验性分支
- **状态持久化** - 消息存储

---

## 参与贡献

欢迎通过以下方式参与：

- **报告问题** - 发现教程中的错误或不清之处
- **补充内容** - 添加更多实践案例
- **改进文档** - 优化解释和代码示例

---

## 许可证

[CC BY-NC 4.0](LICENSE)

---

## 相关资源

- [OpenCode 官方网站](https://opencode.ai)
- [OpenCode GitHub](https://github.com/anomalyco/opencode)
