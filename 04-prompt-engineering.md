# 第四章：Prompt Engineering

> 本章目标：掌握 Agent 系统中的 Prompt 设计技巧

---

## 4.1 System Prompt 的重要性

### 什么是 System Prompt

System Prompt（系统提示词）是发送给 LLM 的第一条消息，用于：
- 定义 AI 的身份和角色
- 设定行为规范和限制
- 提供背景知识和上下文
- 指导工具使用方式

```
┌─────────────────────────────────────────────────────────────┐
│                    消息结构                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ System Prompt (系统提示词)                          │   │
│  │                                                     │   │
│  │ "你是一个专业的编程助手，擅长..."                    │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ User Message (用户消息)                             │   │
│  │                                                     │   │
│  │ "帮我写一个排序函数"                                │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Assistant Response (AI 回复)                        │   │
│  │                                                     │   │
│  │ "好的，这是一个快速排序的实现..."                    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### System Prompt 的影响

| 方面 | 影响 |
|------|------|
| 回复风格 | 正式/随意、详细/简洁 |
| 专业领域 | 编程/写作/分析 |
| 行为边界 | 能做什么、不能做什么 |
| 工具使用 | 何时使用、如何使用 |

---

## 4.2 OpenCode 的 Prompt 结构

### Prompt 组成层次

```
┌─────────────────────────────────────────────────────────────┐
│                   OpenCode Prompt 结构                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Layer 1: Header (基础身份)                                 │
│  ├── 模型特定的身份设定                                     │
│  └── 基础能力声明                                           │
│                                                             │
│  Layer 2: Provider Prompt (模型优化)                        │
│  ├── 针对不同模型的优化指令                                 │
│  └── 模型特定的最佳实践                                     │
│                                                             │
│  Layer 3: Agent Prompt (Agent 定制)                         │
│  ├── Agent 特定的行为指令                                   │
│  └── 权限和限制说明                                         │
│                                                             │
│  Layer 4: Environment (环境信息)                            │
│  ├── 工作目录                                               │
│  ├── 操作系统                                               │
│  └── 项目类型                                               │
│                                                             │
│  Layer 5: Custom Instructions (用户自定义)                  │
│  ├── AGENTS.md / CLAUDE.md                                  │
│  └── 配置文件中的 instructions                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 代码实现

打开 `packages/opencode/src/session/system.ts`：

```typescript
export namespace SystemPrompt {
  // Layer 1: Header - 基础身份
  export function header(providerID: string) {
    if (providerID.includes("anthropic")) {
      return [PROMPT_ANTHROPIC_SPOOF.trim()]
    }
    return []
  }
  
  // Layer 2: Provider Prompt - 模型特定优化
  export function provider(model: Provider.Model) {
    if (model.api.id.includes("gpt-5")) return [PROMPT_CODEX]
    if (model.api.id.includes("gpt-")) return [PROMPT_BEAST]
    if (model.api.id.includes("gemini-")) return [PROMPT_GEMINI]
    if (model.api.id.includes("claude")) return [PROMPT_ANTHROPIC]
    return [PROMPT_ANTHROPIC_WITHOUT_TODO]
  }
  
  // Layer 4: Environment - 环境信息
  export async function environment() {
    return [
      `<env>`,
      `  Working directory: ${Instance.directory}`,
      `  Is directory a git repo: ${project.vcs === "git" ? "yes" : "no"}`,
      `  Platform: ${process.platform}`,
      `  Today's date: ${new Date().toDateString()}`,
      `</env>`,
    ].join("\n")
  }
  
  // Layer 5: Custom Instructions - 用户自定义
  export async function custom() {
    const paths = new Set<string>()
    
    // 查找 AGENTS.md, CLAUDE.md 等文件
    for (const localRuleFile of LOCAL_RULE_FILES) {
      const matches = await Filesystem.findUp(localRuleFile, Instance.directory)
      matches.forEach(path => paths.add(path))
    }
    
    // 读取并返回内容
    return Promise.all(
      Array.from(paths).map(p => Bun.file(p).text())
    )
  }
}
```


---

## 4.3 Prompt 组装过程

### LLM.stream() 中的 Prompt 组装

打开 `packages/opencode/src/session/llm.ts`：

```typescript
export async function stream(input: StreamInput) {
  // 1. 获取 Header
  const system = SystemPrompt.header(input.model.providerID)
  
  // 2. 组装主体 Prompt
  system.push(
    [
      // Agent 自定义 prompt（如果有）
      ...(input.agent.prompt 
        ? [input.agent.prompt] 
        : SystemPrompt.provider(input.model)),
      
      // 调用时传入的额外 prompt
      ...input.system,
      
      // 用户消息中的 system prompt
      ...(input.user.system ? [input.user.system] : []),
    ]
    .filter(x => x)
    .join("\n")
  )
  
  // 3. 插件可以修改 system prompt
  await Plugin.trigger("experimental.chat.system.transform", {}, { system })
  
  // 4. 发送给 LLM
  return streamText({
    messages: [
      // System prompt 作为第一条消息
      ...system.map(x => ({ role: "system", content: x })),
      // 对话历史
      ...input.messages,
    ],
    // ...
  })
}
```

### 完整的 Prompt 示例

一个实际发送给 LLM 的 prompt 可能是这样的：

```
[System Message 1 - Header]
You are Claude, an AI assistant created by Anthropic...

