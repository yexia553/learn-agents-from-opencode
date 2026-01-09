# Learn Agents from OpenCode

## 项目简介

OpenCode 是一个优秀的开源 Coding Agent, **Agent**、**Tool**、**Permission**、**Skill**、**Session**(Context Engineering) 等Agent开发的核心概念均在其中得到了全面实现。

这使得OpenCode成为了学习Agent开发的绝佳案例。

我也在做Agent开发相关的工作，但是做的远没有OpenCode全面和深入，所以在
本项目旨在通过对 OpenCode 源码的深入分析，帮助开发者理解其核心架构和设计理念，从而掌握 Agent 系统的开发方法。

---

## 教程列表

| 序号 | 教程 | 说明 | 难度 |
|------|------|------|------|
| 00 | [学习规划](00_OPENCODE_LEARNING_PLAN.md) | 完整的架构学习路线图 | - |
| 01 | [System Prompt 系统](01_SYSTEM_PROMPT_TUTORIAL.md) | Prompt 构建、Provider 适配、环境注入 | ⭐⭐ |
| 02 | [权限审核系统](02_PERMISSION_SYSTEM_TUTORIAL.md) | 权限规则、请求流程、BashArity | ⭐⭐⭐ |
| 03 | [Agent 系统](03_AGENT_SYSTEM_TUTORIAL.md) | 内置 Agent、配置系统、权限集成 | ⭐⭐ |
| 04 | [Tool 系统](04_TOOL_SYSTEM_TUTORIAL.md) | 工具定义、注册、执行流程 | ⭐⭐⭐ |
| 05 | [迭代信息收集模式](05_ITERATIVE_INFO_GATHERING_TUTORIAL.md) | 核心循环、Tool Chaining、Explore Agent | ⭐⭐⭐⭐ |
| 06 | [Skill 系统](06_SKILL_SYSTEM_TUTORIAL.md) | Skill 定义、发现机制、工具集成 | ⭐⭐ |
| 07 | [Session 系统](07_SESSION_SYSTEM_TUTORIAL.md) | 会话管理、消息流转、上下文压缩 | ⭐⭐⭐ |
| 08 | [Provider 系统](08_PROVIDER_SYSTEM_TUTORIAL.md) | 多模型适配、SDK 初始化、成本计算 | ⭐⭐⭐ |


---

## 核心架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        OpenCode 架构                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌──────────────────┐    ┌─────────────┐    │
│  │   CLI/TUI   │───▶│   Agent System   │───▶│  Provider   │    │
│  └─────────────┘    └────────┬─────────┘    └─────────────┘    │
│                              │                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              迭代信息收集循环 (核心灵魂)                   │   │
│  │   while (true) {                                         │   │
│  │     LLM 分析 → 探索收集 → 评估决策 → 继续/结束            │   │
│  │   }                                                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │    Tool     │  │  Permission │  │   Session   │             │
│  │   System    │  │   System    │  │   System    │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                              │                                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │    Skill    │  │  System     │  │   LLM       │             │
│  │   System    │  │   Prompt    │  │  Provider   │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

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
