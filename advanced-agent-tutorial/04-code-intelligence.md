# 第四章：代码智能

> 🎯 本章目标：理解代码智能对 Agent 的价值，掌握 LSP 协议的核心概念，学会让 Agent 真正"理解"代码

## 什么是代码智能？

想象一下，你在使用 VS Code 时：
- 鼠标悬停在函数上，自动显示类型和文档
- 按下 F12，直接跳转到定义
- 输入 `.`，智能提示可用的方法和属性
- 保存文件，立即看到语法错误和警告

这些功能的背后，就是**代码智能**（Code Intelligence）。

```
┌─────────────────────────────────────────────────────────────┐
│                    代码智能能力一览                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  语法层面：                                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  • 语法高亮：区分关键字、变量、字符串等             │   │
│  │  • 语法检查：发现拼写错误、缺少分号等               │   │
│  │  • 代码折叠：按结构折叠代码块                       │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  语义层面：                                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  • 类型检查：发现类型不匹配、未定义变量等           │   │
│  │  • 智能补全：根据上下文提示可用的方法和属性         │   │
│  │  • 悬停信息：显示类型、文档、参数说明               │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  导航层面：                                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  • 跳转定义：从使用处跳转到定义处                   │   │
│  │  • 查找引用：找到所有使用某符号的位置               │   │
│  │  • 符号搜索：在项目中搜索函数、类、变量             │   │
│  │  • 调用层次：查看函数的调用关系                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  重构层面：                                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  • 重命名：安全地重命名符号及其所有引用             │   │
│  │  • 提取函数：将代码块提取为独立函数                 │   │
│  │  • 代码修复：自动修复常见问题                       │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 代码智能对 Agent 的价值

对于 AI Agent 来说，代码智能不是"锦上添花"，而是"雪中送炭"。


```
┌─────────────────────────────────────────────────────────────┐
│                Agent 为什么需要代码智能？                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  问题 1：LLM 的"幻觉"问题                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  用户: 帮我调用 user.getName() 方法                 │   │
│  │                                                     │   │
│  │  没有代码智能的 Agent:                              │   │
│  │  → 直接生成 user.getName()                          │   │
│  │  → 但实际上方法名是 user.getFullName() ❌           │   │
│  │                                                     │   │
│  │  有代码智能的 Agent:                                │   │
│  │  → 查询 user 对象的可用方法                         │   │
│  │  → 发现只有 getFullName()，没有 getName()           │   │
│  │  → 生成正确的代码 ✅                                │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  问题 2：大型代码库的理解                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  用户: 帮我修改 handleSubmit 函数                   │   │
│  │                                                     │   │
│  │  没有代码智能的 Agent:                              │   │
│  │  → 需要读取大量文件来找到函数位置                   │   │
│  │  → 不确定函数被哪些地方调用                         │   │
│  │  → 修改可能破坏其他功能 ❌                          │   │
│  │                                                     │   │
│  │  有代码智能的 Agent:                                │   │
│  │  → 使用符号搜索直接定位函数                         │   │
│  │  → 使用引用查找了解所有调用点                       │   │
│  │  → 安全地进行修改 ✅                                │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  问题 3：实时错误反馈                                       │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  没有代码智能的 Agent:                              │   │
│  │  → 生成代码后需要运行才能发现错误                   │   │
│  │  → 错误信息可能不够详细                             │   │
│  │  → 修复-运行-修复循环耗时 ❌                        │   │
│  │                                                     │   │
│  │  有代码智能的 Agent:                                │   │
│  │  → 生成代码后立即获得诊断信息                       │   │
│  │  → 类型错误、未定义变量等即时发现                   │   │
│  │  → 一次性生成正确代码 ✅                            │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

| 能力 | 没有代码智能 | 有代码智能 |
|------|-------------|-----------|
| 定位代码 | 全文搜索，可能找错 | 精确符号定位 |
| 理解类型 | 靠猜测和推断 | 精确类型信息 |
| 发现错误 | 运行时才知道 | 编辑时即知道 |
| 重构代码 | 可能遗漏引用 | 完整引用分析 |
| 代码补全 | 基于模式匹配 | 基于语义理解 |

---

## 1. LSP 协议入门

### 1.1 什么是 LSP？

LSP（Language Server Protocol）是微软在 2016 年提出的开放协议，用于标准化编辑器/IDE 与语言服务器之间的通信。

在 LSP 出现之前，每个编辑器都需要为每种语言单独实现代码智能功能：

```
┌─────────────────────────────────────────────────────────────┐
│                    LSP 之前 vs LSP 之后                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  LSP 之前：M × N 问题                                       │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                     │   │
│  │  VS Code ──┬── TypeScript 支持                      │   │
│  │            ├── Python 支持                          │   │
│  │            └── Go 支持                              │   │
│  │                                                     │   │
│  │  Vim ──────┬── TypeScript 支持  (重复实现)          │   │
│  │            ├── Python 支持      (重复实现)          │   │
│  │            └── Go 支持          (重复实现)          │   │
│  │                                                     │   │
│  │  Emacs ────┬── TypeScript 支持  (重复实现)          │   │
│  │            ├── Python 支持      (重复实现)          │   │
│  │            └── Go 支持          (重复实现)          │   │
│  │                                                     │   │
│  │  3 个编辑器 × 3 种语言 = 9 个实现                   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  LSP 之后：M + N 问题                                       │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                     │   │
│  │  VS Code ──┐     ┌── TypeScript Language Server     │   │
│  │            │     │                                  │   │
│  │  Vim ──────┼─LSP─┼── Python Language Server         │   │
│  │            │     │                                  │   │
│  │  Emacs ────┘     └── Go Language Server             │   │
│  │                                                     │   │
│  │  3 个编辑器 + 3 个语言服务器 = 6 个实现             │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```


### 1.2 LSP 架构

LSP 采用客户端-服务器架构：