[System Message 2 - Main Prompt]
You are an expert software engineer with deep knowledge of...

When using tools:
- Always explain what you're about to do before calling a tool
- If a tool fails, try an alternative approach
- Use the read tool to examine files before editing them

<env>
  Working directory: /home/user/project
  Is directory a git repo: yes
  Platform: linux
  Today's date: Mon Jan 06 2025
</env>

Instructions from: /home/user/project/AGENTS.md
- Follow the coding style in this project
- Always run tests after making changes
- Use TypeScript for all new files
```

---

## 4.4 工具描述的最佳实践

### 工具描述的作用

工具描述是 System Prompt 的一部分，告诉 LLM：
- 工具能做什么
- 什么时候应该使用
- 如何正确调用

### 好的工具描述结构

```
1. 功能概述（一句话）
2. 使用场景（什么时候用）
3. 参数说明（每个参数的含义）
4. 使用示例（具体的调用例子）
5. 注意事项（限制、边界情况）
```

### 实际例子对比

❌ **差的描述**：
```
Reads a file from disk.
```

✅ **好的描述**（来自 `read.txt`）：
```
Reads a file from the local filesystem. Use this tool to access file 
contents when you need to examine code, configuration files, 
documentation, or any other text-based files.

IMPORTANT GUIDELINES:
- Always use relative paths from the current working directory
- For large files, use the offset and limit parameters to read in chunks
- The output includes line numbers for easy reference (e.g., "00001| content")
- Binary files cannot be read with this tool
- Files like .env are blocked for security reasons

PARAMETERS:
- filePath (required): Path to the file, relative or absolute
- offset (optional): Starting line number (0-based)
- limit (optional): Number of lines to read (default: 2000)

EXAMPLES:
- Read entire file: {"filePath": "src/index.ts"}
- Read from line 100: {"filePath": "src/index.ts", "offset": 100}
- Read 50 lines: {"filePath": "src/index.ts", "offset": 200, "limit": 50}

OUTPUT FORMAT:
<file>
00001| first line of content
00002| second line of content
...
</file>
```

### 描述技巧

| 技巧 | 说明 | 例子 |
|------|------|------|
| 明确场景 | 告诉 LLM 什么时候用 | "Use when you need to examine code" |
| 给出示例 | 具体的调用例子 | `{"filePath": "src/index.ts"}` |
| 说明限制 | 工具不能做什么 | "Binary files cannot be read" |
| 描述输出 | 输出格式是什么样 | "Output includes line numbers" |

---

## 4.5 Agent 特定 Prompt

### 不同 Agent 的 Prompt 差异

OpenCode 的不同 Agent 有不同的 prompt：

```typescript
// build Agent - 默认，全能型
const buildPrompt = `
You are a skilled software engineer...
You have full access to read, write, and execute commands.
`

