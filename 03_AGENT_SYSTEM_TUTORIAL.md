# OpenCode Agent 系统学习教程

> 基于源码分析的完整学习指南，涵盖 Agent 系统的架构设计、内置 Agent 和定制方法。

---

## 目录

| 章节 | 标题 | 难度 |
|------|------|------|
| 一 | 系统概述 | 入门 |
| 二 | Agent Info 结构 | 进阶 |
| 三 | 内置 Agent 详解 | 进阶 |
| 四 | Agent 配置系统 | 进阶 |
| 五 | Agent 与权限集成 | 高级 |
| 六 | Agent Prompt 系统 | 进阶 |
| 七 | 自定义 Agent 实战 | 实践 |
| 八 | 常见问题 | 排查 |

---

## 一、系统概述

### 1.1 什么是 Agent 系统

**Agent 系统** 是 OpenCode 的核心执行引擎，负责协调 LLM 与工具的交互、管理会话状态和执行用户指令。

**核心职责：**

| 职责 | 说明 |
|------|------|
| **会话管理** | 创建和管理 Agent 会话 |
| **指令执行** | 接收用户指令，调用工具执行 |
| **状态维护** | 维护会话上下文和历史消息 |
| **权限控制** | 结合权限系统控制操作范围 |
| **上下文压缩** | 管理长对话的上下文摘要 |

### 1.2 Agent 系统的位置

