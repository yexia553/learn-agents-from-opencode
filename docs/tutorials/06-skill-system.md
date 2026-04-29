# OpenCode Skill 系统学习教程

> 基于源码分析的完整学习指南，涵盖 Skill 系统的架构设计、实现原理和定制方法。

---

## 目录

| 章节 | 标题 | 难度 |
|------|------|------|
| 一 | 系统概述 | 入门 |
| 二 | 核心架构 | 进阶 |
| 三 | Skill 定义格式 | 进阶 |
| 四 | 发现与加载机制 | 进阶 |
| 五 | Skill 工具集成 | 进阶 |
| 六 | 权限系统集成 | 高级 |
| 七 | 自定义 Skill 实战 | 实践 |
| 八 | 常见问题 | 排查 |

---

## 一、系统概述

### 1.1 什么是 Skill 系统

**Skill 系统** 是 OpenCode 的专业技能扩展机制，允许用户定义和管理特定领域的专家知识。

**核心职责：**

| 职责 | 说明 |
|------|------|
| **技能发现** | 自动扫描并发现可用的 Skill |
| **技能管理** | 提供技能的加载、解析和执行机制 |
| **权限控制** | 结合权限系统控制 Skill 的访问 |
| **内容呈现** | 将 Skill 内容格式化为可读格式 |

**设计目标：**

- **专业化**：为特定任务提供专家级指导
- **可扩展**：用户可自由添加自定义 Skill
- **模块化**：每个 Skill 独立管理，易于维护
- **安全性**：通过权限系统控制 Skill 访问

### 1.2 Skill 系统的位置

```
┌─────────────────────────────────────────────────────────────────┐
│                        OpenCode 架构                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌──────────────────┐    ┌─────────────┐    │
│  │   Agent     │───▶│   Tool System    │───▶│  Provider   │    │
│  └─────────────┘    └────────┬─────────┘    └─────────────┘    │
│                              │                                    │
│              ┌───────────────┼───────────────┐                  │
│              ▼               ▼               ▼                  │
│      ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│      │   skill     │ │   grep      │ │   read      │           │
│      │   tool      │ │   tool      │ │   tool      │           │
│      └─────────────┘ └─────────────┘ └─────────────┘           │
│              │                                                    │
│              ▼                                                    │
│      ┌─────────────┐                                             │
│      │  Skill      │                                             │
│      │  Registry   │                                             │
│      │ (skill.ts)  │                                             │
│      └─────────────┘                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**相关文件：**

| 文件 | 路径 | 说明 |
|------|------|------|
| **Skill 核心** | `packages/opencode/src/skill/skill.ts` | Skill 发现与管理 |
| **Skill 工具** | `packages/opencode/src/tool/skill.ts` | Skill 工具实现 |
| **Skill 导出** | `packages/opencode/src/skill/index.ts` | 模块导出 |
| **配置解析** | `packages/opencode/src/config/markdown.ts` | SKILL.md 解析 |

---

## 二、核心架构

### 2.1 类型定义

Skill 的核心数据结构定义在 `skill.ts:13-18`：

```typescript
// packages/opencode/src/skill/skill.ts:13-18
export const Info = z.object({
  name: z.string(),           // Skill 标识符
  description: z.string(),    // Skill 功能描述
  location: z.string(),       // SKILL.md 文件路径
})
export type Info = z.infer<typeof Info>
```

### 2.2 错误类型

```typescript
// packages/opencode/src/skill/skill.ts:20-36

// 无效 Skill 错误
export const InvalidError = NamedError.create(
  "SkillInvalidError",
  z.object({
    path: z.string(),                    // 无效的 Skill 文件路径
    message: z.string().optional(),      // 错误消息
    issues: z.custom<z.core.$ZodIssue[]>().optional(),  // Zod 验证问题
  }),
)

