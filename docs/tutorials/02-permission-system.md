# OpenCode 权限审核系统学习教程

> 基于源码分析的完整学习指南，涵盖权限系统的架构设计、实现原理和定制方法。

---

## 目录

| 章节 | 标题 | 难度 |
|------|------|------|
| 一 | 系统概述 | 入门 |
| 二 | 核心架构 | 进阶 |
| 三 | 权限类型详解 | 进阶 |
| 四 | 权限请求流程 | 进阶 |
| 五 | 权限规则配置 | 进阶 |
| 六 | BashArity 命令分组 | 高级 |
| 七 | 错误处理机制 | 进阶 |
| 八 | 实践演练 | 实践 |
| 九 | 常见问题 | 排查 |

---

## 一、系统概述

### 1.1 什么是权限审核系统

**权限审核系统** 是 OpenCode 安全架构的核心组件，负责在工具执行前验证操作是否被允许。

**核心职责：**

| 职责 | 说明 |
|------|------|
| **权限检查** | 根据规则判断操作是否被允许 |
| **用户交互** | 弹窗请求用户授权 |
| **规则持久化** | 保存用户的授权决策 |
| **访问控制** | 控制对敏感资源的访问 |

**设计目标：**

- **安全性**：防止未经授权的敏感操作
- **可控性**：用户完全掌控权限决策
- **灵活性**：支持细粒度的规则配置
- **用户体验**：清晰的权限请求界面

### 1.2 权限系统的位置

```
┌─────────────────────────────────────────────────────────────────┐
│                        OpenCode 架构                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌──────────────────┐    ┌─────────────┐    │
│  │   Agent     │───▶│  Tool Execution  │───▶│  Provider   │    │
│  └─────────────┘    └────────┬─────────┘    └─────────────┘    │
│                              │                                    │
│                              ▼                                    │
│                    ┌──────────────────┐                          │
│                    │   权限审核系统    │                          │
│                    │  PermissionNext  │                          │
│                    └──────────────────┘                          │
│                              │                                    │
│                              ▼                                    │
│                    ┌──────────────────┐                          │
│                    │   实际工具执行    │                          │
│                    └──────────────────┘                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 权限类型概览

OpenCode 支持以下权限类型：

| 权限名 | 说明 | 默认动作 |
|--------|------|----------|
| `read` | 读取文件内容 | allow |
| `edit` | 编辑/修改文件 | ask |
| `write` | 写入新文件 | ask |
| `glob` | 文件 glob 匹配 | allow |
| `grep` | 代码搜索 | allow |
| `list` / `ls` | 列出目录内容 | allow |
| `bash` | 执行 bash 命令 | ask |
| `task` | 调用子 agent | ask |
| `webfetch` | 抓取网页内容 | allow |
| `websearch` | 网络搜索 | allow |
| `codesearch` | 代码搜索 | allow |
| `external_directory` | 访问项目外目录 | ask |
| `doom_loop` | 死循环检测 | ask |

---

## 二、核心架构

### 2.1 核心文件结构

**权限系统核心文件：**

| 文件 | 路径 | 说明 |
|------|------|------|
| **PermissionNext** | `packages/opencode/src/permission/next.ts` | 新版权限系统核心 |
| **Permission** | `packages/opencode/src/permission/index.ts` | 旧版权限系统 |
| **BashArity** | `packages/opencode/src/permission/arity.ts` | bash 命令参数分组 |
| **TUI 权限对话框** | `cli/cmd/tui/routes/session/permission.tsx` | 前端权限 UI |
| **权限事件同步** | `cli/cmd/tui/context/sync.tsx` | 事件总线同步 |

### 2.2 PermissionNext 命名空间

**位置**：`packages/opencode/src/permission/next.ts`

**核心类型定义：**

```typescript
// 权限动作类型
export const Action = z.enum(["allow", "deny", "ask"])
export type Action = z.infer<typeof Action>

// 单条规则
export const Rule = z.object({
  permission: z.string(),  // 权限类型
  pattern: z.string(),     // 匹配模式
  action: Action,          // 允许/拒绝/询问
})
export type Rule = z.infer<typeof Rule>

// 规则集
export const Ruleset = Rule.array()
export type Ruleset = z.infer<typeof Ruleset>

