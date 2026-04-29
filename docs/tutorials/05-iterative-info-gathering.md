# OpenCode 迭代信息收集模式学习教程

> 基于源码分析的完整学习指南，涵盖 OpenCode 核心灵魂——智能决策循环机制。

---

## 目录

| 章节 | 标题 | 难度 |
|------|------|------|
| 一 | 核心概念 | 入门 |
| 二 | 处理器主循环 | 进阶 |
| 三 | 工具调用链机制 | 进阶 |
| 四 | 死循环检测机制 | 高级 |
| 五 | Explore Agent 详解 | 进阶 |
| 六 | SubAgent 调用机制 | 高级 |
| 七 | 完整执行流程 | 实践 |
| 八 | 常见问题 | 排查 |

---

## 一、核心概念

### 1.1 什么是迭代信息收集

**迭代信息收集模式** 是 OpenCode 的核心决策机制，让 Agent 能够：

1. **理解用户意图** - 分析用户问题的本质
2. **评估信息需求** - 判断当前信息是否足够
3. **探索收集信息** - 调用工具搜索代码库
4. **迭代决策** - 根据收集结果决定是否继续
5. **采取行动** - 信息足够后执行最终任务

**与传统编程的区别：**

```
传统程序: 输入 → 处理 → 输出 (单次执行)

OpenCode:  输入 → 分析 → 探索? → 收集 → 分析 → 探索? → ... → 行动
                                    ↑___________|
                                    (循环迭代)
```

### 1.2 为什么需要迭代

**场景示例：**

```
用户: "修复认证模块的漏洞"

直接行动的问题:
├── 不了解认证模块的结构
├── 不知道有哪些文件需要修改
├── 不清楚现有的安全措施
└── 可能引入新的问题

迭代收集的优势:
├── 找到认证相关文件
├── 分析现有代码逻辑
├── 识别潜在漏洞点
├── 理解代码依赖关系
└── 制定修复方案后行动
```

### 1.3 核心架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    迭代信息收集架构                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   Session Processor                      │   │
│  │                                                           │   │
│  │    while (true) {  ← 外层循环: 迭代决策                   │   │
│  │      LLM.stream()  ← 调用 LLM                            │   │
│  │        ↓                                                  │   │
│  │      for await (value) {  ← 内层循环: 处理流              │   │
│  │        switch (value.type) {                              │   │
│  │          case "tool-call":    执行工具                    │   │
│  │          case "tool-result":  处理结果                    │   │
│  │          case "text-delta":   构建响应                    │   │
│  │        }                                                  │   │
│  │      }                                                    │   │
│  │    }                                                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              ↓                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    工具层                                 │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐    │   │
│  │  │  glob   │→ │  grep   │→ │  read   │→ │  bash   │    │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘    │   │
│  │      ↓            ↓            ↓            ↓           │   │
│  │    发现文件     搜索内容      读取代码     验证结果      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              ↓                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Agent 层                              │   │
│  │  ┌─────────────────────────────────────────────────┐    │   │
│  │  │              Explore Agent                       │    │   │
│  │  │  - 专用探索优化的 Agent                          │    │   │
│  │  │  - 白名单权限模式                                │    │   │
│  │  │  - 快速文件搜索专家                              │    │   │
│  │  └─────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、处理器主循环

### 2.1 循环结构解析

**位置：** `packages/opencode/src/session/processor.ts:44-397`

处理器是整个迭代模式的核心，包含双层循环结构：

```typescript
// processor.ts:44-397
export function create(input: {...}) {
  const result = {
    async process(streamInput: LLM.StreamInput) {
      // === 外层循环：迭代决策 ===
      while (true) {
        try {
          let currentText: MessageV2.TextPart | undefined
          let reasoningMap: Record<string, MessageV2.ReasoningPart> = {}

          // 调用 LLM 获取响应流
          const stream = await LLM.stream(streamInput)

          // === 内层循环：处理流事件 ===
          for await (const value of stream.fullStream) {
            input.abort.throwIfAborted()

            switch (value.type) {
              case "start":
                // 开始处理
                break

              case "reasoning-start/delta/end":
                // 处理思考过程
                break

              case "tool-call":
                // 执行工具调用
                break

              case "tool-result":
                // 处理工具结果
                break

              case "text-delta/end":
                // 构建文本响应
                break

              case "finish-step":
                // 步骤完成
                break
            }

            // 检查是否需要压缩上下文
            if (needsCompaction) break
          }
        } catch (e) {
          // 错误处理和重试
        }

        // === 循环退出条件 ===
        if (needsCompaction) return "compact"   // 需要压缩
        if (blocked) return "stop"              // 被阻止
        if (input.assistantMessage.error) return "stop"  // 有错误
        return "continue"                       // 继续迭代
      }
    }
  }
  return result
}
```

