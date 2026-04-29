# OpenCode Session 系统学习教程

> 基于源码分析的完整学习指南，涵盖 Session 系统的架构设计、消息流转和状态管理。

---

## 目录

| 章节 | 标题 | 难度 |
|------|------|------|
| 一 | 系统概述 | 入门 |
| 二 | Session Info 结构 | 进阶 |
| 三 | 消息系统 (MessageV2) | 进阶 |
| 四 | 会话生命周期 | 进阶 |
| 五 | 上下文压缩与摘要 | 高级 |
| 六 | 权限继承机制 | 进阶 |
| 七 | 消息处理流程 | 高级 |
| 八 | 常见问题 | 排查 |

---

## 一、系统概述

### 1.1 什么是 Session 系统

**Session 系统** 是 OpenCode 的核心状态管理组件，负责管理整个对话生命周期的所有数据和状态。

**核心职责：**

| 职责 | 说明 |
|------|------|
| **会话管理** | 创建、更新、删除会话 |
| **消息流转** | 管理用户/助手消息的生命周期 |
| **上下文压缩** | 长对话的 Token 优化 |
| **状态持久化** | 消息和会话的存储 |
| **权限继承** | 权限规则在会话间的传递 |

**设计目标：**

- **可靠性**：确保消息和状态不会丢失
- **可追溯性**：支持会话分叉(fork)和历史回溯
- **高效性**：自动压缩长对话上下文
- **灵活性**：支持父子会话继承

### 1.2 Session 系统的位置

```
┌─────────────────────────────────────────────────────────────────┐
│                        OpenCode 架构                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌──────────────────┐    ┌─────────────┐    │
│  │   CLI/TUI   │───▶│   Session System │───▶│  Provider   │    │
│  └─────────────┘    └────────┬─────────┘    └─────────────┘    │
│                              │                                    │
│              ┌───────────────┼───────────────┐                  │
│              ▼               ▼               ▼                  │
│      ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│      │   Session   │ │  MessageV2  │ │  Compaction │           │
│      │   Manager   │ │  Message    │ │  Context    │           │
│      └─────────────┘ └─────────────┘ └─────────────┘           │
│                              │                                    │
│              ┌───────────────┼───────────────┐                  │
│              ▼               ▼               ▼                  │
│      ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│      │  Storage    │ │   Bus       │ │  Permission │           │
│      │  Persistence│ │  Event Bus  │ │  Inheritance│           │
│      └─────────────┘ └─────────────┘ └─────────────┘           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**相关文件：**

| 文件 | 路径 | 说明 |
|------|------|------|
| **Session 核心** | `packages/opencode/src/session/index.ts` | Session 管理 (470行) |
| **消息定义** | `packages/opencode/src/session/message-v2.ts` | MessageV2 类型 |
| **消息处理** | `packages/opencode/src/session/processor.ts` | 消息处理器 |
| **上下文压缩** | `packages/opencode/src/session/compaction.ts` | 压缩机制 |
| **消息流转换** | `packages/opencode/src/session/llm.ts` | LLM 流处理 |
| **摘要生成** | `packages/opencode/src/session/summary.ts` | 摘要生成 |

---

## 二、Session Info 结构

### 2.1 类型定义

Session 的核心数据结构定义在 `index.ts:39-79`：

```typescript
// packages/opencode/src/session/index.ts:39-79
export const Info = z.object({
  id: Identifier.schema("session"),           // 会话唯一标识
  projectID: z.string(),                       // 项目 ID
  directory: z.string(),                       // 工作目录
  parentID: Identifier.schema("session").optional(),  // 父会话 ID
  summary: z.object({
    additions: z.number(),                     // 新增行数
    deletions: z.number(),                     // 删除行数
    files: z.number(),                         // 修改文件数
    diffs: Snapshot.FileDiff.array().optional(),  // 文件差异
  }).optional(),                               // 会话摘要
  share: z.object({
    url: z.string(),                           // 分享 URL
  }).optional(),                               // 分享信息
  title: z.string(),                           // 会话标题
  version: z.string(),                         // OpenCode 版本
  time: z.object({
    created: z.number(),                       // 创建时间
    updated: z.number(),                       // 更新时间
    compacting: z.number().optional(),         // 压缩时间
    archived: z.number().optional(),           // 归档时间
  }),                                          // 时间戳
  permission: PermissionNext.Ruleset.optional(),  // 权限规则
  revert: z.object({
    messageID: z.string(),                     // 消息 ID
    partID: z.string().optional(),             // 部分 ID
    snapshot: z.string().optional(),           // 快照
    diff: z.string().optional(),               // 差异
  }).optional(),                               // 回滚信息
})
```

### 2.2 核心字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| **id** | `string` | Session 唯一标识，格式 `session_xxx` |
| **projectID** | `string` | 所属项目 ID |
| **directory** | `string` | 当前工作目录 |
| **parentID** | `string` | 父 Session ID（用于分叉） |
| **summary** | `Object` | 会话成果统计（新增/删除/修改） |
| **share** | `Object` | 分享 URL（可选） |
| **title** | `string` | 会话标题 |
| **version** | `string` | OpenCode 版本号 |
| **time** | `Object` | 时间戳（创建/更新/压缩） |
| **permission** | `Ruleset` | 权限规则集 |
| **revert** | `Object` | 回滚信息 |

### 2.3 默认标题生成

```typescript
// index.ts:26-37 - 标题生成规则
const parentTitlePrefix = "New session - "
const childTitlePrefix = "Child session - "

