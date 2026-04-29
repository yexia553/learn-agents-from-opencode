# OpenCode Provider 系统学习教程

> 基于源码分析的完整学习指南，涵盖 Provider 系统的架构设计、多模型适配和 API 集成。

---

## 目录

| 章节 | 标题 | 难度 |
|------|------|------|
| 一 | 系统概述 | 入门 |
| 二 | 核心架构 | 进阶 |
| 三 | Provider 类型定义 | 进阶 |
| 四 | Model 类型定义 | 进阶 |
| 五 | Provider 加载机制 | 高级 |
| 六 | SDK 初始化流程 | 高级 |
| 七 | 内置 Provider 详解 | 进阶 |
| 八 | 自定义 Provider 配置 | 实践 |
| 九 | 常见问题 | 排查 |

---

## 一、系统概述

### 1.1 什么是 Provider 系统

**Provider 系统** 是 OpenCode 的多模型提供商适配层，负责统一管理不同 LLM 提供商的 API 调用。

**核心职责：**

| 职责 | 说明 |
|------|------|
| **多提供商适配** | 统一 Anthropic、OpenAI、Google 等 20+ 提供商 |
| **SDK 管理** | 动态加载和缓存各提供商 SDK |
| **模型发现** | 自动发现和加载可用模型 |
| **配置合并** | 优先级配置覆盖（env > config > default） |
| **成本计算** | Token 消耗和成本统计 |

**设计目标：**

- **统一性**：对上层提供统一的 Model 接口
- **可扩展性**：支持动态添加新的 Provider
- **灵活性**：支持配置覆盖和环境变量
- **高性能**：SDK 缓存和复用

### 1.2 Provider 系统的位置

```
┌─────────────────────────────────────────────────────────────────┐
│                        OpenCode 架构                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌──────────────────┐    ┌─────────────┐    │
│  │   Session   │───▶│   Provider       │───▶│  External   │    │
│  │   System    │    │   System         │    │  APIs       │    │
│  └─────────────┘    └────────┬─────────┘    └─────────────┘    │
│                              │                                    │
│              ┌───────────────┼───────────────┐                  │
│              ▼               ▼               ▼                  │
│      ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│      │  Provider   │ │   Models    │ │  Transform  │           │
│      │   Core      │ │   Macro     │ │             │           │
│      └─────────────┘ └─────────────┘ └─────────────┘           │
│                              │                                    │
│              ┌───────────────┼───────────────┐                  │
│              ▼               ▼               ▼                  │
│      ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│      │ @ai-sdk/    │ │ @ai-sdk/    │ │ @ai-sdk/    │           │
│      │ anthropic   │ │  openai     │ │  google     │           │
│      └─────────────┘ └─────────────┘ └─────────────┘           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**相关文件：**

| 文件 | 路径 | 说明 |
|------|------|------|
| **Provider 核心** | `packages/opencode/src/provider/provider.ts` | 核心实现 (1135行) |
| **模型数据** | `packages/opencode/src/provider/models.ts` | 模型定义和加载 |
| **模型宏** | `packages/opencode/src/provider/models-macro.ts` | 内置模型数据 |
| **转换逻辑** | `packages/opencode/src/provider/transform.ts` | 工具 schema 转换 |

---

## 二、核心架构

### 2.1 Provider 加载流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    Provider 加载流程                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 加载 ModelsDev 数据库                                        │
│     ├── 从 models.json 读取                                      │
│     └── 从 models.dev/api.json 刷新                              │
│           │                                                       │
│           ▼                                                       │
│  2. 构建 Provider 数据库                                          │
│     ├── 从 models.dev 转换 Provider                              │
│     └── 合并配置文件 (opencode.json)                              │
│           │                                                       │
│           ▼                                                       │
│  3. 加载认证信息                                                  │
│     ├── 环境变量 (ANTHROPIC_API_KEY 等)                          │
│     ├── Auth 存储 (API Key)                                      │
│     └── Plugin 认证                                              │
│           │                                                       │
│           ▼                                                       │
│  4. 应用 Custom Loaders                                          │
│     ├── 自定义 Provider 配置                                      │
│     └── 特殊处理逻辑                                              │
│           │                                                       │
│           ▼                                                       │
│  5. 过滤和验证                                                    │
│     ├── 禁用 Provider 过滤                                        │
│     ├── Alpha 模型过滤                                           │
│     └── 黑名单/白名单过滤                                         │
│           │                                                       │
│           ▼                                                       │
│  6. 返回最终 Provider 列表                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 状态初始化

```typescript
// provider.ts:605-886 - Provider 状态初始化
const state = Instance.state(async () => {
  const config = await Config.get()
  const modelsDev = await ModelsDev.get()
  const database = mapValues(modelsDev, fromModelsDevProvider)

  const disabled = new Set(config.disabled_providers ?? [])
  const enabled = config.enabled_providers ? new Set(config.enabled_providers) : null

  function isProviderAllowed(providerID: string): boolean {
    if (enabled && !enabled.has(providerID)) return false
    if (disabled.has(providerID)) return false
    return true
  }

  const providers: { [providerID: string]: Info } = {}
  const languages = new Map<string, LanguageModelV2>()
  const modelLoaders: { [providerID: string]: CustomModelLoader } = {}
  const sdk = new Map<number, SDK>()

  // ... 加载逻辑
})
```

### 2.3 架构组件

| 组件 | 说明 | 作用 |
|------|------|------|
| **BUNDLED_PROVIDERS** | 内置 Provider 工厂 | 直接导入的 SDK |
| **CUSTOM_LOADERS** | 自定义加载器 | 特殊 Provider 处理 |
| **state()** | 状态缓存 | 延迟初始化和缓存 |
| **getSDK()** | SDK 获取 | 动态加载 Provider SDK |
| **getLanguage()** | 模型获取 | 返回 LLM 接口 |

---

## 三、Provider 类型定义

### 3.1 Provider Info 结构

```typescript
// provider.ts:513-526 - Provider 定义
export const Info = z
  .object({
    id: z.string(),                    // Provider 标识符
    name: z.string(),                  // 显示名称
    source: z.enum(["env", "config", "custom", "api"]),  // 来源
    env: z.string().array(),           // 环境变量列表
    key: z.string().optional(),        // API Key
    options: z.record(z.string(), z.any()),  // Provider 选项
    models: z.record(z.string(), Model),  // 模型列表
  })
  .meta({ ref: "Provider" })