// 名称不匹配错误
export const NameMismatchError = NamedError.create(
  "SkillNameMismatchError",
  z.object({
    path: z.string(),                    // Skill 文件路径
    expected: z.string(),                // 期望的名称
    actual: z.string(),                  // 实际的名称
  }),
)
```

### 2.3 核心函数

| 函数 | 签名 | 说明 |
|------|------|------|
| **state** | `Instance.state(async () => {...})` | 异步初始化 Skill 状态 |
| **get** | `async function get(name: string)` | 根据名称获取单个 Skill |
| **all** | `async function all()` | 获取所有可用 Skill |

**state 实现 (`skill.ts:41-115`)：**

```typescript
// 状态初始化
export const state = Instance.state(async () => {
  const skills: Record<string, Info> = {}

  // 添加 Skill 的辅助函数
  const addSkill = async (match: string) => {
    const md = await ConfigMarkdown.parse(match)
    if (!md) return

    const parsed = Info.pick({ name: true, description: true }).safeParse(md.data)
    if (!parsed.success) return

    // 检测重复名称
    if (skills[parsed.data.name]) {
      log.warn("duplicate skill name", {
        name: parsed.data.name,
        existing: skills[parsed.data.name].location,
        duplicate: match,
      })
    }

    skills[parsed.data.name] = {
      name: parsed.data.name,
      description: parsed.data.description,
      location: match,
    }
  }

  // ... 扫描逻辑（见第四章）

  return skills
})
```

### 2.4 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                      Skill 系统架构                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   扫描层 (Scan Layer)                     │   │
│  │                                                          │   │
│  │  .claude/skills/**/*.SKILL.md    ← 项目级全局            │   │
│  │  ~/.claude/skills/**/*.SKILL.md  ← 用户级全局            │   │
│  │  .opencode/skill/**/*.SKILL.md   ← 项目级本地            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│              ┌───────────────┼───────────────┐                  │
│              ▼               ▼               ▼                  │
│      ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│      │   解析      │ │   验证      │ │   存储      │           │
│      │ ConfigMarkdown│ │   name/    │ │ skills{}   │           │
│      │   .parse()  │ │ description │ │  Record    │           │
│      └─────────────┘ └─────────────┘ └─────────────┘           │
│                              │                                  │
│              ┌───────────────┼───────────────┐                  │
│              ▼               ▼               ▼                  │
│      ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│      │  Skill.get()│ │ Skill.all() │ │  Tool 调用  │           │
│      └─────────────┘ └─────────────┘ └─────────────┘           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 三、Skill 定义格式

### 3.1 SKILL.md 结构

每个 Skill 通过一个 `SKILL.md` 文件定义，必须包含 YAML frontmatter：

```markdown
---
name: skill-identifier
description: A brief description of what this skill does
---

# Skill 文档标题

这里是技能的详细说明和使用指南...
```

**文件命名规则：**

```
skill-name/
└── SKILL.md           ← 必须命名为 SKILL.md
```

### 3.2 Frontmatter 字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| **name** | `string` | ✅ | Skill 唯一标识符 |
| **description** | `string` | ✅ | Skill 功能描述（用于 Skill 选择） |

**完整示例：**

```markdown
---
name: code-reviewer
description: Use this skill when asked to review code for bugs, security issues, or style violations
---

# Code Review Skill

## 何时使用

当需要审查代码时使用此 Skill：
- 检测潜在 bug
- 识别安全漏洞
- 检查代码风格
- 验证最佳实践

## 使用步骤

1. **阅读代码**
   - 使用 Read 工具查看目标文件
   - 分析代码结构和逻辑

2. **发现问题**
   - 使用 Grep 搜索敏感操作
   - 检查错误处理

3. **输出报告**
   - 整理发现的问题
   - 提供修复建议
```

### 3.3 目录结构规范

Skill 可以放在以下目录结构中：

```
项目结构/
├── .claude/
│   └── skills/
│       ├── code-reviewer/
│       │   └── SKILL.md
│       └── database-expert/
│           └── SKILL.md
│
├── .opencode/
│   └── skill/
│       ├── api-generator/
│       │   └── SKILL.md
│       └── test-writer/
│           └── SKILL.md
│
└── ~/.claude/                    ← 用户全局目录
    └── skills/
        └── my-custom-skill/
            └── SKILL.md
