# 第六章：实战项目

> 本章目标：通过实际项目巩固所学知识

---

## 6.1 项目一：创建专用 Agent

### 目标

创建一个专门用于代码审查的 Agent，具有以下特点：
- 只读权限，不能修改代码
- 专注于代码质量、安全性、性能
- 输出结构化的审查报告

### 步骤 1：定义 Agent 配置

创建或编辑 `~/.config/opencode/config.json`：

```json
{
  "agent": {
    "reviewer": {
      "description": "Code review agent that analyzes code quality, security, and performance",
      "prompt": "You are an expert code reviewer. Your job is to analyze code and provide constructive feedback.\n\n## Review Focus Areas\n1. **Code Quality**: Readability, maintainability, naming conventions\n2. **Security**: Potential vulnerabilities, input validation, authentication\n3. **Performance**: Inefficient algorithms, memory leaks, unnecessary operations\n4. **Best Practices**: Design patterns, error handling, testing\n\n## Output Format\nAlways structure your review as:\n\n### Summary\nBrief overview of the code and overall assessment.\n\n### Issues Found\n- 🔴 Critical: [description]\n- 🟡 Warning: [description]\n- 🔵 Suggestion: [description]\n\n### Recommendations\nSpecific actionable improvements.\n\n## Rules\n- Be constructive, not critical\n- Explain WHY something is an issue\n- Provide code examples for fixes\n- Acknowledge good practices you find",
      "permission": {
        "edit": { "*": "deny" },
        "write": { "*": "deny" },
        "bash": { "*": "deny" }
      }
    }
  }
}
```

### 步骤 2：测试 Agent

启动 OpenCode 并切换到 reviewer Agent：

```bash
cd packages/opencode && bun dev
```

在 OpenCode 中：
```
@reviewer 请审查 src/tool/read.ts 这个文件
```

### 步骤 3：观察输出

Agent 应该：
1. 读取文件内容
2. 分析代码质量
3. 输出结构化的审查报告
4. 不会尝试修改任何文件

### 扩展练习

1. 添加对特定语言的支持（如 TypeScript 特定规则）
2. 创建不同严格程度的审查模式
3. 添加自动生成修复建议的功能

---

## 6.2 项目二：开发自定义工具

### 目标

创建一个 `git-summary` 工具，用于分析 Git 仓库的提交历史。

### 步骤 1：创建工具文件

创建 `~/.config/opencode/tool/git-summary.ts`：

```typescript
import z from "zod"
import { execSync } from "child_process"

export default {
  description: `
Analyzes Git repository history and provides a summary.

USE CASES:
- Understanding recent changes in a project
- Finding who worked on specific files
- Analyzing commit patterns

PARAMETERS:
- days: Number of days to look back (default: 7)
- author: Filter by author name (optional)
- path: Filter by file path (optional)

EXAMPLES:
- Recent activity: {"days": 7}
- Specific author: {"days": 30, "author": "john"}
- Specific path: {"days": 14, "path": "src/"}

OUTPUT:
Returns a structured summary including:
- Total commits
- Authors and their commit counts
- Most changed files
- Recent commit messages
  `,

  args: {
    days: z.number().default(7).describe("Number of days to analyze"),
    author: z.string().optional().describe("Filter by author name"),
    path: z.string().optional().describe("Filter by file path"),
  },

  async execute(args: { days: number; author?: string; path?: string }) {
    const since = `--since="${args.days} days ago"`
    const author = args.author ? `--author="${args.author}"` : ""
    const pathFilter = args.path || ""

    try {
      // Get commit count
      const commitCount = execSync(
        `git log ${since} ${author} --oneline ${pathFilter} | wc -l`,
        { encoding: "utf-8" }
      ).trim()

      // Get authors
      const authors = execSync(
        `git log ${since} ${author} --format="%an" ${pathFilter} | sort | uniq -c | sort -rn | head -10`,
        { encoding: "utf-8" }
      ).trim()

      // Get most changed files
      const changedFiles = execSync(
        `git log ${since} ${author} --name-only --format="" ${pathFilter} | sort | uniq -c | sort -rn | head -10`,
        { encoding: "utf-8" }
      ).trim()

      // Get recent commits
      const recentCommits = execSync(
        `git log ${since} ${author} --format="%h %s (%an, %ar)" ${pathFilter} | head -10`,
        { encoding: "utf-8" }
      ).trim()

      return `
## Git Summary (Last ${args.days} days)
${args.author ? `Filtered by author: ${args.author}` : ""}
${args.path ? `Filtered by path: ${args.path}` : ""}

### Overview
- Total commits: ${commitCount}

### Top Contributors
${authors || "No commits found"}

### Most Changed Files
${changedFiles || "No files changed"}

### Recent Commits
${recentCommits || "No recent commits"}
      `.trim()
    } catch (error) {
      return `Error analyzing git history: ${error}`
    }
  },
}
```

### 步骤 2：测试工具

重启 OpenCode，然后测试：

```
最近一周有哪些代码变更？
```

