# Learn Agents from OpenCode

> [中文](README.zh.md) | English

> A Chinese tutorial series on learning Coding Agent engineering architecture from the OpenCode source code.

OpenCode is an excellent open-source Coding Agent project with complete implementations of core concepts such as Agent, Tool, Permission, Skill, Session, and Provider. This repository documents the tutorials I compiled while studying the OpenCode source code, aiming to help readers understand the engineering design of a real-world Agent project rather than building a toy example from scratch.

These tutorials were primarily generated with assistance from OpenCode, Claude Code, and Codex, and have been manually reviewed. It is recommended to read them alongside the [OpenCode source code](https://github.com/anomalyco/opencode).

## Target Audience

- Users who have already used coding agents like Claude Code, Codex, or OpenCode and want to understand their internal workings
- Developers looking to implement their own controllable Coding Agent or other types of Agents
- Engineers with requirements for Agent permissions, tool calling, session management, prompt assembly, and other modules

## Quick Start

This repo includes the OpenCode source code as a Git submodule so you can follow the tutorials while reading the actual implementation.

```bash
# Clone with submodule (recommended)
git clone --recursive https://github.com/yexia553/learn-agents-from-opencode.git

# Or if already cloned, initialize the submodule
git submodule update --init --recursive

# The OpenCode source will be available at ./opencode/
cd opencode && bun install
```

> **Note:** Tutorials reference source files under `opencode/packages/opencode/src/...`.

## Tutorial Index

For full navigation, see [docs/README.md](docs/README.md).

| No. | Tutorial | Description |
| --- | --- | --- |
| 00 | [Learning Plan](docs/00-learning-plan.md) | Complete architecture learning roadmap |
| 01 | [System Prompt](docs/tutorials/01-system-prompt.md) | Prompt construction, Provider adaptation, environment injection |
| 02 | [Permission System](docs/tutorials/02-permission-system.md) | Permission rules, request flow, BashArity |
| 03 | [Agent System](docs/tutorials/03-agent-system.md) | Built-in Agents, configuration system, permission integration |
| 04 | [Tool System](docs/tutorials/04-tool-system.md) | Tool definition, registration, execution flow |
| 05 | [Iterative Info Gathering](docs/tutorials/05-iterative-info-gathering.md) | Tool Chaining, Explore Agent, SubAgent |
| 06 | [Skill System](docs/tutorials/06-skill-system.md) | Skill definition, discovery mechanism, tool integration |
| 07 | [Session System](docs/tutorials/07-session-system.md) | Session management, message flow, context compression |
| 08 | [Provider System](docs/tutorials/08-provider-system.md) | Multi-model adaptation, SDK initialization, cost calculation |

> **Note:** The tutorial documents below are currently available in Chinese only. English translations will be added progressively.

## Repository Structure

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
├── opencode/                  # OpenCode source code (submodule)
│   └── packages/
│       └── opencode/
│           └── src/
├── .github/
│   ├── ISSUE_TEMPLATE/
│   ├── workflows/
│   └── pull_request_template.md
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
├── SECURITY.md
├── CHANGELOG.md
├── LICENSE
├── README.md
└── README.zh.md
```

## Contributing

Contributions via issues or pull requests are welcome. Please read [CONTRIBUTING.md](CONTRIBUTING.md) first.

## License

This project is licensed under the [MIT License](LICENSE).

## Related Resources

- [OpenCode Official Website](https://opencode.ai)
- [OpenCode GitHub](https://github.com/anomalyco/opencode)

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=yexia553/learn-agents-from-opencode&type=Date)](https://star-history.com/#yexia553/learn-agents-from-opencode&Date)