function createDefaultTitle(isChild = false) {
  return (isChild ? childTitlePrefix : parentTitlePrefix) + new Date().toISOString()
}

export function isDefaultTitle(title: string) {
  return new RegExp(
    `^(${parentTitlePrefix}|${childTitlePrefix})\\d{4}-\\d{2}-\\d{2}T\\d{2}:\\d{2}:\\d{2}\\.\\d{3}Z$`,
  ).test(title)
}
```

**标题格式示例：**

```
父会话: "New session - 2024-01-15T10:30:45.123Z"
子会话: "Child session - 2024-01-15T10:35:12.456Z"
```

### 2.4 Session 事件

```typescript
// index.ts:91-124 - Session 事件定义
export const Event = {
  Created: BusEvent.define("session.created", { info: Info }),      // 创建事件
  Updated: BusEvent.define("session.updated", { info: Info }),      // 更新事件
  Deleted: BusEvent.define("session.deleted", { info: Info }),      // 删除事件
  Diff: BusEvent.define("session.diff", {                            // 差异事件
    sessionID: z.string(),
    diff: Snapshot.FileDiff.array(),
  }),
  Error: BusEvent.define("session.error", {                          // 错误事件
    sessionID: z.string().optional(),
    error: MessageV2.Assistant.shape.error,
  }),
}
```

---

## 三、消息系统 (MessageV2)

### 3.1 消息类型定义

消息系统定义了两种核心消息类型：

```typescript
// index.ts:298-321 - User 消息
export const User = Base.extend({
  role: z.literal("user"),
  time: z.object({ created: z.number() }),
  summary: z.object({
    title: z.string().optional(),
    body: z.string().optional(),
    diffs: Snapshot.FileDiff.array(),
  }).optional(),
  agent: z.string(),                              // 使用的 Agent
  model: z.object({ providerID: z.string(), modelID: z.string() }),
  system: z.string().optional(),                  // System Prompt
  tools: z.record(z.string(), z.boolean()).optional(),
  variant: z.string().optional(),
})

// index.ts:343-385 - Assistant 消息
export const Assistant = Base.extend({
  role: z.literal("assistant"),
  time: z.object({ created: z.number(), completed: z.number().optional() }),
  error: z.discriminatedUnion(...).optional(),    // 错误信息
  parentID: z.string(),                           // 父消息 ID
  modelID: z.string(),                            // 模型 ID
  providerID: z.string(),                         // 提供商 ID
  mode: z.string(),                               // 模式（已废弃）
  agent: z.string(),                              // Agent 名称
  path: z.object({ cwd: z.string(), root: z.string() }),
  summary: z.boolean().optional(),                // 是否为摘要消息
  cost: z.number(),                               // 成本
  tokens: z.object({                              // Token 统计
    input: z.number(),
    output: z.number(),
    reasoning: z.number(),
    cache: z.object({ read: z.number(), write: z.number() }),
  }),
  finish: z.string().optional(),                  // 完成原因
})
```

### 3.2 消息 Part 类型

MessageV2 将消息拆分为多种 Part：

| Part 类型 | 说明 | 用途 |
|-----------|------|------|
| **TextPart** | 文本内容 | 普通对话文本 |
| **ToolPart** | 工具调用 | 工具输入/输出 |
| **ReasoningPart** | 推理过程 | 模型思考过程 |
| **FilePart** | 文件附件 | 图片、PDF 等 |
| **AgentPart** | Agent 切换 | Agent 状态记录 |
| **SubtaskPart** | 子任务 | 子任务定义 |
| **CompactionPart** | 压缩标记 | 上下文压缩 |
| **RetryPart** | 重试信息 | 错误重试记录 |
| **StepStartPart** | 步骤开始 | 步骤标记 |
| **StepFinishPart** | 步骤完成 | 步骤统计 |
| **SnapshotPart** | 快照 | 文件快照 |
| **PatchPart** | 补丁 | 代码修改 |

### 3.3 ToolState 状态机

```typescript
// message-v2.ts:214-281 - 工具状态定义