// 权限请求
export const Request = z.object({
  id: Identifier.schema("permission"),
  sessionID: Identifier.schema("session"),
  permission: z.string(),
  patterns: z.string().array(),
  metadata: z.record(z.string(), z.any()),
  always: z.string().array(),
  tool: z.object({
    messageID: z.string(),
    callID: z.string(),
  }).optional(),
})

// 用户回复
export const Reply = z.enum(["once", "always", "reject"])
export type Reply = z.infer<typeof Reply>
```

### 2.3 核心函数

**权限系统提供以下核心函数：**

| 函数 | 说明 | 用途 |
|------|------|------|
| `fromConfig()` | 从配置创建规则集 | 初始化权限规则 |
| `merge()` | 合并多个规则集 | 组合权限规则 |
| `ask()` | 请求权限 | 工具调用时触发 |
| `reply()` | 回复权限请求 | 用户操作后响应 |
| `evaluate()` | 评估单条规则 | 判断权限动作 |
| `disabled()` | 检查禁用的工具 | UI 工具栏控制 |

### 2.4 权限状态管理

**状态存储** (`next.ts:97-114`)：

```typescript
const state = Instance.state(async () => {
  const projectID = Instance.project.id
  // 从存储读取已批准的规则
  const stored = await Storage.read<Ruleset>(["permission", projectID]).catch(() => [])

  // 待处理的权限请求
  const pending: Record<string, {
    info: Request
    resolve: () => void
    reject: (e: any) => void
  }> = {}

  return {
    pending,      // 待处理请求
    approved: stored,  // 已批准的规则
  }
})
```

**状态结构：**

| 状态字段 | 类型 | 说明 |
|----------|------|------|
| `pending` | Record | 待处理的权限请求队列 |
| `approved` | Ruleset | 已批准的权限规则（持久化） |

---

## 三、权限请求流程

### 3.1 整体流程图

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          权限请求完整流程                                 │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Tool.execute()                                                           │
│       │                                                                   │
│       ▼                                                                   │
│  ctx.ask({ permission, patterns, always, metadata })                      │
│       │                                                                   │
│       ▼                                                                   │
│  PermissionNext.ask()                                                     │
│       │                                                                   │
│       ├─────────────────────────────────────────────────────────────┐     │
│       │                                                             │     │
│       ▼                                                             ▼     │
│  evaluate() ──命中 deny ──→ DeniedError                      evaluate()    │
│       │                                                        │          │
│       │                                                     未命中         │
│       │◀──────────────────────────────────────────────────────┘          │
│       │                                                             ▲     │
│       ├─────────────────────────────────────────────────────────────┼     │
│       │                                                             │     │
│       ▼                                                             │     │
│  命中 ask ──→ Promise 阻塞 ──→ Bus.publish(Event.Asked)            │     │
│       │                     │                                     │     │
│       │                     ▼                                     │     │
│       │             TUI 显示权限对话框                              │     │
│       │                     │                                     │     │
│       │         ┌───────────┼───────────┐                         │     │
│       │         ▼           ▼           ▼                         │     │
│       │       Once      Always      Reject                        │     │
│       │         │           │           │                         │     │
│       │         └───────────┴───────────┘                         │     │
│       │                     │                                     │     │
│       │                     ▼                                     │     │
│       │             Bus.publish(Event.Replied)                    │     │
│       │                     │                                     │     │
│       │                     ▼                                     │     │
│       │             Promise resolve/reject                        │     │
│       │                     │                                     │     │
│       └─────────────────────┴─────────────────────────────────────┘     │
│                               │                                         │
│                               ▼                                         │
│                    继续执行或抛出错误                                     │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.2 核心函数：ask()

**位置**：`next.ts:116-146`

**功能**：发起权限请求

```typescript
export const ask = fn(
  Request.partial({ id: true }).extend({
    ruleset: Ruleset,  // 权限规则集
  }),
  async (input) => {
    const s = await state()
    const { ruleset, ...request } = input

    // 遍历所有请求的模式
    for (const pattern of request.patterns ?? []) {
      // 评估权限
      const rule = evaluate(request.permission, pattern, ruleset, s.approved)
      log.info("evaluated", { permission: request.permission, pattern, action: rule })

      // deny 动作：直接抛出错误
      if (rule.action === "deny")
        throw new DeniedError(ruleset.filter((r) => Wildcard.match(request.permission, r.permission)))

      // ask 动作：阻塞等待用户回复
      if (rule.action === "ask") {
        const id = input.id ?? Identifier.ascending("permission")
        return new Promise<void>((resolve, reject) => {
          const info: Request = { id, ...request }
          s.pending[id] = { info, resolve, reject }
          Bus.publish(Event.Asked, info)  // 发布事件通知 UI
        })
      }

      // allow 动作：继续检查下一个模式
      if (rule.action === "allow") continue
    }
  },
)
```

**参数说明：**

| 参数 | 类型 | 说明 |
|------|------|------|
| `permission` | string | 权限类型（如 "bash"、"read"） |
| `patterns` | string[] | 要检查的模式列表 |
| `always` | string[] | 始终允许的模式（"always" 回复时使用） |
| `metadata` | Record | 元数据（用于 UI 显示） |
| `ruleset` | Ruleset | 权限规则集 |
| `tool` | object | 工具调用信息（可选） |

**返回值：** Promise，阻塞直到用户回复

### 3.3 核心函数：reply()

**位置**：`next.ts:148-218`

**功能**：处理用户回复

```typescript
export const reply = fn(
  z.object({
    requestID: Identifier.schema("permission"),
    reply: Reply,  // once / always / reject
    message: z.string().optional(),  // 拒绝时的说明
  }),
  async (input) => {
    const s = await state()
    const existing = s.pending[input.requestID]
    if (!existing) return

    // 清除待处理记录
    delete s.pending[input.requestID]

    // 发布回复事件
    Bus.publish(Event.Replied, {
      sessionID: existing.info.sessionID,
      requestID: existing.info.id,
      reply: input.reply,
    })

    // 处理不同回复
    if (input.reply === "reject") {
      // 用户拒绝（带可选说明）
      existing.reject(input.message ? new CorrectedError(input.message) : new RejectedError())
      // 拒绝当前会话的所有待处理请求
      // ...
      return
    }

    if (input.reply === "once") {
      // 仅本次允许
      existing.resolve()
      return
    }

    if (input.reply === "always") {
      // 永久保存到 approved 规则集
      for (const pattern of existing.info.always) {
        s.approved.push({
          permission: existing.info.permission,
          pattern,
          action: "allow",
        })
      }
      existing.resolve()

      // 检查是否有其他待处理请求可以被自动批准
      // ...
      return
    }
  },
)
```

**回复类型说明：**

| 回复 | 说明 | 效果 |
|------|------|------|
| `once` | 仅本次允许 | 只允许当前请求，下次相同请求需重新询问 |
| `always` | 始终允许 | 永久保存到规则集，下次自动允许 |
| `reject` | 拒绝 | 拒绝当前请求，抛出错误 |

### 3.4 核心函数：evaluate()

**位置**：`next.ts:220-227`

**功能**：评估单条权限规则

```typescript
export function evaluate(permission: string, pattern: string, ...rulesets: Ruleset[]): Rule {
  const merged = merge(...rulesets)  // 合并所有规则集
  log.info("evaluate", { permission, pattern, ruleset: merged })

  // 查找匹配的规则（使用 findLast，后出现的规则优先级更高）
  const match = merged.findLast(
    (rule) => Wildcard.match(permission, rule.permission) && Wildcard.match(pattern, rule.pattern),
  )

  // 未匹配时默认返回 ask
  return match ?? { action: "ask", permission, pattern: "*" }
}
```

**匹配规则：**
- 使用 `Wildcard.match()` 进行模式匹配
- `findLast()` 确保后出现的规则优先级更高
- 未匹配任何规则时默认返回 `ask`

---

## 四、权限规则配置

### 4.1 从配置创建规则集

**函数**：`fromConfig()` (`next.ts:36-50`)

```typescript
export function fromConfig(permission: Config.Permission) {
  const ruleset: Ruleset = []
  for (const [key, value] of Object.entries(permission)) {
    if (typeof value === "string") {
      // 简单格式："read": "allow"
      ruleset.push({
        permission: key,
        action: value,
        pattern: "*",
      })
      continue
    }
    // 复杂格式：
    // "read": { "*.env": "deny", "*.txt": "allow" }
    ruleset.push(...Object.entries(value).map(([pattern, action]) => ({
      permission: key,
      pattern,
      action,
    })))
  }
  return ruleset
}
```

**配置格式示例：**

```typescript
// opencode.json 配置
{
  "permission": {
    "*": "allow",
    "doom_loop": "ask",
    "external_directory": "ask",
    "read": {
      "*": "allow",
      "*.env": "deny",
      "*.env.*": "deny",
      "*.env.example": "allow"
    }
  }
}
```

### 4.2 规则合并

**函数**：`merge()` (`next.ts:52-54`)

```typescript
export function merge(...rulesets: Ruleset[]): Ruleset {
  return rulesets.flat()
}
```

**合并顺序（优先级从高到低）：**

1. **Agent 默认规则** (`agent.ts:43-57`)
2. **项目配置规则** (`opencode.json`)
3. **用户全局规则** (`~/.claude/`)
4. **运行时已批准的规则** (内存中)

### 4.3 Agent 权限配置示例

**build Agent** (`agent.ts:43-57`)：

```typescript
const defaults = PermissionNext.fromConfig({
  "*": "allow",  // 默认允许所有操作
  doom_loop: "ask",  // 死循环检测需要询问
  external_directory: "ask",  // 外部目录访问需要询问
  read: {
    "*": "allow",
    "*.env": "deny",  // 禁止读取 .env 文件
    "*.env.*": "deny",
    "*.env.example": "allow",  // 但允许读取示例文件
  },
})
```

**plan Agent**（只读模式）：

```typescript
plan: {
  permission: PermissionNext.fromConfig({
    edit: { "*": "deny" },  // 禁止所有编辑操作
  }),
}
```

---

## 五、权限请求示例

### 5.1 Bash 工具的权限请求

**位置**：`packages/opencode/src/tool/bash.ts:139-155`

```typescript
// 请求外部目录访问权限
if (directories.size > 0) {
  await ctx.ask({
    permission: "external_directory",
    patterns: Array.from(directories),
    always: Array.from(directories).map((x) => path.dirname(x) + "*"),
    metadata: {},
  })
}