```
┌─────────────────────────────────────────────────────────────┐
│                    LSP 架构详解                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────┐         ┌─────────────────────────┐   │
│  │   LSP Client    │         │    Language Server      │   │
│  │   (编辑器/IDE)  │         │    (语言服务器)         │   │
│  │                 │         │                         │   │
│  │  ┌───────────┐  │  JSON   │  ┌─────────────────┐   │   │
│  │  │ 用户界面  │  │   RPC   │  │  语言分析引擎   │   │   │
│  │  └───────────┘  │ ◄─────► │  └─────────────────┘   │   │
│  │        │        │         │          │             │   │
│  │        ▼        │         │          ▼             │   │
│  │  ┌───────────┐  │         │  ┌─────────────────┐   │   │
│  │  │ LSP 客户端│  │         │  │  类型检查器     │   │   │
│  │  │ 协议实现  │  │         │  └─────────────────┘   │   │
│  │  └───────────┘  │         │          │             │   │
│  │                 │         │          ▼             │   │
│  │                 │         │  ┌─────────────────┐   │   │
│  │                 │         │  │  符号索引       │   │   │
│  │                 │         │  └─────────────────┘   │   │
│  └─────────────────┘         └─────────────────────────┘   │
│                                                             │
│  通信方式：                                                 │
│  • stdio：标准输入输出（最常用）                           │
│  • socket：TCP/IP 连接                                     │
│  • pipe：命名管道                                          │
│                                                             │
│  消息格式：                                                 │
│  • 基于 JSON-RPC 2.0                                       │
│  • 请求-响应模式                                           │
│  • 通知模式（无需响应）                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.3 LSP 核心能力

LSP 定义了丰富的语言特性，主要分为以下几类：

| 类别 | 能力 | 说明 |
|------|------|------|
| 文档同步 | `textDocument/didOpen` | 通知服务器文件已打开 |
| | `textDocument/didChange` | 通知服务器文件内容变化 |
| | `textDocument/didClose` | 通知服务器文件已关闭 |
| 诊断 | `textDocument/publishDiagnostics` | 服务器推送诊断信息 |
| 补全 | `textDocument/completion` | 获取代码补全建议 |
| 悬停 | `textDocument/hover` | 获取悬停信息 |
| 定义 | `textDocument/definition` | 跳转到定义 |
| 引用 | `textDocument/references` | 查找所有引用 |
| 符号 | `textDocument/documentSymbol` | 获取文档符号 |
| | `workspace/symbol` | 搜索工作区符号 |
| 调用层次 | `callHierarchy/incomingCalls` | 查找调用者 |
| | `callHierarchy/outgoingCalls` | 查找被调用者 |

### 1.4 LSP 通信协议

LSP 使用 JSON-RPC 2.0 协议进行通信：

```typescript
// 请求消息格式
interface RequestMessage {
  jsonrpc: "2.0"
  id: number | string      // 请求 ID，用于匹配响应
  method: string           // 方法名，如 "textDocument/definition"
  params?: object          // 参数
}

// 响应消息格式
interface ResponseMessage {
  jsonrpc: "2.0"
  id: number | string      // 对应请求的 ID
  result?: any             // 成功时的结果
  error?: {                // 失败时的错误
    code: number
    message: string
    data?: any
  }
}

// 通知消息格式（无需响应）
interface NotificationMessage {
  jsonrpc: "2.0"
  method: string
  params?: object
}
```

**示例：跳转到定义**

```json
// 客户端请求
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "textDocument/definition",
  "params": {
    "textDocument": { "uri": "file:///src/index.ts" },
    "position": { "line": 10, "character": 15 }
  }
}

// 服务器响应
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "uri": "file:///src/utils.ts",
    "range": {
      "start": { "line": 5, "character": 0 },
      "end": { "line": 5, "character": 20 }
    }
  }
}
```

---

## 2. 代码诊断

### 2.1 什么是代码诊断？

代码诊断是 LSP 最重要的功能之一，它能在你编写代码时实时发现问题。

```
┌─────────────────────────────────────────────────────────────┐
│                    诊断信息结构                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  interface Diagnostic {                                     │
│    range: Range           // 问题位置（起始行列到结束行列） │
│    severity: number       // 严重程度                       │
│    code?: string | number // 错误代码                       │
│    source?: string        // 来源（如 "typescript"）        │
│    message: string        // 错误描述                       │
│  }                                                          │
│                                                             │
│  严重程度：                                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  1 = Error   → 必须修复的错误                       │   │
│  │  2 = Warning → 警告，建议修复                       │   │
│  │  3 = Info    → 信息提示                             │   │
│  │  4 = Hint    → 轻微提示                             │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  示例：                                                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  {                                                  │   │
│  │    "range": {                                       │   │
│  │      "start": { "line": 10, "character": 5 },       │   │
│  │      "end": { "line": 10, "character": 15 }         │   │
│  │    },                                               │   │
│  │    "severity": 1,                                   │   │
│  │    "code": "2304",                                  │   │
│  │    "source": "typescript",                          │   │
│  │    "message": "Cannot find name 'userName'."        │   │
│  │  }                                                  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```


### 2.2 Agent 如何利用诊断

对于 Agent 来说，诊断信息是验证代码正确性的重要手段：

```
┌─────────────────────────────────────────────────────────────┐
│                Agent 利用诊断的工作流                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 生成代码                                                │
│     ┌─────────────────────────────────────────────────┐    │
│     │  Agent 根据用户需求生成代码                     │    │
│     │  → 写入文件 src/feature.ts                      │    │
│     └─────────────────────────────────────────────────┘    │
│                          │                                  │
│                          ▼                                  │
│  2. 触发诊断                                                │
│     ┌─────────────────────────────────────────────────┐    │
│     │  通知 LSP 服务器文件已更新                      │    │
│     │  → textDocument/didOpen 或 didChange            │    │
│     └─────────────────────────────────────────────────┘    │
│                          │                                  │
│                          ▼                                  │
│  3. 等待诊断结果                                            │
│     ┌─────────────────────────────────────────────────┐    │
│     │  LSP 服务器分析代码                             │    │
│     │  → 推送 textDocument/publishDiagnostics         │    │
│     └─────────────────────────────────────────────────┘    │
│                          │                                  │
│                          ▼                                  │
│  4. 处理诊断结果                                            │
│     ┌─────────────────────────────────────────────────┐    │
│     │  if (有错误) {                                  │    │
│     │    分析错误信息                                 │    │
│     │    修复代码                                     │    │
│     │    重复步骤 1-4                                 │    │
│     │  } else {                                       │    │
│     │    代码验证通过 ✅                              │    │
│     │  }                                              │    │
│     └─────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. 符号与导航