// plan Agent - 只读分析
const planPrompt = `
You are a code analyst and planner...
You can only READ files, not modify them.
Focus on understanding and planning, not implementation.
`

// explore Agent - 快速搜索
const explorePrompt = `
You are a fast code explorer...
Use search tools efficiently to find relevant code.
Provide concise summaries of what you find.
`
```

### explore Agent 的 Prompt 示例

打开 `packages/opencode/src/agent/prompt/explore.txt`：

```
You are a fast, focused code exploration agent. Your job is to quickly 
find and summarize relevant code based on the user's query.

GUIDELINES:
1. Start with broad searches, then narrow down
2. Use glob for file patterns, grep for content search
3. Read only the most relevant files
4. Provide concise summaries, not full file contents
5. Note file paths so the user can explore further

SEARCH STRATEGY:
- "quick": 1-2 searches, surface-level results
- "medium": 3-5 searches, moderate depth
- "very thorough": 5+ searches, comprehensive coverage

Always state your search strategy at the beginning.
```

---

## 4.6 用户自定义指令

### AGENTS.md / CLAUDE.md

用户可以在项目根目录创建 `AGENTS.md` 或 `CLAUDE.md` 文件，添加项目特定的指令：

```markdown
# Project Instructions

## Coding Style
- Use TypeScript for all new files
- Follow the existing code style
- Add JSDoc comments for public functions

## Testing
- Run `bun test` after making changes
- Add tests for new features

## Git
- Use conventional commits
- Create feature branches for new work

## Project Structure
- Components go in `src/components/`
- Utilities go in `src/utils/`
- Types go in `src/types/`
```

### 配置文件中的 instructions

在 `~/.config/opencode/config.json` 中：

```json
{
  "instructions": [
    "~/global-instructions.md",
    ".cursor/rules.md"
  ]
}
```

### 指令加载顺序

```
1. 全局指令 (~/.config/opencode/AGENTS.md)
2. 项目指令 (./AGENTS.md)
3. 配置文件指令 (config.instructions)
```

后加载的指令优先级更高。


---

## 4.7 Prompt 优化技巧

### 1. 结构化指令

使用清晰的结构让 LLM 更容易理解：

```
❌ 不好：
你是一个编程助手，可以帮用户写代码，也可以读文件，
执行命令的时候要小心，不要删除重要文件...

✅ 好：
# Role
You are a programming assistant.

# Capabilities
- Read and write files
- Execute shell commands
- Search code

# Guidelines
1. Always explain before taking action
2. Be careful with destructive commands
3. Ask for confirmation when unsure
```

### 2. 使用 XML 标签

XML 标签帮助 LLM 区分不同类型的内容：

```
<env>
  Working directory: /home/user/project
  Platform: linux
</env>

<instructions>
  Follow the coding style in this project.
</instructions>

<context>
  The user is working on a React application.
</context>
```

### 3. 提供示例

Few-shot learning 非常有效：

```
When the user asks to create a file, respond like this:

User: Create a hello.ts file
Assistant: I'll create a TypeScript file with a simple hello world function.
[calls write tool with appropriate content]

User: Make a config file
Assistant: I'll create a configuration file. What format would you prefer 
(JSON, YAML, or TOML)?
```

### 4. 明确边界

告诉 LLM 什么不应该做：

```
RESTRICTIONS:
- Never modify files outside the project directory without asking
- Never execute commands that could delete data (rm -rf, etc.) without confirmation
- Never expose sensitive information like API keys
- If unsure, ask the user for clarification
```

### 5. 处理歧义

指导 LLM 如何处理不明确的请求：

```
When the user's request is ambiguous:
1. State your interpretation
2. Ask for confirmation if the action is significant
3. Provide alternatives if multiple approaches exist

