# OpenCode 架构学习规划

> 基于代码库探索的完整学习路线图

## 核心架构模块

### 1. System Prompt 系统 (重点学习)

**System Prompt 构建流程** (`packages/opencode/src/session/prompt.ts`)

```
System Prompt 组成:
├── Provider 特定 prompt (provider/model 相关)
│   ├── anthropic.txt - Claude 系列模型
│   ├── beast.txt - GPT-4/o 系列模型
│   ├── gemini.txt - Google 模型
│   ├── codex.txt - Codex/GPT-5 模型
│   └── qwen.txt - Qwen 等其他模型
│
├── 环境信息 (SystemPrompt.environment)
│   ├── 工作目录
│   ├── Git 仓库状态
│   ├── 平台信息
│   └── 当前日期
│
├── 自定义规则 (SystemPrompt.custom)
│   ├── 项目级: AGENTS.md, CLAUDE.md, CONTEXT.md
│   ├── 全局级: ~/.claude/CLAUDE.md
│   ├── 配置指令: config.instructions
│   └── URL 远程规则
│
└── Agent 特定 prompt
    ├── build: 默认开发 agent
    ├── plan: 只读计划模式
    ├── explore: 代码探索 agent
    ├── general: 通用子 agent
    └── 自定义 agent
```

**关键文件位置:**

| 文件               | 路径                                                 | 用途                |
| ------------------ | ---------------------------------------------------- | ------------------- |
| Provider Prompt    | `packages/opencode/src/session/prompt/anthropic.txt` | Claude 系统提示     |
| Agent 定义         | `packages/opencode/src/agent/agent.ts`        | Agent 配置与 prompt |
| Agent Prompt       | `packages/opencode/src/agent/prompt/*.txt`           | 专用 agent 提示词   |
| System Prompt 构建 | `packages/opencode/src/session/system.ts`            | Prompt 组装逻辑     |
| Prompt 处理        | `packages/opencode/src/session/prompt.ts`    | LLM 调用时注入      |

---

### 2. 权限审核系统 (重点学习)

**权限流程概览:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    权限请求流程                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Tool.execute()                                                 │
│       │                                                         │
│       ▼                                                         │
│  ctx.ask({ permission, patterns, always, metadata })            │
│       │                                                         │
│       ▼                                                         │
│  PermissionNext.ask()                                           │
│       │                                                         │
│       ├─→ 检查 approved 规则 ──命中──→ 返回 Promise.resolve()   │
│       │                                                         │
│       ├─→ 检查 deny 规则 ──命中──→ 抛出 DeniedError             │
│       │                                                         │
│       └─→ action === "ask"                                      │
│                   │                                             │
│                   ▼                                             │
│           发布 permission.asked 事件                             │
│                   │                                             │
│                   ▼                                             │
│           TUI 接收事件，显示权限对话框                            │
│                   │                                             │
│         ┌────────┼────────┐                                     │
│         ▼        ▼        ▼                                     │
│       Once   Always   Reject                                    │
│         │        │        │                                     │
│         └────────┴────────┘                                     │
│                   │                                             │
│                   ▼                                             │
│           Bus.publish(Event.Replied)                            │
│                   │                                             │
│                   ▼                                             │
│           Promise resolve/reject                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**核心文件:**

| 文件           | 路径                                        | 说明                       |
| -------------- | ------------------------------------------- | -------------------------- |
| **权限核心**   | `packages/opencode/src/permission/next.ts`          | PermissionNext 命名空间    |
| **旧权限**     | `packages/opencode/src/permission/index.ts`         | Permission 命名空间 (旧版) |
| **Arity 计算** | `packages/opencode/src/permission/arity.ts`         | bash 命令参数分组          |
| **权限 UI**    | `packages/opencode/src/cli/cmd/tui/routes/session/permission.tsx` | TUI 权限对话框             |
| **Bus 事件**   | `packages/opencode/src/cli/cmd/tui/context/sync.tsx` | 事件同步                   |

**权限类型:**