### 2.2 外层循环功能

| 功能 | 说明 | 代码位置 |
|------|------|----------|
| **迭代控制** | 控制是否继续收集信息 | `:48` |
| **错误处理** | 捕获和处理执行错误 | `:335-359` |
| **重试机制** | 自动重试可恢复错误 | `:341-352` |
| **上下文检查** | 检查是否需要压缩 | `:270-272` |
| **退出决策** | 决定继续或停止 | `:393-396` |

### 2.3 内层循环事件处理

**事件类型表：**

| 事件类型 | 说明 | 处理逻辑 |
|----------|------|----------|
| `start` | 会话开始 | 设置状态为 busy |
| `reasoning-start` | 思考开始 | 创建推理 Part |
| `reasoning-delta` | 思考内容 | 增量更新推理 |
| `reasoning-end` | 思考结束 | 完成推理 Part |
| `tool-input-start` | 工具输入开始 | 创建工具 Part |
| `tool-call` | 工具调用 | 执行工具 |
| `tool-result` | 工具结果 | 更新结果状态 |
| `tool-error` | 工具错误 | 处理错误 |
| `text-start` | 文本开始 | 创建文本 Part |
| `text-delta` | 文本内容 | 增量更新文本 |
| `text-end` | 文本结束 | 完成文本 Part |
| `finish-step` | 步骤完成 | 统计成本 Token |
| `finish` | 完成 | 结束处理 |

### 2.4 循环退出条件

```typescript
// processor.ts:393-396
if (needsCompaction) return "compact"   // 上下文超限，需要压缩
if (blocked) return "stop"              // 权限被拒绝
if (input.assistantMessage.error) return "stop"  // 发生错误
return "continue"                       // 继续下一次迭代
```

---

## 三、工具调用链机制

### 3.1 工具执行流程

**工具调用状态机：**

```
┌─────────────────────────────────────────────────────────────────┐
│                    工具调用状态机                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  pending (等待)                                                  │
│      │                                                          │
│      ▼                                                          │
│  running (执行中)                                                │
│      │                                                          │
│      ├─────────────────────────────────────────────────────┐    │
│      │                                                     │    │
│      ▼                                                     ▼    │
│  completed (完成)                                     error (错误)
│      │                                                     │
│      └─────────────────────────────────────────────────────┘    │
│                            │                                    │
│                            ▼                                    │
│                    结果反馈给 LLM                               │
│                            │                                    │
│                            ▼                                    │
│              LLM 决定：继续调用工具？还是结束？                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 工具调用代码解析

**步骤 1: 创建工具 Part** (`processor.ts:102-117`)

```typescript
case "tool-input-start": {
  // 创建工具调用 Part
  const part = await Session.updatePart({
    id: toolcalls[value.id]?.id ?? Identifier.ascending("part"),
    messageID: input.assistantMessage.id,
    sessionID: input.assistantMessage.sessionID,
    type: "tool",
    tool: value.toolName,
    callID: value.id,
    state: {
      status: "pending",  // 初始状态：待执行
      input: {},
      raw: "",
    },
  })
  toolcalls[value.id] = part as MessageV2.ToolPart
  break
}
```

**步骤 2: 开始执行** (`processor.ts:125-170`)

```typescript
case "tool-call": {
  const match = toolcalls[value.toolCallId]
  if (match) {
    // 更新状态为运行中
    const part = await Session.updatePart({
      ...match,
      tool: value.toolName,
      state: {
        status: "running",  // 标记为运行中
        input: value.input,  // 记录输入参数
        time: {
          start: Date.now(),  // 记录开始时间
        },
      },
      metadata: value.providerMetadata,
    })
    toolcalls[value.toolCallId] = part as MessageV2.ToolPart

    // === 死循环检测 ===
    const parts = await MessageV2.parts(input.assistantMessage.id)
    const lastThree = parts.slice(-DOOM_LOOP_THRESHOLD)
    // ... 检测逻辑（见第四章）
  }
  break
}
```

**步骤 3: 完成执行** (`processor.ts:171-193`)

```typescript
case "tool-result": {
  const match = toolcalls[value.toolCallId]
  if (match && match.state.status === "running") {
    await Session.updatePart({
      ...match,
      state: {
        status: "completed",  // 标记为完成
        input: value.input,
        output: value.output.output,  // 工具输出
        metadata: value.output.metadata,
        title: value.output.title,
        time: {
          start: match.state.time.start,
          end: Date.now(),  // 记录结束时间
        },
        attachments: value.output.attachments,
      },
    })
    delete toolcalls[value.toolCallId]  // 从追踪中移除
  }
  break
}
```

### 3.3 典型工具链示例

**探索代码库的典型流程：**

```typescript
// 1. 使用 glob 查找文件
await ctx.call("glob", {
  pattern: "**/*.ts",
})

