# 第一章：Agent 是什么

> 本章目标：理解 Agent 的核心概念，建立对 AI Agent 的整体认知

---

## 1.1 从 ChatGPT 到 Agent

### 普通聊天机器人的局限

当你使用 ChatGPT 网页版时，它只能：
- 回答问题
- 生成文本
- 进行对话

但它**不能**：
- 读取你电脑上的文件
- 执行命令
- 访问互联网获取实时信息
- 修改代码

这是因为普通的 LLM（大语言模型）只是一个"大脑"，它没有"手脚"去与外部世界交互。

### Agent = LLM + 行动能力

**Agent（智能体）** 是在 LLM 基础上，赋予它与外部世界交互能力的系统。

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   普通 LLM:    用户 ──▶ LLM ──▶ 文本回复                    │
│                                                             │
│   Agent:       用户 ──▶ LLM ──▶ 思考 ──▶ 行动 ──▶ 观察     │
│                              ◀────────────────────┘        │
│                                    (循环)                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 一个具体的例子

假设你对 Agent 说："帮我看看 package.json 里的依赖版本"

**普通 LLM 的回答：**
> "我无法访问你的文件系统，请把 package.json 的内容粘贴给我。"

**Agent 的行为：**
1. 思考：用户想知道依赖版本，我需要读取 package.json
2. 行动：调用 `read` 工具读取文件
3. 观察：获取到文件内容
4. 回答：基于文件内容告诉用户依赖版本

这就是 Agent 的核心价值：**它能自主决定采取什么行动来完成任务**。

---

## 1.2 Agent 的核心组成

一个完整的 Agent 系统由以下部分组成：

```
┌─────────────────────────────────────────────────────────────┐
│                        Agent 系统                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    LLM (大脑)                        │   │
│  │                                                     │   │
│  │  • 理解用户意图                                      │   │
│  │  • 决定下一步行动                                    │   │
│  │  • 分析工具执行结果                                  │   │
│  │  • 生成最终回复                                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  Tools (工具/手脚)                   │   │
│  │                                                     │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │   │
│  │  │  read   │ │  write  │ │  bash   │ │  search │   │   │
│  │  │ 读文件  │ │ 写文件  │ │执行命令 │ │  搜索   │   │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │               Agent Loop (控制循环)                  │   │
│  │                                                     │   │
│  │  思考 ──▶ 行动 ──▶ 观察 ──▶ 思考 ──▶ ... ──▶ 完成  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 三大核心组件

| 组件 | 作用 | 类比 |
|------|------|------|
| **LLM** | 理解、推理、决策 | 大脑 |
| **Tools** | 与外部世界交互 | 手脚 |
| **Agent Loop** | 协调 LLM 和 Tools | 神经系统 |

### 关键概念：Function Calling / Tool Use

现代 LLM（如 Claude、GPT-4）都支持一种叫 **Function Calling** 或 **Tool Use** 的能力。

简单说，就是 LLM 可以输出一种特殊格式，表示"我想调用某个工具"：

```json
{
  "tool": "read",
  "arguments": {
    "filePath": "package.json"
  }
}
```

Agent 系统检测到这种输出后，会：
1. 解析出要调用的工具和参数
2. 执行工具
3. 把结果返回给 LLM
4. LLM 继续思考下一步

这就是 Agent 能"行动"的技术基础。

---

## 1.3 OpenCode 中的 Agent

现在让我们看看 OpenCode 是如何实现 Agent 的。

### 核心文件位置

```
packages/opencode/src/
├── agent/
│   └── agent.ts        ← Agent 定义
├── session/
│   ├── index.ts        ← 会话管理
│   ├── llm.ts          ← LLM 调用
│   └── processor.ts    ← Agent Loop 实现
├── tool/
│   ├── tool.ts         ← 工具基础定义
│   ├── read.ts         ← 读文件工具
│   ├── bash.ts         ← 执行命令工具
│   └── ...             ← 其他工具
└── provider/
    └── provider.ts     ← LLM 提供商管理
```

### Agent 的定义

打开 `packages/opencode/src/agent/agent.ts`，你会看到 Agent 的类型定义：

```typescript
// 简化版，突出核心字段
export const Info = z.object({
  name: z.string(),                    // Agent 名称，如 "build"
  description: z.string().optional(),  // 描述，告诉 LLM 这个 Agent 的用途
  mode: z.enum(["subagent", "primary", "all"]),  // 运行模式
  
  // 权限控制：决定 Agent 能使用哪些工具
  permission: PermissionNext.Ruleset,
  
  // 可选：指定使用的模型
  model: z.object({
    modelID: z.string(),
    providerID: z.string(),
  }).optional(),
  
  // 自定义系统提示词
  prompt: z.string().optional(),
  
  // LLM 参数
  temperature: z.number().optional(),
  topP: z.number().optional(),
})
```

**关键理解：**
- `name`：Agent 的唯一标识
- `permission`：控制 Agent 能做什么（非常重要！）
- `prompt`：定制 Agent 的行为风格
- `model`：可以为不同 Agent 指定不同的 LLM

### OpenCode 的内置 Agent

OpenCode 预定义了几个 Agent，各有不同的用途：

```typescript
// 来自 agent.ts 的内置 Agent 定义

