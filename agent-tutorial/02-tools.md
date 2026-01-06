# 第二章：工具系统 (Tools)

> 本章目标：深入理解 Agent 的工具系统，学会阅读和创建工具

---

## 2.1 为什么 Agent 需要工具

### LLM 的能力边界

LLM 本质上是一个**文本生成模型**，它只能：
- 接收文本输入
- 输出文本

它**无法**直接：
- 读写文件
- 执行代码
- 访问网络
- 查询数据库

### 工具是 Agent 的"手脚"

工具（Tools）让 Agent 能够与外部世界交互：

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│                        LLM (大脑)                           │
│                           │                                 │
│              "我需要读取 package.json"                       │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    工具系统                          │   │
│  │                                                     │   │
│  │   read ──▶ 读取文件系统 ──▶ 返回文件内容            │   │
│  │   bash ──▶ 执行 Shell   ──▶ 返回命令输出            │   │
│  │   grep ──▶ 搜索文件     ──▶ 返回匹配结果            │   │
│  │                                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│                      外部世界                               │
│           (文件系统、网络、数据库...)                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 工具的本质

从技术角度看，工具就是一个**函数**：
- 有明确的输入参数（schema）
- 有明确的输出格式
- 执行某种副作用或查询

```typescript
// 工具的抽象结构
interface Tool {
  name: string           // 工具名称
  description: string    // 描述（给 LLM 看）
  parameters: Schema     // 参数定义
  execute(args): Result  // 执行函数
}
```

---

## 2.2 Function Calling 机制详解

### 工作流程

```
┌─────────────────────────────────────────────────────────────┐
│                   Function Calling 流程                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 用户输入                                                │
│     │                                                       │
│     ▼                                                       │
│  2. 发送给 LLM（附带可用工具列表）                           │
│     │                                                       │
│     │  请求体包含：                                         │
│     │  - messages: 对话历史                                 │
│     │  - tools: [                                          │
│     │      { name: "read", description: "...", schema: {} } │
│     │      { name: "bash", description: "...", schema: {} } │
│     │    ]                                                  │
│     │                                                       │
│     ▼                                                       │
│  3. LLM 决定是否调用工具                                    │
│     │                                                       │
│     ├──▶ 不需要工具 ──▶ 直接返回文本回复                    │
│     │                                                       │
│     └──▶ 需要工具 ──▶ 返回 tool_call：                      │
│                        {                                    │
│                          "tool": "read",                    │
│                          "arguments": {                     │
│                            "filePath": "package.json"       │
│                          }                                  │
│                        }                                    │
│                        │                                    │
│                        ▼                                    │
│  4. Agent 执行工具，获取结果                                │
│     │                                                       │
│     ▼                                                       │
│  5. 将结果作为 tool_result 发回给 LLM                       │
│     │                                                       │
│     ▼                                                       │
│  6. LLM 基于结果继续思考（可能再次调用工具，或给出最终回复） │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 消息格式示例

一次完整的工具调用，消息流如下：

```typescript
// 1. 用户消息
{ role: "user", content: "读取 package.json 的内容" }

// 2. LLM 返回工具调用
{ 
  role: "assistant", 
  tool_calls: [{
    id: "call_123",
    type: "function",
    function: {
      name: "read",
      arguments: '{"filePath": "package.json"}'
    }
  }]
}

// 3. 工具执行结果
{
  role: "tool",
  tool_call_id: "call_123",
  content: '{"name": "opencode", "version": "1.0.0", ...}'
}

// 4. LLM 最终回复
{
  role: "assistant",
  content: "package.json 的内容显示这是一个名为 opencode 的项目..."
}
```

---

## 2.3 OpenCode 的工具系统

### 工具定义基础

打开 `packages/opencode/src/tool/tool.ts`：

```typescript
export namespace Tool {
  // 工具执行时的上下文
  export type Context = {
    sessionID: string      // 当前会话 ID
    messageID: string      // 当前消息 ID
    agent: string          // 当前 Agent 名称
    abort: AbortSignal     // 用于取消执行
    
    // 更新工具执行状态（实时显示进度）
    metadata(input: { title?: string; metadata?: any }): void
    
    // 请求用户授权
    ask(input: PermissionRequest): Promise<void>
  }