export type Info = z.infer<typeof Info>
```

### 3.2 Provider 来源优先级

| 来源 | 说明 | 优先级 |
|------|------|--------|
| **env** | 环境变量 | 最高 |
| **api** | Auth 存储 | 高 |
| **custom** | Plugin/Custom Loader | 中 |
| **config** | opencode.json | 低 |
| **custom** | models.dev 默认 | 最低 |

### 3.3 Provider 来源示例

```typescript
// 1. 环境变量加载 (provider.ts:741-751)
for (const [providerID, provider] of Object.entries(database)) {
  const apiKey = provider.env.map((item) => env[item]).find(Boolean)
  if (!apiKey) continue
  mergeProvider(providerID, { source: "env", key: apiKey })
}

// 2. Auth 存储加载 (provider.ts:753-762)
for (const [providerID, provider] of Object.entries(await Auth.all())) {
  if (provider.type === "api") {
    mergeProvider(providerID, { source: "api", key: provider.key })
  }
}

// 3. 配置加载 (provider.ts:823-830)
for (const [providerID, provider] of configProviders) {
  mergeProvider(providerID, { source: "config" })
}
```

---

## 四、Model 类型定义

### 4.1 Model 结构

```typescript
// provider.ts:443-511 - Model 定义
export const Model = z
  .object({
    id: z.string(),                    // 模型 ID
    providerID: z.string(),            // Provider ID
    api: z.object({
      id: z.string(),                  // API 模型 ID
      url: z.string(),                 // API URL
      npm: z.string(),                 // SDK 包名
    }),
    name: z.string(),                  // 显示名称
    family: z.string().optional(),     // 模型系列
    capabilities: z.object({          // 能力支持
      temperature: z.boolean(),        // 温度参数
      reasoning: z.boolean(),          // 推理能力
      attachment: z.boolean(),         // 附件支持
      toolcall: z.boolean(),           // 工具调用
      input: z.object({               // 输入模态
        text: z.boolean(),
        audio: z.boolean(),
        image: z.boolean(),
        video: z.boolean(),
        pdf: z.boolean(),
      }),
      output: z.object({              // 输出模态
        text: z.boolean(),
        audio: z.boolean(),
        image: z.boolean(),
        video: z.boolean(),
        pdf: z.boolean(),
      }),
      interleaved: z.union([z.boolean(), z.object({ field: z.enum(...) })]),
    }),
    cost: z.object({                  // 成本信息
      input: z.number(),               // 输入价格 (per 1M)
      output: z.number(),              // 输出价格
      cache: z.object({
        read: z.number(),              // 缓存读取
        write: z.number(),             // 缓存写入
      }),
      experimentalOver200K: z.object({  // >200K context 成本
        input: z.number(),
        output: z.number(),
        cache: z.object({ read: z.number(), write: z.number() }),
      }).optional(),
    }),
    limit: z.object({                 // 限制
      context: z.number(),             // 上下文长度
      output: z.number(),              // 输出长度
    }),
    status: z.enum(["alpha", "beta", "deprecated", "active"]),  // 状态
    options: z.record(z.string(), z.any()),  // 模型选项
    headers: z.record(z.string(), z.string()),  // 自定义 headers
    release_date: z.string(),          // 发布日期
    variants: z.record(z.string(), z.record(z.string(), z.any())).optional(),  // 变体
  })
  .meta({ ref: "Model" })