```

### 3.4 Skill 名称规则

| 规则 | 说明 |
|------|------|
| **唯一性** | 同一名称只能出现一次，重复会报警告 |
| **命名风格** | 建议使用 kebab-case（如 `code-reviewer`） |
| **可嵌套** | 支持多级目录，名称包含路径（如 `category/skill-name`） |

**名称冲突处理：**

```typescript
// skill.ts:54-60 - 重复名称检测
if (skills[parsed.data.name]) {
  log.warn("duplicate skill name", {
    name: parsed.data.name,
    existing: skills[parsed.data.name].location,   // 已存在的路径
    duplicate: match,                              // 重复的路径
  })
}
// 后续的同名 Skill 会被忽略，只保留第一个
```

---

## 四、发现与加载机制

### 4.1 扫描路径

Skill 系统支持三个扫描路径：

| 路径 | 类型 | 优先级 | 说明 |
|------|------|--------|------|
| `.claude/skills/**/*.SKILL.md` | 项目级 | 高 | 项目内全局 Skill |
| `~/.claude/skills/**/*.SKILL.md` | 用户级 | 中 | 用户全局 Skill |
| `.opencode/skill/**/*.SKILL.md` | 项目级 | 低 | 项目本地 Skill |

**扫描模式：**

```typescript
// skill.ts:38-39 - glob 模式定义
const OPENCODE_SKILL_GLOB = new Bun.Glob("{skill,skills}/**/SKILL.md")
const CLAUDE_SKILL_GLOB = new Bun.Glob("skills/**/SKILL.md")
```

**注意：** `{skill,skills}` 语法支持两种目录名。

### 4.2 扫描流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    Skill 扫描流程                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 扫描 .claude/skills/                                        │
│     ├── 向上遍历目录查找 .claude                                 │
│     ├── 包含全局 ~/.claude/                                      │
│     └── 使用 CLAUDE_SKILL_GLOB 匹配                              │
│                                                                 │
│  2. 扫描 .opencode/skill/                                       │
│     ├── 使用 Config.directories() 获取配置目录                   │
│     └── 使用 OPENCODE_SKILL_GLOB 匹配                            │
│                                                                 │
│  3. 解析每个匹配的 SKILL.md                                      │
│     ├── 解析 YAML frontmatter                                   │
│     ├── 验证 name 和 description                                 │
│     └── 存储到 skills{} 对象                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 .claude 目录扫描

```typescript
// skill.ts:69-100 - .claude 目录扫描

// 步骤 1：查找所有 .claude 目录
const claudeDirs = await Array.fromAsync(
  Filesystem.up({
    targets: [".claude"],
    start: Instance.directory,
    stop: Instance.worktree,
  }),
)

// 步骤 2：包含全局 ~/.claude/
const globalClaude = `${Global.Path.home}/.claude`
if (await exists(globalClaude)) {
  claudeDirs.push(globalClaude)
}

// 步骤 3：扫描每个目录
for (const dir of claudeDirs) {
  const matches = await Array.fromAsync(
    CLAUDE_SKILL_GLOB.scan({
      cwd: dir,
      absolute: true,
      onlyFiles: true,
      followSymlinks: true,
      dot: true,
    }),
  ).catch((error) => {
    log.error("failed .claude directory scan for skills", { dir, error })
    return []
  })

  for (const match of matches) {
    await addSkill(match)
  }
}
```

**Filesystem.up() 函数作用：**

```typescript
// 从当前目录向上遍历，查找包含 .claude 的目录
Filesystem.up({
  targets: [".claude"],     // 目标目录名
  start: Instance.directory, // 起始目录
  stop: Instance.worktree,   // 停止目录（工作树根）
})
```

**示例：**

```
工作目录: /project/user/feature/
工作树:   /project/

Filesystem.up() 查找结果:
├── /project/user/.claude/
└── /project/.claude/
```

### 4.4 .opencode/skill 目录扫描

```typescript
// skill.ts:102-112 - .opencode/skill 目录扫描

for (const dir of await Config.directories()) {
  for await (const match of OPENCODE_SKILL_GLOB.scan({
    cwd: dir,
    absolute: true,
    onlyFiles: true,
    followSymlinks: true,
  })) {
    await addSkill(match)
  }
}
```

### 4.5 技能获取 API

```typescript
// skill.ts:117-123 - Skill 获取函数

// 获取单个 Skill
export async function get(name: string) {
  return state().then((x) => x[name])
}