// 请求 bash 执行权限
if (patterns.size > 0) {
  await ctx.ask({
    permission: "bash",
    patterns: Array.from(patterns),      // 当前命令模式
    always: Array.from(always),           // 持久化的模式
    metadata: {},
  })
}
```

**请求参数说明：**

| 参数 | 说明 |
|------|------|
| `patterns` | 当前要执行的命令模式 |
| `always` | 要持久化的匹配模式（"always" 回复时使用） |
| `permission` | 权限类型 |

### 5.2 其他工具的权限请求

**Read 工具**（通常不需要权限请求）：

```typescript
// read 权限通常被默认允许，不需要显式请求
```

**Edit 工具**：

```typescript
await ctx.ask({
  permission: "edit",
  patterns: [filePath],
  always: [filePath],
  metadata: {},
})
```

---

## 六、BashArity 命令分组

### 6.1 什么是 BashArity

**BashArity** 是一个用于解析 bash 命令参数分组的工具，帮助将具体命令转换为可读的"人类可理解命令"。

**位置**：`packages/opencode/src/permission/arity.ts`

**核心功能**：`prefix()` 函数

```typescript
export namespace BashArity {
  export function prefix(tokens: string[]) {
    // 从后往前查找最长匹配的前缀
    for (let len = tokens.length; len > 0; len--) {
      const prefix = tokens.slice(0, len).join(" ")
      const arity = ARITY[prefix]
      if (arity !== undefined) return tokens.slice(0, arity)
    }
    if (tokens.length === 0) return []
    return tokens.slice(0, 1)  // 默认只取第一个 token
  }
}
```

### 6.2 Arity 字典

**部分命令的 arity 定义**：

| 命令前缀 | Arity | 示例 |
|----------|-------|------|
| `cat` | 1 | `cat file.txt` |
| `cd` | 1 | `cd /path` |
| `git` | 2 | `git checkout main` |
| `git remote` | 3 | `git remote add origin` |
| `npm` | 2 | `npm install` |
| `npm run` | 3 | `npm run dev` |
| `docker` | 2 | `docker run nginx` |
| `docker compose` | 3 | `docker compose up` |
| `kubectl` | 2 | `kubectl get pods` |
| `terraform` | 2 | `terraform apply` |

### 6.3 使用示例

**命令解析流程：**

```
输入命令：git checkout -b feature/new-feature
         ↓