```typescript
// 支持的权限类型
  edit - // 文件编辑 (包含创建和修改)
  read - // 文件读取
  glob - // 文件 glob
  grep - // 代码搜索
  list - // 目录列表
  bash - // 命令执行
  task - // 子 agent 调用
  todoread - // 待办事项读取
  todowrite - // 待办事项写入
  question - // 向用户提问
  webfetch - // 网页抓取
  websearch - // 网络搜索
  codesearch - // 代码搜索
  external_directory - // 外部目录
  doom_loop - // 死循环检测
```

**权限动作:**

```typescript
Action = "allow" | "deny" | "ask"

// Reply = "once" | "always" | "reject"
```

**Agent 权限配置示例** (`packages/opencode/src/agent/agent.ts`):

```typescript
const defaults = PermissionNext.fromConfig({
  "*": "allow",
  doom_loop: "ask",
  external_directory: "ask",
  read: {
    "*": "allow",
    "*.env": "deny", // 禁止读取 .env
    "*.env.*": "deny",
    "*.env.example": "allow",
  },
})
```

**工具调用权限请求示例** (`packages/opencode/src/tool/bash.ts`):

```typescript
// 请求外部目录访问权限
await ctx.ask({
  permission: "external_directory",
  patterns: Array.from(directories),
  always: Array.from(directories).map((x) => path.dirname(x) + "*"),
  metadata: {},
})

// 请求 bash 执行权限
await ctx.ask({
  permission: "bash",
  patterns: Array.from(patterns),
  always: Array.from(always),
  metadata: {},
})
```

---

### 3. Agent 系统 (`packages/opencode/src/agent/`)

```typescript
// Agent 定义示例 (packages/opencode/src/agent/agent.ts)
build: {
  name: "build"
  mode: "primary"      // primary/subagent/all
  native: true
  permission: {...}
}

plan: {
  name: "plan"
  mode: "primary"
  prompt: PROMPT_PLAN  // 读-only 模式
  permission: {
    edit: "*": "deny"  // 禁止编辑
  }
}
```

---

### 4. Tool 系统 (`packages/opencode/src/tool/`)

**工具定义模式:**

```typescript
export const ReadTool = Tool.define("read", async (ctx) => {
  return {
    description: "Read a file...",
    parameters: z.object({
      filePath: z.string(),
    }),
    async execute(params, ctx) {
      // 工具实现
    },
  }
})
```

---

### 5. 迭代信息收集模式 (`packages/opencode/src/session/processor.ts`)

> **核心灵魂** - OpenCode 的智能决策循环，让 Agent 能够根据用户问题探索代码库、迭代收集信息、直到获取足够信息后再采取行动。

**迭代决策流程：**

```
┌─────────────────────────────────────────────────────────────────┐
│                    迭代信息收集流程                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  用户提问                                                        │
│     │                                                           │
│     ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  步骤 1: LLM 分析问题                                      │   │
│  │  - 理解用户意图                                            │   │
│  │  - 决定：需要更多信息？还是直接行动？                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│           │                    │                                 │
│           ▼                    ▼                                 │
│     ┌──────────┐          ┌──────────────────┐                 │
│     │ 需要探索  │          │ 信息足够          │                 │
│     └──────────┘          └──────────────────┘                 │
│           │                    │                                 │
│           ▼                    ▼                                 │
│     ┌──────────┐          ┌──────────────────┐                 │
│     │ 调用工具  │          │ 执行最终行动      │                 │
│     │ 收集信息  │          │ (write/edit等)   │                 │
│     └──────────┘          └──────────────────┘                 │
│           │                                                       │
│           ▼                                                       │
│     ┌─────────────────────────────────────────────────────────┐ │
│  │  步骤 2: 处理工具结果                                        │ │
│  │  - LLM 评估结果是否足够                                      │ │
│  │  - 决定：继续探索？还是结束？                                │ │
│  └─────────────────────────────────────────────────────────┘   │
│           │                    │                                 │
│           └────────────────────┘                                 │
│                  (循环)                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**核心实现：**

```typescript
// processor.ts - 处理器主循环
async process(streamInput: LLM.StreamInput) {
  while (true) {  // 外层循环：迭代决策
    const stream = await LLM.stream(streamInput)

    for await (const value of stream.fullStream) {  // 内层循环：工具调用
      switch (value.type) {
        case "tool-call":
          // 执行工具，收集信息
          break
        case "tool-result":
          // 处理结果，LLM 决定是否继续
          break
      }
    }
    // 根据上下文决定是否继续循环
  }
}