```

### 4.2 字段速查

| 字段 | 类型 | 说明 |
|------|------|------|
| **id** | `string` | 模型唯一标识 |
| **api.id** | `string` | API 调用时的模型 ID |
| **api.npm** | `string` | SDK 包名 |
| **capabilities** | `Object` | 模型能力标志 |
| **cost** | `Object` | Token 价格 (per 1M tokens) |
| **limit.context** | `number` | 最大上下文 Token |
| **limit.output** | `number` | 最大输出 Token |
| **status** | `enum` | alpha/beta/deprecated/active |

### 4.3 能力标志

```typescript
// capabilities 决定工具可用性
capabilities: {
  temperature: true,    // 可调整温度
  reasoning: true,      // 支持思考过程
  attachment: true,     // 支持文件附件
  toolcall: true,       // 支持工具调用
  input: {
    text: true,         // 支持文本输入
    image: true,        // 支持图片输入
    audio: false,       // 不支持音频
    video: false,       // 不支持视频
    pdf: true,          // 支持 PDF
  },
  output: {
    text: true,
    image: false,
    audio: false,
    video: false,
    pdf: false,
  },
}
```

---

## 五、Provider 加载机制

### 5.1 配置合并策略

```typescript
// provider.ts:645-656 - Provider 合并
function mergeProvider(providerID: string, provider: Partial<Info>) {
  const existing = providers[providerID]
  if (existing) {
    // 深度合并现有配置
    providers[providerID] = mergeDeep(existing, provider)
    return
  }
  const match = database[providerID]
  if (!match) return
  providers[providerID] = mergeDeep(match, provider)
}
```

**合并优先级（从高到低）：**

```
1. Auth 存储 (api)
2. 环境变量 (env)
3. Custom Loader (custom)
4. 配置文件 (config)
5. 默认配置 (models.dev)
```

### 5.2 配置文件覆盖

```typescript
// provider.ts:659-739 - 从配置扩展数据库
for (const [providerID, provider] of configProviders) {
  const existing = database[providerID]
  const parsed: Info = {
    id: providerID,
    name: provider.name ?? existing?.name ?? providerID,
    env: provider.env ?? existing?.env ?? [],
    options: mergeDeep(existing?.options ?? {}, provider.options ?? {}),
    source: "config",
    models: existing?.models ?? {},
  }

  for (const [modelID, model] of Object.entries(provider.models ?? {})) {
    // 合并模型配置
    const parsedModel = {
      ...existingModel,
      ...model,
      capabilities: { ...existingModel.capabilities, ...model },
      cost: { ...existingModel.cost, ...model.cost },
    }
  }
}
```

### 5.3 模型过滤

```typescript
// provider.ts:850-869 - 模型过滤逻辑
for (const [modelID, model] of Object.entries(provider.models)) {
  // 1. 特殊模型移除
  if (modelID === "gpt-5-chat-latest") delete provider.models[modelID]

  // 2. Alpha 模型过滤（除非启用实验功能）
  if (model.status === "alpha" && !Flag.OPENCODE_ENABLE_EXPERIMENTAL_MODELS)
    delete provider.models[modelID]

  // 3. 黑名单过滤
  if (configProvider?.blacklist && configProvider.blacklist.includes(modelID))
    delete provider.models[modelID]

  // 4. 白名单过滤
  if (configProvider?.whitelist && !configProvider.whitelist.includes(modelID))
    delete provider.models[modelID]
}
```

### 5.4 禁用配置示例

```json
// opencode.json - Provider 配置
{
  "enabled_providers": ["anthropic", "openai"],
  "disabled_providers": ["google-vertex"],
  "provider": {
    "anthropic": {
      "options": {
        "timeout": 120000
      }
    },
    "openai": {
      "env": ["OPENAI_API_KEY"],
      "models": {
        "gpt-4": {
          "cost": {
            "input": 30,
            "output": 60
          }
        }
      }
    },
    "custom-provider": {
      "api": "https://custom-api.example.com/v1",
      "models": {
        "my-model": {
          "limit": {
            "context": 128000
          }
        }
      }
    }
  }
}
```

---

## 六、SDK 初始化流程

### 6.1 SDK 获取

```typescript
// provider.ts:892-975 - SDK 获取逻辑
async function getSDK(model: Model) {
  const s = await state()
  const provider = s.providers[model.providerID]
  const options = { ...provider.options }

  // 1. 设置 baseURL
  if (!options["baseURL"]) options["baseURL"] = model.api.url

  // 2. 设置 API Key
  if (options["apiKey"] === undefined && provider.key)
    options["apiKey"] = provider.key

  // 3. 合并自定义 headers
  if (model.headers) {
    options["headers"] = { ...options["headers"], ...model.headers }
  }

  // 4. 计算缓存键
  const key = Bun.hash.xxHash32(JSON.stringify({ npm: model.api.npm, options }))
  const existing = s.sdk.get(key)
  if (existing) return existing

  // 5. 创建带超时的 fetch
  options["fetch"] = async (input: any, init?: BunFetchRequestInit) => {
    const fetchFn = customFetch ?? fetch
    // 超时控制
    if (options["timeout"] !== undefined) {
      const signals = []
      if (init?.signal) signals.push(init.signal)
      signals.push(AbortSignal.timeout(options["timeout"]))
      init.signal = signals.length > 1 ? AbortSignal.any(signals) : signals[0]
    }
    return fetchFn(input, { ...init, timeout: false })
  }

  // 6. 加载 Provider SDK
  if (bundledFn) {
    // 内置 Provider 直接使用
    const loaded = bundledFn({ name: model.providerID, ...options })
    s.sdk.set(key, loaded)
    return loaded
  }

  // 7. 动态安装和加载第三方 Provider
  const installedPath = await BunProc.install(model.api.npm, "latest")
  const mod = await import(installedPath)
  const fn = mod[Object.keys(mod).find((key) => key.startsWith("create"))!]
  const loaded = fn({ name: model.providerID, ...options })
  s.sdk.set(key, loaded)
  return loaded
}
```

### 6.2 语言模型获取

```typescript
// provider.ts:1001-1026 - 获取语言模型
export async function getLanguage(model: Model): Promise<LanguageModelV2> {
  const s = await state()
  const key = `${model.providerID}/${model.id}`
  if (s.models.has(key)) return s.models.get(key)!

  const provider = s.providers[model.providerID]
  const sdk = await getSDK(model)

  try {
    // 使用自定义 loader 或默认 languageModel
    const language = s.modelLoaders[model.providerID]
      ? await s.modelLoaders[model.providerID](sdk, model.api.id, provider.options)
      : sdk.languageModel(model.api.id)
    s.models.set(key, language)
    return language
  } catch (e) {
    if (e instanceof NoSuchModelError)
      throw new ModelNotFoundError({ modelID: model.id, providerID: model.providerID })
    throw e
  }
}
```

### 6.3 SDK 缓存策略

```
┌─────────────────────────────────────────────────────────────────┐
│                    SDK 缓存机制                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  缓存键: hash(npm 包名 + options)                                │
│                                                                 │
│  相同 options 的请求复用同一 SDK 实例                             │
│                                                                 │
│  示例:                                                           │
│  ├── anthropic/claude-sonnet-4-20250514                         │
│  │   └── SDK: @ai-sdk/anthropic                                 │
│  └── openai/gpt-4o                                              │
│      └── SDK: @ai-sdk/openai                                    │
│                                                                 │
│  SDK 生命周期:                                                   │
│  ├── 首次请求: 安装包 + 初始化 + 缓存                            │
│  └── 后续请求: 直接返回缓存                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 七、内置 Provider 详解