Agent 应该调用 `git-summary` 工具并返回分析结果。

### 步骤 3：改进工具

添加更多功能：

```typescript
// 添加到 args
format: z.enum(["summary", "detailed", "json"]).default("summary"),

// 在 execute 中根据 format 返回不同格式
if (args.format === "json") {
  return JSON.stringify({
    commits: parseInt(commitCount),
    authors: parseAuthors(authors),
    files: parseFiles(changedFiles),
  }, null, 2)
}
```

### 扩展练习

1. 添加分支比较功能
2. 生成代码变更统计图表（ASCII）
3. 集成 GitHub/GitLab API 获取 PR 信息


---

## 6.3 项目三：构建 MCP Server

### 目标

创建一个简单的 MCP Server，提供天气查询功能。

### 步骤 1：初始化项目

```bash
mkdir weather-mcp-server
cd weather-mcp-server
npm init -y
npm install @modelcontextprotocol/sdk zod
```

### 步骤 2：创建 Server

创建 `index.ts`：

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js"
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js"
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js"

// 模拟天气数据
const weatherData: Record<string, { temp: number; condition: string; humidity: number }> = {
  "beijing": { temp: 25, condition: "Sunny", humidity: 45 },
  "shanghai": { temp: 28, condition: "Cloudy", humidity: 65 },
  "new york": { temp: 20, condition: "Rainy", humidity: 80 },
  "london": { temp: 15, condition: "Foggy", humidity: 90 },
  "tokyo": { temp: 22, condition: "Clear", humidity: 55 },
}

// 创建 Server
const server = new Server(
  { name: "weather-server", version: "1.0.0" },
  { capabilities: { tools: {} } }
)

// 注册工具列表
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "get_weather",
      description: "Get current weather for a city. Returns temperature, condition, and humidity.",
      inputSchema: {
        type: "object",
        properties: {
          city: {
            type: "string",
            description: "City name (e.g., 'Beijing', 'New York')",
          },
        },
        required: ["city"],
      },
    },
    {
      name: "list_cities",
      description: "List all available cities with weather data.",
      inputSchema: {
        type: "object",
        properties: {},
      },
    },
  ],
}))

// 处理工具调用
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params

  if (name === "get_weather") {
    const city = (args?.city as string)?.toLowerCase()
    const weather = weatherData[city]

    if (!weather) {
      return {
        content: [
          {
            type: "text",
            text: `Weather data not available for "${args?.city}". Available cities: ${Object.keys(weatherData).join(", ")}`,
          },
        ],
      }
    }

    return {
      content: [
        {
          type: "text",
          text: `Weather in ${args?.city}:\n- Temperature: ${weather.temp}°C\n- Condition: ${weather.condition}\n- Humidity: ${weather.humidity}%`,
        },
      ],
    }
  }

  if (name === "list_cities") {
    return {
      content: [
        {
          type: "text",
          text: `Available cities:\n${Object.entries(weatherData)
            .map(([city, data]) => `- ${city}: ${data.temp}°C, ${data.condition}`)
            .join("\n")}`,
        },
      ],
    }
  }

  return {
    content: [{ type: "text", text: `Unknown tool: ${name}` }],
  }
})

// 启动 Server
async function main() {
  const transport = new StdioServerTransport()
  await server.connect(transport)
  console.error("Weather MCP Server running...")
}

main().catch(console.error)
```

### 步骤 3：配置 OpenCode

在 `~/.config/opencode/config.json` 中添加：

```json
{
  "mcp": {
    "weather": {
      "type": "local",
      "command": ["npx", "ts-node", "/path/to/weather-mcp-server/index.ts"]
    }
  }
}
```

### 步骤 4：测试

重启 OpenCode，然后：

```
北京今天天气怎么样？
```

Agent 应该调用 `weather_get_weather` 工具。

### 扩展练习

1. 接入真实的天气 API（如 OpenWeatherMap）
2. 添加天气预报功能
3. 添加多语言支持

---

## 6.4 项目四：完整的 Agent 应用

### 目标

创建一个"项目分析助手"，能够：
1. 分析项目结构
2. 生成文档
3. 发现潜在问题
4. 提供改进建议

### 架构设计

```
┌─────────────────────────────────────────────────────────────┐
│                   项目分析助手                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Primary Agent (analyzer)               │   │
│  │                                                     │   │
│  │  接收用户请求，协调子任务                            │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│         ┌─────────────────┼─────────────────┐              │
│         ▼                 ▼                 ▼              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │   explore   │  │   general   │  │  reviewer   │        │
│  │             │  │             │  │             │        │
│  │  探索结构    │  │  生成文档    │  │  代码审查    │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
│                                                             │
│  自定义工具:                                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │ git-summary │  │ dep-check   │  │ loc-count   │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 步骤 1：创建分析 Agent