### 3.1 符号查找

符号是代码中有意义的命名实体，如函数、类、变量、接口等。

```
┌─────────────────────────────────────────────────────────────┐
│                    符号类型（SymbolKind）                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  常用符号类型：                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  1  = File          文件                            │   │
│  │  2  = Module        模块                            │   │
│  │  3  = Namespace     命名空间                        │   │
│  │  5  = Class         类                              │   │
│  │  6  = Method        方法                            │   │
│  │  7  = Property      属性                            │   │
│  │  8  = Field         字段                            │   │
│  │  9  = Constructor   构造函数                        │   │
│  │  10 = Enum          枚举                            │   │
│  │  11 = Interface     接口                            │   │
│  │  12 = Function      函数                            │   │
│  │  13 = Variable      变量                            │   │
│  │  14 = Constant      常量                            │   │
│  │  23 = Struct        结构体                          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  两种符号查询方式：                                         │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  documentSymbol：获取单个文件中的所有符号           │   │
│  │  → 用于文件大纲、快速导航                          │   │
│  │                                                     │   │
│  │  workspaceSymbol：在整个工作区搜索符号              │   │
│  │  → 用于全局搜索、跨文件导航                        │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 定义跳转与引用查找

这是 Agent 理解代码结构最重要的两个能力：

```
┌─────────────────────────────────────────────────────────────┐
│                定义跳转 vs 引用查找                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  goToDefinition（跳转到定义）：                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  从使用处 → 定义处                                  │   │
│  │                                                     │   │
│  │  // 使用处                                          │   │
│  │  const result = calculateSum(a, b)                  │   │
│  │                  ^^^^^^^^^^^^                       │   │
│  │                  光标在这里，按 F12                 │   │
│  │                        │                            │   │
│  │                        ▼                            │   │
│  │  // 定义处                                          │   │
│  │  function calculateSum(x: number, y: number) {      │   │
│  │    return x + y                                     │   │
│  │  }                                                  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  findReferences（查找引用）：                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  从定义处 → 所有使用处                              │   │
│  │                                                     │   │
│  │  // 定义处                                          │   │
│  │  function calculateSum(x, y) { ... }                │   │
│  │           ^^^^^^^^^^^^                              │   │
│  │           光标在这里，查找引用                      │   │
│  │                        │                            │   │
│  │                        ▼                            │   │
│  │  // 所有使用处                                      │   │
│  │  • src/math.ts:15 - calculateSum(1, 2)              │   │
│  │  • src/calc.ts:8  - calculateSum(a, b)              │   │
│  │  • test/math.test.ts:20 - calculateSum(0, 0)        │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 调用层次

调用层次帮助理解函数之间的调用关系：

```
┌─────────────────────────────────────────────────────────────┐
│                    调用层次分析                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  incomingCalls（谁调用了我？）：                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                     │   │
│  │  handleClick() ──┐                                  │   │
│  │                  │                                  │   │
│  │  onSubmit() ─────┼──► validateForm() ◄── 目标函数  │   │
│  │                  │                                  │   │
│  │  processData() ──┘                                  │   │
│  │                                                     │   │
│  │  结果：handleClick, onSubmit, processData          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  outgoingCalls（我调用了谁？）：                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                     │   │
│  │                  ┌──► checkEmail()                  │   │
│  │                  │                                  │   │
│  │  validateForm() ─┼──► checkPassword()               │   │
│  │  ▲               │                                  │   │
│  │  目标函数        └──► showError()                   │   │
│  │                                                     │   │
│  │  结果：checkEmail, checkPassword, showError        │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. AST 与代码理解

### 4.1 什么是 AST？

AST（Abstract Syntax Tree，抽象语法树）是源代码的树状结构表示。

```
┌─────────────────────────────────────────────────────────────┐
│                    代码到 AST 的转换                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  源代码：                                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  function add(a, b) {                               │   │
│  │    return a + b;                                    │   │
│  │  }                                                  │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│                          ▼                                  │
│  AST 结构：                                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Program                                            │   │
│  │  └── FunctionDeclaration                            │   │
│  │      ├── name: "add"                                │   │
│  │      ├── params:                                    │   │
│  │      │   ├── Identifier: "a"                        │   │
│  │      │   └── Identifier: "b"                        │   │
│  │      └── body: BlockStatement                       │   │
│  │          └── ReturnStatement                        │   │
│  │              └── BinaryExpression                   │   │
│  │                  ├── operator: "+"                  │   │
│  │                  ├── left: Identifier "a"           │   │
│  │                  └── right: Identifier "b"          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```


### 4.2 AST 对 Agent 的价值

AST 让 Agent 能够进行结构化的代码分析和修改：

| 应用场景 | 基于文本 | 基于 AST |
|----------|----------|----------|
| 查找函数 | 正则匹配，可能误匹配注释 | 精确定位 FunctionDeclaration 节点 |
| 重命名变量 | 全局替换，可能改错 | 只修改对应的 Identifier 节点 |
| 提取函数 | 手动计算缩进和位置 | 基于节点边界精确提取 |
| 代码格式化 | 基于规则猜测 | 基于语法结构重新生成 |

### 4.3 Tree-sitter：轻量级语法分析

Tree-sitter 是一个增量式语法分析库，被广泛用于代码编辑器：

```
┌─────────────────────────────────────────────────────────────┐
│                    Tree-sitter 特点                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  优势：                                                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  • 增量解析：只重新解析变化的部分，速度快           │   │
│  │  • 容错性强：即使代码有语法错误也能解析             │   │
│  │  • 多语言支持：支持 100+ 种编程语言                 │   │
│  │  • 查询语言：提供类似 CSS 选择器的查询语法          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  查询示例（查找所有函数定义）：                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  (function_declaration                              │   │
│  │    name: (identifier) @function.name                │   │
│  │    parameters: (formal_parameters) @function.params │   │
│  │    body: (statement_block) @function.body)          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. 代码生成与重构