// 获取所有 Skill
export async function all() {
  return state().then((x) => Object.values(x))
}
```

**使用示例：**

```typescript
// 获取单个 Skill
const skill = await Skill.get("code-reviewer")
console.log(skill)
// 输出: { name: "code-reviewer", description: "...", location: "/path/to/SKILL.md" }

// 获取所有 Skill
const allSkills = await Skill.all()
console.log(allSkills)
// 输出: [{ name: "code-reviewer", ... }, { name: "database-expert", ... }]
```

---

## 五、Skill 工具集成

### 5.1 SkillTool 定义

Skill 通过 `SkillTool` 暴露给 LLM：

```typescript
// packages/opencode/src/tool/skill.ts:12-75
export const SkillTool = Tool.define("skill", async (ctx) => {
  const skills = await Skill.all()

  // 过滤基于权限的 Skill
  const agent = ctx?.agent
  const accessibleSkills = agent
    ? skills.filter((skill) => {
        const rule = PermissionNext.evaluate("skill", skill.name, agent.permission)
        return rule.action !== "deny"
      })
    : skills

  // 构建工具描述
  const description = [
    "Load a skill to get detailed instructions for a specific task.",
    "Skills provide specialized knowledge and step-by-step guidance.",
    "<available_skills>",
    ...accessibleSkills.flatMap((skill) => [
      `  <skill>`,
      `    <name>${skill.name}</name>`,
      `    <description>${skill.description}</description>`,
      `  </skill>`,
    ]),
    "</available_skills>",
  ].join(" ")

  return { description, parameters }
})
```

### 5.2 工具参数

```typescript
// packages/opencode/src/tool/skill.ts:8-10
const parameters = z.object({
  name: z.string().describe("The skill identifier from available_skills"),
})
```

### 5.3 工具执行流程

```typescript
// packages/opencode/src/tool/skill.ts:44-73
async execute(params: z.infer<typeof parameters>, ctx) {
  // 1. 获取 Skill 信息
  const skill = await Skill.get(params.name)

  if (!skill) {
    const available = await Skill.all().then((x) => Object.keys(x).join(", "))
    throw new Error(`Skill "${params.name}" not found. Available skills: ${available || "none"}`)
  }

  // 2. 请求权限
  await ctx.ask({
    permission: "skill",
    patterns: [params.name],
    always: [params.name],
    metadata: {},
  })

  // 3. 解析 Skill 内容
  const parsed = await ConfigMarkdown.parse(skill.location)
  const dir = path.dirname(skill.location)

  // 4. 格式化输出
  const output = [
    `## Skill: ${skill.name}`,
    "",
    `**Base directory**: ${dir}`,
    "",
    parsed.content.trim(),
  ].join("\n")

  // 5. 返回结果
  return {
    title: `Loaded skill: ${skill.name}`,
    output,
    metadata: { name: skill.name, dir },
  }
}
```

### 5.4 工具描述示例

当 LLM 调用 Skill 工具时，看到的描述如下：

```
Load a skill to get detailed instructions for a specific task.
Skills provide specialized knowledge and step-by-step guidance.
<available_skills>
  <skill>
    <name>code-reviewer</name>
    <description>Use this skill when asked to review code...</description>
  </skill>
  <skill>
    <name>database-expert</name>
    <description>Database expert for SQL optimization...</description>
  </skill>
  <skill>
    <name>api-designer</name>
    <description>Design RESTful APIs following best practices...</description>
  </skill>
</available_skills>
```

### 5.5 Skill 调用示例

```
用户: 请帮我审查这段代码

Agent 思考: 用户请求代码审查，匹配 code-reviewer Skill

Agent 执行: call(skill, { name: "code-reviewer" })

输出:
## Skill: code-reviewer

**Base directory**: /project/.claude/skills/code-reviewer

# Code Review Skill

## 何时使用
...
```

---

## 六、权限系统集成

### 6.1 Skill 权限类型

Skill 作为一种权限类型进行管理：

| 权限类型 | 说明 |
|----------|------|
| **skill** | 控制是否允许加载特定 Skill |

### 6.2 权限过滤

SkillTool 在提供可用 Skill 列表时，会根据 Agent 权限进行过滤：

```typescript
// packages/opencode/src/tool/skill.ts:15-22
const agent = ctx?.agent
const accessibleSkills = agent
  ? skills.filter((skill) => {
      // 评估权限规则
      const rule = PermissionNext.evaluate("skill", skill.name, agent.permission)
      return rule.action !== "deny"  // 排除被拒绝的 Skill
    })
    : skills  // 无 agent 时返回全部