### 7.1 内置 Provider 列表

```typescript
// provider.ts:43-65 - 内置 Provider 工厂
const BUNDLED_PROVIDERS: Record<string, (options: any) => SDK> = {
  "@ai-sdk/amazon-bedrock": createAmazonBedrock,
  "@ai-sdk/anthropic": createAnthropic,
  "@ai-sdk/azure": createAzure,
  "@ai-sdk/google": createGoogleGenerativeAI,
  "@ai-sdk/google-vertex": createVertex,
  "@ai-sdk/google-vertex/anthropic": createVertexAnthropic,
  "@ai-sdk/openai": createOpenAI,
  "@ai-sdk/openai-compatible": createOpenAICompatible,
  "@openrouter/ai-sdk-provider": createOpenRouter,
  "@ai-sdk/xai": createXai,
  "@ai-sdk/mistral": createMistral,
  "@ai-sdk/groq": createGroq,
  "@ai-sdk/deepinfra": createDeepInfra,
  "@ai-sdk/cerebras": createCerebras,
  "@ai-sdk/cohere": createCohere,
  "@ai-sdk/gateway": createGateway,
  "@ai-sdk/togetherai": createTogetherAI,
  "@ai-sdk/perplexity": createPerplexity,
  "@ai-sdk/vercel": createVercel,
  "@ai-sdk/github-copilot": createGitHubCopilotOpenAICompatible,
}
```