```json
{
  "agent": {
    "analyzer": {
      "description": "Project analysis agent that provides comprehensive insights",
      "prompt": "You are a project analysis expert. Your job is to thoroughly analyze codebases and provide actionable insights.\n\n## Analysis Process\n1. First, explore the project structure using the explore agent\n2. Analyze dependencies and potential issues\n3. Review code quality in key files\n4. Generate a comprehensive report\n\n## Report Structure\n### Project Overview\n- Name, description, tech stack\n- Directory structure\n- Key files\n\n### Dependencies\n- Main dependencies\n- Outdated packages\n- Security vulnerabilities\n\n### Code Quality\n- Code style consistency\n- Test coverage\n- Documentation status\n\n### Recommendations\n- Priority improvements\n- Technical debt\n- Best practices to adopt\n\n## Guidelines\n- Use subagents for specialized tasks\n- Be thorough but concise\n- Prioritize actionable insights"
    }
  }
}
```

### 步骤 2：创建辅助工具

创建 `~/.config/opencode/tool/dep-check.ts`：

```typescript
import z from "zod"
import { execSync } from "child_process"
import * as fs from "fs"

export default {
  description: "Check project dependencies for issues",
  args: {
    type: z.enum(["npm", "pip", "auto"]).default("auto"),
  },
  async execute(args: { type: string }) {
    let type = args.type
    
    // Auto-detect project type
    if (type === "auto") {
      if (fs.existsSync("package.json")) type = "npm"
      else if (fs.existsSync("requirements.txt")) type = "pip"
      else return "Could not detect project type"
    }
    
    if (type === "npm") {
      try {
        const outdated = execSync("npm outdated --json", { encoding: "utf-8" })
        const audit = execSync("npm audit --json", { encoding: "utf-8" })
        return `## NPM Dependencies\n\n### Outdated\n${outdated}\n\n### Security\n${audit}`
      } catch (e) {
        return `Error checking npm dependencies: ${e}`
      }
    }
    
    return "Unsupported project type"
  }
}
```

创建 `~/.config/opencode/tool/loc-count.ts`：

```typescript
import z from "zod"
import { execSync } from "child_process"

export default {
  description: "Count lines of code in the project",
  args: {
    extensions: z.string().default("ts,js,py").describe("File extensions to count"),
  },
  async execute(args: { extensions: string }) {
    const exts = args.extensions.split(",").map(e => `*.${e.trim()}`).join(" ")
    try {
      const result = execSync(
        `find . -type f \\( -name "${exts.split(" ").join('" -o -name "')}" \\) | xargs wc -l | tail -1`,
        { encoding: "utf-8" }
      )
      return `Total lines of code: ${result.trim()}`
    } catch {
      return "Error counting lines of code"
    }
  }
}
```

### 步骤 3：使用分析助手

```
@analyzer 请全面分析这个项目，包括：
1. 项目结构和技术栈
2. 依赖状态
3. 代码质量
4. 改进建议
```

---

## 6.5 学习总结

### 你学到了什么

通过这份教程，你应该掌握了：

| 章节 | 核心知识 |
|------|---------|
| 第一章 | Agent 的基本概念和组成 |
| 第二章 | 工具系统的设计和实现 |
| 第三章 | Agent Loop 的工作原理 |
| 第四章 | Prompt Engineering 技巧 |
| 第五章 | 高级特性：多Agent、权限、MCP |
| 第六章 | 实战项目开发 |

### OpenCode 代码地图

```
packages/opencode/src/
├── agent/
│   └── agent.ts          # Agent 定义
├── session/
│   ├── index.ts          # 会话管理
│   ├── llm.ts            # LLM 调用
│   ├── processor.ts      # Agent Loop
│   ├── system.ts         # System Prompt
│   └── compaction.ts     # 上下文压缩
├── tool/
│   ├── tool.ts           # 工具基础
│   ├── registry.ts       # 工具注册
│   ├── read.ts           # 读文件工具
│   ├── bash.ts           # 执行命令工具
│   └── task.ts           # 子Agent工具
├── provider/
│   └── provider.ts       # LLM 提供商
├── permission/
│   └── next.ts           # 权限系统
└── mcp/
    └── index.ts          # MCP 协议
```

### 下一步学习建议

1. **深入源码**：阅读 OpenCode 的完整实现
2. **贡献代码**：参与 OpenCode 的开发
3. **构建项目**：用所学知识构建自己的 Agent 应用
4. **关注发展**：跟踪 Agent 领域的最新进展

### 推荐资源

- [OpenCode 官方文档](https://opencode.ai/docs)
- [Vercel AI SDK](https://sdk.vercel.ai/docs)
- [MCP 协议规范](https://modelcontextprotocol.io/)
- [ReAct 论文](https://arxiv.org/abs/2210.03629)

---

## 恭喜完成！🎉

你已经完成了 OpenCode Agent 教程的全部内容。

现在你具备了：
- 理解 Agent 系统的核心原理
- 阅读和修改 Agent 代码的能力
- 创建自定义 Agent 和工具的技能
- 构建完整 Agent 应用的知识

继续探索，构建属于你的 AI Agent！

---

[← 上一章：高级主题](./05-advanced-topics.md) | [返回目录](./00-introduction.md)