```

**权限评估流程：**

```
Skill: "code-reviewer"
Agent 权限: { "skill": { "code-*": "allow", "security-*": "deny" } }

评估步骤:
1. 匹配 "code-reviewer" ~ "code-*"  → allow
2. 不匹配 "security-*" 规则
3. 最终结果: allow (可见且可用)
```

### 6.3 权限请求

当实际加载 Skill 时，会请求 `skill` 权限：

```typescript
// packages/opencode/src/tool/skill.ts:52-57
await ctx.ask({
  permission: "skill",           // 权限类型
  patterns: [params.name],       // Skill 名称
  always: [params.name],         // 记住选择
  metadata: {},
})
```

### 6.4 权限配置示例

```typescript
// Agent 权限配置
{
  "skill": {
    "code-reviewer": "allow",        // 允许代码审查
    "database-expert": "allow",      // 允许数据库专家
    "*": "ask"                       // 其他 Skill 需要询问
  }
}

// 禁止特定 Skill
{
  "skill": {
    "experimental-*": "deny",        // 禁止实验性 Skill
    "test-skill": "deny"             // 禁止测试 Skill
  }
}
```

### 6.5 完整权限流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                    Skill 权限流程                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LLM 调用 SkillTool                                             │
│       │                                                         │
│       ▼                                                         │
│  Skill.all() 获取所有 Skill                                     │
│       │                                                         │
│       ▼                                                         │
│  权限过滤 (agent.permission)                                     │
│       │                                                         │
│       ├─── deny ──→ 排除该 Skill                                │
│       │                                                         │
│       └─── allow/ask ──→ 保留该 Skill                           │
│                                                                 │
│       ▼                                                         │
│  返回可用 Skill 列表给 LLM                                       │
│                                                                 │
│  ───────────────────────────────────────────────────────────    │
│                                                                 │
│  LLM 选择加载特定 Skill                                          │
│       │                                                         │
│       ▼                                                         │
│  ctx.ask({ permission: "skill", ... })                         │
│       │                                                         │
│       ▼                                                         │
│  权限再次评估                                                    │
│       │                                                         │
│       ├─── deny ──→ 抛出 DeniedError                           │
│       │                                                         │
│       ├─── allow ──→ 直接执行                                   │
│       │                                                         │
│       └─── ask ──→ 等待用户授权                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 七、自定义 Skill 实战

### 7.1 实战 1：创建代码审查 Skill

**步骤 1：创建目录和文件**

```bash
mkdir -p .claude/skills/code-reviewer
touch .claude/skills/code-reviewer/SKILL.md
```

**步骤 2：编写 SKILL.md**

```markdown
---
name: code-reviewer
description: Use this skill when asked to review code for bugs, security issues, or style violations
---

# Code Review Skill

## 何时使用

当需要执行以下任务时使用此 Skill：
- 审查代码质量
- 检测潜在 bug
- 识别安全漏洞
- 检查代码风格
- 验证最佳实践

## 审查步骤

### 1. 阅读代码
使用 Read 工具查看目标文件：
- 理解代码结构和逻辑
- 识别主要功能和组件

### 2. 安全检查
使用 Grep 搜索敏感操作：
```bash
# 搜索硬编码密钥
grep -r "api_key\|password\|secret" .
# 搜索 SQL 注入风险
grep -r "SELECT.*\+.*" .
```

### 3. 错误处理
检查错误处理模式：
- 是否正确处理异常
- 是否有适当的验证
- 是否有用户输入过滤

### 4. 输出报告
整理发现的问题：
```markdown
## 代码审查报告

### 发现的问题

1. **安全问题**: 硬编码密钥
   - 文件: `src/auth.ts`
   - 行号: 42
   - 建议: 使用环境变量

2. **错误处理**: 缺少异常处理
   - 文件: `src/api.ts`
   - 建议: 添加 try-catch
```

## 最佳实践