// 死循环检测
const DOOM_LOOP_THRESHOLD = 3
if (lastThree.length === DOOM_LOOP_THRESHOLD &&
    all_same_tool_calls &&
    no_errors) {
  throw new DoomLoopError()  // 防止无限循环
}
```

**Tool Chaining 模式：**

```typescript
// 典型探索链示例
1. glob("**/*.ts")      // 发现文件
   ↓
2. grep("API|export")   // 搜索关键内容
   ↓
3. read("file.ts")      // 读取具体文件
   ↓
4. bash("npm test")     // 验证发现
```

**Explore Agent 专用探索：**

```typescript
// packages/opencode/src/agent/agent.ts - 专为快速探索优化的 Agent
explore: {
  name: "explore",
  mode: "subagent",
  permission: PermissionNext.merge(defaults, {
    "*": "deny",            // 禁止所有操作
    grep: "allow",          // 显式允许搜索
    glob: "allow",          // 允许文件匹配
    read: "allow",          // 允许读取
    list: "allow",          // 允许目录列表
    bash: "allow",          // 允许命令
    codesearch: "allow",    // 允许代码搜索
    external_directory: {   // 允许外部目录访问
      [Truncate.DIR]: "allow",
    },
    webfetch: "allow",
    websearch: "allow",
  }),
  description: "Fast agent specialized for exploring codebases...",
  prompt: PROMPT_EXPLORE,
}
```

**关键学习点：**

| 文件 | 关键函数/行号 | 说明 |
|------|--------------|------|
| `packages/opencode/src/session/processor.ts` | `process()` 主循环 | 迭代决策核心 |
| `packages/opencode/src/session/processor.ts` | 死循环检测 | `DOOM_LOOP_THRESHOLD` |
| `packages/opencode/src/tool/grep.ts` | grep 工具 | 代码搜索 |
| `packages/opencode/src/tool/glob.ts` | glob 工具 | 文件发现 |
| `packages/opencode/src/tool/read.ts` | read 工具 | 内容读取 |
| `packages/opencode/src/agent/agent.ts` | explore agent | 专用探索 |

---

### 6. Skill 系统 (`packages/opencode/src/skill/`)

**Skill 定义格式** (`.opencode/skill/test-skill/SKILL.md`):

```yaml
---
name: test-skill
description: use this when asked to test skill
---
# Skill 文档
这里是技能的详细说明和使用指南
```

---

### 7. Session 管理 (`packages/opencode/src/session/`)

- 会话创建与状态管理
- 消息流处理 (MessageV2)
- 上下文压缩与摘要
- 权限继承

---

### 8. Provider 系统 (`packages/opencode/src/provider/`)

- 多模型提供商适配
- 工具 schema 转换
- API 集成

---

## 学习路线建议

### 第一阶段：System Prompt + 权限基础 (1周)

1. **System Prompt 基础**

   ```bash
   cat packages/opencode/src/session/prompt/anthropic.txt
   cat packages/opencode/src/session/prompt/beast.txt
   ```

   - 阅读 `packages/opencode/src/session/system.ts` 了解 header/environment/custom 如何组合
   - 阅读 `packages/opencode/src/session/prompt.ts` 了解如何注入到 LLM 调用

2. **权限系统基础**
   - 阅读 `packages/opencode/src/permission/next.ts` 理解核心逻辑
   - 理解 `ask()` → `evaluate()` → 决策流程
   - 学习 TUI 权限对话框 (`packages/opencode/src/cli/cmd/tui/routes/session/permission.tsx`)

3. **实践**
   - 修改 `AGENTS.md` 添加项目规范
   - 配置 agent 权限规则测试

### 第二阶段：Agent + Tool 定制 (1周)

1. **学习 Agent 定义**
   - 阅读 `packages/opencode/src/agent/agent.ts` 理解 build/plan/explore agent
   - 理解 permission 系统与 agent 的关联

2. **创建自定义 Agent**
   - 在 `opencode.json` 中配置新 agent
   - 编写专用的 system prompt
   - 设置权限规则

3. **深入 Tool Registry**
   - 理解工具如何注册和暴露给 LLM
   - 工具权限请求的实现

### 第三阶段：Tool 迭代与探索 (3-5天)

1. **迭代信息收集模式**
   - 阅读 `packages/opencode/src/session/processor.ts` 理解主循环
   - 理解 `while(true)` + `for await` 双层循环
   - 学习死循环检测机制 (`DOOM_LOOP_THRESHOLD`)
   - 理解 LLM 如何决定探索还是行动

2. **Tool Chaining**
   - 学习 glob → grep → read → bash 探索链
   - 理解工具结果如何反馈给 LLM
   - 掌握迭代决策流程

3. **Explore Agent**
   - 阅读 `packages/opencode/src/agent/agent.ts` 理解专用探索 Agent
   - 理解白名单权限模式 (`*": "deny"`)
   - 学习 `task` 工具调用 sub-agent

