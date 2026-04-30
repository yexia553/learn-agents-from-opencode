# OpenCode Tool 系统学习教程

> 本教程旨在深入理解 OpenCode 的 Tool 系统，通过源码分析和实践演练，帮助读者掌握 Tool 的架构设计、实现原理和定制方法。

---

## 目录

| 章节 | 标题                 |
| ---- | -------------------- |
| 一   | 系统概述             |
| 二   | 核心架构分析         |
| 三   | 工具注册机制         |
| 四   | 核心工具详解         |
| 五   | 权限请求集成         |
| 六   | 输出截断处理         |
| 七   | Provider Schema 转换 |
| 八   | 实践演练             |
| 九   | 高级定制             |
| 十   | 常见问题             |

---

## 一、系统概述

### 1.1 什么是 Tool 系统

**Tool**（工具）是 OpenCode 中 Agent 与外部环境交互的桥梁。每个 Tool 都是一个封装了特定功能的可调用单元，Agent 通过调用 Tool 来执行文件操作、命令执行、代码搜索等任务。

> **核心概念**：在 OpenCode 中，Tool 是一个轻量级的函数式封装，定义了工具的元数据（ID、描述、参数模式）和执行逻辑。Tool 系统与权限系统紧密集成，每个工具调用都需要经过权限检查。

**Tool 系统的核心职责：**

| 职责         | 说明                                             |
| ------------ | ------------------------------------------------ |
| **功能封装** | 将文件系统操作、命令执行等功能封装为可调用的工具 |
| **参数验证** | 使用 Zod Schema 定义和验证工具参数               |
| **权限集成** | 在执行前请求必要的权限                           |
| **输出处理** | 对工具输出进行截断、格式化处理                   |
| **注册管理** | 管理所有可用工具的注册和发现                     |

### 1.2 Tool 的组成结构

每个 Tool 由以下 **四个核心部分** 组成：

| 部分            | 说明                        | 代码位置                       |
| --------------- | --------------------------- | ------------------------------ |
| **ID**          | 工具唯一标识符              | `Tool.Info.id`                 |
| **Description** | 工具功能描述（供 LLM 理解） | `Tool.Info.init().description` |
| **Parameters**  | 参数模式（Zod Schema）      | `Tool.Info.init().parameters`  |
| **Execute**     | 执行逻辑                    | `Tool.Info.init().execute`     |

