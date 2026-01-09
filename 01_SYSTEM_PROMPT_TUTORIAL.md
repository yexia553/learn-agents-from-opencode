# OpenCode System Prompt 系统学习教程

> 本教程旨在深入理解 OpenCode 的 System Prompt 系统，通过源码分析和实践演练，帮助读者掌握 Agent开发中System Prompt 的架构设计、实现细节。

---

## 目录

| 章节 | 标题 |
|------|------|
| 一 | 系统概述 |
| 二 | 核心架构分析 |
| 三 | Provider 特定 Prompt |
| 四 | 环境信息注入 |
| 五 | 自定义规则加载 |
| 六 | Prompt 注入流程 |
| 七 | Agent 特定 Prompt |
| 八 | 实践演练 |
| 九 | 高级定制 |
| 十 | 常见问题 |

---

## 一、系统概述

### 1.1 什么是 System Prompt

**System Prompt** 是与大语言模型交互时传递给模型的第一条系统级消息，它定义了模型的身份定位、行为规范、工具权限和工作流程。

> **核心概念**：在 OpenCode 中，System Prompt 不仅仅是一段静态文本，而是一个由多个组件动态组合而成的复杂系统，涵盖了 LLM Provider 适配、环境感知、自定义规则和 Agent 特定配置等多个维度。

**分层设计的三大优势：**

- **灵活性高**：可以针对不同模型提供商定制不同的提示风格以适应不同模型的特性
- **可扩展性强**：新增提示来源只需添加对应的加载逻辑
- **维护性好**：各部分职责清晰，便于独立修改和测试

### 1.2 System Prompt 的组成结构

在 OpenCode 中，一个完整的 System Prompt 由以下 **四个核心部分** 组成：

| 部分 | 说明 |
|------|------|
| **Provider 特定提示** | 根据模型提供商选择对应的提示模板 |
| **环境信息** | 动态注入运行环境的上下文 |
| **自定义规则** | 从多个来源加载用户/项目级别的配置指令 |
| **Agent 特定提示** | 根据 Agent 类型注入额外的指令 |

这四个部分在运行时被组装成一个数组，作为系统消息传递给大语言模型。

**注入逻辑源码** (`packages/opencode/src/session/prompt.ts:591-610`)：

```typescript
const result = await processor.process({
  user: lastUser,
  agent,
  abort,
  sessionID,
  system: [...(await SystemPrompt.environment()), ...(await SystemPrompt.custom())],
  messages: [
    ...MessageV2.toModelMessage(sessionMessages),
    ...(isLastStep
      ? [
          {
            role: "assistant" as const,
            content: MAX_STEPS,
          },
        ]
      : []),
  ],
  tools,
  model,
})
```

从这段代码可以看出，`environment()` 和 `custom()` 两部分被合并后传递给 `processor.process` 方法。

### 1.3 核心文件概览

要深入理解 System Prompt 系统，需要重点关注以下核心文件：

| 文件 | 说明 |
|------|------|
| `packages/opencode/src/session/system.ts` | 定义 header、provider、environment、custom 四个关键函数 |
| `packages/opencode/src/session/prompt.ts` | Prompt 处理主文件，实现 System Prompt 注入逻辑 |
| `packages/opencode/src/agent/agent.ts` | 定义所有内置 Agent 的配置 |
| `packages/opencode/src/session/prompt/` | 包含所有 Provider 和 Agent 特定的提示模板 |

---

## 二、核心架构分析

### 2.1 SystemPrompt 命名空间详解

**SystemPrompt** 命名空间是整个 System Prompt 系统的核心，位于 `packages/opencode/src/session/system.ts`。

**四个公开函数：**

| 函数 | 职责 |
|------|------|
| `header()` | 根据 Provider ID 注入特定的标识提示 |
| `provider()` | 根据模型标识选择对应的提示模板 |
| `environment()` | 收集和格式化当前运行环境的上下文信息 |
| `custom()` | 从多个来源加载用户自定义的指令规则 |

**文件导入依赖：**

```typescript
import { Ripgrep } from "../file/ripgrep"
import { Global } from "../global"
import { Filesystem } from "../util/filesystem"
import { Config } from "../config/config"
import { Instance } from "../project/instance"
import path from "path"
import os from "os"

import PROMPT_ANTHROPIC from "./prompt/anthropic.txt"
import PROMPT_ANTHROPIC_WITHOUT_TODO from "./prompt/qwen.txt"
import PROMPT_BEAST from "./prompt/beast.txt"
import PROMPT_GEMINI from "./prompt/gemini.txt"
import PROMPT_ANTHROPIC_SPOOF from "./prompt/anthropic_spoof.txt"
import PROMPT_CODEX from "./prompt/codex.txt"
import type { Provider } from "@/provider/provider"
import { Flag } from "@/flag/flag"

export namespace SystemPrompt {
  // 四个公开函数定义
}
```

**核心依赖模块：**

- **Global**：提供全局配置路径
- **Config**：提供用户配置读取能力
- **Instance**：提供当前项目和工作目录信息
- **Filesystem**：提供文件系统操作能力

---

### 2.2 header 函数：Provider 标识注入

**功能**：根据 Provider ID 注入特定的标识提示。

```typescript
export function header(providerID: string) {
  if (providerID.includes("anthropic")) return [PROMPT_ANTHROPIC_SPOOF.trim()]
  return []
}
```

**逻辑说明：**
- 只有当 Provider ID 包含 `"anthropic"` 时，才返回特殊的 spoof 提示
- 否则返回空数组

**调用时机**：在 Agent 生成阶段调用，确保 Agent 配置能正确反映所使用的模型提供商身份。

```typescript
export async function generate(input: { description: string; model?: { providerID: string; modelID: string } }) {
  const cfg = await Config.get()
  const defaultModel = input.model ?? (await Provider.defaultModel())
  const model = await Provider.getModel(defaultModel.providerID, defaultModel.modelID)
  const language = await Provider.getLanguage(model)
  const system = SystemPrompt.header(defaultModel.providerID)
  system.push(PROMPT_GENERATE)
  // ... 后续处理
}
```