// 2. 使用 grep 搜索内容
await ctx.call("grep", {
  pattern: "export.*class.*Controller",
  files: ["src/**/*.ts"],
})

// 3. 使用 read 读取文件
await ctx.call("read", {
  filePath: "/path/to/file.ts",
})

// 4. 使用 bash 验证
await ctx.call("bash", {
  command: "npm test",
})
```

**工具链说明：**

| 步骤 | 工具 | 目的 | 输出 |
|------|------|------|------|
| 1 | glob | 发现文件 | 文件路径列表 |
| 2 | grep | 筛选内容 | 匹配行和位置 |
| 3 | read | 分析代码 | 文件内容 |
| 4 | bash | 验证结果 | 命令输出 |

### 3.4 工具结果反馈

**结果如何影响 LLM 决策：**

```typescript
// 工具执行完成后，结果被格式化并返回给 LLM
// LLM 根据结果决定下一步操作

// 示例：LLM 的决策逻辑
if (tool_result.is_sufficient) {
  // 信息足够，可以采取行动
  return "action"
} else {
  // 信息不足，继续探索
  return "explore_more"
}
```

---

## 四、死循环检测机制

### 4.1 为什么需要死循环检测

**问题场景：**

```
用户: "查找所有文件"
LLM: glob("**/*")
结果: 10000 个文件

LLM: glob("**/*")  // 重复调用
结果: 同样的 10000 个文件

LLM: glob("**/*")  // 无限循环
结果: ...
```

**检测阈值：** `DOOM_LOOP_THRESHOLD = 3`

### 4.2 检测代码解析

**位置：** `processor.ts:19, 142-167`

```typescript
// processor.ts:19
const DOOM_LOOP_THRESHOLD = 3

// processor.ts:142-167
case "tool-call": {
  const match = toolcalls[value.toolCallId]
  if (match) {
    const part = await Session.updatePart({...})
    toolcalls[value.toolCallId] = part

    // === 死循环检测 ===
    const parts = await MessageV2.parts(input.assistantMessage.id)
    const lastThree = parts.slice(-DOOM_LOOP_THRESHOLD)

    if (
      lastThree.length === DOOM_LOOP_THRESHOLD &&
      lastThree.every(
        (p) =>
          p.type === "tool" &&
          p.tool === value.toolName &&
          p.state.status !== "pending" &&
          JSON.stringify(p.state.input) === JSON.stringify(value.input),
      )
    ) {
      // === 检测到死循环，请求权限 ===
      const agent = await Agent.get(input.assistantMessage.agent)
      await PermissionNext.ask({
        permission: "doom_loop",  // 特殊权限类型
        patterns: [value.toolName],
        sessionID: input.assistantMessage.sessionID,
        metadata: {
          tool: value.toolName,
          input: value.input,
        },
        always: [value.toolName],
        ruleset: agent.permission,
      })
    }
  }
  break
}
```

### 4.3 检测条件详解

```typescript
// 条件 1: 至少有 3 个最近的工具调用
lastThree.length === DOOM_LOOP_THRESHOLD