### 5.1 智能代码补全

代码补全是 Agent 生成代码时的重要辅助：

```
┌─────────────────────────────────────────────────────────────┐
│                    代码补全类型                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 成员补全（Member Completion）                           │
│     ┌─────────────────────────────────────────────────┐    │
│     │  user.                                          │    │
│     │       ↓                                         │    │
│     │  • name: string                                 │    │
│     │  • email: string                                │    │
│     │  • getFullName(): string                        │    │
│     │  • updateProfile(data): void                    │    │
│     └─────────────────────────────────────────────────┘    │
│                                                             │
│  2. 导入补全（Import Completion）                           │
│     ┌─────────────────────────────────────────────────┐    │
│     │  import { useState } from                       │    │
│     │                          ↓                      │    │
│     │  • "react"                                      │    │
│     │  • "preact/hooks"                               │    │
│     └─────────────────────────────────────────────────┘    │
│                                                             │
│  3. 代码片段（Snippet Completion）                          │
│     ┌─────────────────────────────────────────────────┐    │
│     │  for                                            │    │
│     │     ↓                                           │    │
│     │  for (let i = 0; i < array.length; i++) {       │    │
│     │    const element = array[i];                    │    │
│     │    |                                            │    │
│     │  }                                              │    │
│     └─────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 代码重构模式

常见的代码重构操作：

| 重构类型 | 说明 | LSP 支持 |
|----------|------|----------|
| 重命名 | 重命名符号及其所有引用 | `textDocument/rename` |
| 提取函数 | 将代码块提取为函数 | Code Action |
| 提取变量 | 将表达式提取为变量 | Code Action |
| 内联 | 将变量/函数内联到使用处 | Code Action |
| 移动 | 移动符号到其他文件 | Code Action |

---

## 6. OpenCode 实践案例

让我们深入分析 OpenCode 是如何实现代码智能功能的。

### 6.1 LSP 客户端实现

OpenCode 在 `packages/opencode/src/lsp/client.ts` 中实现了 LSP 客户端：

```typescript
// packages/opencode/src/lsp/client.ts（简化版）

export namespace LSPClient {
  // 创建 LSP 客户端连接
  export async function create(input: { 
    serverID: string
    server: LSPServer.Handle
    root: string 
  }) {
    // 1. 创建 JSON-RPC 连接
    // 使用 vscode-jsonrpc 库处理 LSP 协议通信
    const connection = createMessageConnection(
      new StreamMessageReader(input.server.process.stdout),
      new StreamMessageWriter(input.server.process.stdin),
    )

    // 2. 存储诊断信息
    // 每个文件路径对应一组诊断结果
    const diagnostics = new Map<string, Diagnostic[]>()
    
    // 3. 监听诊断推送
    // LSP 服务器会主动推送诊断信息
    connection.onNotification("textDocument/publishDiagnostics", (params) => {
      const filePath = fileURLToPath(params.uri)
      diagnostics.set(filePath, params.diagnostics)
      // 发布事件，通知其他组件诊断已更新
      Bus.publish(Event.Diagnostics, { path: filePath, serverID: input.serverID })
    })

    // 4. 处理服务器请求
    // 服务器可能请求工作区配置等信息
    connection.onRequest("workspace/configuration", async () => {
      return [input.server.initialization ?? {}]
    })
    
    connection.onRequest("workspace/workspaceFolders", async () => [{
      name: "workspace",
      uri: pathToFileURL(input.root).href,
    }])

    // 5. 启动连接监听
    connection.listen()

    // 6. 发送初始化请求
    // 这是 LSP 协议的第一步，告诉服务器客户端的能力
    await connection.sendRequest("initialize", {
      rootUri: pathToFileURL(input.root).href,
      processId: input.server.process.pid,
      workspaceFolders: [{
        name: "workspace",
        uri: pathToFileURL(input.root).href,
      }],
      // 声明客户端支持的能力
      capabilities: {
        textDocument: {
          synchronization: { didOpen: true, didChange: true },
          publishDiagnostics: { versionSupport: true },
        },
        workspace: {
          configuration: true,
          didChangeWatchedFiles: { dynamicRegistration: true },
        },
      },
    })

    // 7. 发送初始化完成通知
    await connection.sendNotification("initialized", {})

    // 8. 返回客户端接口
    return {
      connection,
      diagnostics,
      // 通知服务器文件已打开/更新
      notify: {
        async open(input: { path: string }) {
          const text = await Bun.file(input.path).text()
          const languageId = LANGUAGE_EXTENSIONS[path.extname(input.path)]
          
          await connection.sendNotification("textDocument/didOpen", {
            textDocument: {
              uri: pathToFileURL(input.path).href,
              languageId,
              version: 0,
              text,
            },
          })
        },
      },
      // 等待诊断结果
      async waitForDiagnostics(input: { path: string }) {
        // 使用防抖等待，因为 LSP 可能发送多次诊断
        return new Promise((resolve) => {
          const unsub = Bus.subscribe(Event.Diagnostics, (event) => {
            if (event.properties.path === input.path) {
              setTimeout(() => { unsub(); resolve() }, 150)
            }
          })
        })
      },
    }
  }
}
```

**关键设计点**：

1. **JSON-RPC 通信**：使用 `vscode-jsonrpc` 库处理协议细节
2. **诊断缓存**：使用 Map 缓存每个文件的诊断结果
3. **事件驱动**：通过事件总线通知诊断更新
4. **能力协商**：初始化时声明客户端支持的功能


### 6.2 LSP 服务器配置

OpenCode 在 `packages/opencode/src/lsp/server.ts` 中定义了多种语言服务器：

```typescript
// packages/opencode/src/lsp/server.ts（简化版）