// 状态 1：待执行
ToolStatePending = {
  status: "pending",
  input: Record<string, any>,
  raw: string,
}

// 状态 2：执行中
ToolStateRunning = {
  status: "running",
  input: Record<string, any>,
  title: string,
  time: { start: number },
}

// 状态 3：已完成
ToolStateCompleted = {
  status: "completed",
  input: Record<string, any>,
  output: string,
  title: string,
  metadata: Record<string, any>,
  time: { start: number, end: number, compacted?: number },
  attachments: FilePart[],
}

// 状态 4：错误
ToolStateError = {
  status: "error",
  input: Record<string, any>,
  error: string,
  time: { start: number, end: number },
}
```

### 3.4 消息转换

```typescript
// message-v2.ts:429-447 - 转换为 LLM 消息格式
export function toModelMessage(input: WithParts[]): ModelMessage[] {
  const result: UIMessage[] = []

  for (const msg of input) {
    if (msg.parts.length === 0) continue

    if (msg.info.role === "user") {
      const userMessage: UIMessage = {
        id: msg.info.id,
        role: "user",
        parts: [],
      }
      result.push(userMessage)
      for (const part of msg.parts) {
        if (part.type === "text" && !part.ignored)
          userMessage.parts.push({ type: "text", text: part.text })
      }
    }
    // ... more conversions
  }
  return result
}
```

---

## 四、会话生命周期

### 4.1 会话创建

```typescript
// index.ts:181-221 - 创建新会话
export async function createNext(input: {
  id?: string
  title?: string
  parentID?: string
  directory: string
  permission?: PermissionNext.Ruleset
}) {
  const result: Info = {
    id: Identifier.descending("session", input.id),    // 生成 ID
    version: Installation.VERSION,                     // 版本号
    projectID: Instance.project.id,                    // 项目 ID
    directory: input.directory,                        // 目录
    parentID: input.parentID,                          // 父会话
    title: input.title ?? createDefaultTitle(!!input.parentID),
    permission: input.permission,                      // 权限
    time: {
      created: Date.now(),                             // 创建时间
      updated: Date.now(),                             // 更新时间
    },
  }

  // 持久化
  await Storage.write(["session", Instance.project.id, result.id], result)

  // 发布事件
  Bus.publish(Event.Created, { info: result })
  Bus.publish(Event.Updated, { info: result })

  // 自动分享
  const cfg = await Config.get()
  if (!result.parentID && (Flag.OPENCODE_AUTO_SHARE || cfg.share === "auto"))
    share(result.id).then((share) => {
      update(result.id, (draft) => { draft.share = share })
    })

  return result
}
```

### 4.2 会话分叉 (Fork)

```typescript
// index.ts:144-173 - 会话分叉
export const fork = fn(
  z.object({
    sessionID: Identifier.schema("session"),
    messageID: Identifier.schema("message").optional(),
  }),
  async (input) => {
    // 1. 创建新会话
    const session = await createNext({ directory: Instance.directory })

    // 2. 复制消息
    const msgs = await messages({ sessionID: input.sessionID })
    for (const msg of msgs) {
      // 复制到指定消息为止
      if (input.messageID && msg.info.id >= input.messageID) break

      // 复制消息信息
      const cloned = await updateMessage({
        ...msg.info,
        sessionID: session.id,
        id: Identifier.ascending("message"),
      })

      // 复制所有部分
      for (const part of msg.parts) {
        await updatePart({
          ...part,
          id: Identifier.ascending("part"),
          messageID: cloned.id,
          sessionID: session.id,
        })
      }
    }
    return session
  },
)
```

### 4.3 生命周期流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                    Session 生命周期                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 创建会话                                                    │
│     Session.create()                                            │
│         │                                                       │
│         ▼                                                       │
│     生成 ID、标题、时间戳                                        │
│         │                                                       │
│         ▼                                                       │
│     Storage.write() + Bus.publish()                             │
│                                                                 │
│  2. 消息循环                                                    │
│     用户输入 → Assistant 生成 → 工具调用 → 结果                  │
│         │                                                       │
│         ▼                                                       │
│     Session.updateMessage() + Session.updatePart()              │
│                                                                 │
│  3. 会话分叉                                                    │
│     Session.fork()                                              │
│         │                                                       │
│         ▼                                                       │
│     复制消息 + 部分到新会话                                      │
│                                                                 │
│  4. 上下文压缩                                                  │
│     SessionCompaction.process()                                 │
│         │                                                       │
│         ▼                                                       │
│     生成摘要 + 清理旧工具调用                                    │
│                                                                 │
│  5. 会话结束                                                    │
│     Session.remove()                                            │
│         │                                                       │
│         ▼                                                       │
│     删除消息 + 部分 + 会话 + 分享                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.4 核心 API

| 函数 | 签名 | 说明 |
|------|------|------|
| **create** | `create(input?)` | 创建新会话 |
| **createNext** | `createNext(input)` | 创建会话（底层） |
| **fork** | `fork({sessionID, messageID})` | 分叉会话 |
| **get** | `get(id)` | 获取会话 |
| **update** | `update(id, editor)` | 更新会话 |
| **remove** | `remove(id)` | 删除会话 |
| **messages** | `messages({sessionID, limit})` | 获取消息列表 |
| **touch** | `touch(id)` | 更新会话时间 |
| **children** | `children(parentID)` | 获取子会话 |
| **share** | `share(id)` | 创建分享链接 |
| **unshare** | `unshare(id)` | 取消分享 |

---

## 五、上下文压缩与摘要

### 5.1 压缩触发条件

```typescript
// compaction.ts:30-39 - 判断是否需要压缩
export async function isOverflow(input: {
  tokens: MessageV2.Assistant["tokens"]
  model: Provider.Model
}) {
  const config = await Config.get()
  if (config.compaction?.auto === false) return false

  const context = input.model.limit.context
  if (context === 0) return false

  const count = input.tokens.input + input.tokens.cache.read + input.tokens.output
  const output = Math.min(input.model.limit.output, SessionPrompt.OUTPUT_TOKEN_MAX)
  const usable = context - output

  return count > usable
}
```

### 5.2 压缩处理流程

```typescript
// compaction.ts:92-193 - 压缩处理
export async function process(input: {
  parentID: string
  messages: MessageV2.WithParts[]
  sessionID: string
  abort: AbortSignal
  auto: boolean
}) {
  // 1. 获取用户消息
  const userMessage = input.messages.findLast((m) => m.info.id === input.parentID)!.info

  // 2. 使用 compaction Agent 生成摘要
  const agent = await Agent.get("compaction")
  const model = agent.model ? await Provider.getModel(...) : ...

  // 3. 创建摘要消息
  const msg = await Session.updateMessage({
    id: Identifier.ascending("message"),
    role: "assistant",
    parentID: input.parentID,
    sessionID: input.sessionID,
    mode: "compaction",
    agent: "compaction",
    summary: true,  // 标记为摘要消息
    ...
  })

  // 4. 调用 LLM 生成摘要
  const processor = SessionProcessor.create({ ... })
  const result = await processor.process({
    user: userMessage,
    agent,
    messages: [...MessageV2.toModelMessage(input.messages), {
      role: "user",
      content: [{
        type: "text",
        text: "Provide a detailed prompt for continuing our conversation...",
      }],
    }],
    ...
  })

  // 5. 如果 auto 模式，添加继续指令
  if (result === "continue" && input.auto) {
    await Session.updatePart({
      type: "text",
      synthetic: true,
      text: "Continue if you have next steps",
    })
  }

  return result
}
```

### 5.3 旧工具调用清理 (Prune)

```typescript
// compaction.ts:46-90 - 清理旧工具调用
export async function prune(input: { sessionID: string }) {
  const config = await Config.get()
  if (config.compaction?.prune === false) return

  const msgs = await Session.messages({ sessionID: input.sessionID })
  let total = 0
  let pruned = 0
  const toPrune = []

  // 从后向前扫描消息
  loop: for (let msgIndex = msgs.length - 1; msgIndex >= 0; msgIndex--) {
    const msg = msgs[msgIndex]

    // 找到摘要消息则停止
    if (msg.info.role === "assistant" && msg.info.summary) break loop

    for (let partIndex = msg.parts.length - 1; partIndex >= 0; partIndex--) {
      const part = msg.parts[partIndex]

      if (part.type === "tool" && part.state.status === "completed") {
        // 保护 skill 工具调用
        if (PRUNE_PROTECTED_TOOLS.includes(part.tool)) continue

        // 累计 Token
        const estimate = Token.estimate(part.state.output)
        total += estimate

        // 超过阈值则标记清理
        if (total > PRUNE_PROTECT) {
          pruned += estimate
          toPrune.push(part)
        }
      }
    }
  }

  // 清理旧工具调用
  if (pruned > PRUNE_MINIMUM) {
    for (const part of toPrune) {
      part.state.time.compacted = Date.now()
      await Session.updatePart(part)
    }
  }
}
```

### 5.4 压缩配置参数

```typescript
// compaction.ts:41-44 - 压缩参数
export const PRUNE_MINIMUM = 20_000    // 最小清理 Token
export const PRUNE_PROTECT = 40_000    // 保护阈值