tokenize: ["git", "checkout", "-b", "feature/new-feature"]
         ↓
查找 arity：git → 2, git checkout 未定义
         ↓
返回 prefix：["git", "checkout"]
         ↓
生成模式："git checkout *"（用于权限匹配）
```

**在 Bash 工具中的使用** (`bash.ts:135`)：

```typescript
// 为每条命令生成权限模式
patterns.add(command.join(" "))                    // 具体命令
always.add(BashArity.prefix(command).join(" ") + "*")  // 泛化命令
```

**效果：**

- 首次执行 `git checkout -b feature/new-feature` 需要权限
- 批准后，后续执行 `git checkout main` 自动允许（匹配 `git checkout *`）

---

## 七、错误处理机制

### 7.1 错误类型

**位置**：`next.ts:243-264`

```typescript
// 用户拒绝（无说明）- 停止执行
export class RejectedError extends Error {
  constructor() {
    super(`The user rejected permission to use this specific tool call.`)
  }
}

// 用户拒绝（有说明）- 继续执行但带指导
export class CorrectedError extends Error {
  constructor(message: string) {
    super(`The user rejected permission to use this specific tool call with the following feedback: ${message}`)
  }
}

// 配置规则拒绝 - 自动停止执行
export class DeniedError extends Error {
  constructor(public readonly ruleset: Ruleset) {
    super(`The user has specified a rule which prevents you from using this specific tool call...`)
  }
}
```

**错误处理对比：**

| 错误类型 | 触发条件 | 行为 | 后续操作 |
|----------|----------|------|----------|
| `RejectedError` | 用户拒绝操作 | 停止执行 | 需要修改后重试 |
| `CorrectedError` | 用户拒绝+说明 | 停止执行 | 需按指导修改 |
| `DeniedError` | 规则命中 deny | 停止执行 | 无法重试 |

### 7.2 错误处理流程

```
Tool.execute()
      │
      ▼