// 条件 2: 所有调用都是同一个工具
lastThree.every((p) => p.tool === value.toolName)

// 条件 3: 所有调用都已完成（非 pending）
p.state.status !== "pending"

// 条件 4: 所有调用的输入完全相同
JSON.stringify(p.state.input) === JSON.stringify(value.input)
```

### 4.4 处理流程

```
检测到死循环
    │
    ▼
请求 doom_loop 权限
    │
    ├── 允许 (allow) → 继续执行
    │
    ├── 拒绝 (deny) → 抛出 DeniedError
    │
    └── 询问 (ask) → 用户决定
```

---

## 五、Explore Agent 详解

### 5.1 Explore Agent 定位

**Explore Agent** 是专为快速代码探索优化的子 Agent：

```typescript
// processor.ts:103-128
explore: {
  name: "explore",
  mode: "subagent",  // 只作为子 Agent 调用
  native: true,
  description: `Fast agent specialized for exploring codebases.
    Use this when you need to quickly find files by patterns,
    search code for keywords, or answer questions about the codebase.`,
  prompt: PROMPT_EXPLORE,
}
```

### 5.2 权限配置（白名单模式）

```typescript
// processor.ts:105-122
permission: PermissionNext.merge(
  defaults,  // 默认规则
  PermissionNext.fromConfig({
    "*": "deny",  // 默认禁止所有操作（白名单模式）

    // 显式允许的操作
    grep: "allow",           // 允许代码搜索
    glob: "allow",           // 允许文件匹配
    list: "allow",           // 允许目录列表
    bash: "allow",           // 允许命令执行
    webfetch: "allow",       // 允许网页抓取
    websearch: "allow",      // 允许网络搜索
    codesearch: "allow",     // 允许代码搜索
    read: "allow",           // 允许读取文件

    // 特殊目录访问
    external_directory: {
      [Truncate.DIR]: "allow",  // 允许截断目录
    },
  }),
  user,  // 用户自定义规则
)
```

### 5.3 System Prompt

**位置：** `packages/opencode/src/agent/prompt/explore.txt`

```text
You are a file search specialist. You excel at thoroughly navigating and exploring codebases.

Your strengths:
- Rapidly finding files using glob patterns
- Searching code and text with powerful regex patterns
- Reading and analyzing file contents

Guidelines:
- Use Glob for broad file pattern matching
- Use Grep for searching file contents with regex
- Use Read when you know the specific file path you need to read
- Use Bash for file operations like copying, moving, or listing directory contents
- Adapt your search approach based on the thoroughness level specified by the caller
- Return file paths as absolute paths in your final response
- For clear communication, avoid using emojis
- Do not create any files, or run bash commands that modify the user's system state in any way

Complete the user's search request efficiently and report your findings clearly.
```

### 5.4 使用场景

| 场景 | 指令示例 | 说明 |
|------|----------|------|
| 文件查找 | "Find all React components" | 使用 glob 查找文件 |
| 代码搜索 | "Find API endpoints" | 使用 grep 搜索 |
| 代码理解 | "How does authentication work?" | 综合探索 |
| 依赖分析 | "Find all imports of utils.ts" | 追踪引用 |

### 5.5 调用示例

```
用户: "查找项目中的所有 API 端点"

build Agent 决策:
1. 分析问题 → 需要探索代码库
2. 调用 @explore subagent
3. explore 使用工具链:
   - glob("**/*.{ts,js}")
   - grep("router|get|post|put|delete")
   - read(匹配的路由文件)
