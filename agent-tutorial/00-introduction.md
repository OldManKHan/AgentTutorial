# OpenCode Agent 完整教程

> 🎯 面向 Agent 小白的系统教程，理论与实践并重

## 写在前面

欢迎来到 AI Agent 的世界！

如果你是一个开发者，已经用过 ChatGPT、Claude 这类 AI 助手，你可能会好奇：

- 为什么 AI 能帮我写代码、读文件、执行命令？
- 它是怎么"思考"并决定下一步做什么的？
- 我能不能自己做一个这样的 AI 助手？

这份教程将通过 **OpenCode** —— 一个开源的 AI 编程助手项目，带你从零理解 Agent 的工作原理，并学会如何开发自己的 Agent。

---

## 为什么选择 OpenCode 作为学习项目？

| 特点 | 说明 |
|------|------|
| 100% 开源 | 代码完全公开，可以看到每一个实现细节 |
| 生产级质量 | 不是玩具项目，是真正可用的产品 |
| 架构清晰 | 模块划分合理，代码质量高 |
| 功能完整 | 涵盖了 Agent 开发的所有核心概念 |
| 活跃维护 | 持续更新，代表最新的 Agent 开发实践 |

---

## 教程结构

本教程共 6 章，建议学习时间 2-3 周：

| 章节 | 内容 | 时间 |
|------|------|------|
| [第一章](./01-what-is-agent.md) | Agent 是什么 | 1-2 天 |
| [第二章](./02-tools.md) | 工具系统 (Tools) | 2-3 天 |
| [第三章](./03-agent-loop.md) | Agent Loop 核心循环 | 2-3 天 |
| [第四章](./04-prompt-engineering.md) | Prompt Engineering | 2-3 天 |
| [第五章](./05-advanced-topics.md) | 高级主题 | 3-4 天 |
| [第六章](./06-projects.md) | 实战项目 | 3-5 天 |

---

## 学习前提

**必须具备：**
- 基础的 TypeScript/JavaScript 知识
- 了解 async/await 异步编程
- 基本的命令行操作能力

**建议了解：**
- 用过 OpenAI/Claude API
- 了解 HTTP 请求和 JSON

---

## 环境准备

### 1. 克隆项目

```bash
git clone https://github.com/anomalyco/opencode.git
cd opencode
git checkout dev  # 切换到开发分支
```

### 2. 安装依赖

```bash
# 安装 bun (如果没有)
curl -fsSL https://bun.sh/install | bash

# 安装项目依赖
bun install
```

### 3. 配置 API Key

设置环境变量（选择一个你有的）：

```bash
# Anthropic Claude
export ANTHROPIC_API_KEY="your-api-key"

# 或 OpenAI
export OPENAI_API_KEY="your-api-key"

# 或 Google
export GOOGLE_GENERATIVE_AI_API_KEY="your-api-key"
```

### 4. 启动开发模式

```bash
cd packages/opencode
bun dev
```

看到终端界面就说明成功了！

---

## 如何使用这份教程

1. **按顺序学习**：每一章都建立在前一章的基础上
2. **边读边做**：每章都有「动手实验」环节，一定要实际操作
3. **看代码**：教程会指向具体的源码文件，务必打开看
4. **做笔记**：记录你的理解和疑问

---

## 目录

- [第一章：Agent 是什么](./01-what-is-agent.md)
- [第二章：工具系统](./02-tools.md)
- [第三章：Agent Loop 核心循环](./03-agent-loop.md)
- [第四章：Prompt Engineering](./04-prompt-engineering.md)
- [第五章：高级主题](./05-advanced-topics.md)
- [第六章：实战项目](./06-projects.md)

---

准备好了吗？让我们开始吧！

[下一章：Agent 是什么 →](./01-what-is-agent.md)