ctx.ask() ──命中 deny──→ throw DeniedError
      │
  阻塞等待
      │
      ▼
用户选择 Reject
      │
      ▼
throw RejectedError / CorrectedError
      │
      ▼
Agent 捕获错误，决定是否重试或报错
```

### 7.3 Agent 中的错误处理

```typescript
try {
  await ctx.ask({ ... })
} catch (e) {
  if (e instanceof PermissionNext.DeniedError) {
    // 配置规则拒绝，无法继续
    throw new Error("Permission denied by configuration")
  }
  if (e instanceof PermissionNext.RejectedError) {
    // 用户拒绝，可以提示用户或修改请求
    throw new Error("User rejected the permission request")
  }
  if (e instanceof PermissionNext.CorrectedError) {
    // 用户提供了修正建议
    throw new Error(`User suggestion: ${e.message}`)
  }
}
```

---

## 八、实践演练

### 8.1 实践一：配置项目权限规则

**目标**：通过配置文件控制项目权限

**步骤**：

1. 在 `opencode.json` 中添加权限配置：

```json
{
  "permission": {
    "bash": {
      "rm -rf *": "deny",
      "chmod 777 *": "deny"
    },
    "read": {
      "*.key": "deny",
      "*.pem": "deny",
      "secrets/**": "deny"
    }
  }
}
```

2. 测试效果：
   - 执行被禁止的命令应该立即失败
   - 执行敏感文件读取应该被拒绝

---

### 8.2 实践二：创建只读 Agent

**目标**：创建一个禁止所有编辑操作的 Agent

**Agent 配置**：

```json
{
  "agent": {
    "readonly": {
      "name": "readonly",
      "description": "Read-only analysis agent",
      "mode": "primary",
      "permission": {
        "edit": "*:deny",
        "write": "*:deny",
        "bash": {
          "*": "ask",
          "rm *": "deny",
          "mv *": "deny"
        }
      }
    }
  }
}
```

**使用方式**：`@readonly 分析这段代码`

---

### 8.3 实践三：理解 BashArity 模式匹配

**目标**：理解命令模式如何匹配

**测试命令**：

```bash
# 这些命令应该共享相同的 "always" 模式
git checkout main        → 匹配 "git checkout *"
git checkout feature/x   → 匹配 "git checkout *"
git checkout -b dev      → 匹配 "git checkout *"