**完整 Tool 定义示例** ([`/opencode/packages/opencode/src/tool/read.ts:16-152`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/read.ts#L16-L152))：

```typescript
export const ReadTool = Tool.define("read", {
  description: DESCRIPTION,
  parameters: z.object({
    filePath: z.string().describe("The path to the file to read"),
    offset: z.coerce.number().describe("...").optional(),
    limit: z.coerce.number().describe("...").optional(),
  }),
  async execute(params, ctx) {
    // 工具实现逻辑
    return {
      title: "...",
      output: "...",
      metadata: { ... },
    }
  },
})
```

### 1.3 内置工具列表

OpenCode 提供了以下 **14 个核心工具**：

| 工具         | 功能                 | 权限类型   |
| ------------ | -------------------- | ---------- |
| `read`       | 读取文件内容         | read       |
| `write`      | 写入文件（完整覆盖） | edit       |
| `edit`       | 编辑文件（局部修改） | edit       |
| `glob`       | 文件 glob 模式匹配   | glob       |
| `grep`       | 代码内容搜索         | grep       |
| `bash`       | 执行 Shell 命令      | bash       |
| `task`       | 调用子 Agent         | task       |
| `skill`      | 加载 Skill 技能      | skill      |
| `webfetch`   | 抓取网页内容         | webfetch   |
| `websearch`  | 网络搜索             | websearch  |
| `codesearch` | 代码库搜索           | codesearch |
| `todowrite`  | 写入待办事项         | todowrite  |
| `todoread`   | 读取待办事项         | todoread   |
| `list`       | 列出目录内容         | list       |

### 1.4 核心文件概览

要深入理解 Tool 系统，需要重点关注以下核心文件：

| 文件                                          | 说明                              |
| --------------------------------------------- | --------------------------------- |
| [`/opencode/packages/opencode/src/tool/tool.ts`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/tool.ts)          | Tool 接口定义和 `define` 函数     |
| [`/opencode/packages/opencode/src/tool/registry.ts`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/registry.ts)      | ToolRegistry 命名空间，工具注册表 |
| [`/opencode/packages/opencode/src/tool/*.ts`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/)             | 各工具实现（read、bash、edit 等） |
| [`/opencode/packages/opencode/src/tool/truncation.ts`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/truncation.ts)    | 输出截断处理逻辑                  |
| [`/opencode/packages/opencode/src/provider/transform.ts`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/provider/transform.ts) | Provider Schema 转换              |

---

## 二、核心架构分析

### 2.1 Tool 命名空间详解

**Tool** 命名空间是整个 Tool 系统的核心，位于 [`/opencode/packages/opencode/src/tool/tool.ts`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/tool.ts)。

**三个核心类型：**

| 类型               | 说明             |
| ------------------ | ---------------- |
| `Tool.Info`        | 工具定义接口     |
| `Tool.Context`     | 工具执行上下文   |
| `Tool.InitContext` | 工具初始化上下文 |

**文件导入依赖：**

```typescript
import z from "zod"
import type { MessageV2 } from "../session/message-v2"
import type { Agent } from "../agent/agent"
import type { PermissionNext } from "../permission/next"
import { Truncate } from "./truncation"

export namespace Tool {
  // 核心接口和函数定义
}
```

---

### 2.2 Context 接口：工具执行上下文

**Context** 是工具执行时接收的上下文对象，提供了与系统交互的能力。

**完整接口定义** ([`/opencode/packages/opencode/src/tool/tool.ts:16-25`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/tool.ts#L16-L25))：

```typescript
export type Context<M extends Metadata = Metadata> = {
  sessionID: string // 会话 ID
  messageID: string // 消息 ID
  agent: string // 当前 Agent 名称
  abort: AbortSignal // 中止信号
  callID?: string // 调用 ID（可选）
  extra?: { [key: string]: any } // 额外数据
  metadata(input: { title?: string; metadata?: M }): void // 更新元数据
  ask(input: Omit<PermissionNext.Request, "id" | "sessionID" | "tool">): Promise<void> // 请求权限
}
```

**Context 的核心能力：**

| 能力           | 方法             | 作用                                            |
| -------------- | ---------------- | ----------------------------------------------- |
| **元数据更新** | `ctx.metadata()` | 更新工具执行的元数据（输出摘要、诊断信息等）    |
| **权限请求**   | `ctx.ask()`      | 请求执行所需的权限                              |
| **中止控制**   | `ctx.abort`      | 监听中止信号，支持用户取消操作                  |
| **额外数据**   | `ctx.extra`      | 接收调用者传入的额外数据（如 `bypassCwdCheck`） |

**元数据更新示例** ([`/opencode/packages/opencode/src/tool/edit.ts:124-130`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/edit.ts#L124-L130))：

```typescript
ctx.metadata({
  metadata: {
    diff,
    filediff,
    diagnostics: {},
  },
})
```

**权限请求示例** ([`/opencode/packages/opencode/src/tool/read.ts:32-48`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/read.ts#L32-L48))：

```typescript
// 请求外部目录访问权限
await ctx.ask({
  permission: "external_directory",
  patterns: [parentDir],
  always: [parentDir + "/*"],
  metadata: { filepath, parentDir },
})

// 请求文件读取权限
await ctx.ask({
  permission: "read",
  patterns: [filepath],
  always: ["*"],
  metadata: {},
})
```

---

### 2.3 Info 接口：工具定义

**Info** 是工具的静态定义接口，描述了工具的元数据和初始化逻辑。

**完整接口定义** ([`/opencode/packages/opencode/src/tool/tool.ts:26-42`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/tool.ts#L26-L42))：

```typescript
export interface Info<Parameters extends z.ZodType = z.ZodType, M extends Metadata = Metadata> {
  id: string
  init: (ctx?: InitContext) => Promise<{
    description: string
    parameters: Parameters
    execute(
      args: z.infer<Parameters>,
      ctx: Context,
    ): Promise<{
      title: string
      metadata: M
      output: string
      attachments?: MessageV2.FilePart[]
    }>
    formatValidationError?(error: z.ZodError): string
  }>
}
```

**接口泛型参数：**

| 参数         | 说明            | 默认类型    |
| ------------ | --------------- | ----------- |
| `Parameters` | 参数 Zod Schema | `z.ZodType` |
| `M`          | 元数据类型      | `Metadata`  |

**类型推断工具：**

```typescript
export type InferParameters<T extends Info> = z.infer<P> // 提取参数类型
export type InferMetadata<T extends Info> = M // 提取元数据类型
```

---

### 2.4 define 函数：工具定义核心

**define** 是定义工具的核心函数，封装了参数验证和输出截断逻辑。

**完整实现** ([`/opencode/packages/opencode/src/tool/tool.ts:47-87`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/tool.ts#L47-L87))：

```typescript
export function define<Parameters extends z.ZodType, Result extends Metadata>(
  id: string,
  init: Info<Parameters, Result>["init"] | Awaited<ReturnType<Info<Parameters, Result>["init"]>>,
): Info<Parameters, Result> {
  return {
    id,
    init: async (initCtx) => {
      const toolInfo = init instanceof Function ? await init(initCtx) : init
      const execute = toolInfo.execute

      // 包装 execute 函数，添加参数验证和输出截断
      toolInfo.execute = async (args, ctx) => {
        try {
          toolInfo.parameters.parse(args)
        } catch (error) {
          if (error instanceof z.ZodError && toolInfo.formatValidationError) {
            throw new Error(toolInfo.formatValidationError(error), { cause: error })
          }
          throw new Error(
            `The ${id} tool was called with invalid arguments: ${error}.\nPlease rewrite the input so it satisfies the expected schema.`,
            { cause: error },
          )
        }
        const result = await execute(args, ctx)

        // 如果工具自己处理了截断，跳过自动截断
        if (result.metadata.truncated !== undefined) {
          return result
        }

        // 自动截断输出
        const truncated = await Truncate.output(result.output, {}, initCtx?.agent)
        return {
          ...result,
          output: truncated.content,
          metadata: {
            ...result.metadata,
            truncated: truncated.truncated,
            ...(truncated.truncated && { outputPath: truncated.outputPath }),
          },
        }
      }
      return toolInfo
    },
  }
}
```

**define 函数的三层封装：**

| 层级       | 功能                                                    |
| ---------- | ------------------------------------------------------- |
| **第一层** | 参数验证：调用 `parameters.parse(args)` 验证参数        |
| **第二层** | 错误格式化：调用 `formatValidationError` 自定义错误消息 |
| **第三层** | 输出截断：调用 `Truncate.output` 处理大输出             |

---

### 2.5 InitContext 接口：初始化上下文

**InitContext** 是工具初始化时接收的上下文，用于获取 Agent 信息。

**完整接口定义** ([`/opencode/packages/opencode/src/tool/tool.ts:12-14`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/tool.ts#L12-L14))：

```typescript
export interface InitContext {
  agent?: Agent.Info // 当前 Agent 信息（可选）
}
```

**使用场景：** 工具可以根据当前 Agent 调整行为，如 TaskTool 根据调用者权限过滤可用的子 Agent。

**使用示例** ([`/opencode/packages/opencode/src/tool/task.ts:27-30`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/task.ts#L27-L30))：

```typescript
const caller = ctx?.agent
const accessibleAgents = caller
  ? agents.filter((a) => PermissionNext.evaluate("task", a.name, caller.permission).action !== "deny")
  : agents
```

---

## 三、工具注册机制

### 3.1 ToolRegistry 命名空间概述

**ToolRegistry** 负责管理所有工具的注册、发现和查询。

**核心职责：**

| 职责         | 说明                                    |
| ------------ | --------------------------------------- |
| **工具注册** | 注册自定义工具                          |
| **工具发现** | 发现和加载自定义工具（从 `tool/` 目录） |
| **工具过滤** | 根据 Provider 和 Agent 过滤可用工具     |
| **插件集成** | 从 Plugin 加载工具定义                  |

---

### 3.2 工具注册流程

**工具列表来源** ([`/opencode/packages/opencode/src/tool/registry.ts:89-112`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/registry.ts#L89-L112))：

```typescript
async function all(): Promise<Tool.Info[]> {
  const custom = await state().then((x) => x.custom)
  const config = await Config.get()

  return [
    InvalidTool,
    BashTool,
    ReadTool,
    GlobTool,
    GrepTool,
    EditTool,
    WriteTool,
    TaskTool,
    WebFetchTool,
    TodoWriteTool,
    TodoReadTool,
    WebSearchTool,
    CodeSearchTool,
    SkillTool,
    ...(Flag.OPENCODE_EXPERIMENTAL_LSP_TOOL ? [LspTool] : []),
    ...(config.experimental?.batch_tool === true ? [BatchTool] : []),
    ...custom,
  ]
}
```

**工具来源优先级：**

| 优先级 | 来源       | 说明                                |
| ------ | ---------- | ----------------------------------- |
| 1      | 内置工具   | 核心功能工具（read、bash、edit 等） |
| 2      | 实验性工具 | 根据 Flag 启用的工具（lsp、batch）  |
| 3      | 自定义工具 | 从 `tool/` 目录加载的工具           |
| 4      | 插件工具   | 从 Plugin 加载的工具                |

---

### 3.3 自定义工具加载

**自定义工具目录扫描** ([`/opencode/packages/opencode/src/tool/registry.ts:31-48`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/registry.ts#L31-L48))：

```typescript
export const state = Instance.state(async () => {
  const custom = [] as Tool.Info[]
  const glob = new Bun.Glob("tool/*.{js,ts}")

  for (const dir of await Config.directories()) {
    for await (const match of glob.scan({
      cwd: dir,
      absolute: true,
      followSymlinks: true,
      dot: true,
    })) {
      const namespace = path.basename(match, path.extname(match))
      const mod = await import(match)
      for (const [id, def] of Object.entries<ToolDefinition>(mod)) {
        custom.push(fromPlugin(id === "default" ? namespace : `${namespace}_${id}`, def))
      }
    }
  }

  const plugins = await Plugin.list()
  for (const plugin of plugins) {
    for (const [id, def] of Object.entries(plugin.tool ?? {})) {
      custom.push(fromPlugin(id, def))
    }
  }

  return { custom }
})
```

**加载规则：**

| 规则         | 说明                                           |
| ------------ | ---------------------------------------------- |
| **扫描目录** | 从 Config 配置的目录扫描 `tool/*.{js,ts}` 文件 |
| **命名空间** | 文件名作为命名空间，id 作为工具名              |
| **插件支持** | 从 Plugin 定义中加载工具                       |
| **动态导入** | 使用 `import()` 动态加载模块                   |

---

### 3.4 工具查询接口

**获取所有工具 ID** ([`/opencode/packages/opencode/src/tool/registry.ts:114-116`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/registry.ts#L114-L116))：

```typescript
export async function ids() {
  return all().then((x) => x.map((t) => t.id))
}
```

**获取可用工具** ([`/opencode/packages/opencode/src/tool/registry.ts:118-138`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/registry.ts#L118-L138))：

```typescript
export async function tools(providerID: string, agent?: Agent.Info) {
  const tools = await all()
  const result = await Promise.all(
    tools
      .filter((t) => {
        // 根据 Provider 过滤工具
        if (t.id === "codesearch" || t.id === "websearch") {
          return providerID === "opencode" || Flag.OPENCODE_ENABLE_EXA
        }
        return true
      })
      .map(async (t) => {
        using _ = log.time(t.id)
        return {
          id: t.id,
          ...(await t.init({ agent })),  // 初始化工具
        },
      }),
  )
  return result
}
```

**工具过滤规则：**

| 工具                       | 过滤条件                                      |
| -------------------------- | --------------------------------------------- |
| `codesearch` / `websearch` | 仅对 opencode Provider 或启用 EXA Flag 时可用 |

---

## 四、核心工具详解

### 4.1 ReadTool：文件读取

**功能**：安全读取文件内容，支持大文件截断和二进制文件检测。

**定义位置**：[`/opencode/packages/opencode/src/tool/read.ts`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/read.ts)

**参数模式**：

```typescript
parameters: z.object({
  filePath: z.string().describe("The path to the file to read"),
  offset: z.coerce.number().describe("The line number to start reading from (0-based)").optional(),
  limit: z.coerce.number().describe("The number of lines to read (defaults to 2000)").optional(),
})
```

**执行流程**：

```
1. 路径规范化 → 相对路径转换为绝对路径
2. 权限检查 → 请求 external_directory 和 read 权限
3. 文件存在检查 → 不存在则提供相似文件名建议
4. 文件类型检测 → 图片/PDF 返回 base64，抛出二进制文件错误
5. 读取内容 → 限制 2000 行，50KB 字节
6. 输出格式化 → 添加行号前缀，标注截断信息
7. LSP 预热 → 调用 LSP.touchFile 预热语言服务器
```

**特殊处理：**

| 处理类型       | 说明                             |
| -------------- | -------------------------------- |
| **图片/PDF**   | 返回 base64 编码的附件           |
| **二进制文件** | 抛出错误，禁止读取               |
| **大文件**     | 截断输出，保存完整内容到临时文件 |
| **文件不存在** | 提供相似文件名建议               |

**输出示例**：

```
00001| import z from "zod"
00002| import * as fs from "fs"
00003| import * as path from "path"
...
00010|
00011| (End of file - total 210 lines)
```

---

### 4.2 EditTool：文件编辑

**功能**：使用 diff 语义安全编辑文件，支持多种匹配策略。

**定义位置**：[`/opencode/packages/opencode/src/tool/edit.ts`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/edit.ts)

**参数模式**：

```typescript
parameters: z.object({
  filePath: z.string().describe("The absolute path to the file to modify"),
  oldString: z.string().describe("The text to replace"),
  newString: z.string().describe("The text to replace it with (must be different from oldString)"),
  replaceAll: z.boolean().optional().describe("Replace all occurrences of oldString"),
})
```

**匹配策略优先级** ([`/opencode/packages/opencode/src/tool/edit.ts:618-655`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/edit.ts#L618-L655))：

| 策略                           | 说明                | 阈值      |
| ------------------------------ | ------------------- | --------- |
| `SimpleReplacer`               | 精确匹配            | 精确      |
| `LineTrimmedReplacer`          | 行首尾空白忽略匹配  | 精确      |
| `BlockAnchorReplacer`          | 锚定行匹配 + 相似度 | 0.0 / 0.3 |
| `WhitespaceNormalizedReplacer` | 空白标准化匹配      | 精确      |
| `IndentationFlexibleReplacer`  | 缩进灵活匹配        | 精确      |
| `EscapeNormalizedReplacer`     | 转义字符标准化匹配  | 精确      |
| `TrimmedBoundaryReplacer`      | 边界截断匹配        | 精确      |
| `ContextAwareReplacer`         | 上下文感知匹配      | 50%       |
| `MultiOccurrenceReplacer`      | 多occurrence匹配    | 精确      |

**执行流程**：

```
1. 参数验证 → oldString ≠ newString
2. 路径规范化 → 相对路径转换为绝对路径
3. 权限检查 → 请求 external_directory 和 edit 权限
4. 文件锁定 → 使用 FileTime.withLock 防止并发写入
5. 匹配查找 → 按优先级尝试各匹配策略
6. 内容替换 → 替换匹配的文本
7. 文件写入 → 使用 Bun.write 写入
8. 事件发布 → 发布 File.Event.Edited 事件
9. 诊断更新 → 调用 LSP.touchFile 更新诊断
```

---

### 4.3 WriteTool：文件写入

**功能**：完整覆盖写入文件，保留文件历史差异。

**定义位置**：[`/opencode/packages/opencode/src/tool/write.ts`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/write.ts)

**参数模式**：

```typescript
parameters: z.object({
  content: z.string().describe("The content to write to the file"),
  filePath: z.string().describe("The absolute path to the file to write"),
})
```

**执行流程**：

```
1. 路径规范化 → 相对路径转换为绝对路径
2. 权限检查 → 请求 edit 权限
3. 文件存在检查 → 存在则读取旧内容
4. 差异计算 → 使用 createTwoFilesPatch 计算差异
5. 文件写入 → 使用 Bun.write 写入
6. 事件发布 → 发布 File.Event.Edited 事件
7. 诊断更新 → 调用 LSP.touchFile 更新诊断
```

---

### 4.4 BashTool：命令执行

**功能**：安全执行 Shell 命令，支持命令解析和权限控制。

**定义位置**：[`/opencode/packages/opencode/src/tool/bash.ts`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/bash.ts)

**参数模式**：

```typescript
parameters: z.object({
  command: z.string().describe("The command to execute"),
  timeout: z.number().describe("Optional timeout in milliseconds").optional(),
  workdir: z.string().describe("The working directory...").optional(),
  description: z.string().describe("Clear, concise description..."),
})
```

**命令解析** ([`/opencode/packages/opencode/src/tool/bash.ts:92-137`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/bash.ts#L92-L137))：

```typescript
// 使用 tree-sitter 解析 bash 命令
const tree = await parser().then((p) => p.parse(params.command))

for (const node of tree.rootNode.descendantsOfType("command")) {
  // 提取命令名和参数
  const command = node.children
    .filter((child) => ["command_name", "word", "string", "raw_string", "concatenation"].includes(child.type))
    .map((child) => child.text)

  // 识别文件操作命令
  if (["cd", "rm", "cp", "mv", "mkdir", "touch", "chmod", "chown"].includes(command[0])) {
    for (const arg of command.slice(1)) {
      if (arg.startsWith("-")) continue
      // 解析真实路径
      const resolved = await $`realpath ${arg}`.cwd(cwd).quiet().nothrow().text()
      if (resolved) {
        if (!Filesystem.contains(Instance.directory, normalized)) {
          directories.add(normalized)
        }
      }
    }
  }
}
```

**权限请求** ([`/opencode/packages/opencode/src/tool/bash.ts:139-155`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/bash.ts#L139-L155))：

```typescript
if (directories.size > 0) {
  await ctx.ask({
    permission: "external_directory",
    patterns: Array.from(directories),
    always: Array.from(directories).map((x) => path.dirname(x) + "*"),
    metadata: {},
  })
}

if (patterns.size > 0) {
  await ctx.ask({
    permission: "bash",
    patterns: Array.from(patterns),
    always: Array.from(always),
    metadata: {},
  })
}
```

**超时处理**：

```typescript
const timeout = params.timeout ?? DEFAULT_TIMEOUT // 默认 2 分钟
const timeoutTimer = setTimeout(() => {
  timedOut = true
  void kill()
}, timeout + 100)
```

---

### 4.5 TaskTool：子 Agent 调用

**功能**：调用其他 Agent 执行任务，实现 Agent 组合。

**定义位置**：[`/opencode/packages/opencode/src/tool/task.ts`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/task.ts)

**参数模式**：

```typescript
parameters: z.object({
  description: z.string().describe("A short (3-5 words) description of the task"),
  prompt: z.string().describe("The task for the agent to perform"),
  subagent_type: z.string().describe("The type of specialized agent to use"),
  session_id: z.string().optional().describe("Existing Task session to continue"),
  command: z.string().optional().describe("The command that triggered this task"),
})
```

**Agent 权限过滤** ([`/opencode/packages/opencode/src/tool/task.ts:27-30`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/task.ts#L27-L30))：

```typescript
const caller = ctx?.agent
const accessibleAgents = caller
  ? agents.filter((a) => PermissionNext.evaluate("task", a.name, caller.permission).action !== "deny")
  : agents
```

**子会话创建** ([`/opencode/packages/opencode/src/tool/task.ts:59-91`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/task.ts#L59-L91))：

```typescript
return await Session.create({
  parentID: ctx.sessionID,
  title: params.description + ` (@${agent.name} subagent)`,
  permission: [
    { permission: "todowrite", pattern: "*", action: "deny" },
    { permission: "todoread", pattern: "*", action: "deny" },
    { permission: "task", pattern: "*", action: "deny" },
    ...(config.experimental?.primary_tools?.map((t) => ({
      pattern: "*",
      action: "allow" as const,
      permission: t,
    })) ?? []),
  ],
})
```

---

### 4.6 SkillTool：技能加载

**功能**：加载和执行 Skill，提供专业领域的指导。

**定义位置**：[`/opencode/packages/opencode/src/tool/skill.ts`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/skill.ts)

**参数模式**：

```typescript
parameters: z.object({
  name: z.string().describe("The skill identifier..."),
})
```

**Skill 权限过滤** ([`/opencode/packages/opencode/src/tool/skill.ts:16-22`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/skill.ts#L16-L22))：

```typescript
const agent = ctx?.agent
const accessibleSkills = agent
  ? skills.filter((skill) => {
      const rule = PermissionNext.evaluate("skill", skill.name, agent.permission)
      return rule.action !== "deny"
    })
  : skills
```

---

### 4.7 GrepTool：代码搜索

**功能**：使用 ripgrep 进行代码内容搜索。

**定义位置**：[`/opencode/packages/opencode/src/tool/grep.ts`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/grep.ts)

**参数模式**：

```typescript
parameters: z.object({
  pattern: z.string().describe("The regex pattern to search for..."),
  path: z.string().optional().describe("The directory to search in..."),
  include: z.string().optional().describe("File pattern to include..."),
})
```

**搜索流程**：

```
1. 权限检查 → 请求 grep 权限
2. 构建命令 → 构建 ripgrep 命令参数
3. 执行搜索 → 使用 Bun.spawn 执行 ripgrep
4. 结果解析 → 解析 file|lineNum|lineText 格式
5. 排序输出 → 按修改时间排序
6. 截断结果 → 限制 100 条结果
```

---

### 4.8 GlobTool：文件匹配

**功能**：使用 glob 模式匹配文件。

**定义位置**：[`/opencode/packages/opencode/src/tool/glob.ts`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/glob.ts)

**参数模式**：

```typescript
parameters: z.object({
  pattern: z.string().describe("The glob pattern to match files against"),
  path: z.string().optional().describe("The directory to search in..."),
})
```

---

## 五、权限请求集成

### 5.1 工具权限请求模式

每个工具在执行敏感操作前都需要调用 `ctx.ask()` 请求权限。

**权限请求结构**：

```typescript
await ctx.ask({
  permission: "read",           // 权限类型
  patterns: [filepath],         // 匹配模式
  always: ["*"],                // "始终允许" 模式
  metadata: { ... },            // 元数据
})
```

**权限类型与工具映射**：

| 权限类型             | 工具                | 说明           |
| -------------------- | ------------------- | -------------- |
| `read`               | ReadTool            | 文件读取       |
| `edit`               | EditTool, WriteTool | 文件编辑       |
| `glob`               | GlobTool            | 文件匹配       |
| `grep`               | GrepTool            | 代码搜索       |
| `bash`               | BashTool            | 命令执行       |
| `task`               | TaskTool            | Agent 调用     |
| `skill`              | SkillTool           | Skill 加载     |
| `external_directory` | 所有工具            | 访问项目外目录 |
| `webfetch`           | WebFetchTool        | 网页抓取       |
| `websearch`          | WebSearchTool       | 网络搜索       |
| `codesearch`         | CodeSearchTool      | 代码搜索       |

---

### 5.2 patterns 与 always 的区别

**patterns**：当前请求的模式

- 用于精确匹配当前操作
- 用户需要逐个确认或拒绝

**always**："始终允许"的模式

- 用于设置自动批准的规则
- 匹配该模式的后续操作无需确认

**示例** ([`/opencode/packages/opencode/src/tool/read.ts:32-48`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/read.ts#L32-L48))：

```typescript
// 读取项目内文件
await ctx.ask({
  permission: "read",
  patterns: [filepath],
  always: ["*"], // 始终允许读取任何文件
})

// 读取项目外文件
await ctx.ask({
  permission: "external_directory",
  patterns: [parentDir], // 请求访问特定目录
  always: [parentDir + "/*"], // 始终允许访问该目录下的文件
})
```

---

### 5.3 权限请求与执行分离

OpenCode 采用权限请求与执行分离的架构：

```
┌─────────────────────────────────────────────────────────────────┐
│                    权限请求流程                                   │
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
│           TUI 接收事件，显示权限对话框                            │
│                   │                                             │
│         ┌────────┼────────┐                                     │
│         ▼        ▼        ▼                                     │
│       Once   Always   Reject                                    │
│         │        │        │                                     │
│         └────────┴────────┘                                     │
│                   │                                             │
│                   ▼                                             │
│           Promise resolve/reject                                 │
│                   │                                             │
│                   ▼                                             │
│           继续执行 Tool.execute() 剩余逻辑                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### 5.4 外部目录权限请求

**外部目录判断** ([`/opencode/packages/opencode/src/tool/read.ts:30-41`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/read.ts#L30-L41))：

```typescript
if (!ctx.extra?.["bypassCwdCheck"] && !Filesystem.contains(Instance.directory, filepath)) {
  const parentDir = path.dirname(filepath)
  await ctx.ask({
    permission: "external_directory",
    patterns: [parentDir],
    always: [parentDir + "/*"],
    metadata: { filepath, parentDir },
  })
}
```

**判断逻辑**：使用 `Filesystem.contains()` 检查文件是否在项目目录内。

---

## 六、输出截断处理

### 6.1 Truncate 命名空间概述

**Truncate** 负责处理工具输出的大文件截断，避免超出上下文限制。

**核心常量** ([`/opencode/packages/opencode/src/tool/truncation.ts:9-13`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/truncation.ts#L9-L13))：

```typescript
export const MAX_LINES = 2000
export const MAX_BYTES = 50 * 1024 // 50KB
export const DIR = path.join(Global.Path.data, "tool-output")
const RETENTION_MS = 7 * 24 * 60 * 60 * 1000 // 7 天
```

---

### 6.2 输出截断逻辑

**完整实现** ([`/opencode/packages/opencode/src/tool/truncation.ts:41-97`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/truncation.ts#L41-L97))：

```typescript
export async function output(text: string, options: Options = {}, agent?: Agent.Info): Promise<Result> {
  const maxLines = options.maxLines ?? MAX_LINES
  const maxBytes = options.maxBytes ?? MAX_BYTES
  const direction = options.direction ?? "head"
  const lines = text.split("\n")
  const totalBytes = Buffer.byteLength(text, "utf-8")

  // 未超出限制，直接返回
  if (lines.length <= maxLines && totalBytes <= maxBytes) {
    return { content: text, truncated: false }
  }

  // 截断处理
  const out: string[] = []
  let bytes = 0
  let hitBytes = false

  if (direction === "head") {
    for (i = 0; i < lines.length && i < maxLines; i++) {
      const size = Buffer.byteLength(lines[i], "utf-8") + (i > 0 ? 1 : 0)
      if (bytes + size > maxBytes) {
        hitBytes = true
        break
      }
      out.push(lines[i])
      bytes += size
    }
  } else {
    // tail 模式：从尾部开始
    for (i = lines.length - 1; i >= 0 && out.length < maxLines; i--) {
      const size = Buffer.byteLength(lines[i], "utf-8") + (out.length > 0 ? 1 : 0)
      if (bytes + size > maxBytes) {
        hitBytes = true
        break
      }
      out.unshift(lines[i])
      bytes += size
    }
  }

  // 保存完整内容到临时文件
  await init()
  const id = Identifier.ascending("tool")
  const filepath = path.join(DIR, id)
  await Bun.write(Bun.file(filepath), text)

  // 生成提示消息
  const hint = hasTaskTool(agent)
    ? `The tool call succeeded but the output was truncated. Full output saved to: ${filepath}\nUse the Task tool to have a subagent process this file with Grep and Read (with offset/limit).`
    : `The tool call succeeded but the output was truncated. Full output saved to: ${filepath}\nUse Grep to search the full content or Read with offset/limit.`

  return { content: message, truncated: true, outputPath: filepath }
}
```

**截断策略：**

| 策略          | 说明                       |
| ------------- | -------------------------- |
| **head 模式** | 从头部截断，保留前面的内容 |
| **tail 模式** | 从尾部截断，保留后面的内容 |
| **字节限制**  | 50KB 字节限制              |
| **行数限制**  | 2000 行限制                |

---

### 6.3 临时文件管理

**临时文件目录**：`~/Library/Application Support/opencode/tool-output/`

**清理机制** ([`/opencode/packages/opencode/src/tool/truncation.ts:23-31`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/truncation.ts#L23-L31))：

```typescript
export async function cleanup() {
  const cutoff = Identifier.timestamp(Identifier.create("tool", false, Date.now() - RETENTION_MS))
  const glob = new Bun.Glob("tool_*")
  const entries = await Array.fromAsync(glob.scan({ cwd: DIR, onlyFiles: true })).catch(() => [])
  for (const entry of entries) {
    if (Identifier.timestamp(entry) >= cutoff) continue
    await fs.unlink(path.join(DIR, entry)).catch(() => {})
  }
}
```

**保留策略**：保留 7 天内的临时文件。

---

### 6.4 TaskTool 特殊处理

**Agent 权限感知** ([`/opencode/packages/opencode/src/tool/truncation.ts:35-39`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/truncation.ts#L35-L39))：

```typescript
function hasTaskTool(agent?: Agent.Info): boolean {
  if (!agent?.permission) return false
  const rule = PermissionNext.evaluate("task", "*", agent.permission)
  return rule.action !== "deny"
}
```

**处理建议差异：**

| 场景             | 建议                              |
| ---------------- | --------------------------------- |
| 有 TaskTool 权限 | 建议使用 Task 工具让子 Agent 处理 |
| 无 TaskTool 权限 | 建议使用 Grep/Read 工具自行处理   |

---

## 七、Provider Schema 转换

### 7.1 工具 Schema 转换概述

OpenCode 使用统一的工具定义格式，需要转换为各 Provider 兼容的 Schema 格式。

**转换入口**：[`/opencode/packages/opencode/src/provider/transform.ts`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/provider/transform.ts)

---

### 7.2 Schema 转换函数

**主转换函数** ([`/opencode/packages/opencode/src/provider/transform.ts:576-638`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/provider/transform.ts#L576-L638))：

```typescript
export function schema(model: Provider.Model, schema: JSONSchema.BaseSchema) {
  // Google/Gemini 特殊处理：整数枚举转字符串枚举
  if (model.providerID === "google" || model.api.id.includes("gemini")) {
    const sanitizeGemini = (obj: any): any => {
      if (obj === null || typeof obj !== "object") {
        return obj
      }

      if (Array.isArray(obj)) {
        return obj.map(sanitizeGemini)
      }

      const result: any = {}
      for (const [key, value] of Object.entries(obj)) {
        if (key === "enum" && Array.isArray(value)) {
          result[key] = value.map((v) => String(v))
          if (result.type === "integer" || result.type === "number") {
            result.type = "string"
          }
        } else if (typeof value === "object" && value !== null) {
          result[key] = sanitizeGemini(value)
        } else {
          result[key] = value
        }
      }

      // 过滤 required 数组
      if (result.type === "object" && result.properties && Array.isArray(result.required)) {
        result.required = result.required.filter((field: any) => field in result.properties)
      }

      if (result.type === "array" && result.items == null) {
        result.items = {}
      }

      return result
    }

    schema = sanitizeGemini(schema)
  }

  return schema
}
```

**Provider 特定转换：**

| Provider      | 转换类型                 |
| ------------- | ------------------------ |
| Google/Gemini | 整数枚举转字符串枚举     |
| OpenAI/Azure  | 可选字段添加 `null` 类型 |
| Anthropic     | 保持原始 Schema          |

---

### 7.3 Zod 到 JSON Schema 转换

**转换来源**：Zod Schema 转换为 JSON Schema。

**示例转换**：

```typescript
// Zod 定义
z.object({
  filePath: z.string(),
  offset: z.coerce.number().optional(),
  limit: z.coerce.number().optional(),
})

// 转换为 JSON Schema
{
  type: "object",
  properties: {
    filePath: { type: "string", description: "The path to the file to read" },
    offset: { type: "number", description: "The line number to start reading from (0-based)" },
    limit: { type: "number", description: "The number of lines to read" },
  },
  required: ["filePath"]
}
```

---

### 7.4 消息格式标准化

**消息标准化** ([`/opencode/packages/opencode/src/provider/transform.ts:19-139`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/provider/transform.ts#L19-L139))：

```typescript
function normalizeMessages(msgs: ModelMessage[], model: Provider.Model): ModelMessage[] {
  // Anthropic：过滤空消息
  if (model.api.npm === "@ai-sdk/anthropic") {
    msgs = msgs
      .map((msg) => {
        if (typeof msg.content === "string") {
          if (msg.content === "") return undefined
          return msg
        }
        if (!Array.isArray(msg.content)) return msg
        const filtered = msg.content.filter((part) => {
          if (part.type === "text" || part.type === "reasoning") {
            return part.text !== ""
          }
          return true
        })
        if (filtered.length === 0) return undefined
        return { ...msg, content: filtered }
      })
      .filter((msg): msg is ModelMessage => msg !== undefined && msg.content !== "")
  }

  // Claude：规范化 tool call ID
  if (model.api.id.includes("claude")) {
    msgs = msgs.map((msg) => {
      if ((msg.role === "assistant" || msg.role === "tool") && Array.isArray(msg.content)) {
        msg.content = msg.content.map((part) => {
          if ((part.type === "tool-call" || part.type === "tool-result") && "toolCallId" in part) {
            return {
              ...part,
              toolCallId: part.toolCallId.replace(/[^a-zA-Z0-9_-]/g, "_"),
            }
          }
          return part
        })
      }
      return msg
    })
  }

  // Mistral：9 字符 ID 规范化
  if (model.providerID === "mistral" || model.api.id.toLowerCase().includes("mistral")) {
    // ...
  }

  return msgs
}
```

---

## 八、实践演练

### 8.1 实践一：分析内置工具实现

**目标**：通过阅读源码深入理解工具实现模式。

**步骤一：阅读 ReadTool 实现**

```bash
cat packages/opencode/src/tool/read.ts
```

**关键观察点：**

| 观察点     | 说明                  |
| ---------- | --------------------- |
| 参数定义   | 如何使用 Zod 定义参数 |
| 权限请求   | 何时请求什么权限      |
| 错误处理   | 如何处理各种错误情况  |
| 输出格式化 | 如何格式化输出内容    |

**步骤二：阅读 EditTool 实现**

```bash
cat packages/opencode/src/tool/edit.ts
```

**关键观察点：**

| 观察点   | 说明                 |
| -------- | -------------------- |
| 匹配策略 | 8 种匹配策略的优先级 |
| 差异计算 | 如何计算和应用差异   |
| 并发安全 | 如何使用文件锁       |

---

### 8.2 实践二：创建自定义工具

**目标**：创建一个简单的自定义工具，用于生成 UUID。

**步骤一：创建工具文件**

```typescript
// uuid-tool.ts
import z from "zod"
import { Tool } from "./tool"

export const UuidTool = Tool.define("uuid", {
  description: "Generate a UUID v4 string",
  parameters: z.object({
    count: z.number().describe("Number of UUIDs to generate").optional().default(1),
  }),
  async execute(params) {
    const count = params.count ?? 1
    const uuids = []
    for (let i = 0; i < count; i++) {
      uuids.push(crypto.randomUUID())
    }
    return {
      title: `Generated ${count} UUID(s)`,
      output: uuids.join("\n"),
      metadata: { count },
    }
  },
})
```

**步骤二：放置到工具目录**

将文件放到项目的 `tool/` 目录下。

**步骤三：测试使用**

```
请生成 3 个 UUID
```

---

### 8.3 实践三：工具调试技巧

**方法一：检查工具注册**

```typescript
// 在 registry.ts 中添加日志
export async function ids() {
  const ids = await all().then((x) => x.map((t) => t.id))
  console.log("Registered tools:", ids)
  return ids
}
```

**方法二：检查工具初始化**

```typescript
// 在 tools() 函数中添加日志
.map(async (t) => {
  console.log(`Initializing tool: ${t.id}`)
  using _ = log.time(t.id)
  return { ... }
})
```

**方法三：检查参数验证**

```typescript
// 在 define 函数中添加日志
toolInfo.parameters.parse(args)
console.log("Tool", id, "received valid args:", args)
```

---

### 8.4 实践四：理解权限请求流程

**目标**：追踪一次权限请求的完整流程。

**关键断点位置：**

| 文件                 | 行号 | 说明         |
| -------------------- | ---- | ------------ |
| `tool.ts`            | 56   | 参数验证后   |
| `tool.ts`            | 68   | 工具执行前   |
| `permission/next.ts` | ~100 | 权限评估入口 |
| `permission/next.ts` | ~150 | 规则匹配     |

**追踪步骤：**

1. 在 `ctx.ask()` 调用处设置断点
2. 观察 `permission` 和 `patterns` 参数
3. 检查权限评估结果
4. 观察 TUI 权限对话框的显示

---

### 8.5 实践五：测试输出截断

**目标**：验证输出截断机制。

**测试方法**：

```bash
# 生成大输出
bash -c "for i in \$(seq 1 10000); do echo \$i; done"
```

**观察点：**

| 观察点   | 说明                 |
| -------- | -------------------- |
| 截断位置 | 在哪一行/字节处截断  |
| 提示消息 | 是否显示临时文件路径 |
| 临时文件 | 是否正确保存完整内容 |

---

## 九、高级定制

### 9.1 自定义工具加载器

**从插件加载工具** ([`/opencode/packages/opencode/src/tool/registry.ts:50-56`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/registry.ts#L50-L56))：

```typescript
const plugins = await Plugin.list()
for (const plugin of plugins) {
  for (const [id, def] of Object.entries(plugin.tool ?? {})) {
    custom.push(fromPlugin(id, def))
  }
}
```

**插件工具定义格式**：

```typescript
// plugin.ts
export const myTool = {
  args: {
    input: z.string(),
  },
  description: "My custom tool",
  execute: async (args, ctx) => {
    return "result"
  },
}
```

---

### 9.2 工具执行上下文扩展

**添加额外数据**：

```typescript
// 调用工具时传入额外数据
await tool.execute(args, {
  ...ctx,
  extra: {
    bypassCwdCheck: true, // 绕过工作目录检查
    customData: "value",
  },
})
```

**在工具中访问**：

```typescript
const bypassCwdCheck = ctx.extra?.["bypassCwdCheck"]
```

---

### 9.3 工具输出附件

**支持附件类型** ([`/opencode/packages/opencode/src/tool/read.ts:83-93`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/read.ts#L83-L93))：

```typescript
return {
  title,
  output: msg,
  metadata: {
    preview: msg,
    truncated: false,
  },
  attachments: [
    {
      id: Identifier.ascending("part"),
      sessionID: ctx.sessionID,
      messageID: ctx.messageID,
      type: "file",
      mime,
      url: `data:${mime};base64,...`,
    },
  ],
}
```

**附件类型**：

| 类型   | 说明                     |
| ------ | ------------------------ |
| `file` | 文件附件（图片、PDF 等） |
| 自定义 | 扩展支持其他类型         |

---

### 9.4 实验性工具启用

**通过 Flag 启用** ([`/opencode/packages/opencode/src/tool/registry.ts:108`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/registry.ts#L108))：

```typescript
...(Flag.OPENCODE_EXPERIMENTAL_LSP_TOOL ? [LspTool] : []),
```

**通过配置启用** ([`/opencode/packages/opencode/src/tool/registry.ts:109`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/registry.ts#L109))：

```typescript
...(config.experimental?.batch_tool === true ? [BatchTool] : []),
```

---

### 9.5 工具与 LSP 集成

**LSP 预热** ([`/opencode/packages/opencode/src/tool/read.ts:140`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/read.ts#L140))：

```typescript
// 预热 LSP
LSP.touchFile(filepath, false)
```

**诊断获取** ([`/opencode/packages/opencode/src/tool/edit.ts:133-143`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/edit.ts#L133-L143))：

```typescript
const diagnostics = await LSP.diagnostics()
const normalizedFilePath = Filesystem.normalizePath(filePath)
const issues = diagnostics[normalizedFilePath] ?? []
const errors = issues.filter((item) => item.severity === 1)
```

---

## 十、常见问题

### Q1：工具参数验证失败怎么办？

**问题表现**：

```
The read tool was called with invalid arguments: ...
```

**解决方案**：

1. 检查参数类型是否正确（字符串 vs 数字）
2. 检查必需参数是否提供
3. 检查可选参数格式

**调试方法**：

```typescript
// 在 define 函数中添加参数日志
console.log("Received args:", args)
console.log("Expected schema:", toolInfo.parameters)
```

---

### Q2：权限请求被拒绝怎么办？

**问题表现**：工具执行抛出 `DeniedError`

**解决方案**：

1. 检查 Agent 权限配置
2. 调整 patterns 模式
3. 使用 "Always" 选项避免重复请求

**检查权限配置**：

```typescript
// 查看 Agent 权限
const agent = await Agent.get("build")
console.log(agent.permission)
```

---

### Q3：输出被截断怎么办？

**问题表现**：工具输出显示截断提示和临时文件路径

**解决方案**：

1. 使用 `Read` 工具读取临时文件
2. 使用 `Grep` 搜索特定内容
3. 使用 `offset/limit` 参数读取部分内容

**示例**：

```
# 读取临时文件的特定部分
Read { filePath: "/path/to/tool_xxx", offset: 100, limit: 50 }
```

---

### Q4：自定义工具不被加载怎么办？

**问题表现**：工具 ID 不在可用工具列表中

**调试步骤**：

1. 检查文件位置是否正确
2. 检查文件名格式
3. 检查模块导出格式
4. 查看注册日志

**检查方法**：

```typescript
// 在 registry.ts 中添加调试日志
console.log("Scanning directory:", dir)
console.log("Found tool file:", match)
```

---

### Q5：Bash 工具超时怎么办？

**问题表现**：命令执行超时被终止

**解决方案**：

1. 增加 `timeout` 参数
2. 将长时间任务分解为多个步骤
3. 使用后台执行（`&`）

**示例**：

```typescript
Bash {
  command: "npm install",
  timeout: 300000,  // 5 分钟
  description: "Install npm dependencies",
}
```

---

### Q6：工具间如何共享状态？

**问题表现**：多个工具调用需要共享数据

**解决方案**：

1. 使用 `ctx.extra` 传递数据（同一消息内）
2. 使用 `Session` 存储状态（跨消息）
3. 使用 `File` 写入临时文件

**示例**（使用 extra）：

```typescript
// Tool A
ctx.extra = { intermediateResult: "data" }

// Tool B
const data = ctx.extra?.["intermediateResult"]
```

---

## 参考资料

### 核心文件路径

| 文件                                          | 说明          |
| --------------------------------------------- | ------------- |
| [`/opencode/packages/opencode/src/tool/tool.ts`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/tool.ts)          | Tool 接口定义 |
| [`/opencode/packages/opencode/src/tool/registry.ts`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/registry.ts)      | 工具注册表    |
| [`/opencode/packages/opencode/src/tool/read.ts`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/read.ts)          | ReadTool 实现 |
| [`/opencode/packages/opencode/src/tool/bash.ts`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/bash.ts)          | BashTool 实现 |
| [`/opencode/packages/opencode/src/tool/edit.ts`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/edit.ts)          | EditTool 实现 |
| [`/opencode/packages/opencode/src/tool/truncation.ts`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/tool/truncation.ts)    | 输出截断处理  |
| [`/opencode/packages/opencode/src/provider/transform.ts`](https://github.com/anomalyco/opencode/blob/v1.14.30/packages/opencode/src/provider/transform.ts) | Schema 转换   |

### 相关文档

- [System Prompt 教程](./01-system-prompt.md)
- [权限系统教程](./02-permission-system.md)
- [Agent 系统教程](./03-agent-system.md)
- [OpenCode 学习规划](../00-learning-plan.md)