4. 汇总结果返回给 build Agent
5. build Agent 基于结果采取行动
```

---

## 六、SubAgent 调用机制

### 6.1 Task Tool 概述

**Task Tool** 是调用 subagent 的核心工具：

**位置：** `packages/opencode/src/tool/task.ts`

```typescript
// task.ts:15-21
const parameters = z.object({
  description: z.string(),  // 任务描述（3-5词）
  prompt: z.string(),       // 任务提示
  subagent_type: z.string(), // 使用的 Agent 类型
  session_id: z.string().optional(),  // 继续现有会话
  command: z.string().optional(),     // 触发命令
})
```

### 6.2 SubAgent 调用流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    SubAgent 调用流程                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 权限检查                                                     │
│     ctx.ask({ permission: "task", patterns: [subagent_type] })  │
│           │                                                     │
│           ▼                                                     │
│  2. 创建子会话                                                   │
│     Session.create({                                            │
│       parentID: ctx.sessionID,                                   │
│       title: description + "(@agent_name subagent)",             │
│       permission: [...]  // 限制权限                             │
│     })                                                          │
│           │                                                     │
│           ▼                                                     │
│  3. 执行子会话                                                   │
│     SessionPrompt.prompt({                                       │
│       messageID,                                                 │
│       sessionID: subagent_session.id,                            │
│       model,                                                     │
│       agent: agent.name,                                         │
│     })                                                          │
│           │                                                     │
│           ▼                                                     │
│  4. 汇总结果                                                     │
│     - 收集所有工具调用                                           │
│     - 提取文本响应                                               │
│     - 返回给调用方                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.3 核心代码解析

**步骤 1: 权限检查** (`task.ts:44-55`)

```typescript
// Skip permission check when user explicitly invoked via @ or command subtask
if (!ctx.extra?.bypassAgentCheck) {
  await ctx.ask({
    permission: "task",
    patterns: [params.subagent_type],
    always: ["*"],
    metadata: {
      description: params.description,
      subagent_type: params.subagent_type,
    },
  })
}
```

**步骤 2: 创建子会话** (`task.ts:59-91`)

```typescript
const session = await iife(async () => {
  if (params.session_id) {
    const found = await Session.get(params.session_id).catch(() => {})
    if (found) return found  // 继续现有会话
  }

  return await Session.create({
    parentID: ctx.sessionID,  // 关联父会话
    title: params.description + ` (@${agent.name} subagent)`,
    permission: [
      // 禁止修改 todo 列表
      { permission: "todowrite", pattern: "*", action: "deny" },
      { permission: "todoread", pattern: "*", action: "deny" },
      // 禁止递归调用 subagent
      { permission: "task", pattern: "*", action: "deny" },
      // 允许主要工具
      ...(config.experimental?.primary_tools?.map((t) => ({
        pattern: "*",
        action: "allow" as const,
        permission: t,
      })) ?? []),
    ],
  })
})
```

**步骤 3: 执行子会话** (`task.ts:138-153`)

```typescript
const result = await SessionPrompt.prompt({
  messageID,
  sessionID: session.id,
  model: {
    modelID: model.modelID,
    providerID: model.providerID,
  },
  agent: agent.name,  // 使用指定的 Agent
  tools: {
    todowrite: false,  // 禁用 todo 工具
    todoread: false,
    task: false,       // 禁用任务工具（防止递归）
    ...Object.fromEntries(
      (config.experimental?.primary_tools ?? []).map((t) => [t, false])
    ),
  },
  parts: promptParts,
})
```

**步骤 4: 汇总结果** (`task.ts:155-169`)

```typescript
const messages = await Session.messages({ sessionID: session.id })
const summary = messages
  .filter((x) => x.info.role === "assistant")
  .flatMap((msg) => msg.parts.filter((x: any) => x.type === "tool"))
  .map((part) => ({
    id: part.id,
    tool: part.tool,
    state: {
      status: part.state.status,
      title: part.state.status === "completed" ? part.state.title : undefined,
    },
  }))

const text = result.parts.findLast((x) => x.type === "text")?.text ?? ""
const output = text + "\n\n" + [
  "<task_metadata>",
  `session_id: ${session.id}`,
  "</task_metadata>",
].join("\n")