const PRUNE_PROTECTED_TOOLS = ["skill"]  // 保护的工具列表
```

---

## 六、权限继承机制

### 6.1 权限在 Session 中的存储

```typescript
// index.ts:66 - Session 权限字段
permission: PermissionNext.Ruleset.optional(),
```

**权限来源：**

```
权限优先级（从高到低）
├── 1. Session.permission（创建时指定）
├── 2. Agent.permission（当前 Agent 配置）
└── 3. Config.permission（用户全局配置）
```

### 6.2 权限创建时指定

```typescript
// index.ts:126-142 - 创建会话时指定权限
export const create = fn(
  z.object({
    parentID: Identifier.schema("session").optional(),
    title: z.string().optional(),
    permission: Info.shape.permission,  // 可指定权限
  }).optional(),
  async (input) => {
    return createNext({
      parentID: input?.parentID,
      directory: Instance.directory,
      title: input?.title,
      permission: input?.permission,  // 传递权限
    })
  },
)
```

### 6.3 权限继承流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    权限继承流程                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 父会话权限                                                   │
│     parent.permission = { "read": "allow", "edit": "deny" }     │
│           │                                                       │
│           ▼                                                       │
│  2. 创建子会话                                                   │
│     Session.create({ parentID: "xxx" })                         │
│           │                                                       │
│           ▼                                                       │
│  3. 权限合并                                                     │
│     child.permission = merge(parent.permission, config)         │
│           │                                                       │
│           ▼                                                       │
│  4. 分叉会话                                                     │
│     Session.fork({ sessionID: "xxx" })                          │
│           │                                                       │
│           ▼                                                       │
│  5. 完全复制权限                                                 │
│     fork 会话继承原会话的所有权限                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.4 权限与消息关联

```typescript
// message-v2.ts:310-314 - User 消息关联 Agent
export const User = Base.extend({
  ...
  agent: z.string(),  // 使用的 Agent
  model: z.object({ providerID: z.string(), modelID: z.string() }),
  ...
})
```

**关联流程：**

```
消息 → Agent → Agent.permission → 有效权限规则
```

### 6.5 权限检查示例

```typescript
// 工具调用时的权限检查
const agent = ctx?.agent
const accessibleSkills = agent
  ? skills.filter((skill) => {
      const rule = PermissionNext.evaluate("skill", skill.name, agent.permission)
      return rule.action !== "deny"
    })
  : skills
