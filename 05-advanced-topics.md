# 第五章：高级主题

> 本章目标：深入学习 Agent 系统的高级特性

---

## 5.1 多 Agent 协作

### 为什么需要多 Agent

单个 Agent 处理复杂任务时可能遇到：
- 上下文过长，超出模型限制
- 任务太复杂，需要专业分工
- 需要并行处理多个子任务

### OpenCode 的 Task 工具

OpenCode 通过 `task` 工具实现多 Agent 协作。

打开 `packages/opencode/src/tool/task.ts`：

```typescript
export const TaskTool = Tool.define("task", async () => {
  // 获取可用的子 Agent
  const agents = await Agent.list()
    .then(x => x.filter(a => a.mode !== "primary"))
  
  return {
    description: DESCRIPTION.replace("{agents}", 
      agents.map(a => `- ${a.name}: ${a.description}`).join("\n")
    ),
    
    parameters: z.object({
      description: z.string().describe("A short description of the task"),
      prompt: z.string().describe("The task for the agent to perform"),
      subagent_type: z.string().describe("The type of agent to use"),
    }),
    
    async execute(params, ctx) {
      // 1. 获取指定的子 Agent
      const agent = await Agent.get(params.subagent_type)
      
      // 2. 创建子会话
      const session = await Session.create({
        parentID: ctx.sessionID,
        title: params.description,
        // 子 Agent 的权限限制
        permission: [
          { permission: "todowrite", pattern: "*", action: "deny" },
          { permission: "todoread", pattern: "*", action: "deny" },
          { permission: "task", pattern: "*", action: "deny" },  // 防止无限嵌套
        ],
      })
      
      // 3. 执行子任务
      const result = await SessionPrompt.prompt({
        sessionID: session.id,
        model: agent.model,
        agent: agent.name,
        parts: promptParts,
      })
      
      // 4. 返回结果
      return {
        title: params.description,
        output: result.text,
        metadata: { sessionId: session.id },
      }
    },
  }
})
```

### 多 Agent 协作流程

```
┌─────────────────────────────────────────────────────────────┐
│                    多 Agent 协作                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  用户请求: "分析这个项目并生成文档"                          │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Primary Agent (build)                  │   │
│  │                                                     │   │
│  │  Thought: 这个任务需要两步：                         │   │
│  │           1. 探索代码结构                            │   │
│  │           2. 生成文档                                │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│              ┌────────────┴────────────┐                   │
│              ▼                         ▼                   │
│  ┌───────────────────┐    ┌───────────────────┐           │
│  │  task(explore)    │    │  task(general)    │           │
│  │                   │    │                   │           │
│  │  探索代码结构      │    │  生成文档         │           │
│  │  返回结构摘要      │    │  返回文档内容      │           │
│  └───────────────────┘    └───────────────────┘           │
│              │                         │                   │
│              └────────────┬────────────┘                   │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Primary Agent (build)                  │   │
│  │                                                     │   │
│  │  汇总子任务结果，生成最终回复                         │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 子 Agent 的限制

为了安全和防止无限循环，子 Agent 有以下限制：

```typescript
permission: [
  // 不能使用 todo 工具（避免干扰主任务）
  { permission: "todowrite", pattern: "*", action: "deny" },
  { permission: "todoread", pattern: "*", action: "deny" },
  // 不能再调用 task（防止无限嵌套）
  { permission: "task", pattern: "*", action: "deny" },
]
```


---

## 5.2 上下文管理

### 上下文窗口限制

每个 LLM 都有上下文窗口限制：

| 模型 | 上下文窗口 |
|------|-----------|
| Claude 3.5 Sonnet | 200K tokens |
| GPT-4 Turbo | 128K tokens |
| Claude 3 Opus | 200K tokens |

当对话过长时，需要进行上下文管理。

### OpenCode 的上下文压缩

打开 `packages/opencode/src/session/compaction.ts`：

```typescript
export namespace SessionCompaction {
  // 检查是否需要压缩
  export async function isOverflow(input: { 
    tokens: { input: number }
    model: Provider.Model 
  }) {
    const limit = input.model.limit.context
    const threshold = limit * 0.8  // 80% 阈值
    return input.tokens.input > threshold
  }
  