---

### 2.3 provider 函数：Provider 适配选择

**功能**：根据模型标识选择对应的提示模板。

```typescript
export function provider(model: Provider.Model) {
  if (model.api.id.includes("gpt-5")) return [PROMPT_CODEX]
  if (model.api.id.includes("gpt-") || model.api.id.includes("o1") || model.api.id.includes("o3"))
    return [PROMPT_BEAST]
  if (model.api.id.includes("gemini-")) return [PROMPT_GEMINI]
  if (model.api.id.includes("claude")) return [PROMPT_ANTHROPIC]
  return [PROMPT_ANTHROPIC_WITHOUT_TODO]
}
```

**模型适配对照表：**

| 模型系列 | 提示模板 | 说明 |
|----------|----------|------|
| GPT-5 / Codex | `codex.txt` | 专门的代码生成优化 |
| GPT-4 / GPT-3.5 / o1 / o3 | `beast.txt` | OpenAI 系列通用模板 |
| Gemini | `gemini.txt` | Google 模型专用模板 |
| Claude | `anthropic.txt` | Anthropic 模型默认模板 |
| 其他 | `qwen.txt` | 默认回退模板 |

**核心思想**：不同模型提供商对系统提示的理解和处理方式不同，需要针对每个提供商的特点调整提示的表述方式、指令格式和风格。

---

### 2.4 environment 函数：运行时环境信息

**功能**：收集和格式化当前运行环境的上下文信息。

```typescript
export async function environment() {
  const project = Instance.project
  return [
    [
      `Here is some useful information about the environment you are running in:`,
      `<env>`,
      `  Working directory: ${Instance.directory}`,
      `  Is directory a git repo: ${project.vcs === "git" ? "yes" : "no"}`,
      `  Platform: ${process.platform}`,
      `  Today's date: ${new Date().toDateString()}`,
      `</env>`,
      `<files>`,
      `${
        project.vcs === "git" && false
          ? await Ripgrep.tree({
              cwd: Instance.directory,
              limit: 200,
            })
          : ""
      }`,
      `</files>`,
    ].join("\n"),
  ]
}
```

**收集的环境信息字段：**

| 字段 | 说明 | 作用 |
|------|------|------|
| `Working directory` | 当前工作目录绝对路径 | 解析相对路径的基础 |
| `Is directory a git repo` | 是否为 Git 仓库 | 判断是否可以使用 Git 工具 |
| `Platform` | 运行平台标识 (darwin/linux/win32) | 生成适合当前平台的命令 |
| `Today's date` | 当前日期 | 理解时间敏感的上下文 |

**格式化示例：**

```xml
<env>
  Working directory: /Users/felix/learningspace/opencode
  Is directory a git repo: yes
  Platform: darwin
  Today's date: Wed Jan 07 2026
</env>
<files>
  [文件列表，被禁用]
</files>
```

> **注意**：文件列表显示功能目前被 `false` 条件禁用。如需启用，将 `false` 改为 `true` 即可。

---

### 2.5 custom 函数：自定义规则加载

**功能**：从多个来源加载用户自定义的指令规则。

**加载策略分为三个层次：**

| 层次 | 说明 | 优先级 |
|------|------|--------|
| **本地项目级** | 向上查找 AGENTS.md/CLAUDE.md/CONTEXT.md | 高 |
| **全局用户级** | 检查全局配置目录 | 中 |
| **配置指令** | 从 config.instructions 加载 | 低 |

**文件查找配置：**

```typescript
const LOCAL_RULE_FILES = [
  "AGENTS.md",
  "CLAUDE.md",
  "CONTEXT.md", // deprecated
]
const GLOBAL_RULE_FILES = [
  path.join(Global.Path.config, "AGENTS.md"),
  path.join(os.homedir(), ".claude", "CLAUDE.md")
]

if (Flag.OPENCODE_CONFIG_DIR) {
  GLOBAL_RULE_FILES.push(path.join(Flag.OPENCODE_CONFIG_DIR, "AGENTS.md"))
}
```

**本地规则加载策略：**
- 从当前目录向上遍历目录树查找
- 优先级：`AGENTS.md > CLAUDE.md > CONTEXT.md`
- 找到第一个即停止（break）

**全局规则加载策略：**
- 检查所有定义的位置
- 一旦找到任何存在的文件即停止

**配置指令支持的格式：**
- 本地文件路径（支持相对/绝对路径，波浪号展开）
- glob 模式（如 `**/*.md`）
- 远程 URL

**完整代码实现：**

```typescript
export async function custom() {
  const config = await Config.get()
  const paths = new Set<string>()

  // 本地规则查找
  for (const localRuleFile of LOCAL_RULE_FILES) {
    const matches = await Filesystem.findUp(localRuleFile, Instance.directory, Instance.worktree)
    if (matches.length > 0) {
      matches.forEach((path) => paths.add(path))
      break
    }
  }

  // 全局规则查找
  for (const globalRuleFile of GLOBAL_RULE_FILES) {
    if (await Bun.file(globalRuleFile).exists()) {
      paths.add(globalRuleFile)
      break
    }
  }

  // 配置指令处理
  const urls: string[] = []
  if (config.instructions) {
    for (let instruction of config.instructions) {
      // URL 处理
      if (instruction.startsWith("https://") || instruction.startsWith("http://")) {
        urls.push(instruction)
        continue
      }
      // 路径展开
      if (instruction.startsWith("~/")) {
        instruction = path.join(os.homedir(), instruction.slice(2))
      }
      // 文件匹配
      let matches: string[] = []
      if (path.isAbsolute(instruction)) {
        matches = await Array.fromAsync(
          new Bun.Glob(path.basename(instruction)).scan({
            cwd: path.dirname(instruction),
            absolute: true,
            onlyFiles: true,
          }),
        ).catch(() => [])
      } else {
        matches = await Filesystem.globUp(instruction, Instance.directory, Instance.worktree).catch(() => [])
      }
      matches.forEach((path) => paths.add(path))
    }
  }

  // 异步加载文件内容
  const foundFiles = Array.from(paths).map((p) =>
    Bun.file(p)
      .text()
      .catch(() => "")
      .then((x) => "Instructions from: " + p + "\n" + x),
  )

  // 异步获取远程内容
  const foundUrls = urls.map((url) =>
    fetch(url, { signal: AbortSignal.timeout(5000) })
      .then((res) => (res.ok ? res.text() : ""))
      .catch(() => "")
      .then((x) => (x ? "Instructions from: " + url + "\n" + x : "")),
  )

  return Promise.all([...foundFiles, ...foundUrls]).then((result) => result.filter(Boolean))
}
```