  // 工具的完整定义
  export interface Info<Parameters, Metadata> {
    id: string  // 工具唯一标识
    init: () => Promise<{
      description: string           // 工具描述
      parameters: Parameters        // 参数 Schema
      execute(args, ctx): Promise<{
        title: string               // 执行结果标题
        metadata: Metadata          // 元数据
        output: string              // 输出内容（给 LLM 看）
      }>
    }>
  }

  // 定义工具的辅助函数
  export function define<P, M>(
    id: string,
    init: Info<P, M>["init"]
  ): Info<P, M>
}
```

### 工具注册表

打开 `packages/opencode/src/tool/registry.ts`：

```typescript
export namespace ToolRegistry {
  // 获取所有可用工具
  async function all(): Promise<Tool.Info[]> {
    return [
      InvalidTool,      // 处理无效工具调用
      BashTool,         // 执行 Shell 命令
      ReadTool,         // 读取文件
      GlobTool,         // 文件模式匹配
      GrepTool,         // 搜索文件内容
      EditTool,         // 编辑文件
      WriteTool,        // 写入文件
      TaskTool,         // 调用子 Agent
      WebFetchTool,     // 获取网页内容
      TodoWriteTool,    // 写入待办事项
      TodoReadTool,     // 读取待办事项
      WebSearchTool,    // 网络搜索
      CodeSearchTool,   // 代码搜索
      SkillTool,        // 技能调用
      ...custom,        // 自定义工具
    ]
  }
}
```

### OpenCode 内置工具一览

| 工具 | 用途 | 文件 |
|------|------|------|
| `read` | 读取文件内容 | `read.ts` |
| `write` | 写入文件 | `write.ts` |
| `edit` | 编辑文件（diff 方式） | `edit.ts` |
| `bash` | 执行 Shell 命令 | `bash.ts` |
| `glob` | 文件模式匹配搜索 | `glob.ts` |
| `grep` | 搜索文件内容 | `grep.ts` |
| `task` | 调用子 Agent | `task.ts` |
| `webfetch` | 获取网页内容 | `webfetch.ts` |
| `websearch` | 网络搜索 | `websearch.ts` |
| `codesearch` | 代码语义搜索 | `codesearch.ts` |

---

## 2.4 深入分析：read 工具

让我们详细分析一个简单但完整的工具实现。

### 完整代码解析

打开 `packages/opencode/src/tool/read.ts`：

```typescript
import z from "zod"
import * as fs from "fs"
import * as path from "path"
import { Tool } from "./tool"
import DESCRIPTION from "./read.txt"  // 工具描述从单独文件加载

const DEFAULT_READ_LIMIT = 2000  // 默认读取行数限制
const MAX_LINE_LENGTH = 2000    // 单行最大长度