  // 执行压缩
  export async function compact(input: {
    sessionID: string
    model: Provider.Model
  }) {
    // 1. 获取所有消息
    const messages = await Session.messages({ sessionID: input.sessionID })
    
    // 2. 使用 compaction Agent 生成摘要
    const summary = await generateSummary(messages)
    
    // 3. 用摘要替换旧消息
    await replaceWithSummary(input.sessionID, summary)
  }
}
```

### 压缩策略

```
┌─────────────────────────────────────────────────────────────┐
│                    上下文压缩策略                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  原始对话:                                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ User: 帮我分析 src 目录                              │   │
│  │ Assistant: [调用 glob] [调用 read] 分析结果...       │   │
│  │ User: 修改 index.ts                                 │   │
│  │ Assistant: [调用 edit] 已修改...                    │   │
│  │ User: 运行测试                                      │   │
│  │ Assistant: [调用 bash] 测试结果...                  │   │
│  │ ... (更多消息)                                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  压缩后:                                                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ [Summary]                                           │   │
│  │ 之前的对话中：                                       │   │
│  │ 1. 分析了 src 目录结构                              │   │
│  │ 2. 修改了 index.ts 文件                             │   │
│  │ 3. 运行了测试，全部通过                              │   │
│  │ 关键文件：src/index.ts, src/utils/helper.ts         │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ (最近的几条消息保持原样)                             │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 会话摘要

打开 `packages/opencode/src/session/summary.ts`：

```typescript
export namespace SessionSummary {
  export async function summarize(input: {
    sessionID: string
    messageID: string
  }) {
    // 获取会话的文件变更
    const diffs = await Snapshot.diff(input.sessionID)
    
    // 更新会话摘要
    await Session.update(input.sessionID, draft => {
      draft.summary = {
        additions: diffs.additions,
        deletions: diffs.deletions,
        files: diffs.files.length,
        diffs: diffs.files,
      }
    })
  }
}
```

---

## 5.3 权限与安全

### 权限系统设计

OpenCode 的权限系统基于规则匹配：

```typescript
// 权限规则结构
type PermissionRule = {
  permission: string    // 权限类型：read, edit, bash, etc.
  pattern: string       // 匹配模式：文件路径或命令
  action: "allow" | "deny" | "ask"  // 动作
}
```

### 权限类型

| 权限 | 说明 | 示例 |
|------|------|------|
| `read` | 读取文件 | `read: { "*.ts": "allow" }` |
| `edit` | 编辑文件 | `edit: { "*": "deny" }` |
| `bash` | 执行命令 | `bash: { "npm *": "allow" }` |
| `external_directory` | 访问项目外目录 | 默认 `ask` |
| `doom_loop` | 重复操作检测 | 默认 `ask` |

### 权限配置示例

在 `~/.config/opencode/config.json` 中：

```json
{
  "permission": {
    "read": {
      "*": "allow",
      ".env": "deny",
      "*.pem": "deny"
    },
    "edit": {
      "*": "allow",
      "node_modules/**": "deny"
    },
    "bash": {
      "npm *": "allow",
      "git *": "allow",
      "rm -rf *": "deny"
    },
    "external_directory": "ask"
  }
}
```

### 权限检查流程

```
┌─────────────────────────────────────────────────────────────┐
│                    权限检查流程                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  工具调用请求                                               │
│       │                                                     │
│       ▼                                                     │
│  ┌─────────────────┐                                       │
│  │ 提取权限类型    │  read / edit / bash / ...             │
│  └────────┬────────┘                                       │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────┐                                       │
│  │ 匹配权限规则    │  检查 pattern 是否匹配                 │
│  └────────┬────────┘                                       │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────┐                                       │
│  │ 执行动作        │                                       │
│  │                 │                                       │
│  │ allow ──▶ 直接执行                                      │
│  │ deny  ──▶ 拒绝并报错                                    │
│  │ ask   ──▶ 询问用户                                      │
│  └─────────────────┘                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 安全最佳实践

1. **最小权限原则**：只授予必要的权限
2. **敏感文件保护**：默认拒绝 `.env`、密钥文件
3. **危险命令拦截**：`rm -rf`、`sudo` 等需要确认
4. **外部访问控制**：项目外目录默认询问


---

## 5.4 MCP 协议扩展

### 什么是 MCP

MCP（Model Context Protocol）是 Anthropic 提出的开放协议，用于扩展 AI 助手的能力。

```
┌─────────────────────────────────────────────────────────────┐
│                      MCP 架构                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────┐         ┌─────────────────┐           │
│  │   OpenCode      │         │   MCP Server    │           │
│  │   (MCP Client)  │◀───────▶│                 │           │
│  └─────────────────┘   MCP   │  - 数据库查询    │           │
│                      Protocol │  - API 调用     │           │
│                               │  - 自定义功能    │           │
│                               └─────────────────┘           │
│                                                             │
│  MCP 协议定义了：                                           │
│  - 工具发现（list tools）                                   │
│  - 工具调用（call tool）                                    │
│  - 资源访问（prompts）                                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### OpenCode 的 MCP 实现

打开 `packages/opencode/src/mcp/index.ts`：

