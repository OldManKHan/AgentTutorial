# OpenCode Agent 教程

## 项目简介

这是一个基于 [OpenCode](https://github.com/anomalyco/opencode.git) 开源项目的 AI Agent 完整教程。教程面向 Agent 小白，通过系统讲解理论与实践，帮助你从零理解 AI Agent 的工作原理，并学会如何开发自己的 Agent。

OpenCode 是一个开源的 AI 编程助手项目，具有 100% 开源、生产级质量、架构清晰、功能完整等特点，非常适合作为学习 Agent 开发的入门项目。

## 教程结构

本教程分为基础教程和高级教程两部分。基础教程面向 Agent 小白，高级教程面向有基础的开发者。

### 基础教程

基础教程共分为 6 章，建议学习时间为 2-3 周：

1. **[第一章：Agent 是什么](./agent-tutorial/01-what-is-agent.md)** - 介绍 Agent 的基本概念和 OpenCode 项目
2. **[第二章：工具系统 (Tools)](./agent-tutorial/02-tools.md)** - 深入理解 Agent 的核心组成
3. **[第三章：Agent Loop 核心循环](./agent-tutorial/03-agent-loop.md)** - 学习 Agent 的执行流程
4. **[第四章：Prompt Engineering](./agent-tutorial/04-prompt-engineering.md)** - 掌握提示词工程技巧
5. **[第五章：高级主题](./agent-tutorial/05-advanced-topics.md)** - 探索 Agent 开发的高级话题
6. **[第六章：实战项目](./agent-tutorial/06-projects.md)** - 通过项目实践巩固所学

### 高级教程

高级教程在基础教程的基础上，深入探讨 Agent 系统的核心技术，适合希望构建生产级 Agent 的开发者。共分为 6 章：

1. **[第一章：上下文工程](./advanced-agent-tutorial/01-context-engineering.md)** - 掌握上下文窗口管理技术，在有限 Token 预算内高效组织信息
2. **[第二章：Multi-Agent 系统](./advanced-agent-tutorial/02-multi-agent.md)** - 理解多 Agent 协作设计原理，构建多个 Agent 完成复杂任务
3. **[第三章：工具系统设计](./advanced-agent-tutorial/03-tool-system.md)** - 掌握 Agent 工具系统的设计原则，构建安全、可扩展的工具
4. **[第四章：代码智能](./advanced-agent-tutorial/04-code-intelligence.md)** - 理解代码智能对 Agent 的价值，掌握 LSP 协议核心概念
5. **[第五章：记忆系统](./advanced-agent-tutorial/05-memory-system.md)** - 理解 Agent 记忆系统设计，掌握向量数据库和 RAG 技术
6. **[第六章：可观测性与调试](./advanced-agent-tutorial/06-observability.md)** - 理解 Agent 系统可观测性，掌握链路追踪、日志设计和监控系统

## 学习前提

- 基础的 TypeScript/JavaScript 知识
- 了解 async/await 异步编程
- 有基本的 LLM API 调用经验（如 OpenAI API）

## 如何开始学习

1. 克隆本仓库到本地：
   ```bash
   git clone https://github.com/OldManKHan/AgentTutorial.git
   cd AgentTutorial
   ```

2. 按照章节顺序阅读 Markdown 文件，从 [基础教程第一章](./agent-tutorial/01-what-is-agent.md) 开始。

3. 每个章节都包含理论讲解和代码示例，建议边读边实践。

4. 完成基础教程后，可以继续学习 [高级教程](./advanced-agent-tutorial/00-introduction.md)。

5. 遇到问题时，可以参考 [OpenCode 项目](https://github.com/anomalyco/opencode.git) 的源代码。


## 贡献

欢迎提交 Issue 和 Pull Request 来改进这个教程！

## 许可证

本教程遵循 MIT 许可证。