export const ReadTool = Tool.define("read", {
  // 1. 工具描述 - 这是给 LLM 看的，非常重要！
  description: DESCRIPTION,
  
  // 2. 参数定义 - 使用 Zod schema
  parameters: z.object({
    filePath: z.string().describe("The path to the file to read"),
    offset: z.coerce.number()
      .describe("The line number to start reading from (0-based)")
      .optional(),
    limit: z.coerce.number()
      .describe("The number of lines to read (defaults to 2000)")
      .optional(),
  }),
  
  // 3. 执行函数
  async execute(params, ctx) {
    // 3.1 路径处理
    let filepath = params.filePath
    if (!path.isAbsolute(filepath)) {
      filepath = path.join(process.cwd(), filepath)
    }
    
    // 3.2 权限检查 - 外部目录需要用户确认
    if (!Filesystem.contains(Instance.directory, filepath)) {
      await ctx.ask({
        permission: "external_directory",
        patterns: [parentDir],
        // ...
      })
    }
    
    // 3.3 读取权限检查
    await ctx.ask({
      permission: "read",
      patterns: [filepath],
      // ...
    })
    
    // 3.4 安全检查 - 阻止读取敏感文件
    if (/^\.env(\.|$)/.test(basename)) {
      throw new Error(`Blocked from reading ${filepath}`)
    }
    
    // 3.5 文件存在性检查
    const file = Bun.file(filepath)
    if (!(await file.exists())) {
      // 提供相似文件建议
      throw new Error(`File not found: ${filepath}\n\nDid you mean...`)
    }
    
    // 3.6 处理特殊文件类型（图片、PDF）
    if (isImage || isPdf) {
      return {
        title,
        output: "Image read successfully",
        attachments: [/* base64 编码的文件 */],
      }
    }
    
    // 3.7 二进制文件检查
    if (await isBinaryFile(filepath, file)) {
      throw new Error(`Cannot read binary file: ${filepath}`)
    }
    
    // 3.8 读取文件内容
    const limit = params.limit ?? DEFAULT_READ_LIMIT
    const offset = params.offset || 0
    const lines = await file.text().then(text => text.split("\n"))
    
    // 3.9 格式化输出（带行号）
    const content = lines
      .slice(offset, offset + limit)
      .map((line, index) => {
        // 截断过长的行
        if (line.length > MAX_LINE_LENGTH) {
          line = line.substring(0, MAX_LINE_LENGTH) + "..."
        }
        // 添加行号
        return `${(index + offset + 1).toString().padStart(5, "0")}| ${line}`
      })
    
    // 3.10 构建输出
    let output = "<file>\n" + content.join("\n")
    
    if (hasMoreLines) {
      output += `\n\n(File has more lines. Use 'offset' to read beyond line ${lastReadLine})`
    }
    output += "\n</file>"
    
    // 3.11 返回结果
    return {
      title: path.relative(Instance.worktree, filepath),
      output,
      metadata: {
        preview: content.slice(0, 20).join("\n"),
      },
    }
  },
})
```

### 关键设计点

1. **参数验证**：使用 Zod 定义严格的参数 schema
2. **权限检查**：通过 `ctx.ask()` 请求用户授权
3. **安全防护**：阻止读取 `.env` 等敏感文件
4. **友好错误**：文件不存在时提供相似文件建议
5. **格式化输出**：带行号，方便 LLM 引用具体位置
6. **大文件处理**：支持分页读取（offset/limit）

---

## 2.5 深入分析：bash 工具

bash 工具更复杂，涉及命令解析和安全控制。

### 核心代码解析

打开 `packages/opencode/src/tool/bash.ts`：

```typescript
export const BashTool = Tool.define("bash", async () => {
  const shell = Shell.acceptable()  // 检测可用的 shell
  
  return {
    description: DESCRIPTION.replaceAll("${directory}", Instance.directory),
    
    parameters: z.object({
      command: z.string().describe("The command to execute"),
      timeout: z.number().describe("Optional timeout in milliseconds").optional(),
      workdir: z.string().describe("The working directory").optional(),
      // 重要：要求 LLM 描述命令用途
      description: z.string().describe(
        "Clear, concise description of what this command does in 5-10 words"
      ),
    }),
    
    async execute(params, ctx) {
      const cwd = params.workdir || Instance.directory
      const timeout = params.timeout ?? DEFAULT_TIMEOUT  // 默认 2 分钟
      
      // 1. 解析命令（使用 tree-sitter）
      const tree = await parser().then(p => p.parse(params.command))
      
      // 2. 分析命令中涉及的目录
      const directories = new Set<string>()
      for (const node of tree.rootNode.descendantsOfType("command")) {
        // 检测 cd, rm, cp, mv 等命令的目标路径
        if (["cd", "rm", "cp", "mv", "mkdir"].includes(command[0])) {
          // 解析路径，检查是否在项目目录外
          if (!Filesystem.contains(Instance.directory, resolved)) {
            directories.add(resolved)
          }
        }
      }
      
      // 3. 外部目录权限检查
      if (directories.size > 0) {
        await ctx.ask({
          permission: "external_directory",
          patterns: Array.from(directories),
        })
      }
      
      // 4. 命令执行权限检查
      await ctx.ask({
        permission: "bash",
        patterns: Array.from(patterns),
      })
      
      // 5. 执行命令
      const proc = spawn(params.command, {
        shell,
        cwd,
        stdio: ["ignore", "pipe", "pipe"],
      })
      
      // 6. 实时输出更新
      let output = ""
      const append = (chunk: Buffer) => {
        output += chunk.toString()
        // 实时更新 UI
        ctx.metadata({
          metadata: { output, description: params.description },
        })
      }
      proc.stdout?.on("data", append)
      proc.stderr?.on("data", append)
      
      // 7. 超时处理
      const timeoutTimer = setTimeout(() => {
        timedOut = true
        Shell.killTree(proc)
      }, timeout)
      
      // 8. 等待完成
      await new Promise<void>((resolve) => {
        proc.once("exit", resolve)
      })
      
      // 9. 输出截断（防止过长）
      if (output.length > MAX_OUTPUT_LENGTH) {
        output = output.slice(0, MAX_OUTPUT_LENGTH)
        // 添加截断提示
      }
      
      return {
        title: params.description,
        metadata: { output, exit: proc.exitCode },
        output,
      }
    },
  }
})
```

### bash 工具的安全设计

```
┌─────────────────────────────────────────────────────────────┐
│                    bash 工具安全层                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 命令解析                                                │
│     └── 使用 tree-sitter 解析 bash 语法                     │
│         └── 提取命令名和参数                                │
│                                                             │
│  2. 路径分析                                                │
│     └── 检测命令涉及的文件路径                              │
│         └── 判断是否在项目目录内                            │
│                                                             │
│  3. 权限检查                                                │
│     ├── 外部目录访问 → 需要用户确认                         │
│     └── 命令执行 → 根据权限规则判断                         │
│                                                             │
│  4. 执行限制                                                │
│     ├── 超时控制（默认 2 分钟）                             │
│     ├── 输出长度限制（30KB）                                │
│     └── 支持中断（abort signal）                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 2.6 工具描述的重要性