**错误处理机制**：
- 文件读取失败静默返回空字符串
- 网络请求超时静默返回空字符串
- 最终通过 `filter(Boolean)` 过滤空值

---

## 三、Provider 特定 Prompt

### 3.1 Prompt 模板文件概述

**模板目录**：`packages/opencode/src/session/prompt/`

**可用模板文件：**

| 文件 | 用途 |
|------|------|
| `anthropic.txt` | Claude 系列模型的默认提示 |
| `beast.txt` | GPT-4/o 系列模型的提示 |
| `gemini.txt` | Google 模型的提示 |
| `codex.txt` | GPT-5/Codex 系列的提示 |
| `qwen.txt` | 其他模型的默认提示 |
| `anthropic_spoof.txt` | 针对 Anthropic 的特殊标识提示 |
| `plan.txt` | plan agent 的提示 |
| `explore.txt` | explore agent 的提示 |
| `build-switch.txt` | agent 切换时的提示 |
| `max-steps.txt` | 最大步数限制的提示 |
| `plan-reminder-anthropic.txt` | 针对 Anthropic 的计划提醒提示 |

---

### 3.2 提示模板对比分析

**三大 Provider 模板的设计差异：**

| 维度 | anthropic.txt | beast.txt | gemini.txt |
|------|---------------|-----------|------------|
| **风格** | 散文式结构 | 指令化 | 列表式 |
| **长度** | 详细冗长 | 简洁直接 | 中等 |
| **结构** | 章节分层 | 连续段落 | 规则列表 |
| **特点** | 大量示例说明 | 强调自主性 | 明确的强制规则 |

**三大设计考量：**

1. **身份定位一致性**
   - 无论使用哪个 Provider，OpenCode 都被定位为专注于软件工程任务的命令行工具

2. **风格结构差异性**
   - anthropic.txt：宽松的散文式结构
   - beast.txt：指令化，强调自主性
   - gemini.txt：核心规则列表

3. **安全策略统一性**
   - 所有模板都包含 URL 生成、文件创建、安全实践等方面的安全规则

---

### 3.3 anthropic.txt 深度分析

**文件路径**：`packages/opencode/src/session/prompt/anthropic.txt`

**完整内容：**

```markdown
You are OpenCode, the best coding agent on the planet.

You are an interactive CLI tool that helps users with software engineering tasks. Use the instructions below and the tools available to you to assist the user.

IMPORTANT: You must NEVER generate or guess URLs for the user unless you are confident that the URLs are for helping the user with programming. You may use URLs provided by the user in their messages or local files.

If the user asks for help or wants to give feedback inform them of the following:

- ctrl+p to list available actions
- To give feedback, users should report the issue at
  https://github.com/anomalyco/opencode

When the user directly asks about OpenCode (eg. "can OpenCode do...", "does OpenCode have..."), or asks in second person (eg. "are you able...", "can you do..."), or asks how to use a specific OpenCode feature (eg. implement a hook, write a slash command, or install an MCP server), use the WebFetch tool to gather information to answer the question from OpenCode docs. The list of available docs is available at https://opencode.ai/docs
```

**关键设计要点：**

| 要点 | 说明 |
|------|------|
| **身份定位** | 定义为"最佳编码 agent"，设定高质量输出的期望 |
| **安全原则** | 禁止生成或猜测 URL，防止链接到恶意/失效资源 |
| **反馈渠道** | 引导用户到 GitHub issues 页面 |

---

**风格与语气：**

```markdown
# Tone and style

- Only use emojis if the user explicitly requests it. Avoid using emojis in all communication unless asked.
- Your output will be displayed on a command line interface. Your responses should be short and concise. You can use Github-flavored markdown for formatting, and will be rendered in a monospace font using the CommonCham specification.
- Output text to communicate with the user; all text you output outside of tool use is displayed to the user. Only use tools to complete tasks. Never use tools like Bash or code comments as means to communicate with the user during the session.
- NEVER create files unless they're absolutely necessary for achieving your goal. ALWAYS prefer editing an existing file to creating a new one. This includes markdown files.
```

**风格规范：**

- **emoji 使用**：除非用户明确要求，否则不使用
- **输出格式**：简短直接，使用 GitHub 风格 Markdown
- **沟通方式**：仅使用工具进行实际操作，不用注释沟通
- **文件操作**：优先编辑现有文件，避免创建新文件

---

**专业客观性：**

```markdown
# Professional objectivity

Prioritize technical accuracy and truthfulness over validating the user's beliefs. Focus on facts and problem-solving, providing direct, objective technical info without any unnecessary superlatives, praise, or emotional validation. It is best for the user if OpenCode honestly applies the same rigorous standards to all ideas and disagrees when necessary, even if it may not be what the user wants to hear. Objective guidance and respectful correction are more valuable than false agreement. Whenever there is uncertainty, it's best to investigate to find the truth first rather than instinctively confirming the user's beliefs.
```

**核心价值观**：优先考虑技术准确性和真实性，而非迎合用户信念。

---

**任务管理：**

```markdown
# Task Management

You have access to the TodoWrite tools to help you manage and plan tasks. Use these tools VERY frequently to ensure that you are tracking your tasks and giving the user visibility into your progress.
These tools are also EXTREMELY helpful for planning tasks, and for breaking down larger complex tasks into smaller steps. If you do not use this tool when planning, you may forget to do important tasks - and that is unacceptable.

It is critical that you mark todos as completed as soon as you are done with a task. Do not batch up multiple tasks before marking them as completed.
```