### 7.2 Anthropic 配置

```typescript
// provider.ts:75-85 - Anthropic 自定义加载器
async anthropic() {
  return {
    autoload: false,
    options: {
      headers: {
        "anthropic-beta":
          "claude-code-20250219,interleaved-thinking-2025-05-14,fine-grained-tool-streaming-2025-05-14",
      },
    },
  }
}
```

### 7.3 OpenAI 配置

```typescript
// provider.ts:108-116 - OpenAI 自定义加载器
openai: async () => {
  return {
    autoload: false,
    async getModel(sdk: any, modelID: string) {
      return sdk.responses(modelID)  // 使用 Responses API
    },
    options: {},
  }
}
```

### 7.4 AWS Bedrock 配置

```typescript
// provider.ts:170-302 - Bedrock 复杂配置
async () => {
  // 1. 认证优先级
  const authPriority = [
    providerConfig?.options?.profile,  // 1. 配置
    Env.get("AWS_PROFILE"),            // 2. 环境变量
  ]

  // 2. Region 解析
  const region = providerConfig?.options?.region ??
    Env.get("AWS_REGION") ??
    "us-east-1"

  // 3. Model ID 前缀处理
  switch (region.split("-")[0]) {
    case "us":
      // 美国区域添加前缀
      if (modelRequiresPrefix) modelID = `us.${modelID}`
      break
    case "eu":
      // 欧洲区域
      if (regionRequiresPrefix && modelRequiresPrefix) modelID = `eu.${modelID}`
      break
    case "ap":
      // 亚太区域
      if (isAustraliaRegion) modelID = `au.${modelID}`
      else if (isTokyoRegion) modelID = `jp.${modelID}`
      else modelID = `apac.${modelID}`
      break
  }

  return { autoload: true, options: { region, credentialProvider }, getModel }
}
```

### 7.5 Google Vertex 配置