```

---

## 七、消息处理流程

### 7.1 处理器创建

```typescript
// processor.ts:25-35 - 创建处理器
export function create(input: {
  assistantMessage: MessageV2.Assistant
  sessionID: string
  model: Provider.Model
  abort: AbortSignal
}) {
  const toolcalls: Record<string, MessageV2.ToolPart> = {}
  let snapshot: string | undefined
  let blocked = false
  let attempt = 0
  let needsCompaction = false

  return {
    message: input.assistantMessage,
    partFromToolCall(toolCallID: string) { return toolcalls[toolCallID] },
    async process(streamInput: LLM.StreamInput) { ... },
  }
}
```

### 7.2 流处理主循环

```typescript
// processor.ts:44-164 - 主处理循环
async process(streamInput: LLM.StreamInput) {
  const shouldBreak = (await Config.get()).experimental?.continue_loop_on_deny !== true

  while (true) {
    try {
      const stream = await LLM.stream(streamInput)

      for await (const value of stream.fullStream) {
        input.abort.throwIfAborted()

        switch (value.type) {
          case "start":
            SessionStatus.set(input.sessionID, { type: "busy" })
            break

          case "reasoning-start":
            // 创建推理 Part
            break

          case "reasoning-delta":
            // 更新推理内容
            break

          case "tool-input-start":
            // 创建工具调用 Part
            break

          case "tool-call":
            // 执行工具调用
            // 检测死循环
            const lastThree = parts.slice(-DOOM_LOOP_THRESHOLD)
            if (lastThree.length === DOOM_LOOP_THRESHOLD && ...) {
              if (shouldBreak) throw new DoomLoopError()
            }
            break

          case "tool-result":
            // 更新工具结果
            break

          case "text-delta":
            // 更新文本内容
            break

          case "text-end":
            // 完成文本
            break
        }
      }

      // 检查是否需要压缩
      if (await SessionCompaction.isOverflow({ ... })) {
        await SessionCompaction.prune({ sessionID: input.sessionID })
        if (await SessionCompaction.isOverflow({ ... })) {
          needsCompaction = true
        }
      }

      break
    } catch (e) {
      if (e instanceof RetryError) {
        attempt++
        continue
      }
      throw e
    }
  }

  // 如果需要压缩，触发压缩
  if (needsCompaction) {
    await SessionCompaction.create({ ... })
  }
}
```

### 7.3 死循环检测

```typescript
// processor.ts:143-156 - 检测连续相同工具调用
if (
  lastThree.length === DOOM_LOOP_THRESHOLD &&
  lastThree.every(
    (p) =>
      p.type === "tool" &&
      p.tool === value.toolName &&
      p.state.status === "completed",
  ) &&
  !lastThree.some((p) => p.state.status === "error")
) {
  if (shouldBreak) throw new DoomLoopError()
  input.abort.abort(new DoomLoopError("doom loop detected"))
}
```

### 7.4 完整消息流

```
┌─────────────────────────────────────────────────────────────────┐
│                    消息处理流程                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 用户输入                                                     │
│     User Message + TextPart                                      │
│           │                                                       │
│           ▼                                                       │
│  2. 创建 Assistant Message                                       │
│     Session.updateMessage()                                      │
│           │                                                       │
│           ▼                                                       │
│  3. LLM 流处理                                                   │
│     LLM.stream() → Stream events                                 │
│           │                                                       │
│           ▼                                                       │
│  4. 处理事件                                                     │
│     reasoning-start/delta/end                                    │
│     tool-input-start/call/result                                 │
│     text-delta/end                                               │
│           │                                                       │
│           ▼                                                       │
│  5. 工具执行                                                     │
│     Tool.execute()                                               │
│           │                                                       │
│           ▼                                                       │
│  6. 状态更新                                                     │
│     Session.updatePart()                                         │
│           │                                                       │
│           ▼                                                       │
│  7. 完成处理                                                     │
│     Token 统计 + 成本计算                                         │
│           │                                                       │
│           ▼                                                       │
│  8. 压缩检查                                                     │
│     isOverflow() → prune() → compaction                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 八、常见问题