### 描述是给 LLM 看的

工具的 `description` 直接影响 LLM 是否能正确使用工具。

打开 `packages/opencode/src/tool/read.txt`：

```text
Reads a file from the local filesystem. Use this tool to access file contents
when you need to examine code, configuration files, documentation, or any
other text-based files.

IMPORTANT GUIDELINES:
- Always use relative paths from the current working directory
- For large files, use the offset and limit parameters to read in chunks
- The output includes line numbers for easy reference
- Binary files cannot be read with this tool

PARAMETERS:
- filePath: Path to the file (relative or absolute)
- offset: Starting line number (0-based, optional)
- limit: Number of lines to read (default: 2000)

EXAMPLES:
- Read entire file: {"filePath": "src/index.ts"}
- Read from line 100: {"filePath": "src/index.ts", "offset": 100}
- Read 50 lines from line 200: {"filePath": "src/index.ts", "offset": 200, "limit": 50}
```

### 好的工具描述应该包含

1. **功能说明**：工具做什么
2. **使用场景**：什么时候应该用这个工具
3. **参数说明**：每个参数的含义和格式
4. **使用示例**：具体的调用例子
5. **注意事项**：限制、边界情况

### 描述质量对比

❌ **差的描述**：
```
Reads a file.
```

✅ **好的描述**：
```
Reads a file from the local filesystem. Use this when you need to:
- Examine source code
- Check configuration files
- Read documentation

The output includes line numbers (e.g., "00001| content").
For large files, use offset/limit to read in chunks.
Cannot read binary files.
```

---

## 2.7 动手实验：创建自定义工具

### 实验目标

创建一个 `weather` 工具，获取天气信息。

### 步骤 1：创建工具文件

在 `~/.config/opencode/tool/` 目录下创建 `weather.ts`：