return {
  title: params.description,
  metadata: { summary, sessionId: session.id },
  output,
}
```

### 6.4 权限限制

**子会话的权限限制：**

```typescript
permission: [
  { permission: "todowrite", pattern: "*", action: "deny" },  // 禁止写 TODO
  { permission: "todoread", pattern: "*", action: "deny" },   // 禁止读 TODO
  { permission: "task", pattern: "*", action: "deny" },       // 禁止调用子任务
  { permission: "read", pattern: "*", action: "allow" },      // 允许读取
  { permission: "grep", pattern: "*", action: "allow" },      // 允许搜索
  // ...
]
```

---

## 七、完整执行流程

### 7.1 端到端流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                    完整执行流程                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  用户输入: "修复认证模块的漏洞"                                   │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Session Processor (step 1)                             │   │
│  │  - LLM 分析用户问题                                      │   │
│  │  - 决定: 需要更多信息                                    │   │
│  │  - 调用 @explore subagent                               │   │
│  └─────────────────────────────────────────────────────────┘   │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Explore Agent (step 1.1)                               │   │
│  │  - glob("**/auth*.ts")  → 发现 5 个文件                  │   │
│  │  - grep("auth|login|verify")  → 找到关键函数             │   │
│  │  - read("auth/controller.ts")  → 分析代码               │   │
│  │  - 返回分析结果                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Session Processor (step 2)                             │   │
│  │  - 接收 explore 结果                                     │   │
│  │  - LLM 评估: 信息足够？                                  │   │
│  │  - 决定: 可以开始修复                                    │   │
│  └─────────────────────────────────────────────────────────┘   │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  工具执行 (step 3)                                       │   │
│  │  - edit("auth/controller.ts")  → 修复漏洞               │   │
│  │  - read("auth/service.ts")  → 验证依赖                  │   │
│  │  - bash("npm test")  → 运行测试                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  结果返回                                                │   │
│  │  - 输出修复总结                                          │   │
│  │  - 更新文件差异                                          │   │
│  │  - 完成会话                                              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 详细时序图

```
用户          LLM              SessionProcessor      Explore Agent      工具
 │              │                   │                     │              │
 │ "修复漏洞"   │                   │                     │              │
 │─────────────→│                   │                     │              │
 │              │ 分析问题           │                     │              │
 │              │→──需要探索───────→│                     │              │
 │              │                   │ 调用 @explore       │              │
 │              │                   │────────────────────→│              │
 │              │                   │                     │ glob("auth*") │
 │              │                   │                     │──────────────→│
 │              │                   │                     │ 发现 5 文件   │
 │              │                   │                     │←──────────────│
 │              │                   │                     │ grep("auth")  │
 │              │                   │                     │──────────────→│
 │              │                   │                     │ 找到 10 处    │
 │              │                   │                     │←──────────────│
 │              │                   │                     │ read(文件)    │
 │              │                   │                     │──────────────→│
 │              │                   │                     │ 返回代码      │
 │              │                   │                     │←──────────────│
 │              │                   │ 返回探索结果        │              │
 │              │←──────────────────│←────────────────────│              │
 │              │                   │                     │              │
 │              │ 评估: 信息足够    │                     │              │
 │              │→──可以修复───────→│                     │              │
 │              │                   │ edit("auth.ts")     │              │
 │              │                   │──────────────────────────────────→│
 │              │                   │                     │              │ 修复代码
 │              │                   │                     │              │←─────────│
 │              │                   │                     │              │
 │              │ 总结修复结果      │                     │              │
 │←─────────────│                   │                     │              │
 │              │                   │                     │              │
```

### 7.3 关键决策点

**决策点 1: 是否需要探索？**

```typescript
// LLM 根据问题复杂度决定
if (问题需要了解代码结构) {
  调用探索工具或 explore agent
} else {
  直接执行操作
}
```

**决策点 2: 探索哪些内容？**

```typescript
// 基于问题选择工具
if (需要找文件) {
  使用 glob
} else if (需要搜索内容) {
  使用 grep
} else if (需要读文件) {
  使用 read
} else if (需要验证) {
  使用 bash
}
```

**决策点 3: 是否继续探索？**

```typescript
// 基于收集的信息评估
if (已有足够信息解决问题) {
  停止探索，开始行动
} else if (已达到探索深度上限) {
  停止探索，开始行动
} else {
  继续探索
}
```

---

## 八、常见问题

### Q1: 迭代不停止？

**原因分析：**

1. LLM 一直认为信息不足
2. 死循环检测未触发
3. 工具调用返回空结果

**解决方案：**

```typescript
// 检查点 1: 死循环检测
if (lastThree.length === DOOM_LOOP_THRESHOLD &&
    lastThree.every(same_tool_and_input)) {
  // 应该触发权限请求
}

// 检查点 2: 工具结果
if (tool_result.is_empty) {
  // 返回空结果，LLM 应该停止
}