```typescript
export namespace MCP {
  // MCP 服务器状态
  export const Status = z.discriminatedUnion("status", [
    z.object({ status: z.literal("connected") }),
    z.object({ status: z.literal("disabled") }),
    z.object({ status: z.literal("failed"), error: z.string() }),
    z.object({ status: z.literal("needs_auth") }),
  ])
  
  // 连接 MCP 服务器
  async function create(key: string, mcp: Config.Mcp) {
    if (mcp.type === "local") {
      // 本地 MCP 服务器（通过 stdio）
      const transport = new StdioClientTransport({
        command: mcp.command[0],
        args: mcp.command.slice(1),
        env: mcp.environment,
      })
      const client = new Client({ name: "opencode", version: VERSION })
      await client.connect(transport)
      return client
    }
    
    if (mcp.type === "remote") {
      // 远程 MCP 服务器（通过 HTTP）
      const transport = new StreamableHTTPClientTransport(
        new URL(mcp.url),
        { authProvider }
      )
      const client = new Client({ name: "opencode", version: VERSION })
      await client.connect(transport)
      return client
    }
  }
  
  // 获取 MCP 工具
  export async function tools() {
    const result: Record<string, Tool> = {}
    const clients = await state().then(s => s.clients)
    
    for (const [clientName, client] of Object.entries(clients)) {
      const toolsResult = await client.listTools()
      
      for (const mcpTool of toolsResult.tools) {
        // 转换为 OpenCode 工具格式
        result[`${clientName}_${mcpTool.name}`] = await convertMcpTool(mcpTool, client)
      }
    }
    return result
  }
}
```

### MCP 配置

在 `~/.config/opencode/config.json` 中配置 MCP 服务器：

```json
{
  "mcp": {
    "filesystem": {
      "type": "local",
      "command": ["npx", "@modelcontextprotocol/server-filesystem", "/path/to/dir"]
    },
    "github": {
      "type": "local", 
      "command": ["npx", "@modelcontextprotocol/server-github"],
      "environment": {
        "GITHUB_TOKEN": "your-token"
      }
    },
    "custom-api": {
      "type": "remote",
      "url": "https://your-mcp-server.com/mcp"
    }
  }
}
```

### MCP 工具转换

```typescript
// 将 MCP 工具转换为 OpenCode 工具
async function convertMcpTool(mcpTool: MCPToolDef, client: MCPClient): Promise<Tool> {
  return dynamicTool({
    description: mcpTool.description ?? "",
    inputSchema: jsonSchema(mcpTool.inputSchema),
    execute: async (args: unknown) => {
      // 调用 MCP 服务器执行工具
      return client.callTool({
        name: mcpTool.name,
        arguments: args as Record<string, unknown>,
      })
    },
  })
}
```

---

## 5.5 插件系统

### OpenCode 插件架构

```
┌─────────────────────────────────────────────────────────────┐
│                     插件系统                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ~/.config/opencode/                                        │
│  ├── plugin/                                                │
│  │   ├── my-plugin/                                        │
│  │   │   ├── index.ts      ← 插件入口                      │
│  │   │   └── package.json                                  │
│  │   └── another-plugin/                                   │
│  └── tool/                                                  │
│      └── custom-tool.ts    ← 自定义工具                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 插件钩子

OpenCode 提供多个扩展点：

```typescript
// 插件可以监听的事件
Plugin.trigger("experimental.chat.system.transform", {}, { system })
Plugin.trigger("experimental.text.complete", { sessionID, messageID }, { text })
Plugin.trigger("chat.params", { sessionID, agent, model }, { temperature, options })
```

### 自定义工具插件

创建 `~/.config/opencode/tool/database.ts`：

```typescript
import z from "zod"

export default {
  description: `
    Query a database. Use this when you need to fetch or modify data.
    
    PARAMETERS:
    - query: SQL query to execute
    - database: Database name (default: "main")
    
    EXAMPLE:
    {"query": "SELECT * FROM users LIMIT 10"}
  `,
  
  args: {
    query: z.string().describe("SQL query to execute"),
    database: z.string().default("main").describe("Database name"),
  },
  
  async execute(args: { query: string; database: string }) {
    // 实际实现中连接数据库
    const result = await executeQuery(args.database, args.query)
    return JSON.stringify(result, null, 2)
  }
}
```

---

## 5.6 流式处理与实时更新

### 为什么需要流式处理

1. **用户体验**：实时看到 AI 的回复，而不是等待完成
2. **工具状态**：实时显示工具执行进度
3. **可中断**：用户可以随时取消

### Vercel AI SDK 的流式 API

```typescript
import { streamText } from "ai"

const stream = await streamText({
  model: language,
  messages: [...],
  tools: {...},
})