Example:
User: "Fix the bug"
Assistant: "I see several potential issues. Could you clarify which one:
1. The null pointer exception in auth.ts
2. The styling issue in Header.tsx
3. The failing test in utils.test.ts"
```

---

## 4.8 动手实验：优化 Agent 行为

### 实验目标

通过修改 prompt，改变 Agent 的行为风格。

### 实验 1：创建简洁风格的 Agent

创建 `~/.config/opencode/config.json`：

```json
{
  "agent": {
    "concise": {
      "prompt": "You are a concise coding assistant. Rules:\n1. Give short, direct answers\n2. No unnecessary explanations\n3. Code only, minimal comments\n4. One-line responses when possible",
      "description": "A concise agent that gives brief responses"
    }
  }
}
```

测试：
```
# 使用默认 Agent
> 写一个函数计算斐波那契数列

# 使用 concise Agent
> @concise 写一个函数计算斐波那契数列
```

比较两者的回复长度和风格。

### 实验 2：创建教学风格的 Agent

```json
{
  "agent": {
    "teacher": {
      "prompt": "You are a patient programming teacher. Rules:\n1. Explain concepts step by step\n2. Use analogies to clarify complex ideas\n3. Ask questions to check understanding\n4. Provide exercises for practice",
      "description": "A teaching agent that explains concepts thoroughly"
    }
  }
}
```

### 实验 3：项目特定指令

在项目根目录创建 `AGENTS.md`：

```markdown
# Project: My React App

## Tech Stack
- React 18 with TypeScript
- Tailwind CSS for styling
- Zustand for state management

## Coding Standards
- Use functional components only
- Prefer hooks over class components
- Use `cn()` utility for conditional classes

## File Naming
- Components: PascalCase (Button.tsx)
- Utilities: camelCase (formatDate.ts)
- Types: PascalCase with .types.ts suffix

## When Creating Components
1. Create in src/components/
2. Export from index.ts
3. Add basic props interface
4. Include a simple test
```

测试 Agent 是否遵循这些指令。

---

## 4.9 知识补充：Prompt Engineering 原则

### 1. 清晰性原则

```
❌ 模糊: "帮我处理一下这个文件"
✅ 清晰: "读取 config.json，将 port 值从 3000 改为 8080"
```

### 2. 具体性原则

```
❌ 抽象: "写好一点的代码"
✅ 具体: "添加错误处理，使用 try-catch 包装异步操作"
```

### 3. 分步原则

复杂任务分解为小步骤：

```
❌ 一步到位: "创建一个完整的用户认证系统"
✅ 分步进行:
1. "创建 User 类型定义"
2. "实现登录函数"
3. "添加 token 验证中间件"
4. "编写测试用例"
```

### 4. 上下文原则

提供足够的背景信息：

```
❌ 缺少上下文: "修复这个 bug"
✅ 有上下文: "用户报告登录后页面空白。错误日志显示 
   'Cannot read property of undefined'。相关代码在 auth.ts"
```

### 5. 约束原则

明确限制条件：

```
✅ 有约束:
- "使用 TypeScript，不要用 any"
- "保持向后兼容"
- "不要引入新的依赖"
- "代码行数控制在 50 行以内"
```

---

## 4.10 本章小结

### 核心概念回顾

1. **System Prompt** 定义 Agent 的身份和行为
2. **Prompt 分层**：Header → Provider → Agent → Environment → Custom
3. **工具描述**直接影响 LLM 的工具使用效果
4. **用户指令**可以通过 AGENTS.md 或配置文件添加

### OpenCode Prompt 架构

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  SystemPrompt.header()     ──▶  基础身份                    │
│         +                                                   │
│  SystemPrompt.provider()   ──▶  模型优化                    │
│         +                                                   │
│  Agent.prompt              ──▶  Agent 定制                  │
│         +                                                   │
│  SystemPrompt.environment()──▶  环境信息                    │
│         +                                                   │
│  SystemPrompt.custom()     ──▶  用户指令                    │
│         =                                                   │
│  完整的 System Prompt                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 检查清单

- [ ] 理解 System Prompt 的作用和结构
- [ ] 知道 OpenCode 如何组装 Prompt
- [ ] 掌握工具描述的最佳实践
- [ ] 学会使用 AGENTS.md 添加项目指令
- [ ] 完成了 Agent 行为优化实验

---

[← 上一章：Agent Loop](./03-agent-loop.md) | [下一章：高级主题 →](./05-advanced-topics.md)