### Q1: 会话没有自动分享？

```typescript
// index.ts:206-216 - 自动分享逻辑
if (!result.parentID && (Flag.OPENCODE_AUTO_SHARE || cfg.share === "auto")) {
  share(result.id).then((share) => {
    update(result.id, (draft) => {
      draft.share = share
    })
  })
}

// 检查项：
// 1. 是否设置了 Flag.OPENCODE_AUTO_SHARE
// 2. Config.get().share 是否为 "auto"
// 3. 是否是子会话（子会话不会自动分享）
```

### Q2: 上下文压缩不触发？

```typescript
// 检查 compaction.ts:30-39 - 触发条件
if (config.compaction?.auto === false) return false
// 原因：配置中设置了 auto: false
```

**解决方案：**

```json
// opencode.json
{
  "compaction": {
    "auto": true,
    "prune": true
  }
}
```

### Q3: 分叉会话消息不完整？

```typescript
// index.ts:154-155 - 分叉逻辑
if (input.messageID && msg.info.id >= input.messageID) break
// 只复制指定消息之前的内容

// 解决方案：
// 1. 不指定 messageID 则复制所有消息
// 2. 指定 messageID 则包含该消息
```

### Q4: 权限不生效？

```typescript
// 检查权限是否正确传递
// 1. Session 创建时是否指定 permission
// 2. Agent 是否具有权限配置
// 3. 权限规则是否正确格式

// 权限格式：
{
  "read": { "*": "allow" },
  "edit": { "*.ts": "ask" }
}
```