### 第四阶段：Skill 系统 (3-5天)

1. **Skill 发现机制**
   - 阅读 `packages/opencode/src/skill/skill.ts` 扫描逻辑
   - 理解 `.claude/skills/` 和 `.opencode/skill/` 优先级

2. **编写自定义 Skill**
   - 创建 `SKILL.md` 文件
   - 定义 name/description/frontmatter

3. **Skill 工具集成**
   - 阅读 `packages/opencode/src/tool/skill.ts` 理解加载执行流程

### 第五阶段：高级特性 (2周)

1. **权限高级**
   - BashArity 命令参数分组
   - 权限规则持久化
   - 批量权限处理

2. **上下文管理**
   - Session compaction 机制
   - 消息压缩策略
   - 会话生命周期管理 (Session.ts)

3. **Provider 系统**
   - 多模型提供商适配
   - SDK 动态加载
   - 成本计算

4. **MCP 集成**
   - Model Context Protocol
   - 服务发现机制

---

## 实践项目建议

| 项目                               | 难度   | 学习目标                    |
| ---------------------------------- | ------ | --------------------------- |
| 1. 修改 System Prompt 添加项目规范 | ⭐     | 理解 Prompt 构建 + 权限基础 |
| 2. 创建代码审查 Agent              | ⭐⭐   | Agent 定义 + 权限配置       |
| 3. 编写 API 文档生成 Skill         | ⭐⭐   | Skill 系统                  |
| 4. 自定义 Tool 实现 + 权限请求     | ⭐⭐⭐ | Tool 注册 + 权限流程        |
| 5. 配置项目权限规则                | ⭐⭐   | 权限规则设计                |

---

## 关键文件速查

### System Prompt 核心

```
├── packages/opencode/src/session/system.ts              # Prompt 组装
├── packages/opencode/src/session/prompt.ts              # LLM 调用注入点
├── packages/opencode/src/agent/agent.ts                 # Agent 定义
└── packages/opencode/src/session/prompt/*.txt           # Provider/Agent prompts
```

### 权限系统核心

```
├── packages/opencode/src/permission/next.ts             # PermissionNext 核心
├── packages/opencode/src/permission/index.ts            # 旧版 Permission
├── packages/opencode/src/permission/arity.ts            # BashArity 命令分组
├── packages/opencode/src/cli/cmd/tui/routes/session/permission.tsx  # TUI 权限UI
└── packages/opencode/src/cli/cmd/tui/context/sync.tsx   # 权限事件同步
```

### Skill 系统

```
├── packages/opencode/src/skill/skill.ts                 # Skill 发现
├── packages/opencode/src/tool/skill.ts                  # Skill 工具
└── .opencode/skill/*/SKILL.md         # Skill 定义
```

### Tool 系统

```
├── packages/opencode/src/tool/tool.ts                   # Tool 接口定义
├── packages/opencode/src/tool/registry.ts               # 工具注册表
└── packages/opencode/src/tool/*.ts                      # 具体工具实现
```

---

## 运行命令速查

```bash
# 启动开发环境
bun install
bun dev

# 运行 OpenCode 测试
bun run --cwd packages/opencode --conditions=browser src/index.ts

# 类型检查
bun run typecheck

# 运行测试
bun test

# 构建本地版本
./packages/opencode/script/build.ts --single
```