- 使用具体示例说明问题
- 提供可执行的修复建议
- 按严重程度分类问题
- 引用相关文档和标准
```

**步骤 3：验证 Skill 是否加载**

启动 OpenCode 后，调用 skill 工具查看：

```
/user @skill

可用技能:
- code-reviewer: Use this skill when asked to review code...
```

### 7.2 实战 2：创建数据库专家 Skill

**目录结构：**

```
.claude/skills/
└── database-expert/
    └── SKILL.md
```

**SKILL.md 内容：**

```markdown
---
name: database-expert
description: Database expert for SQL optimization, schema design, and performance tuning
---

# Database Expert Skill

## 何时使用

- SQL 查询优化
- 数据库 Schema 设计
- 索引策略制定
- 性能调优
- 数据迁移

## 优化检查清单

### 1. 查询分析
- 检查 EXPLAIN 执行计划
- 识别全表扫描
- 优化子查询为 JOIN

### 2. 索引优化
- 检查 WHERE 子句字段
- 检查 JOIN 条件字段
- 考虑复合索引

### 3. Schema 检查
- 规范化程度
- 数据类型选择
- 外键约束

## 示例命令

```sql
-- 查看执行计划
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';

-- 查看索引使用
SHOW INDEX FROM users;

-- 分析慢查询
SELECT * FROM mysql.slow_log ORDER BY start_time DESC LIMIT 10;
```
```

### 7.3 实战 3：创建 API 设计 Skill

**目录结构：**

```
.opencode/skill/
└── api-designer/
    └── SKILL.md
```

**SKILL.md 内容：**

```markdown
---
name: api-designer
description: Design RESTful APIs following best practices and OpenAPI specification
---

# API Designer Skill

## 原则

1. **RESTful 设计**
   - 使用 HTTP 方法语义
   - 资源命名使用名词复数
   - 支持 HATEOAS

2. **版本控制**
   - URL 版本: `/v1/users`
   - Header 版本: `Accept: application/vnd.api+json;version=1`

3. **错误处理**
   - 统一错误格式
   - 适当的状态码
   - 详细的错误消息

## 模板

### 资源定义
```yaml
/users:
  get:
    summary: 列出用户
    parameters:
      - name: page
        in: query
        schema:
          type: integer
          default: 1
    responses:
      '200':
        description: 用户列表
```

### 响应格式
```json
{
  "data": [],
  "meta": {
    "page": 1,
    "per_page": 20,
    "total": 100
  }
}
```
```

### 7.4 实战 4：Skill 权限配置

**场景：限制只有 build Agent 可使用 code-reviewer**

```json
// opencode.json
{
  "agent": {
    "build": {
      "permission": {
        "skill": {
          "code-reviewer": "allow"
        }
      }
    },
    "plan": {
      "permission": {
        "skill": {
          "code-reviewer": "deny"
        }
      }
    }
  }
}
```

### 7.5 实战 5：用户级全局 Skill

**场景：创建跨项目可用的 Skill**

```bash
# 创建用户级 Skill 目录
mkdir -p ~/.claude/skills/my-utils
touch ~/.claude/skills/my-utils/SKILL.md
```

**注意：** 用户级 Skill 对所有项目可见。

---

## 八、常见问题

### Q1: Skill 没有出现在列表中？

1. **检查文件位置**

```bash
# 正确位置
.claude/skills/my-skill/SKILL.md
.opencode/skill/my-skill/SKILL.md
~/.claude/skills/my-skill/SKILL.md

# 错误位置（不会被扫描）
.claude/skills/my-skill.md           # 直接放在目录下
skills/my-skill/something-else.md    # 不是 SKILL.md
```

2. **检查 frontmatter**

```markdown
---
name: my-skill              # ✅ 正确
description: My skill       # ✅ 正确
---

# 技能内容
```

```markdown
---
name: "my-skill"            # ❌ 不需要引号（但可以工作）
description: 'My skill'     # ❌ 不需要引号
---
```

3. **检查文件编码**

```bash
# 使用 UTF-8 编码
file SKILL.md
# 输出: SKILL.md: UTF-8 Unicode text