npm install              → 匹配 "npm install *"
npm install express      → 匹配 "npm install *"

# 这些命令有不同的模式
npm run dev              → 匹配 "npm run dev *"
npm run build            → 匹配 "npm run build *"
```

---

### 8.4 实践四：调试权限系统

**启用调试日志：**

```typescript
// 在 PermissionNext 命名空间中添加调试日志
const log = Log.create({ service: "permission" })

// 每次 evaluate 都会输出日志
log.info("evaluated", { permission, pattern, action: rule })
```

**查看日志：**

```bash
# 启动 OpenCode 并观察权限日志
bun dev 2>&1 | grep -i permission
```

---

## 九、常见问题

### 9.1 权限请求不弹出

**可能原因**：

| 检查项 | 说明 |
|--------|------|
| 规则已命中 allow | 检查规则配置，确保有 ask 动作 |
| 规则已命中 deny | 检查是否被规则阻止 |
| TUI 未连接 | 确认是 TUI 模式而非 CLI 模式 |

**排查步骤**：

```bash
# 检查规则配置
cat opencode.json | grep permission

# 查看日志
bun dev 2>&1 | grep -E "(permission|evaluate|asked)"
```

### 9.2 权限规则不生效

**可能原因**：

| 问题 | 解决方法 |
|------|----------|
| 规则格式错误 | 验证 JSON 格式 |
| 优先级问题 | 规则按顺序匹配，后出现的优先级更高 |
| wildcard 写法错误 | 使用 `*` 而非 `.*` |

**规则格式验证：**

```typescript
// 正确格式
"read": { "*.txt": "allow" }

// 错误格式
"read": { ".txt": "allow" }  // 缺少 *
```

### 9.3 "always" 规则不持久化

**当前限制**：

> 代码注释 (`next.ts:212-214`) 表明规则尚未持久化到磁盘：
> ```typescript
> // TODO: we don't save the permission ruleset to disk yet until there's
> // UI to manage it
> // await Storage.write(["permission", Instance.project.id], s.approved)
> ```

**临时解决方案**：在 `opencode.json` 中显式配置规则

### 9.4 多个 Agent 权限冲突

**问题**：不同 Agent 的权限配置可能冲突

**解决**：了解优先级顺序

```
Agent 默认规则 > 项目配置 > 用户配置 > 已批准规则
```

**建议**：
- 保持规则简单明确
- 避免在多个地方重复配置
- 使用注释说明规则意图

---

## 附录

### 附录一、关键文件路径速查

| 分类 | 文件 | 路径 | 说明 |
|------|------|------|------|
| **核心** | PermissionNext | `packages/opencode/src/permission/next.ts` | 权限系统核心 |
| | BashArity | `packages/opencode/src/permission/arity.ts` | 命令参数分组 |
| | 旧版 Permission | `packages/opencode/src/permission/index.ts` | 旧版权限系统 |
| **工具** | BashTool | `packages/opencode/src/tool/bash.ts:139-155` | 权限请求示例 |
| | EditTool | `packages/opencode/src/tool/edit.ts` | 编辑权限请求 |
| **UI** | 权限对话框 | `cli/cmd/tui/routes/session/permission.tsx` | TUI 权限界面 |
| | 事件同步 | `cli/cmd/tui/context/sync.tsx` | 事件总线同步 |
| **配置** | Agent 定义 | `packages/opencode/src/agent/agent.ts:43-57` | Agent 权限配置 |
| | Config 类型 | `packages/opencode/src/config/config.ts` | 配置类型定义 |

### 附录二、运行命令速查

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

### 附录三、相关资源

| 资源 | 链接 |
|------|------|
| OpenCode GitHub | <https://github.com/anomalyco/opencode> |
| OpenCode 文档 | <https://opencode.ai/docs> |
| 问题反馈 | <https://github.com/anomalyco/opencode/issues> |

---