```typescript
// provider.ts:325-341 - Vertex 自定义加载器
async () => {
  const project = Env.get("GOOGLE_CLOUD_PROJECT") ?? Env.get("GCP_PROJECT")
  const location = Env.get("GOOGLE_CLOUD_LOCATION") ?? "us-east5"
  const autoload = Boolean(project)
  if (!autoload) return { autoload: false }

  return {
    autoload: true,
    options: { project, location },
    async getModel(sdk: any, modelID: string) {
      return sdk.languageModel(String(modelID).trim())
    },
  }
}
```

---

## 八、自定义 Provider 配置

### 8.1 基本配置

```json
{
  "provider": {
    "custom-llm": {
      "api": "https://api.custom-llm.com/v1",
      "env": ["CUSTOM_LLM_API_KEY"],
      "models": {
        "custom-model-1": {
          "name": "Custom LLM v1",
          "limit": {
            "context": 128000,
            "output": 4096
          },
          "cost": {
            "input": 10,
            "output": 20
          }
        }
      }
    }
  }
}
```

### 8.2 完整模型配置

```json
{
  "provider": {
    "my-provider": {
      "name": "My LLM Provider",
      "env": ["MY_API_KEY"],
      "options": {
        "baseURL": "https://api.example.com/v1",
        "timeout": 120000,
        "headers": {
          "X-Custom-Header": "value"
        }
      },
      "models": {
        "model-id": {
          "id": "model-id",
          "name": "My Model",
          "provider": {
            "npm": "@ai-sdk/openai-compatible"
          },
          "status": "active",
          "temperature": true,
          "reasoning": false,
          "attachment": true,
          "tool_call": true,
          "modalities": {
            "input": ["text", "image"],
            "output": ["text"]
          },
          "cost": {
            "input": 5,
            "output": 15,
            "cache_read": 0.5,
            "cache_write": 2
          },
          "limit": {
            "context": 100000,
            "output": 4096
          },
          "variants": {
            "low-cost": {
              "temperature": 0.1,
              "cost": { "input": 1, "output": 2 }
            }
          }
        }
      }
    }
  }
}
```

### 8.3 模型变体配置

```json
{
  "provider": {
    "anthropic": {
      "models": {
        "claude-sonnet-4-20250514": {
          "variants": {
            "fast": {
              "temperature": 0.1,
              "topP": 0.9
            },
            "creative": {
              "temperature": 0.9,
              "topP": 0.99
            }
          }
        }
      }
    }
  }
}
```

### 8.4 黑名单/白名单

```json
{
  "provider": {
    "openai": {
      "whitelist": ["gpt-4o", "gpt-4o-mini"],
      "blacklist": ["gpt-3.5-turbo"]
    },
    "anthropic": {
      "whitelist": ["claude-sonnet-4-20250514", "claude-haiku-4-20250514"]
    }
  }
}
```

---

## 九、常见问题

### Q1: Provider 未加载？

```typescript
// 检查顺序：
// 1. 环境变量是否存在
const apiKey = provider.env.map((item) => env[item]).find(Boolean)
// env: ["ANTHROPIC_API_KEY"]

// 2. Auth 存储是否有 key
const auth = await Auth.get(providerID)

// 3. 是否被禁用
if (disabled.has(providerID)) return

// 4. 是否在 enabled_providers 中
if (enabled && !enabled.has(providerID)) return
```

### Q2: 模型不存在？

```typescript
// provider.ts:981-998 - 模型查找
export async function getModel(providerID: string, modelID: string) {
  const s = await state()
  const provider = s.providers[providerID]
  if (!provider) {
    // 模糊匹配提供建议
    const matches = fuzzysort.go(providerID, Object.keys(s.providers), { limit: 3 })
    throw new ModelNotFoundError({ providerID, modelID, suggestions: matches })
  }

  const info = provider.models[modelID]
  if (!info) {
    const matches = fuzzysort.go(modelID, Object.keys(provider.models), { limit: 3 })
    throw new ModelNotFoundError({ providerID, modelID, suggestions: matches })
  }
  return info
}
```

### Q3: SDK 加载失败？

```typescript
// 1. 检查 npm 包名
model.api.npm  // 应为有效的 npm 包名

// 2. 检查 timeout 设置
options["timeout"] // 设置为 false 禁用超时

// 3. 检查 baseURL
options["baseURL"] = model.api.url

// 错误类型
throw new InitError({ providerID: model.providerID }, { cause: e })
```

