# Learn Agents from OpenCode

> 基于 OpenCode 源码学习 Coding Agent 工程架构的中文教程。

OpenCode 是一个优秀的开源 Coding Agent 项目，Agent、Tool、Permission、Skill、Session、Provider 等核心概念在其中都有完整实现。本仓库记录了我学习 OpenCode 源码过程中整理的教程，目标是帮助读者理解一个真实 Agent 项目的工程设计，而不是从零复刻一个玩具示例。

本教程主要使用 OpenCode、Claude Code、Codex 等工具辅助生成，并经过人工 review。阅读时建议搭配 [OpenCode 源码](https://github.com/anomalyco/opencode) 一起使用。

## 适合读者

- 已经使用过 Claude Code、Codex、OpenCode 等 coding agent，想理解其内部工作方式
- 想自己实现一个可控的 Coding Agent 或其他类型的 Agent
- 对 Agent 的权限、工具调用、会话管理、Prompt 组装等模块有工程需求

## 教程目录

完整课程导航见 [docs/README.md](docs/README.md)。

| 序号 | 教程 | 说明 |
| --- | --- | --- |
| 00 | [学习规划](docs/00-learning-plan.md) | 完整的架构学习路线图 |
| 01 | [System Prompt 系统](docs/tutorials/01-system-prompt.md) | Prompt 构建、Provider 适配、环境注入 |
| 02 | [权限审核系统](docs/tutorials/02-permission-system.md) | 权限规则、请求流程、BashArity |
| 03 | [Agent 系统](docs/tutorials/03-agent-system.md) | 内置 Agent、配置系统、权限集成 |
| 04 | [Tool 系统](docs/tutorials/04-tool-system.md) | 工具定义、注册、执行流程 |
| 05 | [迭代信息收集模式](docs/tutorials/05-iterative-info-gathering.md) | Tool Chaining、Explore Agent、SubAgent |
| 06 | [Skill 系统](docs/tutorials/06-skill-system.md) | Skill 定义、发现机制、工具集成 |
| 07 | [Session 系统](docs/tutorials/07-session-system.md) | 会话管理、消息流转、上下文压缩 |
| 08 | [Provider 系统](docs/tutorials/08-provider-system.md) | 多模型适配、SDK 初始化、成本计算 |

## 仓库结构

```text
.
├── docs/
│   ├── README.md
│   ├── 00-learning-plan.md
│   └── tutorials/
│       ├── 01-system-prompt.md
│       ├── 02-permission-system.md
│       ├── 03-agent-system.md
│       ├── 04-tool-system.md
│       ├── 05-iterative-info-gathering.md
│       ├── 06-skill-system.md
│       ├── 07-session-system.md
│       └── 08-provider-system.md
├── .github/
│   ├── ISSUE_TEMPLATE/
│   ├── workflows/
│   └── pull_request_template.md
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
├── SECURITY.md
├── CHANGELOG.md
├── LICENSE
└── README.md
```

## 贡献

欢迎通过 issue 或 pull request 改进教程内容。开始前请先阅读 [CONTRIBUTING.md](CONTRIBUTING.md)。

## 许可证

本项目使用 [MIT License](LICENSE)。

## 相关资源

- [OpenCode 官方网站](https://opencode.ai)
- [OpenCode GitHub](https://github.com/anomalyco/opencode)