```
┌─────────────────────────────────────────────────────────────────┐
│                        OpenCode 架构                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌──────────────────┐    ┌─────────────┐    │
│  │   CLI/TUI   │───▶│   Agent System   │───▶│  Provider   │    │
│  └─────────────┘    └────────┬─────────┘    └─────────────┘    │
│                              │                                    │
│              ┌───────────────┼───────────────┐                  │
│              ▼               ▼               ▼                  │
│      ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│      │   build     │ │   plan      │ │  explore    │           │
│      │   agent     │ │   agent     │ │   agent     │           │
│      └─────────────┘ └─────────────┘ └─────────────┘           │
│                              │                                    │
│              ┌───────────────┼───────────────┐                  │
│              ▼               ▼               ▼                  │
│      ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│      │   Tool      │ │ Permission  │ │   Session   │           │
│      │  Registry   │ │   System    │ │   Manager   │           │
│      └─────────────┘ └─────────────┘ └─────────────┘           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**相关文件：**

| 文件 | 路径 | 说明 |
|------|------|------|
| **Agent 定义** | `packages/opencode/src/agent/agent.ts` | Agent 核心实现 |
| **Agent Prompt** | `packages/opencode/src/agent/prompt/*.txt` | 各 Agent 系统提示 |
| **Session 管理** | `packages/opencode/src/session/` | 会话管理模块 |
| **权限集成** | `packages/opencode/src/permission/next.ts` | 权限规则处理 |

---

## 二、Agent Info 结构

### 2.1 类型定义

Agent 的核心数据结构定义在 `agent.ts:18-42`：

```typescript
// packages/opencode/src/agent/agent.ts:18-42
export const Info = z.object({
  name: z.string(),                    // Agent 标识符
  description: z.string().optional(),  // Agent 功能描述
  mode: z.enum(["subagent", "primary", "all"]),  // 运行模式
  native: z.boolean().optional(),      // 是否内置 Agent
  hidden: z.boolean().optional(),      // 是否隐藏（不显示在列表）
  topP: z.number().optional(),         // Top-P 采样参数
  temperature: z.number().optional(),  // 温度参数
  color: z.string().optional(),        // UI 颜色标识
  permission: PermissionNext.Ruleset,  // 权限规则集
  model: z.object({
    modelID: z.string(),               // 模型 ID
    providerID: z.string(),            // 提供商 ID
  }).optional(),                       // 指定模型
  prompt: z.string().optional(),       // System Prompt
  options: z.record(z.string(), z.any()),  // 其他选项
  steps: z.number().int().positive().optional(),  // 最大步数
})
```

### 2.2 核心字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| **name** | `string` | Agent 唯一标识符，如 `build`、`plan`、`explore` |
| **description** | `string` | Agent 功能描述，用于 UI 显示和 Agent 选择 |
| **mode** | `"subagent" \| "primary" \| "all"` | 运行模式，控制 Agent 的调用方式 |
| **native** | `boolean` | 是否为内置 Agent，内置 Agent 不可被删除 |
| **hidden** | `boolean` | 是否隐藏，隐藏的 Agent 不会显示在 Agent 列表中 |
| **topP** | `number` | Top-P 采样值，范围 0-1，越高输出越随机 |
| **temperature** | `number` | 温度参数，范围 0-2，越高创意性越强 |
| **color** | `string` | UI 颜色标识，用于终端界面显示 |
| **permission** | `Ruleset` | 权限规则集，控制 Agent 可执行的操作 |
| **model** | `{modelID, providerID}` | 指定使用的模型，覆盖全局模型设置 |
| **prompt** | `string` | System Prompt 内容，覆盖默认 prompt |
| **options** | `Record<string, any>` | 其他配置选项 |
| **steps** | `number` | Agent 执行的最大步数限制 |

### 2.3 Agent 模式详解

```typescript
// agent.ts:22 - 三种运行模式

// 模式 1：Primary Agent
// 直接响应用户请求，作为主对话 Agent
mode: "primary"

// 模式 2：Subagent
// 被其他 Agent 调用，不能直接响应用户
mode: "subagent"

// 模式 3：All
// 既可作为主 Agent，也可作为子 Agent
mode: "all"
```

**模式对比：**

| 模式 | 响应用户 | 可被调用 | 典型用途 |
|------|----------|----------|----------|
| **primary** | ✅ | ❌ | 主要对话 Agent，如 build |
| **subagent** | ❌ | ✅ | 专用任务 Agent，如 explore |
| **all** | ✅ | ✅ | 通用 Agent，可灵活配置 |

---

## 三、内置 Agent 详解

### 3.1 Agent 概览

OpenCode 内置了 7 个核心 Agent：

```
┌─────────────────────────────────────────────────────────────────┐
│                      内置 Agent 矩阵                             │
├────────────┬────────┬────────┬──────────────────────────────────┤
│ Name       │ Mode   │ Hidden │ Purpose                          │
├────────────┼────────┼────────┼──────────────────────────────────┤
│ build      │ primary│ ❌     │ 默认开发 Agent                   │
│ plan       │ primary│ ❌     │ 只读计划模式                     │
│ general    │ subagent│ ❌    │ 通用研究 Agent                   │
│ explore    │ subagent│ ❌    │ 代码探索 Agent                   │
│ compaction │ primary│ ✅     │ 上下文压缩（内部使用）            │
│ title      │ primary│ ✅     │ 标题生成（内部使用）              │
│ summary    │ primary│ ✅     │ 摘要生成（内部使用）              │
└────────────┴────────┴────────┴──────────────────────────────────┘
```

### 3.2 build Agent

**默认开发 Agent**，承担主要的代码编写和修改任务。

```typescript
// agent.ts:65-71
build: {
  name: "build",
  options: {},
  permission: PermissionNext.merge(defaults, user),
  mode: "primary",
  native: true,
}
```

**特性：**

| 特性 | 值 |
|------|-----|
| **模式** | primary - 直接响应用户 |
| **权限** | 继承 defaults + user 配置 |
| **用途** | 日常开发任务（代码编写、调试、重构） |
| **温度** | 未指定（使用模型默认） |

**权限规则（defaults）：**

```typescript
// agent.ts:47-61
const defaults = PermissionNext.fromConfig({
  "*": "allow",                    // 默认允许所有操作
  doom_loop: "ask",                // 死循环检测需要询问
  external_directory: {
    "*": "ask",                    // 外部目录访问需询问
    [Truncate.DIR]: "allow",       // 截断目录除外
  },
  read: {
    "*": "allow",                  // 允许读取所有文件
    "*.env": "deny",               // 禁止读取 .env
    "*.env.*": "deny",             // 禁止读取 .env.* 文件
    "*.env.example": "allow",      // 允许读取示例文件
  },
})
```

### 3.3 plan Agent

**只读计划 Agent**，用于制定开发计划，不允许修改文件。

```typescript
// agent.ts:72-87
plan: {
  name: "plan",
  options: {},
  permission: PermissionNext.merge(
    defaults,
    PermissionNext.fromConfig({
      edit: {
        "*": "deny",                       // 禁止所有编辑操作
        ".opencode/plan/*.md": "allow",    // 允许写入计划文件
      },
    }),
    user,
  ),
  mode: "primary",
  native: true,
}
```

**设计要点：**

| 要点 | 说明 |
|------|------|
| **禁止编辑** | `edit: "*": "deny"` 阻止所有文件修改 |
| **例外放行** | 允许写入 `.opencode/plan/*.md` |
| **用途** | 需求分析、架构设计、计划制定 |

**使用场景：**

```
用户: "帮我设计一个用户认证模块"

plan Agent 行为:
├── 分析需求
├── 设计架构
├── 制定实施计划
└── 输出到 .opencode/plan/auth-module.md

不允许:
├── 修改现有代码
├── 创建实现文件
└── 运行构建命令
```

### 3.4 explore Agent

**代码探索 Agent**，专门用于快速搜索和分析代码库。

```typescript
// agent.ts:103-128
explore: {
  name: "explore",
  permission: PermissionNext.merge(
    defaults,
    PermissionNext.fromConfig({
      "*": "deny",              // 默认禁止所有操作
      grep: "allow",            // 显式允许搜索
      glob: "allow",            // 允许文件匹配
      list: "allow",            // 允许目录列表
      bash: "allow",            // 允许执行命令
      webfetch: "allow",        // 允许网页抓取
      websearch: "allow",       // 允许网络搜索
      codesearch: "allow",      // 允许代码搜索
      read: "allow",            // 允许读取文件
      external_directory: {
        [Truncate.DIR]: "allow", // 允许截断目录
      },
    }),
    user,
  ),
  description: `Fast agent specialized for exploring codebases...`,
  prompt: PROMPT_EXPLORE,
  options: {},
  mode: "subagent",
  native: true,
}
```

**权限白名单模式：**

```typescript
// 显式允许 + 隐式拒绝
{
  "*": "deny",       // 先禁止所有
  grep: "allow",     // 再显式允许需要的操作
  glob: "allow",
  read: "allow",
  // ...
}
```

**System Prompt (`prompt/explore.txt`)：**

```text
You are a file search specialist. You excel at thoroughly navigating and exploring codebases.

Your strengths:
- Rapidly finding files using glob patterns
- Searching code and text with powerful regex patterns
- Reading and analyzing file contents

Guidelines:
- Use Glob for broad file pattern matching
- Use Grep for searching file contents with regex
- Use Read when you know the specific file path
- Adapt your search approach based on the thoroughness level
- Return file paths as absolute paths
- Do not create any files or modify system state
```

### 3.5 general Agent

**通用子 Agent**，用于执行多步骤的研究和复杂任务。

```typescript
// agent.ts:88-102
general: {
  name: "general",
  description: `General-purpose agent for researching complex questions and executing multi-step tasks.`,
  permission: PermissionNext.merge(
    defaults,
    PermissionNext.fromConfig({
      todoread: "deny",    // 禁止读取 todo
      todowrite: "deny",   // 禁止写入 todo
    }),
    user,
  ),
  options: {},
  mode: "subagent",
  native: true,
}
```

**设计特点：**

| 特点 | 说明 |
|------|------|
| **禁止 TODO** | 防止 general agent 读写任务列表 |
| **通用定位** | 可处理各种研究任务 |
| **子 Agent** | 只能被其他 Agent 调用 |

### 3.6 隐藏 Agent

以下 Agent 用于内部功能，对用户不可见：

| Agent | 用途 | 权限 |
|-------|------|------|
| **compaction** | 上下文压缩 | 全部禁止 |
| **title** | 生成消息标题 | 全部禁止 |
| **summary** | 生成消息摘要 | 全部禁止 |

```typescript
// agent.ts:129-143 - compaction 示例
compaction: {
  name: "compaction",
  mode: "primary",
  native: true,
  hidden: true,                    // 隐藏
  prompt: PROMPT_COMPACTION,
  permission: PermissionNext.merge(
    defaults,
    PermissionNext.fromConfig({
      "*": "deny",                // 禁止所有操作
    }),
    user,
  ),
  options: {},
}
```

---

## 四、Agent 配置系统

### 4.1 配置来源优先级

Agent 配置来自多个层级，按优先级合并：

```
┌─────────────────────────────────────────────────────────────────┐
│                    配置优先级（从高到低）                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. opencode.json [agent] 配置                                  │
│     ├── 最高优先级                                              │
│     ├── 可覆盖任何内置配置                                       │
│     └── 可禁用内置 Agent                                        │
│                                                                 │
│  2. Agent 定义的 permission 配置                                 │
│     ├── 内置 Agent 的默认权限                                    │
│     └── 与 defaults、user 合并                                   │
│                                                                 │
│  3. Config.get() 返回的 user 配置                                │
│     ├── 用户全局配置                                             │
│     └── 中等优先级                                               │
│                                                                 │
│  4. defaults（硬编码）                                           │
│     ├── 默认权限规则                                             │
│     └── 最低优先级                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 配置合并逻辑

```typescript
// agent.ts:64-68 - build Agent 配置合并
build: {
  name: "build",
  options: {},
  permission: PermissionNext.merge(defaults, user),  // defaults + user
  mode: "primary",
  native: true,
}

// agent.ts:75-84 - plan Agent 三层合并
permission: PermissionNext.merge(
  defaults,                                    // 第1层：默认规则
  PermissionNext.fromConfig({                 // 第2层：Agent 特定规则
    edit: { "*": "deny", ".opencode/plan/*.md": "allow" },
  }),
  user,                                        // 第3层：用户配置
),
```

**merge 函数行为：**

```typescript
// 权限合并示例
defaults = { read: { "*": "allow" } }
user = { read: { "*.ts": "ask" } }

merge(defaults, user)
// 结果: { read: { "*": "allow", "*.ts": "ask" } }
// 规则按顺序匹配，后者覆盖前者
```

### 4.3 运行时配置覆盖

```typescript
// agent.ts:177-203 - 运行时配置处理
for (const [key, value] of Object.entries(cfg.agent ?? {})) {
  // 禁用 Agent
  if (value.disable) {
    delete result[key]
    continue
  }

  let item = result[key]

  // 创建新 Agent（如果不存在）
  if (!item)
    item = result[key] = {
      name: key,
      mode: "all",
      permission: PermissionNext.merge(defaults, user),
      options: {},
      native: false,
    }

  // 覆盖配置
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
  item.permission = PermissionNext.merge(
    item.permission,
    PermissionNext.fromConfig(value.permission ?? {}),
  )
}
```

### 4.4 opencode.json 配置示例

```json
{
  "agent": {
    "build": {
      "temperature": 0.7,
      "model": "claude-sonnet-4-20250514"
    },
    "plan": {
      "mode": "all",
      "description": "Planning agent with limited editing"
    },
    "custom-agent": {
      "mode": "subagent",
      "prompt": "You are a database expert...",
      "permission": {
        "read": "*",
        "bash": "allow"
      },
      "model": {
        "provider": "anthropic",
        "model": "claude-opus-4"
      }
    },
    "explore": {
      "disable": true
    }
  }
}
```

---

## 五、Agent 与权限集成

### 5.1 权限注入机制

每个 Agent 的权限在定义时就被设定：

```typescript
// agent.ts:44-61 - 默认权限定义
const state = Instance.state(async () => {
  const cfg = await Config.get()

  const defaults = PermissionNext.fromConfig({
    "*": "allow",
    doom_loop: "ask",
    external_directory: {
      "*": "ask",
      [Truncate.DIR]: "allow",
    },
    read: {
      "*": "allow",
      "*.env": "deny",
      "*.env.*": "deny",
      "*.env.example": "allow",
    },
  })

  const user = PermissionNext.fromConfig(cfg.permission ?? {})
  // ...
})
```

### 5.2 权限继承链

```
┌─────────────────────────────────────────────────────────────────┐
│                    权限继承流程                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  defaults (硬编码)                                               │
│       │                                                         │
│       ▼                                                         │
│  PermissionNext.fromConfig({                                     │
│    "*": "allow",                                                │
│    "*.env": "deny",                                             │
│    ...                                                          │
│  })                                                             │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Agent Specific Permission                                │   │
│  │                                                         │   │
│  │ plan: merge(defaults, { edit: { "*": "deny" } }, user) │   │
│  │ explore: merge(defaults, { "*": "deny", grep: "allow" }) │   │
│  │ general: merge(defaults, { todoread: "deny" })          │   │
│  └─────────────────────────────────────────────────────────┘   │
│       │                                                         │
│       ▼                                                         │
│  user (opencode.json [permission])                              │
│       │                                                         │
│       ▼                                                         │
│  Final Ruleset                                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 权限模板模式

**模式 1：宽松模式（build）**

```typescript
PermissionNext.fromConfig({
  "*": "allow",
  doom_loop: "ask",
  external_directory: { "*": "ask", [Truncate.DIR]: "allow" },
  read: { "*": "allow", "*.env": "deny" },
})
```

**模式 2：只读模式（plan）**

```typescript
PermissionNext.fromConfig({
  edit: { "*": "deny", ".opencode/plan/*.md": "allow" },
})
```

**模式 3：白名单模式（explore）**

```typescript
PermissionNext.fromConfig({
  "*": "deny",
  grep: "allow",
  glob: "allow",
  read: "allow",
})
```

**模式 4：功能限制模式（general）**

```typescript
PermissionNext.fromConfig({
  todoread: "deny",
  todowrite: "deny",
})
```

---

## 六、Agent Prompt 系统

### 6.1 Prompt 加载机制

Agent 的 System Prompt 来自多个来源：

```typescript
// 优先级（从高到低）
1. opencode.json [agent][prompt]     // 用户自定义 prompt
2. Agent 定义中的 prompt 字段       // 内置或自定义
3. PROMPT_*.txt 文件                 // 默认 prompt 文件
```

### 6.2 Prompt 文件位置

```
packages/opencode/src/agent/
├── agent.ts                    # Agent 定义
├── generate.txt                # Agent 生成 prompt
└── prompt/
    ├── compaction.txt          # 上下文压缩 prompt
    ├── explore.txt             # 代码探索 prompt
    ├── summary.txt             # 摘要生成 prompt
    └── title.txt               # 标题生成 prompt
```

### 6.3 Prompt 注入到 LLM

System Prompt 在会话创建时注入：

```typescript
// packages/opencode/src/session/system.ts
// Prompt 组装逻辑
export namespace SystemPrompt {
  export function header(providerID: string) { ... }
  export function provider(model: Provider.Model) { ... }
  export async function environment() { ... }
  export async function custom() { ... }
}

// packages/opencode/src/session/prompt.ts:591-610
// LLM 调用时注入
const result = await processor.process({
  system: [
    ...(await SystemPrompt.header(providerID)),      // Provider header
    ...(await SystemPrompt.provider(model)),         // Provider prompt
    ...(await SystemPrompt.environment()),           // 环境信息
    ...(await SystemPrompt.custom()),                // 自定义规则
    agent.prompt ? [agent.prompt] : [],             // Agent prompt
  ],
  ...
})
```

### 6.4 动态 Agent 生成

`generate.txt` 用于根据用户描述生成新 Agent 配置：

```typescript
// agent.ts:224-260
export async function generate(input: { description: string }) {
  const defaultModel = await Provider.defaultModel()
  const model = await Provider.getModel(defaultModel.providerID, defaultModel.modelID)

  // 加载系统 prompt
  const system = SystemPrompt.header(defaultModel.providerID)
  system.push(PROMPT_GENERATE)

  // 调用 LLM 生成 Agent 配置
  const result = await generateObject({
    messages: [
      ...system.map((item) => ({ role: "system", content: item })),
      {
        role: "user",
        content: `Create an agent configuration based on this request: "${input.description}"`,
      },
    ],
    model,
    schema: z.object({
      identifier: z.string(),
      whenToUse: z.string(),
      systemPrompt: z.string(),
    }),
  })

  return result.object
}
```

**生成输出示例：**

```json
{
  "identifier": "code-reviewer",
  "whenToUse": "Use this agent when you need to review code for bugs, security issues, or style violations. Example: Review the recently added authentication module.",
  "systemPrompt": "You are an expert code reviewer..."
}
```

---

## 七、自定义 Agent 实战

### 7.1 实战 1：创建代码审查 Agent

**步骤 1：定义 Agent 配置**

```json
// opencode.json
{
  "agent": {
    "code-reviewer": {
      "mode": "subagent",
      "description": "Expert code reviewer for bugs, security, and best practices",
      "prompt": "You are an expert code reviewer specializing in:\n\n1. Bug detection\n2. Security vulnerability identification\n3. Performance optimization\n4. Code style consistency\n5. Best practices compliance\n\nGuidelines:\n- Review code thoroughly but efficiently\n- Provide specific, actionable feedback\n- Reference security best practices\n- Suggest concrete improvements\n- Never modify code directly",
      "permission": {
        "read": "*",
        "grep": "allow",
        "glob": "allow"
      },
      "temperature": 0.3
    }
  }
}
```

**步骤 2：验证配置**

```bash
# 启动 OpenCode 并查看 Agent 列表
$ bun dev

# 使用 Agent
/user @code-reviewer
请审查 src/auth 中的登录逻辑
```

### 7.2 实战 2：创建数据库专家 Agent

**步骤 1：设计 Agent**

| 属性 | 值 |
|------|-----|
| **名称** | db-expert |
| **模式** | subagent |
| **用途** | SQL 优化、Schema 设计、数据库问题排查 |
| **温度** | 0.5 |

**步骤 2：配置 Agent**

```json
{
  "$schema": "https://opencode.ai/config.json",
  "agent": {
    "db-expert": {
      "mode": "subagent",
      "description": "Database expert for SQL optimization, schema design, and performance tuning",
      "prompt": "You are a database expert with 15 years of experience.\n\nSpecializations:\n- SQL query optimization\n- Index design and usage\n- Schema normalization\n- Performance tuning\n- Migration strategies\n\nApproach:\n- Analyze query execution plans\n- Suggest optimal indexes\n- Recommend schema changes\n- Explain trade-offs clearly",
      "permission": {
        "read": {
          "*.sql": "allow",
          "*.psql": "allow",
          "prisma/schema.prisma": "allow"
        },
        "grep": "allow",
        "bash": {
          "psql": "allow",
          "mysql": "allow"
        }
      }
    }
  }
}
```

### 7.3 实战 3：修改内置 Agent

**场景：增强 build Agent 的 temperature**

```json
{
  "agent": {
    "build": {
      "temperature": 0.8,
      "topP": 0.9
    }
  }
}
```

**场景：禁用 explore Agent**

```json
{
  "agent": {
    "explore": {
      "disable": true
    }
  }
}
```

### 7.4 实战 4：Agent 权限精细控制

**场景：创建只读数据分析 Agent**

```json
{
  "agent": {
    "data-analyst": {
      "mode": "subagent",
      "description": "Data analysis expert for exploring datasets",
      "prompt": "You are a data analysis expert.\n\nTasks:\n- Explore data files\n- Generate statistics\n- Identify patterns\n- Suggest visualizations\n\nConstraints:\n- Never modify source data\n- Only read .csv, .json, .parquet files\n- Output analysis results in Markdown",
      "permission": {
        "read": ["*.csv", "*.json", "*.parquet"],
        "glob": "*.{csv,json,parquet}",
        "bash": {
          "duckdb": "allow",
          "python": "allow"
        }
      }
    }
  }
}
```

---

## 八、常见问题

### Q1: 如何查看当前加载的所有 Agent？

```typescript
// packages/opencode/src/agent/agent.ts:211-218
export async function list() {
  const cfg = await Config.get()
  return pipe(
    await state(),
    values(),
    sortBy([(x) => (cfg.default_agent ? x.name === cfg.default_agent : x.name === "build"), "desc"]),
  )
}
```

### Q2: Agent 不生效怎么办？

1. **检查配置语法**

```json
// 错误：缺少引号
{
  "agent": {
    build: { }  // ❌
  }
}

// 正确
{
  "agent": {
    "build": { }  // ✅
  }
}
```

2. **验证 JSON 有效性**

```bash
# 使用 jq 验证
cat opencode.json | jq empty && echo "Valid JSON" || echo "Invalid JSON"
```

3. **检查配置路径**

```
opencode.json
└── agent          ❌ 应为 "agent"
    └── build

opencode.json
└── "agent"        ✅ 正确
    └── "build"
```

### Q3: 如何调试权限问题？

1. **查看权限规则**

```typescript
// 权限评估函数
PermissionNext.evaluate(permission, pattern, ruleset, approved)
```

2. **启用调试日志**

```bash
# 启动时设置环境变量
OPENCODE_LOG=permission:debug bun dev
```

3. **常见权限问题**

| 问题 | 原因 | 解决 |
|------|------|------|
| 读取被拒绝 | `*.env` 规则 | 检查文件路径 |
| 编辑被拒绝 | plan 模式 | 切换到 build |
| 命令被拒绝 | bash 权限未配置 | 添加 bash 规则 |

### Q4: Agent 之间如何通信？

Agent 通过 Task 工具调用其他 Agent：

```typescript
// 使用 Task 工具
await ctx.call("Task", {
  agent: "explore",
  message: "Find all API endpoints in the codebase",
  mode: "subagent",
})
```

### Q5: 如何为不同 Agent 指定不同模型？

```json
{
  "agent": {
    "build": {
      "model": {
        "provider": "anthropic",
        "model": "claude-sonnet-4-20250514"
      }
    },
    "explore": {
      "model": {
        "provider": "anthropic",
        "model": "claude-haiku-4-20250514"
      }
    },
    "code-reviewer": {
      "model": {
        "provider": "openai",
        "model": "gpt-4o"
      }
    }
  }
}
```

---

## 附录

### A. 内置 Agent 速查表

| Agent | Mode | Hidden | 主要用途 |
|-------|------|--------|----------|
| build | primary | ❌ | 默认开发 |
| plan | primary | ❌ | 只读计划 |
| general | subagent | ❌ | 通用任务 |
| explore | subagent | ❌ | 代码探索 |
| compaction | primary | ✅ | 上下文压缩 |
| title | primary | ✅ | 标题生成 |
| summary | primary | ✅ | 摘要生成 |

### B. 权限配置速查

| 模式 | 配置示例 |
|------|----------|
| 宽松 | `{ "*": "allow" }` |
| 严格 | `{ "*": "deny", "read": "allow" }` |
| 白名单 | `{ "*": "deny", "read": "allow", "grep": "allow" }` |
| 功能限制 | `{ "todoread": "deny", "todowrite": "deny" }` |

### C. 推荐学习路径

```
第 1 天：理解 Agent Info 结构
        ├── 阅读 agent.ts:18-42
        └── 理解各字段含义

第 2 天：学习内置 Agent
        ├── 分析 build Agent 配置
        ├── 对比 plan vs explore 权限
        └── 理解权限继承机制

第 3 天：实践自定义 Agent
        ├── 创建第一个自定义 Agent
        ├── 配置权限规则
        └── 测试 Agent 功能

第 4 天：高级配置
        ├── 模型覆盖配置
        ├── 温度/TopP 调优
        └── Agent 生成功能
```