### Q5: 消息存储在哪里？

```typescript
// 消息存储路径
Storage.write(["message", msg.sessionID, msg.id], msg)  // 消息
Storage.write(["part", part.messageID, part.id], part)  // 部分

// 路径格式：
// message/{sessionID}/{messageID}
// part/{messageID}/{partID}
```

### Q6: 如何回滚会话？

```typescript
// index.ts:67-74 - 回滚信息结构
revert: z.object({
  messageID: z.string(),
  partID: z.string().optional(),
  snapshot: z.string().optional(),
  diff: z.string().optional(),
}).optional(),

// 回滚流程：
// 1. 保存当前状态到 revert
// 2. 恢复时使用 revert 中的信息
```

### Q7: 死循环如何处理？

```typescript
// processor.ts:143-156 - 检测阈值
const DOOM_LOOP_THRESHOLD = 3

// 触发条件：
// 1. 连续 3 次相同工具调用
// 2. 所有调用都成功完成
// 3. 没有错误状态

// 处理方式：
// 1. shouldBreak = true: 抛出 DoomLoopError
// 2. shouldBreak = false: 仅终止请求
```

---

## 附录

### A. Session 速查表

| 场景 | API | 说明 |
|------|-----|------|
| 创建会话 | `Session.create()` | 创建新会话 |
| 分叉会话 | `Session.fork({sessionID, messageID})` | 复制会话 |
| 获取会话 | `Session.get(id)` | 读取会话 |
| 更新会话 | `Session.update(id, editor)` | 修改会话 |
| 删除会话 | `Session.remove(id)` | 删除会话 |
| 获取消息 | `Session.messages({sessionID, limit})` | 列出消息 |
| 更新消息 | `Session.updateMessage(msg)` | 保存消息 |
| 更新部分 | `Session.updatePart(part)` | 保存部分 |

### B. Token 计算速查

| Token 类型 | 说明 |
|------------|------|
| **input** | 输入 Token（用户消息 + System Prompt） |
| **output** | 输出 Token（Assistant 响应） |
| **reasoning** | 推理 Token（思考过程） |
| **cache.read** | 缓存读取 Token |
| **cache.write** | 缓存写入 Token |

### C. 成本计算

```typescript
// index.ts:425-443 - 成本计算
const cost = new Decimal(0)
  .add(new Decimal(tokens.input).mul(costInfo?.input ?? 0).div(1_000_000))
  .add(new Decimal(tokens.output).mul(costInfo?.output ?? 0).div(1_000_000))
  .add(new Decimal(tokens.cache.read).mul(costInfo?.cache?.read ?? 0).div(1_000_000))
  .add(new Decimal(tokens.cache.write).mul(costInfo?.cache?.write ?? 0).div(1_000_000))
  .add(new Decimal(tokens.reasoning).mul(costInfo?.output ?? 0).div(1_000_000))
  .toNumber()
```

### D. 推荐学习路径

```
第 1 天：理解 Session 架构
        ├── 阅读 index.ts 核心结构
        ├── 理解 Info 类型定义
        └── 掌握生命周期管理

第 2 天：学习消息系统
        ├── 阅读 message-v2.ts 类型
        ├── 理解 Part 类型体系
        └── 学习消息转换逻辑

第 3 天：实践会话操作
        ├── 创建和分叉会话
        ├── 测试消息传递
        └── 验证权限继承

第 4 天：深入压缩机制
        ├── 学习 isOverflow 判断
        ├── 理解 prune 清理逻辑
        └── 掌握压缩提示生成
```