// 检查点 3: 手动干预
// 用户可以强制停止
```

### Q2: 探索太慢？

**原因：**

1. glob 模式太宽泛
2. grep 搜索范围过大
3. 读取过多文件

**优化建议：**

```typescript
// 使用更精确的 glob 模式
glob("src/auth/**/*.ts")  // 比 "**/*.ts" 更精确

// 使用文件限制
grep("pattern", { files: "src/**/*.ts" })

// 设置读取深度
read(filePath, { limit: 100 })  // 限制行数
```

### Q3: Explore Agent 不工作？

**检查项：**

```typescript
// 1. Agent 是否存在
const agent = await Agent.get("explore")

// 2. 权限是否正确
if (PermissionNext.evaluate("task", "explore", caller.permission).action === "deny") {
  // 权限被拒绝
}

// 3. 工具是否可用
const tools = await ToolRegistry.all()
// 检查 glob, grep, read 是否在列表中
```

### Q4: SubAgent 权限问题？

**问题：**

```
Error: Permission denied for task: explore
```

**解决方案：**

```json
// opencode.json
{
  "agent": {
    "build": {
      "permission": {
        "task": {
          "explore": "allow"
        }
      }
    }
  }
}
```

### Q5: 工具结果截断？

**原因：**

```typescript
// tool.ts 中有输出截断逻辑
if (output.length > MAX_OUTPUT) {
  output = output.substring(0, MAX_OUTPUT) + "...[truncated]"
}
```

**解决方案：**

```typescript
// 使用更精确的工具参数
await ctx.call("grep", {
  pattern: "specific-pattern",
  files: "specific-file.ts",
  context: 3,  // 减少上下文行数
})
```

### Q6: 如何调试迭代过程？

```typescript
// 启用调试日志
log.info("process")  // processor.ts:45
log.info("tool-call", { tool: value.toolName })
log.info("tool-result", { output: value.output })

// 查看 Session 消息
const messages = await Session.messages({ sessionID })
for (const msg of messages) {
  console.log(msg.info.role, msg.parts.length, "parts")
}
```

---

## 附录

### A. 关键文件速查

| 文件 | 关键函数 | 说明 |
|------|----------|------|
| `session/processor.ts:44-397` | `process()` | 处理器主循环 |
| `session/processor.ts:19` | `DOOM_LOOP_THRESHOLD` | 死循环阈值 |
| `agent/agent.ts:103-128` | `explore` | Explore Agent 定义 |
| `agent/prompt/explore.txt` | - | Explore System Prompt |
| `tool/task.ts:23-181` | `TaskTool` | SubAgent 调用工具 |

### B. 事件类型速查

| 事件类型 | 阶段 | 处理函数 |
|----------|------|----------|
| `tool-input-start` | 工具调用开始 | 创建 Part |
| `tool-call` | 工具执行 | 更新状态 |
| `tool-result` | 工具完成 | 记录结果 |
| `tool-error` | 工具错误 | 处理错误 |
| `text-delta` | 文本响应 | 增量更新 |
| `finish-step` | 步骤结束 | 统计成本 |

### C. 推荐学习路径

```
第 1 天：理解循环结构
├── 阅读 processor.ts:48-100
├── 理解 while(true) 和 for-await 结构
└── 理解事件处理 switch 语句

第 2 天：学习工具调用
├── 阅读 processor.ts:102-193
├── 理解状态机 pending → running → completed
└── 理解工具结果如何反馈

第 3 天：掌握死循环检测
├── 阅读 processor.ts:142-167
├── 理解 DOOM_LOOP_THRESHOLD
└── 测试各种循环场景

第 4 天：实践 SubAgent
├── 阅读 task.ts 完整代码
├── 配置 explore agent
└── 测试 subagent 调用
```

### D. 迭代决策检查表

```
在每次迭代中，LLM 应该检查：

□ 问题是否已理解？
  - 如果否，收集更多信息

□ 是否有足够的上下文？
  - 如果否，继续探索

□ 是否找到了解决方案？
  - 如果是，开始行动

□ 是否达到了深度限制？
  - 如果是，开始行动或请求澄清

□ 是否检测到死循环？
  - 如果是，请求用户干预
```