export namespace LSPServer {
  // 服务器句柄：包含进程和初始化配置
  export interface Handle {
    process: ChildProcessWithoutNullStreams
    initialization?: Record<string, any>
  }

  // 服务器信息接口
  export interface Info {
    id: string              // 服务器标识，如 "typescript"
    extensions: string[]    // 支持的文件扩展名
    root: RootFunction      // 确定项目根目录的函数
    spawn(root: string): Promise<Handle | undefined>  // 启动服务器
  }

  // TypeScript 语言服务器
  export const Typescript: Info = {
    id: "typescript",
    // 支持的文件类型
    extensions: [".ts", ".tsx", ".js", ".jsx", ".mjs", ".cjs"],
    // 通过查找 lock 文件确定项目根目录
    root: NearestRoot([
      "package-lock.json", 
      "bun.lockb", 
      "pnpm-lock.yaml", 
      "yarn.lock"
    ]),
    async spawn(root) {
      // 查找 tsserver 路径
      const tsserver = await Bun.resolve(
        "typescript/lib/tsserver.js", 
        Instance.directory
      ).catch(() => {})
      
      if (!tsserver) return  // 未安装 TypeScript
      
      // 启动 typescript-language-server
      const proc = spawn(
        BunProc.which(), 
        ["x", "typescript-language-server", "--stdio"],
        { cwd: root }
      )
      
      return {
        process: proc,
        initialization: { tsserver: { path: tsserver } },
      }
    },
  }

  // Go 语言服务器
  export const Gopls: Info = {
    id: "gopls",
    extensions: [".go"],
    root: async (file) => {
      // 优先查找 go.work（工作区），然后查找 go.mod
      const work = await NearestRoot(["go.work"])(file)
      if (work) return work
      return NearestRoot(["go.mod", "go.sum"])(file)
    },
    async spawn(root) {
      let bin = Bun.which("gopls")
      
      // 如果未安装，自动安装
      if (!bin) {
        if (!Bun.which("go")) return
        await Bun.spawn({
          cmd: ["go", "install", "golang.org/x/tools/gopls@latest"],
          env: { ...process.env, GOBIN: Global.Path.bin },
        }).exited
        bin = path.join(Global.Path.bin, "gopls")
      }
      
      return { process: spawn(bin, { cwd: root }) }
    },
  }

  // Python 语言服务器（Pyright）
  export const Pyright: Info = {
    id: "pyright",
    extensions: [".py", ".pyi"],
    root: NearestRoot([
      "pyproject.toml", 
      "setup.py", 
      "requirements.txt", 
      "Pipfile"
    ]),
    async spawn(root) {
      // 检测虚拟环境
      const venvPaths = [
        process.env["VIRTUAL_ENV"],
        path.join(root, ".venv"),
        path.join(root, "venv")
      ]
      
      let pythonPath: string | undefined
      for (const venvPath of venvPaths) {
        if (!venvPath) continue
        const potentialPath = path.join(venvPath, "bin", "python")
        if (await Bun.file(potentialPath).exists()) {
          pythonPath = potentialPath
          break
        }
      }
      
      // 启动 pyright-langserver
      const proc = spawn("pyright-langserver", ["--stdio"], { cwd: root })
      
      return {
        process: proc,
        initialization: pythonPath ? { pythonPath } : {},
      }
    },
  }

  // 还支持：Rust (rust-analyzer), Go (gopls), Vue, ESLint, Biome 等
}
```

**支持的语言服务器**：

| 语言 | 服务器 | 自动安装 |
|------|--------|----------|
| TypeScript/JavaScript | typescript-language-server | ✅ |
| Python | pyright | ✅ |
| Go | gopls | ✅ |
| Rust | rust-analyzer | ❌ |
| Vue | vue-language-server | ✅ |
| C/C++ | clangd | ✅ |
| Ruby | rubocop | ✅ |
| Zig | zls | ✅ |

### 6.3 LSP 工具封装

OpenCode 在 `packages/opencode/src/tool/lsp.ts` 中将 LSP 能力封装为 Agent 可用的工具：

```typescript
// packages/opencode/src/tool/lsp.ts（简化版）

// 支持的 LSP 操作
const operations = [
  "goToDefinition",      // 跳转到定义
  "findReferences",      // 查找引用
  "hover",               // 悬停信息
  "documentSymbol",      // 文档符号
  "workspaceSymbol",     // 工作区符号搜索
  "goToImplementation",  // 跳转到实现
  "prepareCallHierarchy",// 准备调用层次
  "incomingCalls",       // 查找调用者
  "outgoingCalls",       // 查找被调用者
] as const