// 处理流式响应
for await (const value of stream.fullStream) {
  switch (value.type) {
    case "text-delta":
      // 文本增量
      console.log(value.text)
      break
    case "tool-call":
      // 工具调用
      console.log(`Calling ${value.toolName}`)
      break
    case "tool-result":
      // 工具结果
      console.log(`Result: ${value.output}`)
      break
  }
}
```

### OpenCode 的实时更新

```typescript
// processor.ts 中的实时更新
case "text-delta":
  if (currentText) {
    currentText.text += value.text
    // 实时更新 UI
    await Session.updatePart({
      part: currentText,
      delta: value.text,  // 只发送增量
    })
  }
  break

case "tool-call":
  // 工具开始执行时更新状态
  ctx.metadata({
    metadata: {
      output: "",
      description: params.description,
    },
  })
  break
```

---

## 5.7 错误处理与重试

### 错误类型

```typescript
// 可重试的错误
const RETRYABLE_ERRORS = [
  "rate_limit",      // 速率限制
  "overloaded",      // 服务过载
  "timeout",         // 超时
  "connection_error" // 连接错误
]

// 不可重试的错误
const FATAL_ERRORS = [
  "invalid_api_key",  // API Key 无效
  "insufficient_quota", // 配额不足
  "content_policy",   // 内容策略违规
]
```

### 重试策略

打开 `packages/opencode/src/session/retry.ts`：

```typescript
export namespace SessionRetry {
  // 判断是否可重试
  export function retryable(error: MessageV2.Error): string | undefined {
    if (error.name === "APIError") {
      if (error.status === 429) return "Rate limited"
      if (error.status === 529) return "Service overloaded"
      if (error.status >= 500) return "Server error"
    }
    return undefined  // 不可重试
  }
  
  // 计算重试延迟（指数退避）
  export function delay(attempt: number, error?: APIError): number {
    // 如果服务器返回了 retry-after，使用它
    if (error?.headers?.["retry-after"]) {
      return parseInt(error.headers["retry-after"]) * 1000
    }
    // 否则使用指数退避：1s, 2s, 4s, 8s...
    return Math.min(1000 * Math.pow(2, attempt - 1), 30000)
  }
}
```

### 重试流程

```
┌─────────────────────────────────────────────────────────────┐
│                     重试流程                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  LLM 调用                                                   │
│      │                                                      │
│      ▼                                                      │
│  ┌─────────┐                                               │
│  │  成功？  │──── Yes ────▶ 继续处理                        │
│  └────┬────┘                                               │
│       │ No                                                  │
│       ▼                                                     │
│  ┌─────────────┐                                           │
│  │ 可重试错误？ │──── No ────▶ 返回错误                     │
│  └──────┬──────┘                                           │
│         │ Yes                                               │
│         ▼                                                   │
│  ┌─────────────┐                                           │
│  │ 重试次数 < 3 │──── No ────▶ 返回错误                     │
│  └──────┬──────┘                                           │
│         │ Yes                                               │
│         ▼                                                   │
│  ┌─────────────┐                                           │
│  │ 等待退避时间 │                                           │
│  └──────┬──────┘                                           │
│         │                                                   │
│         └──────────▶ 重新调用 LLM                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 5.8 本章小结

### 高级特性回顾

| 特性 | 用途 | 关键文件 |
|------|------|---------|
| 多 Agent | 复杂任务分解 | `tool/task.ts` |
| 上下文压缩 | 处理长对话 | `session/compaction.ts` |
| 权限系统 | 安全控制 | `permission/next.ts` |
| MCP 扩展 | 外部工具集成 | `mcp/index.ts` |
| 流式处理 | 实时更新 | `session/processor.ts` |
| 错误重试 | 可靠性保障 | `session/retry.ts` |

### 架构总览

```
┌─────────────────────────────────────────────────────────────┐
│                   OpenCode 高级架构                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    用户界面                          │   │
│  │              (TUI / Desktop / Web)                  │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   Session 管理                       │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │   │
│  │  │ Prompt  │ │Processor│ │Compaction│ │ Retry  │   │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│              ┌────────────┼────────────┐                   │
│              ▼            ▼            ▼                   │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│  │   Agent     │ │    Tool     │ │    MCP      │          │
│  │  (多Agent)  │ │  (内置+自定义)│ │  (外部扩展) │          │
│  └─────────────┘ └─────────────┘ └─────────────┘          │
│              │            │            │                   │
│              └────────────┼────────────┘                   │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  Permission 权限                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   Provider (LLM)                     │   │
│  │         Anthropic / OpenAI / Google / ...           │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 检查清单

- [ ] 理解多 Agent 协作的实现方式
- [ ] 了解上下文压缩的策略
- [ ] 掌握权限系统的配置方法
- [ ] 知道如何配置 MCP 服务器
- [ ] 理解流式处理和错误重试机制

---

[← 上一章：Prompt Engineering](./04-prompt-engineering.md) | [下一章：实战项目 →](./06-projects.md)