**任务管理原则：**
- 频繁使用 TodoWrite 工具跟踪任务进度
- 及时标记任务完成状态
- 避免一次性批量标记多个任务

---

### 3.4 beast.txt 深度分析

**文件路径**：`packages/opencode/src/session/prompt/beast.txt`

**完整内容：**

```markdown
You are opencode, an agent - please keep going until the user's query is completely resolved, before ending your turn and yielding back to the user.

Your thinking should be thorough and it's fine if it's very long. However, avoid unnecessary repetition and verbosity. You should be concise, but thorough.

You MUST iterate and keep going until the problem is solved.

You have everything you need to resolve this problem. I want you to fully solve this autonomously before coming back to me.
```

**核心特点：**

- **自主性**：要求模型持续工作直到问题完全解决
- **彻底性**：鼓励深入思考，但避免冗余
- **持久性**：必须迭代直到问题解决

---

**互联网研究要求：**

```markdown
THE PROBLEM CAN NOT BE SOLVED WITHOUT EXTENSIVE INTERNET RESEARCH.

You must use the webfetch tool to recursively gather all information from URL's provided to you by the user, as well as any links you find in the content of those pages.

Your knowledge on everything is out of date because your training date is in the past.

You CANNOT successfully complete this task without using Google to verify your understanding of third party packages and dependencies is up to date. You must use the webfetch tool to search google for how to properly use libraries, packages, frameworks, dependencies, etc. every single time you install or implement one. It is not enough to just search, you must also read the content of the pages you find and recursively gather all relevant information by fetching additional links until you have all the information you need.
```

**研究原则：**
- 模型知识可能过时，必须进行互联网研究
- 不仅要搜索，还要阅读页面内容
- 递归获取所有相关链接直到获得完整信息

---

**计划与反思：**

```markdown
You MUST plan extensively before each function call, and reflect extensively on the outcomes of the previous function calls. DO NOT do this entire process by making function calls only, as this can impair your ability to solve the problem and think insightfully.
```

**行为准则**：在每次函数调用前进行广泛规划，反思之前调用的结果。

---

### 3.5 gemini.txt 深度分析

**文件路径**：`packages/opencode/src/session/prompt/gemini.txt`

**完整内容：**

```markdown
You are opencode, an interactive CLI agent specializing in software engineering tasks. Your primary goal is to help users safely and efficiently, adhering strictly to the following instructions and utilizing your available tools.

# Core Mandates

- **Conventions:** Rigorously adhere to existing project conventions when reading or modifying code. Analyze surrounding code, tests, and configuration first.
- **Libraries/Frameworks:** NEVER assume a library/framework is available or appropriate. Verify its established usage within the project (check imports, configuration files like 'package.json', 'Cargo.toml', 'requirements.txt', 'build.gradle', etc., or observe neighboring files) before employing it.
- **Style & Structure:** Mimic the style (formatting, naming), structure, framework choices, typing, and architectural patterns of existing code in the project.
- **Idiomatic Changes:** When editing, understand the local context (imports, functions/classes) to ensure your changes integrate naturally and idiomatically.
- **Comments:** Add code comments sparingly. Focus on _why_ something is done, especially for complex logic, rather than _what_ is done. Only add high-value comments if necessary for clarity or if requested by the user. Do not edit comments that are separate from the code you are changing. _NEVER_ talk to the user or describe your changes through comments.
- **Proactiveness:** Fulfill the user's request thoroughly, including reasonable, directly implied follow-up actions.
- **Confirm Ambiguity/Expansion:** Do not take significant actions beyond the clear scope of the request without confirming with the user. If asked _how_ to do something, explain first, don't just do it.
- **Explaining Changes:** After completing a code modification or file operation _do not_ provide summaries unless asked.
- **Path Construction:** Before using any file system tool (e.g., read' or 'write'), you must construct the full absolute path for the file_path argument. Always combine the absolute path of the project's root directory with the file's path relative to the root. For example, if the project root is /path/to/project/ and the file is foo/bar/baz.txt, the final path you must use is /path/to/project/foo/bar/baz.txt. If the user provides a relative path, you must resolve it against the root directory to create an absolute path.
- **Do Not revert changes:** Do not revert changes to the codebase unless asked to do so by the user. Only revert changes made by you if they have resulted in an error or if the user has explicitly asked you to revert the changes.
```

**核心规则 (Core Mandates)：**

| 规则 | 说明 |
|------|------|
| **Conventions** | 严格遵守项目现有约定 |
| **Libraries/Frameworks** | 验证库和框架的可用性 |
| **Style & Structure** | 模仿现有代码风格 |
| **Idiomatic Changes** | 理解本地上下文 |
| **Comments** | 谨慎添加注释 |
| **Proactiveness** | 彻底完成任务 |
| **Confirm Ambiguity** | 不确定时先确认 |
| **Explaining Changes** | 不主动提供修改总结 |
| **Path Construction** | 必须使用绝对路径 |
| **Do Not revert** | 不随意回滚更改 |

**结构特点**：使用核心规则的列表形式，结构清晰，便于模型逐条理解和执行。

---

## 四、环境信息注入

### 4.1 环境信息的组成

**环境信息的作用**：为模型提供运行时的上下文，帮助模型理解当前环境并做出正确的决策。

**核心元素：**

| 元素 | 说明 | 示例值 |
|------|------|--------|
| `Working directory` | 当前工作目录绝对路径 | `/Users/felix/learningspace/opencode` |
| `Is directory a git repo` | 是否为 Git 仓库 | `yes` / `no` |
| `Platform` | 运行平台标识 | `darwin` / `linux` / `win32` |
| `Today's date` | 当前日期 | `Wed Jan 08 2026` |

**格式化示例：**