export const LspTool = Tool.define("lsp", {
  description: DESCRIPTION,
  
  parameters: z.object({
    operation: z.enum(operations)
      .describe("The LSP operation to perform"),
    filePath: z.string()
      .describe("The absolute or relative path to the file"),
    // 注意：使用 1-based 行列号，与编辑器显示一致
    line: z.number().int().min(1)
      .describe("The line number (1-based, as shown in editors)"),
    character: z.number().int().min(1)
      .describe("The character offset (1-based, as shown in editors)"),
  }),
  
  execute: async (args, ctx) => {
    // 1. 请求权限
    await ctx.ask({
      permission: "lsp",
      patterns: ["*"],
      always: ["*"],
      metadata: {},
    })

    // 2. 处理路径和位置
    const file = path.isAbsolute(args.filePath) 
      ? args.filePath 
      : path.join(Instance.directory, args.filePath)
    
    // 转换为 0-based（LSP 协议使用 0-based）
    const position = {
      file,
      line: args.line - 1,
      character: args.character - 1,
    }

    // 3. 检查文件存在性
    if (!await Bun.file(file).exists()) {
      throw new Error(`File not found: ${file}`)
    }

    // 4. 检查是否有可用的 LSP 服务器
    const available = await LSP.hasClients(file)
    if (!available) {
      throw new Error("No LSP server available for this file type.")
    }

    // 5. 触发文件同步，等待诊断
    await LSP.touchFile(file, true)

    // 6. 执行对应的 LSP 操作
    const result = await (async () => {
      switch (args.operation) {
        case "goToDefinition":
          return LSP.definition(position)
        case "findReferences":
          return LSP.references(position)
        case "hover":
          return LSP.hover(position)
        case "documentSymbol":
          return LSP.documentSymbol(pathToFileURL(file).href)
        case "workspaceSymbol":
          return LSP.workspaceSymbol("")
        case "goToImplementation":
          return LSP.implementation(position)
        case "prepareCallHierarchy":
          return LSP.prepareCallHierarchy(position)
        case "incomingCalls":
          return LSP.incomingCalls(position)
        case "outgoingCalls":
          return LSP.outgoingCalls(position)
      }
    })()

    // 7. 格式化输出
    return {
      title: `${args.operation} ${path.relative(Instance.worktree, file)}:${args.line}:${args.character}`,
      metadata: { result },
      output: result.length === 0 
        ? `No results found for ${args.operation}`
        : JSON.stringify(result, null, 2),
    }
  },
})
```

**关键设计点**：

1. **行列号转换**：用户使用 1-based（与编辑器一致），内部转换为 0-based（LSP 协议）
2. **文件同步**：执行操作前先同步文件内容到 LSP 服务器
3. **统一接口**：将多种 LSP 操作封装为单一工具，通过 `operation` 参数区分
4. **错误处理**：检查文件存在性和 LSP 服务器可用性


### 6.4 LSP 核心功能实现

OpenCode 在 `packages/opencode/src/lsp/index.ts` 中实现了 LSP 的核心功能：

```typescript
// packages/opencode/src/lsp/index.ts（简化版）

export namespace LSP {
  // 获取文件的诊断信息
  export async function diagnostics() {
    const results: Record<string, LSPClient.Diagnostic[]> = {}
    // 从所有 LSP 客户端收集诊断
    for (const result of await runAll(async (client) => client.diagnostics)) {
      for (const [path, diagnostics] of result.entries()) {
        const arr = results[path] || []
        arr.push(...diagnostics)
        results[path] = arr
      }
    }
    return results
  }

  // 跳转到定义
  export async function definition(input: { 
    file: string
    line: number
    character: number 
  }) {
    return run(input.file, (client) =>
      client.connection.sendRequest("textDocument/definition", {
        textDocument: { uri: pathToFileURL(input.file).href },
        position: { line: input.line, character: input.character },
      }).catch(() => null)
    ).then((result) => result.flat().filter(Boolean))
  }

  // 查找引用
  export async function references(input: { 
    file: string
    line: number
    character: number 
  }) {
    return run(input.file, (client) =>
      client.connection.sendRequest("textDocument/references", {
        textDocument: { uri: pathToFileURL(input.file).href },
        position: { line: input.line, character: input.character },
        context: { includeDeclaration: true },  // 包含定义本身
      }).catch(() => [])
    ).then((result) => result.flat().filter(Boolean))
  }

  // 悬停信息
  export async function hover(input: { 
    file: string
    line: number
    character: number 
  }) {
    return run(input.file, (client) =>
      client.connection.sendRequest("textDocument/hover", {
        textDocument: { uri: pathToFileURL(input.file).href },
        position: { line: input.line, character: input.character },
      }).catch(() => null)
    )
  }

  // 文档符号
  export async function documentSymbol(uri: string) {
    const file = new URL(uri).pathname
    return run(file, (client) =>
      client.connection.sendRequest("textDocument/documentSymbol", {
        textDocument: { uri },
      }).catch(() => [])
    ).then((result) => result.flat().filter(Boolean))
  }

  // 工作区符号搜索
  export async function workspaceSymbol(query: string) {
    // 过滤只返回重要的符号类型
    const importantKinds = [
      SymbolKind.Class,
      SymbolKind.Function,
      SymbolKind.Method,
      SymbolKind.Interface,
      SymbolKind.Variable,
      SymbolKind.Constant,
      SymbolKind.Struct,
      SymbolKind.Enum,
    ]
    
    return runAll((client) =>
      client.connection.sendRequest("workspace/symbol", { query })
        .then((result: any) => result.filter((x: Symbol) => 
          importantKinds.includes(x.kind)
        ))
        .then((result: any) => result.slice(0, 10))  // 限制结果数量
        .catch(() => [])
    ).then((result) => result.flat())
  }

  // 调用层次：查找调用者
  export async function incomingCalls(input: { 
    file: string
    line: number
    character: number 
  }) {
    return run(input.file, async (client) => {
      // 首先准备调用层次项
      const items = await client.connection.sendRequest(
        "textDocument/prepareCallHierarchy",
        {
          textDocument: { uri: pathToFileURL(input.file).href },
          position: { line: input.line, character: input.character },
        }
      ).catch(() => []) as any[]
      
      if (!items?.length) return []
      
      // 然后查找调用者
      return client.connection.sendRequest(
        "callHierarchy/incomingCalls",
        { item: items[0] }
      ).catch(() => [])
    }).then((result) => result.flat().filter(Boolean))
  }