### Q4: 如何调试 Provider？

```typescript
// 启用日志
const log = Log.create({ service: "provider" })
log.info("init")
log.info("found", { providerID })
log.info("using bundled provider", { providerID, pkg: bundledKey })

// 查看状态
const providers = await Provider.list()
console.log(Object.keys(providers))
```

### Q5: 自定义 Provider 不工作？

```json
// 1. 检查 API 格式
{
  "provider": {
    "custom": {
      "api": "https://api.example.com/v1",  // 必须包含 /v1
      "models": {
        "model-name": {
          "provider": {
            "npm": "@ai-sdk/openai-compatible"  // 使用兼容 SDK
          }
        }
      }
    }
  }
}

// 2. 检查 API Key
ANTHROPIC_API_KEY=sk-...  // 环境变量名需匹配

// 3. 检查模型状态
"status": "alpha"  // alpha 模型需要 OPENCODE_ENABLE_EXPERIMENTAL_MODELS
```

### Q6: Region 前缀问题？

```typescript
// Bedrock region 前缀处理
// 美国: us.claude-3-5-sonnet-20241022
// 欧洲: eu.claude-3-5-sonnet-20241022
// 亚太: apac.claude-3-5-sonnet-20241022

// 解决方案：在 config 中指定 region
{
  "provider": {
    "amazon-bedrock": {
      "options": {
        "region": "us-east-1"
      }
    }
  }
}
```

---

## 附录

### A. Provider 速查表

| Provider | SDK | 认证方式 |
|----------|-----|----------|
| anthropic | @ai-sdk/anthropic | ANTHROPIC_API_KEY |
| openai | @ai-sdk/openai | OPENAI_API_KEY |
| google | @ai-sdk/google | GOOGLE_API_KEY |
| google-vertex | @ai-sdk/google-vertex | GCP 凭证 |
| amazon-bedrock | @ai-sdk/amazon-bedrock | AWS 凭证 |
| azure | @ai-sdk/azure | AZURE_API_KEY |
| groq | @ai-sdk/groq | GROQ_API_KEY |
| mistral | @ai-sdk/mistral | MISTRAL_API_KEY |
| xai | @ai-sdk/xai | XAI_API_KEY |
| openrouter | @openrouter/ai-sdk-provider | OPENROUTER_API_KEY |

### B. 环境变量速查

| Provider | 环境变量 |
|----------|----------|
| anthropic | `ANTHROPIC_API_KEY`, `ANTHROPIC_BETA` |
| openai | `OPENAI_API_KEY`, `OPENAI_ORG_ID` |
| google | `GOOGLE_API_KEY` |
| google-vertex | `GOOGLE_CLOUD_PROJECT`, `GOOGLE_CLOUD_LOCATION` |
| azure | `AZURE_API_KEY`, `AZURE_RESOURCE_NAME` |
| amazon-bedrock | `AWS_REGION`, `AWS_PROFILE`, `AWS_ACCESS_KEY_ID` |

### C. 推荐学习路径

```
第 1 天：理解 Provider 架构
        ├── 阅读 provider.ts 核心结构
        ├── 理解 Provider/Model 类型
        └── 掌握 Provider 加载流程

第 2 天：学习 Provider 配置
        ├── 理解配置合并策略
        ├── 掌握环境变量加载
        └── 学习模型过滤逻辑

第 3 天：实践自定义 Provider
        ├── 配置第三方 Provider
        ├── 添加模型变体
        └── 调试 Provider 问题

第 4 天：深入 SDK 初始化
        ├── 学习 SDK 缓存机制
        ├── 理解自定义 Loaders
        └── 掌握错误处理
```

### D. 成本计算公式

```typescript
// input 成本
cost += tokens.input * model.cost.input / 1_000_000

// output 成本
cost += tokens.output * model.cost.output / 1_000_000

// cache 成本
cost += tokens.cache.read * model.cost.cache.read / 1_000_000
cost += tokens.cache.write * model.cost.cache.write / 1_000_000

// >200K context 特殊定价
if (tokens.input + tokens.cache.read > 200_000) {
  cost = tokens.input * model.cost.experimentalOver200K.input / 1_000_000
  // ...
}
```