```xml
<env>
  Working directory: /Users/felix/learningspace/opencode
  Is directory a git repo: yes
  Platform: darwin
  Today's date: Wed Jan 08 2026
</env>
<files>
  [文件列表，被禁用]
</files>
```

**获取来源**：
- 工作目录：`Instance.directory`
- Git 仓库：`Instance.project.vcs`
- 平台信息：`process.platform`
- 当前日期：`new Date().toDateString()`

---

### 4.2 环境信息的获取机制

**依赖模块**：Instance 模块

**Instance 属性说明：**

| 属性 | 说明 | 类型 |
|------|------|------|
| `Instance.directory` | 当前工作目录绝对路径 | `string` |
| `Instance.worktree` | Git 工作树的根目录 | `string` |
| `Instance.project` | 当前项目配置信息 | `Project` |

**获取示例：**

```typescript
const project = Instance.project
const directory = Instance.directory
const platform = process.platform
const today = new Date().toDateString()
```

---

### 4.3 环境信息的作用

**四大作用：**

| 作用 | 说明 | 示例 |
|------|------|------|
| **路径解析** | 解析相对路径的基础 | 读取文件时确定绝对路径 |
| **工具选择** | 根据平台生成正确的命令 | `bash` vs `PowerShell` |
| **时间感知** | 理解时间敏感的上下文 | 生成符合当前日期的代码 |
| **能力判断** | 判断是否可以使用 Git 工具 | 非 Git 仓库不尝试 Git 操作 |

**具体应用场景：**

- **路径解析**：当模型需要读取或写入文件时，需要知道当前工作目录来解析相对路径
- **工具选择**：不同平台有不同的命令和语法，平台信息帮助模型选择正确的命令格式
- **时间感知**：当前日期帮助模型理解时间敏感的上下文
- **能力判断**：Git 仓库信息帮助模型判断是否可以使用 Git 相关工具

---

## 五、自定义规则加载

### 5.1 自定义规则的来源

**四个主要渠道：**

| 渠道 | 说明 | 配置位置 |
|------|------|----------|
| **本地项目级** | 查找 AGENTS.md / CLAUDE.md / CONTEXT.md | 项目根目录或父目录 |
| **全局用户级** | 查找全局配置目录 | `~/.claude/` 或配置目录 |
| **配置指令** | config.instructions 配置项 | opencode.json |
| **环境变量** | OPENCODE_CONFIG_DIR | 环境变量 |

**加载优先级**：本地 > 全局 > 配置指令

---

### 5.2 文件查找策略

**核心工具函数：**

| 函数 | 作用 |
|------|------|
| `Filesystem.findUp()` | 向上遍历目录树查找目标文件 |
| `Filesystem.globUp()` | 支持 glob 模式的文件查找 |

**查找行为：**

- `findUp`：从指定目录开始，向上遍历，找到文件即返回路径
- `globUp`：匹配符合模式的多个文件，返回所有匹配结果

---

### 5.3 规则文件的加载和合并

**合并策略**：

- 每个文件内容前添加「Instructions from:」前缀
- 远程 URL 的内容同样添加来源标注
- 所有规则都会被加载，不会因为找到一个文件就忽略其他文件

**错误处理机制：**

```typescript
// 文件读取失败静默处理
.catch(() => "")

// 网络请求超时处理
fetch(url, { signal: AbortSignal.timeout(5000) })
  .then((res) => (res.ok ? res.text() : ""))
  .catch(() => "")

// 过滤空值
.filter(Boolean)
```

**特点**：
- 失败后返回空字符串，最终被过滤掉
- 一个来源加载失败不影响其他来源

---

### 5.4 AGENTS.md 文件的格式和用法

**文件作用**：定义项目特定的代码规范和规则

**AGENTS.md 格式与示例**：