  // 触发文件同步
  export async function touchFile(input: string, waitForDiagnostics?: boolean) {
    const clients = await getClients(input)
    await Promise.all(
      clients.map(async (client) => {
        // 可选：等待诊断结果
        const wait = waitForDiagnostics 
          ? client.waitForDiagnostics({ path: input }) 
          : Promise.resolve()
        // 通知服务器文件已打开/更新
        await client.notify.open({ path: input })
        return wait
      })
    )
  }

  // 格式化诊断信息
  export namespace Diagnostic {
    export function pretty(diagnostic: LSPClient.Diagnostic) {
      const severityMap = {
        1: "ERROR",
        2: "WARN",
        3: "INFO",
        4: "HINT",
      }
      const severity = severityMap[diagnostic.severity || 1]
      const line = diagnostic.range.start.line + 1
      const col = diagnostic.range.start.character + 1
      return `${severity} [${line}:${col}] ${diagnostic.message}`
    }
  }
}
```

**关键设计点**：

1. **多服务器支持**：同一文件可能有多个 LSP 服务器（如 TypeScript + ESLint）
2. **结果聚合**：合并多个服务器的结果
3. **错误容忍**：单个服务器失败不影响其他服务器
4. **结果过滤**：只返回重要的符号类型，避免信息过载

---

## 动手实验

### 实验 1：探索 LSP 工具

使用 OpenCode 的 LSP 工具探索代码结构：

```bash
# 1. 启动 OpenCode
cd your-project
opencode

# 2. 在对话中使用 LSP 工具
# 查找函数定义
> 使用 lsp 工具查找 src/index.ts 第 10 行第 5 列的定义

# 查找所有引用
> 使用 lsp 工具查找 handleSubmit 函数的所有引用

# 获取文档符号
> 使用 lsp 工具获取 src/components/Button.tsx 的所有符号
```

### 实验 2：理解诊断信息

观察 Agent 如何利用诊断信息：

```typescript
// 1. 创建一个有错误的文件
// test-diagnostics.ts
const user: { name: string } = {
  name: "Alice",
  age: 25,  // 错误：类型中没有 age 属性
}

console.log(user.email)  // 错误：类型中没有 email 属性

// 2. 让 Agent 修复这个文件
// Agent 会：
// - 触发 LSP 诊断
// - 获取错误信息
// - 根据错误修复代码
```

### 实验 3：调用层次分析

使用调用层次理解代码结构：

```typescript
// 假设有以下代码结构
function validateEmail(email: string) { /* ... */ }
function validatePassword(password: string) { /* ... */ }

function validateForm(data: FormData) {
  validateEmail(data.email)
  validatePassword(data.password)
}

function handleSubmit(event: Event) {
  validateForm(getFormData())
}

// 使用 LSP 工具分析：
// 1. incomingCalls on validateForm → 找到 handleSubmit
// 2. outgoingCalls on validateForm → 找到 validateEmail, validatePassword
```

---

## 实战项目：构建能理解代码结构的 Agent

### 项目目标

构建一个能够：
1. 自动分析代码结构
2. 理解函数调用关系
3. 基于代码理解进行智能修改

### 实现步骤

#### 步骤 1：设置 LSP 环境

```typescript
// code-aware-agent.ts
import { spawn } from "child_process"
import { createMessageConnection, StreamMessageReader, StreamMessageWriter } from "vscode-jsonrpc/node"
import { pathToFileURL } from "url"

// 启动 TypeScript 语言服务器
async function startLSPServer(projectRoot: string) {
  const proc = spawn("npx", ["typescript-language-server", "--stdio"], {
    cwd: projectRoot,
  })

  const connection = createMessageConnection(
    new StreamMessageReader(proc.stdout),
    new StreamMessageWriter(proc.stdin),
  )

  connection.listen()

  // 初始化
  await connection.sendRequest("initialize", {
    rootUri: pathToFileURL(projectRoot).href,
    processId: process.pid,
    capabilities: {
      textDocument: {
        synchronization: { didOpen: true, didChange: true },
        publishDiagnostics: { versionSupport: true },
      },
    },
  })

  await connection.sendNotification("initialized", {})

  return connection
}
```

#### 步骤 2：实现代码分析功能

```typescript
// 获取文件中的所有函数
async function getFunctions(connection: any, filePath: string) {
  const uri = pathToFileURL(filePath).href
  
  // 打开文件
  const content = await Bun.file(filePath).text()
  await connection.sendNotification("textDocument/didOpen", {
    textDocument: { uri, languageId: "typescript", version: 0, text: content },
  })

  // 获取文档符号
  const symbols = await connection.sendRequest("textDocument/documentSymbol", {
    textDocument: { uri },
  })

  // 过滤出函数
  return symbols.filter((s: any) => 
    s.kind === 12 /* Function */ || s.kind === 6 /* Method */
  )
}

// 分析函数的调用关系
async function analyzeCallGraph(
  connection: any, 
  filePath: string, 
  functionName: string
) {
  const uri = pathToFileURL(filePath).href
  const functions = await getFunctions(connection, filePath)
  
  const target = functions.find((f: any) => f.name === functionName)
  if (!target) return null

  // 获取调用者
  const items = await connection.sendRequest("textDocument/prepareCallHierarchy", {
    textDocument: { uri },
    position: target.selectionRange.start,
  })

  if (!items?.length) return { callers: [], callees: [] }

  const [callers, callees] = await Promise.all([
    connection.sendRequest("callHierarchy/incomingCalls", { item: items[0] }),
    connection.sendRequest("callHierarchy/outgoingCalls", { item: items[0] }),
  ])

  return {
    callers: callers.map((c: any) => c.from.name),
    callees: callees.map((c: any) => c.to.name),
  }
}
```


#### 步骤 3：构建智能修改功能

```typescript
// 安全地重命名函数
async function safeRename(
  connection: any,
  filePath: string,
  oldName: string,
  newName: string
) {
  // 1. 找到函数位置
  const functions = await getFunctions(connection, filePath)
  const target = functions.find((f: any) => f.name === oldName)
  
  if (!target) {
    throw new Error(`Function ${oldName} not found`)
  }

  // 2. 查找所有引用
  const references = await connection.sendRequest("textDocument/references", {
    textDocument: { uri: pathToFileURL(filePath).href },
    position: target.selectionRange.start,
    context: { includeDeclaration: true },
  })

  console.log(`Found ${references.length} references to ${oldName}`)

  // 3. 使用 LSP 的重命名功能
  const edits = await connection.sendRequest("textDocument/rename", {
    textDocument: { uri: pathToFileURL(filePath).href },
    position: target.selectionRange.start,
    newName,
  })

  return edits
}