# 如果不是 UTF-8
iconv -f GBK -t UTF-8 SKILL.md > SKILL.md.utf8
```

### Q2: 重复名称警告？

```typescript
// skill.ts:54-60 - 重复名称处理
if (skills[parsed.data.name]) {
  log.warn("duplicate skill name", {
    name: parsed.data.name,
    existing: skills[parsed.data.name].location,  // 保留第一个
    duplicate: match,                             // 忽略后续
  })
}
```

**解决方案：**

```bash
# 重命名其中一个
.claude/skills/
├── code-reviewer-v1/SKILL.md   # 保留这个
└── code-reviewer/SKILL.md      # 重命名或删除这个
```

### Q3: Skill 加载失败？

1. **检查 YAML 语法**

```bash
# 使用工具验证 YAML
cat SKILL.md | python3 -c "import yaml, sys; yaml.safe_load(sys.stdin.read())"
```

2. **检查 frontmatter 字段**

```typescript
// skill.ts:50-51 - 必须包含 name 和 description
const parsed = Info.pick({ name: true, description: true }).safeParse(md.data)
if (!parsed.success) return  // 解析失败会跳过该 Skill
```

### Q4: Skill 权限被拒绝？

1. **检查 Agent 权限**

```json
// opencode.json
{
  "agent": {
    "build": {
      "permission": {
        "skill": {
          "my-skill": "deny"    # 被拒绝
        }
      }
    }
  }
}
```

2. **检查 Skill 权限配置**

```json
// opencode.json - 允许所有 Skill
{
  "agent": {
    "build": {
      "permission": {
        "skill": {
          "*": "allow"
        }
      }
    }
  }
}
```

### Q5: 如何调试 Skill 扫描？

```typescript
// 启用调试日志
log.warn() 和 log.error() 会输出到控制台

// 查看扫描路径
const claudeDirs = await Filesystem.up({...})
// claudeDirs 包含所有被扫描的 .claude 目录

// 查看匹配结果
const matches = await CLAUDE_SKILL_GLOB.scan({...})
// matches 包含所有匹配的 SKILL.md 文件
```

### Q6: Skill 内容不被显示？

```typescript
// tool/skill.ts:63 - 输出格式
const output = [
  `## Skill: ${skill.name}`,           // Skill 标题
  "",                                   // 空行
  `**Base directory**: ${dir}`,         // 基础目录
  "",                                   // 空行
  parsed.content.trim(),                // SKILL.md 内容（不含 frontmatter）
].join("\n")

// 如果内容为空，检查 ConfigMarkdown.parse() 返回值
const parsed = await ConfigMarkdown.parse(skill.location)
console.log(parsed)
// parsed.content 应该包含 frontmatter 后的内容
```

---

## 附录

### A. 目录优先级速查

| 路径 | 优先级 | 作用域 |
|------|--------|--------|
| `.claude/skills/` | 高 | 当前项目 |
| `~/.claude/skills/` | 中 | 所有项目 |
| `.opencode/skill/` | 低 | 当前项目 |

### B. 错误处理速查

| 错误类型 | 触发条件 | 处理方式 |
|----------|----------|----------|
| `SkillInvalidError` | YAML 解析失败 | 检查 frontmatter 语法 |
| `SkillNameMismatchError` | 文件名与 name 不符 | 验证 name 字段 |
| 重复名称警告 | 同名 Skill 多个 | 重命名或删除重复项 |

### C. 推荐学习路径

```
第 1 天：理解 Skill 架构
        ├── 阅读 skill.ts 源码
        ├── 理解扫描机制
        └── 创建第一个测试 Skill

第 2 天：实践 Skill 定义
        ├── 编写完整的 SKILL.md
        ├── 测试 Skill 加载
        └── 理解内容呈现格式

第 3 天：权限集成
        ├── 配置 Agent Skill 权限
        ├── 测试权限过滤
        └── 理解权限评估流程

第 4 天：高级用法
        ├── 创建嵌套路径 Skill
        ├── 配置用户级全局 Skill
        └── 调试 Skill 扫描问题
```

### D. SKILL.md 模板

```markdown
---
name: skill-name
description: Brief description of what this skill does
---

# Skill Title

## 何时使用
- Use case 1
- Use case 2

## 前提条件
- Prerequisite 1

## 使用步骤

### Step 1: xxx
Description...

### Step 2: xxx
Description...

## 示例
```bash
# Example command
```

## 注意事项
- Note 1
- Note 2
```