const result: Record<string, Info> = {
  // 1. build - 默认的开发 Agent
  build: {
    name: "build",
    permission: PermissionNext.merge(defaults, user),  // 全权限
    mode: "primary",
    native: true,
  },
  
  // 2. plan - 只读分析 Agent
  plan: {
    name: "plan",
    permission: PermissionNext.merge(
      defaults,
      PermissionNext.fromConfig({
        edit: {
          "*": "deny",  // 禁止编辑
          ".opencode/plan/*.md": "allow",  // 只能写计划文件
        },
      }),
      user,
    ),
    mode: "primary",
    native: true,
  },
  
  // 3. general - 通用子 Agent
  general: {
    name: "general",
    description: `General-purpose agent for researching complex questions 
                  and executing multi-step tasks.`,
    permission: PermissionNext.merge(
      defaults,
      PermissionNext.fromConfig({
        todoread: "deny",
        todowrite: "deny",
      }),
      user,
    ),
    mode: "subagent",  // 作为子 Agent 被调用
    native: true,
    hidden: true,
  },
  
  // 4. explore - 代码探索 Agent
  explore: {
    name: "explore",
    permission: PermissionNext.merge(
      defaults,
      PermissionNext.fromConfig({
        "*": "deny",
        grep: "allow",
        glob: "allow",
        list: "allow",
        bash: "allow",
        read: "allow",
        // ... 只允许搜索类工具
      }),
      user,
    ),
    description: `Fast agent specialized for exploring codebases.`,
    mode: "subagent",
    native: true,
  },
}
```

### Agent 对比表

| Agent | 用途 | 权限 | 使用场景 |
|-------|------|------|---------|
| `build` | 开发构建 | 全权限 | 写代码、执行命令、修改文件 |
| `plan` | 规划分析 | 只读 | 分析代码、制定计划，不修改 |
| `general` | 子任务 | 受限 | 被其他 Agent 调用执行子任务 |
| `explore` | 代码探索 | 只读+搜索 | 快速搜索和理解代码库 |

---

## 1.4 动手实验：体验不同 Agent

### 实验 1：切换 Agent

1. 启动 OpenCode：
```bash
cd packages/opencode && bun dev
```

2. 默认使用的是 `build` Agent

3. 按 `Tab` 键切换到 `plan` Agent

4. 观察界面上 Agent 名称的变化

### 实验 2：感受权限差异

1. 在 `build` Agent 下，输入：
```
创建一个文件 test.txt，内容是 hello world
```
→ Agent 会成功创建文件

2. 切换到 `plan` Agent，输入同样的请求
→ Agent 会拒绝，因为没有写权限

### 实验 3：查看 Agent 配置

在 OpenCode 中输入：
```
/config
```
可以看到当前的配置，包括 Agent 相关设置。

---

## 1.5 知识补充：Agent 的学术背景

### ReAct 模式

OpenCode 使用的 Agent 模式基于 **ReAct**（Reasoning + Acting）论文。

核心思想是让 LLM 交替进行：
- **Reasoning（推理）**：思考当前情况，决定下一步
- **Acting（行动）**：执行具体操作
- **Observation（观察）**：获取行动结果

```
Thought: 用户想知道 package.json 的内容，我需要读取这个文件
Action: read(filePath="package.json")
Observation: {"name": "opencode", "version": "1.0.0", ...}
Thought: 我已经获取到文件内容，可以回答用户了
Answer: package.json 的内容是...
```

### 为什么 ReAct 有效？

1. **可解释性**：每一步都有明确的思考过程
2. **可控性**：可以在任何一步介入或修正
3. **灵活性**：可以处理复杂的多步骤任务

### 其他 Agent 模式

| 模式 | 特点 | 适用场景 |
|------|------|---------|
| ReAct | 推理+行动交替 | 通用任务 |
| Plan-and-Execute | 先规划后执行 | 复杂任务 |
| Tree of Thoughts | 多路径探索 | 需要回溯的任务 |
| Multi-Agent | 多个 Agent 协作 | 大型复杂任务 |

OpenCode 主要使用 ReAct 模式，但也支持 Multi-Agent（通过 `task` 工具）。

---

## 1.6 本章小结

### 核心概念回顾

1. **Agent = LLM + Tools + Loop**
   - LLM 负责思考和决策
   - Tools 负责与外部世界交互
   - Loop 协调整个过程

2. **Function Calling** 是 Agent 能"行动"的技术基础

3. **权限控制** 是 Agent 安全性的关键

### OpenCode 代码对照

| 概念 | OpenCode 实现 | 文件位置 |
|------|--------------|---------|
| Agent 定义 | `Agent.Info` | `src/agent/agent.ts` |
| 工具系统 | `Tool.define()` | `src/tool/tool.ts` |
| Agent Loop | `SessionProcessor` | `src/session/processor.ts` |
| LLM 调用 | `LLM.stream()` | `src/session/llm.ts` |

### 检查清单

- [ ] 理解 Agent 和普通 LLM 的区别
- [ ] 知道 Agent 的三大核心组件
- [ ] 了解 OpenCode 的内置 Agent 及其差异
- [ ] 完成了动手实验

---

[← 返回目录](./00-introduction.md) | [下一章：工具系统 →](./02-tools.md)