```typescript
import z from "zod"

export default {
  description: `
    Gets current weather information for a specified city.
    Use this when the user asks about weather conditions.
    
    PARAMETERS:
    - city: Name of the city (e.g., "Beijing", "New York")
    
    EXAMPLE:
    {"city": "Shanghai"}
  `,
  
  args: {
    city: z.string().describe("The city name to get weather for"),
  },
  
  async execute(args: { city: string }) {
    // 这里用模拟数据，实际可以调用天气 API
    const mockWeather = {
      "Beijing": { temp: 25, condition: "Sunny" },
      "Shanghai": { temp: 28, condition: "Cloudy" },
      "New York": { temp: 20, condition: "Rainy" },
    }
    
    const weather = mockWeather[args.city] || { temp: 22, condition: "Unknown" }
    
    return `Weather in ${args.city}: ${weather.temp}°C, ${weather.condition}`
  }
}
```

### 步骤 2：测试工具

1. 重启 OpenCode：
```bash
cd packages/opencode && bun dev
```

2. 输入：
```
北京今天天气怎么样？
```

3. 观察 Agent 是否调用了你的 `weather` 工具

### 步骤 3：改进工具

尝试添加更多功能：
- 支持获取未来几天的天气
- 添加湿度、风速等信息
- 接入真实的天气 API

---

## 2.8 知识补充：工具设计最佳实践

### 1. 单一职责

每个工具只做一件事：

```typescript
// ❌ 不好：一个工具做太多事
Tool.define("file", {
  // 既能读又能写又能删除
})

// ✅ 好：拆分成多个工具
Tool.define("read", { /* 只读 */ })
Tool.define("write", { /* 只写 */ })
Tool.define("delete", { /* 只删 */ })
```

### 2. 参数设计

```typescript
// ❌ 不好：参数含义不清
parameters: z.object({
  p: z.string(),
  n: z.number(),
})

// ✅ 好：参数名清晰，有描述
parameters: z.object({
  filePath: z.string().describe("Path to the file to read"),
  lineCount: z.number().describe("Number of lines to read"),
})
```

### 3. 错误处理

```typescript
async execute(params, ctx) {
  // ❌ 不好：直接抛出原始错误
  const content = await fs.readFile(params.path)
  
  // ✅ 好：提供有帮助的错误信息
  try {
    const content = await fs.readFile(params.path)
  } catch (e) {
    if (e.code === 'ENOENT') {
      const suggestions = findSimilarFiles(params.path)
      throw new Error(
        `File not found: ${params.path}\n` +
        `Did you mean: ${suggestions.join(', ')}?`
      )
    }
    throw e
  }
}
```

### 4. 输出格式

```typescript
// ❌ 不好：输出难以解析
return { output: content }

// ✅ 好：结构化输出，带上下文
return {
  output: `<file path="${filepath}">\n${content}\n</file>`,
  metadata: {
    lines: content.split('\n').length,
    size: content.length,
  }
}
```

---

## 2.9 本章小结

### 核心概念回顾

1. **工具是 Agent 与外部世界交互的桥梁**
2. **Function Calling** 是 LLM 调用工具的标准机制
3. **工具描述**直接影响 LLM 的使用效果
4. **安全设计**是工具开发的重要考量

### OpenCode 工具系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                     工具系统架构                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Tool.define()          ToolRegistry              LLM       │
│       │                      │                      │       │
│       │   注册工具           │                      │       │
│       ├─────────────────────▶│                      │       │
│       │                      │                      │       │
│       │                      │   获取工具列表        │       │
│       │                      │◀─────────────────────┤       │
│       │                      │                      │       │
│       │                      │   tool_call          │       │
│       │                      │◀─────────────────────┤       │
│       │                      │                      │       │
│       │   执行工具           │                      │       │
│       │◀─────────────────────┤                      │       │
│       │                      │                      │       │
│       │   返回结果           │   tool_result        │       │
│       ├─────────────────────▶├─────────────────────▶│       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 检查清单

- [ ] 理解 Function Calling 的工作流程
- [ ] 能读懂 OpenCode 的工具实现代码
- [ ] 理解工具描述的重要性
- [ ] 成功创建了自定义工具
- [ ] 了解工具设计的最佳实践

---

[← 上一章：Agent 是什么](./01-what-is-agent.md) | [下一章：Agent Loop →](./03-agent-loop.md)