完整的 AGENTS.md 编写指南请参考 [agents.md](https://agents.md/)。

**加载后影响**：模型在进行代码编写时会遵循这些规范

---

## 六、Prompt 注入流程

### 6.1 Prompt 处理主流程

**文件位置**：`packages/opencode/src/session/prompt.ts`

**核心逻辑**：在主循环中，System Prompt 的注入发生在 `processor.process` 方法调用时。

**注入代码** (`packages/opencode/src/session/prompt.ts:591-610`)：

```typescript
const result = await processor.process({
  user: lastUser,
  agent,
  abort,
  sessionID,
  system: [...(await SystemPrompt.environment()), ...(await SystemPrompt.custom())],
  messages: [
    ...MessageV2.toModelMessage(sessionMessages),
    ...(isLastStep
      ? [
          {
            role: "assistant" as const,
            content: MAX_STEPS,
          },
        ]
      : []),
  ],
  tools,
  model,
})
```

**参数说明：**

| 参数 | 说明 |
|------|------|
| `system` | `environment()` 和 `custom()` 两部分的合并结果 |
| `messages` | 会话消息数组，转换为模型格式 |
| `tools` | 当前会话可用的工具定义 |
| `model` | 模型配置信息 |

**system 参数**：接收 `environment()` 和 `custom()` 两部分的合并结果，作为系统消息传递给模型。

---

### 6.2 Processor 的角色

**SessionProcessor**：处理与模型实际交互的核心组件

**职责**：
- 接收所有必要的信息（用户消息、系统提示、工具定义、模型配置等）
- 构造和发送请求到模型提供商

**Processor 创建** (`packages/opencode/src/session/prompt.ts:571-599`)：

```typescript
const processor = SessionProcessor.create({
  assistantMessage: (await Session.updateMessage({
    id: Identifier.ascending("message"),
    parentID: lastUser.id,
    role: "assistant",
    mode: agent.name,
    agent: agent.name,
    path: {
      cwd: Instance.directory,
      root: Instance.worktree,
    },
    cost: 0,
    tokens: {
      input: 0,
      output: 0,
      reasoning: 0,
      cache: { read: 0, write: 0 },
    },
    modelID: model.id,
    providerID: model.providerID,
    time: {
      created: Date.now(),
    },
    sessionID,
  })) as MessageV2.Assistant,
  sessionID: sessionID,
  model,
  abort,
})
```

**初始化信息**：
- 记录交互状态和结果
- 跟踪成本、令牌使用、工具调用等信息

---

### 6.3 消息转换

**转换函数**：`MessageV2.toModelMessage`

**消息格式**：

```typescript
messages: [
  ...MessageV2.toModelMessage(sessionMessages),
  ...(isLastStep
    ? [
        {
          role: "assistant" as const,
          content: MAX_STEPS,
        },
      ]
    : []),
]
```

**消息来源**：

| 消息类型 | 说明 |
|----------|------|
| `sessionMessages` | 会话中所有消息（用户消息 + assistant 消息） |
| `MAX_STEPS` | 最后一步时的总结提示 |

**系统消息传递**：
- 通过 `system` 参数单独传递
- 符合 OpenAI/Anthropic API 的消息格式要求

---

### 6.4 工具解析

**工具解析函数**：`resolveTools`

**工具处理流程**：

```typescript
const tools = await resolveTools({
  agent,
  session,
  model,
  tools: lastUser.tools,
  processor,
  userInvokedAgents,
})
```

**处理步骤**：
1. 从 `ToolRegistry` 获取可用工具
2. 根据当前 Agent 和模型配置进行过滤
3. 转换为模型可理解的格式（JSON Schema）
4. 与系统提示一起发送给模型

---

### 6.5 最大步数处理

**步数检查**：

```typescript
const agent = await Agent.get(lastUser.agent)
const maxSteps = agent.steps ?? Infinity
const isLastStep = step >= maxSteps
```

**最后一步处理**：

```typescript
...(isLastStep
  ? [
      {
        role: "assistant" as const,
        content: MAX_STEPS,
      },
    ]
  : []),
```

**MAX_STEPS 作用**：要求模型给出完成情况的总结和建议

---

---

## 七、Agent 特定 Prompt

### 7.1 Agent 定义结构

**文件位置**：`packages/opencode/src/agent/agent.ts`

**定义结构：**

```typescript
export const Info = z
  .object({
    name: z.string(),
    description: z.string().optional(),
    mode: z.enum(["subagent", "primary", "all"]),
    native: z.boolean().optional(),
    hidden: z.boolean().optional(),
    topP: z.number().optional(),
    temperature: z.number().optional(),
    color: z.string().optional(),
    permission: PermissionNext.Ruleset,
    model: z
      .object({
        modelID: z.string(),
        providerID: z.string(),
      })
      .optional(),
    prompt: z.string().optional(),
    options: z.record(z.string(), z.any()),
    steps: z.number().int().positive().optional(),
  })
  .meta({
    ref: "Agent",
  })
```

**关键属性说明：**

| 属性 | 类型 | 说明 |
|------|------|------|
| `name` | string | Agent 名称 |
| `mode` | enum | 模式：subagent / primary / all |
| `prompt` | string | Agent 特定提示 |
| `permission` | Ruleset | 权限配置 |
| `steps` | int | 最大步数限制 |

---

### 7.2 内置 Agent 分析

**内置 Agent 列表：**

| Agent | 模式 | 用途 |
|-------|------|------|
| `build` | primary | 默认的主要开发 Agent |
| `plan` | primary | 只读计划模式 |
| `explore` | subagent | 代码探索专用 Agent |

---

**build Agent** - 默认的主要开发 Agent：

```typescript
build: {
  name: "build",
  options: {},
  permission: PermissionNext.merge(defaults, user),
  mode: "primary",
  native: true,
},
```

**特点**：
- 没有额外的 `prompt` 属性
- 只使用 System Prompt（Provider 提示、环境信息、自定义规则）
- 具有完整的权限（除 doom_loop 和 external_directory 需询问）

---

**plan Agent** - 只读计划模式：

```typescript
plan: {
  name: "plan",
  options: {},
  permission: PermissionNext.merge(
    defaults,
    PermissionNext.fromConfig({
      edit: {
        "*": "deny",
        ".opencode/plan/*.md": "allow",
      },
    }),
    user,
  ),
  mode: "primary",
  native: true,
},
```

**特点**：
- 禁止所有编辑操作
- 只允许在 `.opencode/plan/` 目录下创建文件
- 只能用于规划和阅读，不能直接修改代码库

---

**explore Agent** - 代码探索专用 Agent：

```typescript
explore: {
  name: "explore",
  permission: PermissionNext.merge(
    defaults,
    PermissionNext.fromConfig({
      "*": "deny",
      grep: "allow",
      glob: "allow",
      list: "allow",
      bash: "allow",
      webfetch: "allow",
      websearch: "allow",
      codesearch: "allow",
      read: "allow",
    }),
    user,
  ),
  description: `Fast agent specialized for exploring codebases...`,
  prompt: PROMPT_EXPLORE,
  options: {},
  mode: "subagent",
  native: true,
},
```

**特点**：
- 有专门的 `prompt` 属性（引用 `prompt/explore.txt`）
- 权限限制为只读操作
- 禁止其他所有操作（编辑、bash 执行等）

---

### 7.3 Agent Prompt 注入

**两种注入方式：**

| 方式 | 说明 | 优点 |
|------|------|------|
| **Agent 定义** | 在 Agent 配置中设置 `prompt` 属性 | 简单直接 |
| **insertReminders** | 在用户消息中添加合成文本 | 可被上下文压缩处理 |

**方式一：Agent 定义设置**

```typescript
explore: {
  // ...
  prompt: PROMPT_EXPLORE,
  // ...
},
```

**方式二：insertReminders 函数**

```typescript
function insertReminders(input: { messages: MessageV2.WithParts[]; agent: Agent.Info }) {
  const userMessage = input.messages.findLast((msg) => msg.info.role === "user")
  if (!userMessage) return input.messages
  if (input.agent.name === "plan") {
    userMessage.parts.push({
      id: Identifier.ascending("part"),
      messageID: userMessage.info.id,
      sessionID: userMessage.info.sessionID,
      type: "text",
      text: PROMPT_PLAN,
      synthetic: true,
    })
  }
  // ...
}
```

---

### 7.4 自定义 Agent 配置

**配置位置**：`config.agent`

**支持的配置属性：**

| 属性 | 说明 |
|------|------|
| `model` | 指定使用的模型 |
| `prompt` | 指定 Agent 提示 |
| `description` | Agent 描述 |
| `temperature` | 温度参数 |
| `mode` | 模式 |
| `steps` | 最大步数 |
| `permission` | 权限配置 |

**配置加载逻辑：**

```typescript
for (const [key, value] of Object.entries(cfg.agent ?? {})) {
  if (value.disable) {
    delete result[key]
    continue
  }
  let item = result[key]
  if (!item)
    item = result[key] = {
      name: key,
      mode: "all",
      permission: PermissionNext.merge(defaults, user),
      options: {},
      native: false,
    }
  // 属性覆盖
  if (value.model) item.model = Provider.parseModel(value.model)
  item.prompt = value.prompt ?? item.prompt
  item.description = value.description ?? item.description
  item.temperature = value.temperature ?? item.temperature
  item.topP = value.top_p ?? item.topP
  item.mode = value.mode ?? item.mode
  item.color = value.color ?? item.color
  item.hidden = value.hidden ?? item.hidden
  item.name = value.name ?? item.name
  item.steps = value.steps ?? item.steps
  item.options = mergeDeep(item.options, value.options ?? {})
  item.permission = PermissionNext.merge(item.permission, PermissionNext.fromConfig(value.permission ?? {}))
}
```

---

---

## 八、实践演练

### 8.1 实践一：修改 System Prompt 添加项目规范

**目标**：在项目中添加自定义规范，影响 OpenCode 的行为方式

**步骤**：在项目根目录创建 `AGENTS.md` 文件

**AGENTS.md 示例内容：**

```markdown
# 项目代码规范

## 命名规范

- 组件使用 PascalCase 命名
- 变量和函数使用 camelCase 命名
- 常量使用 UPPER_SNAKE_CASE 命名
- 文件名使用 kebab-case 命名

## 代码风格

- 使用 2 空格缩进
- 字符串优先使用单引号
- 每行最大长度 100 字符
- 所有 async 函数必须处理错误

## 测试要求

- 所有新功能必须添加单元测试
- 测试覆盖率不低于 80%
- 使用 Bun 原生测试框架

## 文档要求

- 公开 API 必须添加 JSDoc 注释
- 复杂逻辑必须添加行内注释
- README 保持更新
```

**效果**：OpenCode 会自动加载这个文件，规范会被添加到 System Prompt 中，影响模型的代码生成行为。

---

### 8.2 实践二：创建自定义 Agent

**目标**：创建一个专注于代码审查的 Agent

**配置位置**：`opencode.json` 或通过 Config API

**Agent 配置示例：**

```json
{
  "agent": {
    "review": {
      "description": "Specialized agent for code review and quality analysis",
      "mode": "subagent",
      "prompt": "You are a code review specialist. Your role is to:\n\n1. Analyze code for potential bugs and security vulnerabilities\n2. Check adherence to project coding standards\n3. Suggest performance improvements\n4. Identify code smells and refactoring opportunities\n5. Ensure proper documentation and comments\n\nWhen reviewing:\n- Be specific about issues found\n- Provide examples of improved code when possible\n- Distinguish between critical issues and style suggestions\n- Focus on substantive improvements over minor nitpicks\n",
      "permission": {
        "read": "allow",
        "grep": "allow",
        "glob": "allow",
        "list": "allow",
        "*": "deny"
      },
      "steps": 50
    }
  }
}
```

**Agent 特点：**

| 特性 | 说明 |
|------|------|
| 模式 | subagent（不能被用户直接调用） |
| 权限 | 只读操作（read、grep、glob、list） |
| 步数限制 | 50 步 |

**调用方式**：`@review 请审查 src/utils/ 目录下的代码`

---

### 8.3 实践三：理解 Prompt 加载顺序

**目标**：验证 System Prompt 各部分的加载顺序和优先级

**测试项目结构：**

```
my-project/
├── AGENTS.md          # 优先级 1
├── CLAUDE.md          # 优先级 2（如果 AGENTS.md 不存在）
├── opencode.json      # Agent 配置
└── src/
    └── main.ts
```

**验证步骤**：

1. 在 `AGENTS.md` 中添加测试标记
2. 在 `~/.claude/CLAUDE.md` 中创建全局规则
3. 在 `opencode.json` 中配置远程指令

**加载顺序**：本地 AGENTS.md > 全局规则 > 配置指令

---

### 8.4 实践四：调试 System Prompt

**方法列表：**

| 方法 | 说明 |
|------|------|
| **检查日志输出** | 开发模式下会输出 System Prompt 组成 |
| **检查规则加载** | 在 Filesystem.findUp 调用处添加日志 |
| **检查消息内容** | 查看 Session 中存储的系统消息 |

**调试代码示例：**

```typescript
// 在 prompt.ts 的 processor.process 调用前后添加日志
console.log("System Prompt parts:")
for (const part of [...(await SystemPrompt.environment()), ...(await SystemPrompt.custom())]) {
  console.log("- " + part.substring(0, 100) + "...")
}
```

---

### 8.5 实践五：创建 Provider 特定提示

**步骤一**：确定模型提供商标识

查看 `provider.ts` 中 Provider ID 的格式（提供商名称 + 模型名称）

**步骤二**：添加判断逻辑

```typescript
export function provider(model: Provider.Model) {
  if (model.api.id.includes("new-provider")) return [PROMPT_NEW_PROVIDER]
  // ... 现有逻辑
}
```

**步骤三**：创建提示模板

创建 `prompt/new-provider.txt`，包含：
- 模型身份定位
- 行为规范
- 工具使用说明
- 输出格式要求

**步骤四**：导入新模板

```typescript
import PROMPT_NEW_PROVIDER from "./prompt/new-provider.txt"
```

---

---

## 九、高级定制

### 9.1 动态 System Prompt

**实现方法：**

| 方法 | 说明 |
|------|------|
| **配置指令** | config.instructions 包含动态生成的文件路径 |
| **环境变量** | OPENCODE_CONFIG_DIR 指向不同的配置目录 |
| **修改源码** | 直接修改 system.ts 中的 custom 函数 |

---

### 9.2 条件提示

**实现方式：**

| 方式 | 说明 |
|------|------|
| **文件条件** | 在 AGENTS.md 中包含条件逻辑 |
| **Agent 切换** | 创建不同的 Agent 配置 |
| **配置分离** | 通过环境变量切换不同配置文件 |

---

### 9.3 多语言支持

**实现方法：**

| 方法 | 说明 | 优点 |
|------|------|------|
| **修改提示模板** | 创建不同语言的提示模板文件 | 完全定制 |
| **使用自定义规则** | 在 AGENTS.md 中添加语言指导规则 | 不需要修改源码 |

**示例规则**：`使用中文回复`

---

### 9.4 集成外部规则系统

**集成方式：**

| 方式 | 说明 | 复杂度 |
|------|------|--------|
| **配置指令 URL** | 指向外部规则文件的 URL | 低 |
| **修改 custom 函数** | 从数据库或 API 获取规则 | 高 |
| **文件监控** | 外部系统重写 AGENTS.md | 中 |

---

### 9.5 性能优化

**优化建议：**

| 建议 | 说明 |
|------|------|
| **减少规则文件数量** | 合并相关规则到少数几个文件 |
| **使用简单的 glob 模式** | 避免复杂的文件匹配 |
| **缓存远程内容** | 减少网络请求 |
| **限制规则文件大小** | 控制上下文长度 |

---

---

## 十、常见问题

### 10.1 System Prompt 没有生效

**排查步骤：**

| 步骤 | 检查项 | 说明 |
|------|--------|------|
| 1 | 文件位置和名称 | 确保 AGENTS.md 位于正确的目录 |
| 2 | 文件格式 | 确认是有效的文本文件，不是二进制 |
| 3 | 加载时机 | 修改后需要重新启动会话 |
| 4 | 配置冲突 | 检查 ~/.claude/CLAUDE.md 是否覆盖项目配置 |
| 5 | 调试日志 | 查看规则文件是否被正确找到和加载 |

**支持的规则文件**：`AGENTS.md` / `CLAUDE.md` / `CONTEXT.md`

---

### 10.2 提示模板不匹配模型

**排查步骤：**

1. 确认模型标识（provider.ts 中的 Provider ID 格式）
2. 检查模板选择逻辑（在 system.ts 的 provider 函数中添加日志）
3. 验证模板内容（确保适合该模型的特点）
4. 测试其他模型（排除特定模型问题）

---

### 10.3 规则文件冲突

**加载策略**：
- 本地域查找：`AGENTS.md > CLAUDE.md > CONTEXT.md`
- 找到第一个即停止，不会加载同级别的其他文件

**解决方法**：将多个规则文件的内容合并到一个文件中

---

### 10.4 远程规则加载失败

**可能原因**：
- 网络连接问题
- URL 不存在或返回错误
- 超时

**处理方式**：
- 静默失败（返回空字符串，最终被过滤掉）
- 一个 URL 加载失败不影响其他规则

**验证方法**：
- 使用 curl 测试 URL 可用性
- 查看网络请求日志
- 临时替换为本地文件路径测试

---

### 10.5 Agent 提示不生效

**排查步骤：**

| 步骤 | 检查项 |
|------|--------|
| 1 | JSON 配置格式（prompt 属性是有效的字符串） |
| 2 | Agent 名称（配置的名称与调用的一致） |
| 3 | 属性覆盖逻辑（不会覆盖从外部文件导入的 prompt） |
| 4 | Agent 加载（确认自定义 Agent 已被正确加载） |

---

### 10.6 提示内容过长

**优化建议：**

| 建议 | 说明 |
|------|------|
| 精简内容 | 只保留必要的规则 |
| 移除过时规则 | 删除已不再适用的规则 |
| 简洁表达 | 使用简洁的语言 |
| 文档分离 | 将详细规范独立出来 |

**补充方案**：使用上下文压缩功能（OpenCode 会自动压缩过长的上下文）

---

---

## 附录

### 附录一、关键文件路径速查

| 分类 | 文件 | 路径 | 说明 |
|------|------|------|------|
| **核心模块** | SystemPrompt 核心 | `packages/opencode/src/session/system.ts` | 定义 prompt 加载逻辑 |
| | Prompt 处理 | `packages/opencode/src/session/prompt.ts` | System Prompt 注入点 |
| | Agent 定义 | `packages/opencode/src/agent/agent.ts` | Agent 配置和默认提示 |
| **提示模板** | Anthropic 模板 | `packages/opencode/src/session/prompt/anthropic.txt` | Claude 系列模型提示 |
| | Beast 模板 | `packages/opencode/src/session/prompt/beast.txt` | GPT 系列模型提示 |
| | Gemini 模板 | `packages/opencode/src/session/prompt/gemini.txt` | Google 模型提示 |
| | Explore Agent | `packages/opencode/src/agent/prompt/explore.txt` | Explore Agent 专用提示 |

---

### 附录二、运行命令速查

**开发环境：**

```bash
# 安装依赖并启动开发环境
bun install
bun dev
```

**测试运行：**

```bash
# 运行 OpenCode 测试
bun run --cwd packages/opencode --conditions=browser src/index.ts

# 类型检查
bun run typecheck

# 运行测试
bun test
```

**构建：**

```bash
# 构建本地版本
./packages/opencode/script/build.ts --single
```

---

### 附录三、相关资源

| 资源 | 链接 |
|------|------|
| OpenCode GitHub | https://github.com/anomalyco/opencode |
| OpenCode 文档 | https://opencode.ai/docs |
| 问题反馈 | https://github.com/anomalyco/opencode/issues |

---