// 基于诊断自动修复
async function autoFix(connection: any, filePath: string) {
  const uri = pathToFileURL(filePath).href
  
  // 等待诊断
  await new Promise((resolve) => setTimeout(resolve, 1000))
  
  // 获取代码操作（包括快速修复）
  const diagnostics = [] // 从 publishDiagnostics 通知中收集
  
  const actions = await connection.sendRequest("textDocument/codeAction", {
    textDocument: { uri },
    range: { start: { line: 0, character: 0 }, end: { line: 1000, character: 0 } },
    context: { diagnostics },
  })

  // 过滤出快速修复
  const quickFixes = actions.filter((a: any) => 
    a.kind?.startsWith("quickfix")
  )

  return quickFixes
}
```

#### 步骤 4：整合为 Agent 工具

```typescript
// 将代码分析能力整合为 Agent 可用的工具
const codeAnalysisTool = {
  name: "analyze_code",
  description: "Analyze code structure and relationships",
  parameters: {
    type: "object",
    properties: {
      action: {
        type: "string",
        enum: ["list_functions", "call_graph", "find_references", "get_diagnostics"],
      },
      filePath: { type: "string" },
      symbolName: { type: "string" },
    },
    required: ["action", "filePath"],
  },
  
  async execute(params: any, lspConnection: any) {
    switch (params.action) {
      case "list_functions":
        return getFunctions(lspConnection, params.filePath)
      
      case "call_graph":
        return analyzeCallGraph(lspConnection, params.filePath, params.symbolName)
      
      case "find_references":
        // ... 实现引用查找
        break
      
      case "get_diagnostics":
        // ... 实现诊断获取
        break
    }
  },
}
```

### 扩展思考

1. **增量分析**：如何在大型项目中高效地进行增量代码分析？
2. **跨语言支持**：如何支持多语言项目的代码智能？
3. **缓存策略**：如何缓存 LSP 结果以提高性能？
4. **错误恢复**：当 LSP 服务器崩溃时如何优雅恢复？

---

## 知识补充

### 相关资源

1. **LSP 官方规范**
   - [Language Server Protocol Specification](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/)
   - 完整的协议定义和消息格式

2. **Tree-sitter**
   - [Tree-sitter 官网](https://tree-sitter.github.io/tree-sitter/)
   - 增量式语法分析库

3. **语言服务器实现**
   - [typescript-language-server](https://github.com/typescript-language-server/typescript-language-server)
   - [pyright](https://github.com/microsoft/pyright)
   - [rust-analyzer](https://github.com/rust-lang/rust-analyzer)
   - [gopls](https://github.com/golang/tools/tree/master/gopls)

4. **相关论文**
   - "Language Server Protocol: A Proposal for Standardizing Language Intelligence Tools"
   - "Tree-sitter: A New Parsing System for Programming Tools"

### 进阶阅读

- **VS Code 源码**：了解 LSP 客户端的工业级实现
- **Neovim LSP**：了解轻量级 LSP 客户端实现
- **Monaco Editor**：了解 Web 端的 LSP 集成

---

## 本章小结

### 核心概念回顾

```
┌─────────────────────────────────────────────────────────────┐
│                    代码智能核心概念                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  LSP 协议：                                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  • 标准化的编辑器-语言服务器通信协议                │   │
│  │  • 基于 JSON-RPC 2.0                                │   │
│  │  • 支持请求-响应和通知两种模式                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  核心能力：                                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  • 诊断：实时发现代码错误和警告                     │   │
│  │  • 导航：定义跳转、引用查找、符号搜索               │   │
│  │  • 分析：调用层次、类型信息、文档                   │   │
│  │  • 重构：重命名、代码操作、快速修复                 │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  对 Agent 的价值：                                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  • 减少幻觉：基于真实类型信息生成代码               │   │
│  │  • 精确定位：快速找到需要修改的代码                 │   │
│  │  • 安全重构：理解引用关系，避免破坏性修改           │   │
│  │  • 即时反馈：生成代码后立即验证正确性               │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### OpenCode 代码对照表

| 概念 | OpenCode 实现 | 文件位置 |
|------|--------------|----------|
| LSP 客户端 | `LSPClient.create()` | `lsp/client.ts` |
| LSP 服务器配置 | `LSPServer.Typescript` 等 | `lsp/server.ts` |
| LSP 工具 | `LspTool` | `tool/lsp.ts` |
| 诊断处理 | `LSP.diagnostics()` | `lsp/index.ts` |
| 定义跳转 | `LSP.definition()` | `lsp/index.ts` |
| 引用查找 | `LSP.references()` | `lsp/index.ts` |
| 调用层次 | `LSP.incomingCalls()` | `lsp/index.ts` |

### 检查清单

完成本章学习后，你应该能够：

- [ ] 理解 LSP 协议的架构和通信方式
- [ ] 解释代码诊断对 Agent 的价值
- [ ] 使用 LSP 工具进行代码导航（定义、引用、符号）
- [ ] 理解调用层次分析的应用场景
- [ ] 阅读 OpenCode 的 LSP 实现代码
- [ ] 构建简单的代码分析工具

---

## 导航

- 上一章：[第三章：工具系统设计](./03-tool-system.md)
- 下一章：[第五章：记忆系统](./05-memory-system.md)
- [返回目录](./00-introduction.md)
