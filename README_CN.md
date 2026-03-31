<div align="center">

# Claude Code

**Anthropic 官方 AI 编程助手 CLI 工具 — 源码研究版本**

![TypeScript](https://img.shields.io/badge/TypeScript-严格模式-3178c6?logo=typescript&logoColor=white)
![Bun](https://img.shields.io/badge/Bun-v1.3.5+-fbf0df?logo=bun&logoColor=black)
![Files](https://img.shields.io/badge/源文件-1%2C903-blueviolet)
![Lines](https://img.shields.io/badge/代码行数-512%2C000%2B-orange)

Claude Code 是一款运行在终端中的 AI 编程助手，深度集成 Claude 模型，支持代码编写、调试、重构、代码审查、多代理协作等复杂工程任务。本仓库为其核心源码快照。

<br/>

<img src="assets/cli-preview.jpg" alt="Claude Code 运行截图：bun install + bun run dev 成功启动" width="720"/>

*执行 `bun install` 后，`bun run dev` 直接从源码启动 Claude Code，进入交互式 REPL*

</div>

---

> **✨ [@QuantaAlpha](https://quantaalpha.com) 团队已完成开箱即用改造**
>
> 我们将本源码快照改造为**零配置、即装即跑的 CLI 版本** — `bun install` 即可启动。
> 深度接入 **[EpochX](https://epochx.cc) Agent 社区平台**，让你的 Claude Code 不只是编程助手：
>
> - 💰 自动接取 EpochX 平台悬赏任务
> - 💰 7×24 小时在线赚钱，Agent 替你打工
> - 💰 成为真正的 AI 数字员工
> - 📋 在平台上发布任意任务，让别人的 Claude Code 或 OpenClaw 帮你完成
>
> **技术人别只当工具用，让你的 Agent 替你创收。**
>
> 👉 即用版 Fork：**[github.com/QuantaAlpha/claude-code](https://github.com/QuantaAlpha/claude-code)**
> 🌐 了解更多：**[quantaalpha.com](https://quantaalpha.com)** · **[epochx.cc](https://epochx.cc)**

---

## Quick Start

### 前置要求

- [Bun](https://bun.sh) **v1.3.5+**（`bun --version` 确认；如需升级：`bun upgrade`）
- `ANTHROPIC_API_KEY` 环境变量（运行交互模式时必须）

### 安装

```bash
# 克隆仓库后，在项目根目录执行：
bun install
```

### 启动

```bash
# 方式 1：通过 npm script（推荐）
bun run dev

# 方式 2：直接运行入口文件（等价）
bun run src/dev-entry.ts

# 方式 3：可执行脚本（根目录）
./claude-dev
```

### 常用命令

```bash
# 查看版本
bun run dev --version

# 查看帮助
bun run dev --help

# 交互式 REPL（需要 ANTHROPIC_API_KEY）
export ANTHROPIC_API_KEY=your_key_here
bun run dev

# 非交互打印模式（单次对话后退出）
bun run dev --print "帮我写一个冒泡排序"

# 指定工作目录运行
bun run dev --add-dir /path/to/project
```

---

## 运行逻辑

### 启动入口链路

```
claude-dev  (可执行脚本，根目录)
  └── bun run src/dev-entry.ts
        └── src/entrypoints/cli.tsx   ← 真正的 CLI 入口
              └── main()              ← Commander.js 命令解析 + 路由
                    ├── --version / --help / --print → 直接处理
                    └── 交互模式 → launchRepl() → screens/REPL.tsx
```

#### 为什么需要 `dev-entry.ts`？

`src/entrypoints/cli.tsx` 使用了 Bun 打包专用 API `bun:bundle`（用于编译时特性标志 / 死代码消除），无法直接用 `bun run` 解释执行。`dev-entry.ts` 在运行时通过 `globalThis.MACRO` 模拟打包时注入的常量，使源码可以不经构建直接运行：

```typescript
// dev-entry.ts 做了两件关键事：

// 1. 注入打包时常量（替代 bun build 在编译期注入的值）
globalThis.MACRO = {
  VERSION: pkg.version,
  BUILD_TIME: '',
  PACKAGE_URL: pkg.name,
  // ...
}

// 2. 验证所有相对 import 可解析后，动态导入真正入口
await import('./entrypoints/cli.tsx')
```

### Shims（桩模块）说明

部分 Anthropic 内部包未公开发布，由 `stubs/` 目录下的桩模块替代：

| 包名 | 位置 | 说明 |
|------|------|------|
| `@ant/computer-use-mcp` | `stubs/ant-computer-use-mcp` | macOS 屏幕控制（截图 / 鼠标 / 键盘） |
| `@ant/computer-use-input` | `stubs/ant-computer-use-input` | 底层输入设备控制（Rust/enigo） |
| `@ant/computer-use-swift` | `stubs/ant-computer-use-swift` | macOS Swift 原生截图 API |
| `@ant/claude-for-chrome-mcp` | `stubs/ant-claude-for-chrome-mcp` | Chrome 浏览器控制 MCP 服务 |
| `color-diff-napi` | `stubs/color-diff-napi` | 语法高亮 diff 渲染（Native Addon） |
| `modifiers-napi` | `stubs/modifiers-napi` | 键盘修饰键状态检测 |
| `url-handler-napi` | `stubs/url-handler-napi` | macOS URL scheme 注册 |

这些 shim 均为空实现或返回默认值，不影响核心 AI 对话和代码操作能力。

### 特性标志（`bun:bundle` 编译时决策）

生产包通过 `bun:bundle` 的 `feature()` 宏在**打包阶段**决定是否包含某段代码。在开发模式（`dev-entry.ts` 启动）下，所有 `feature()` 调用返回 `false`，即所有实验性特性默认关闭，只保留核心功能路径。

```typescript
// 示例：仅在生产包中激活特定特性
if (feature('KAIROS')) {
  // 高级助手模式代码 — 开发模式下此分支被跳过
}
```

### 关键源文件

| 文件 | 说明 |
|------|------|
| `src/dev-entry.ts` | 开发启动入口，注入 MACRO 常量并校验依赖完整性 |
| `src/entrypoints/cli.tsx` | CLI bootstrap，处理快速路径（--version 等）和模式路由 |
| `src/main.tsx` | 主应用逻辑：Commander 命令树、REPL 启动、会话初始化 |
| `src/query.ts` | LLM 查询处理主循环 |
| `src/QueryEngine.ts` | 查询引擎核心，管理 API 调用和工具编排 |

---

## 目录

- [架构总览](#架构总览)
- [目录结构](#目录结构)
- [核心概念](#核心概念)
  - [查询引擎](#查询引擎)
  - [工具系统](#工具系统)
  - [命令系统](#命令系统)
  - [Hooks 机制](#hooks-机制)
  - [权限系统](#权限系统)
  - [任务系统](#任务系统)
  - [MCP 集成](#mcp-集成)
- [技术栈](#技术栈)
- [特性标志系统](#特性标志系统)
- [关键数据流](#关键数据流)
- [主要模块说明](#主要模块说明)
- [环境变量参考](#环境变量参考)

---

## 架构总览

```
┌─────────────────────────────────────────────────────────┐
│                      入口层                              │
│  entrypoints/cli.tsx   entrypoints/init.ts   setup.ts  │
└───────────────────────────┬─────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────┐
│                    主应用循环 (main.tsx)                  │
│          命令解析 → 消息处理 → 用户交互管理               │
└──────────┬──────────────────────────┬────────────────────┘
           │                          │
┌──────────▼──────────┐  ┌────────────▼────────────────────┐
│    命令系统          │  │         查询引擎                  │
│  commands/ (103+)   │  │  query.ts + QueryEngine.ts       │
│  PromptCommand      │  │  ↕ services/api/claude.ts        │
│  LocalCommand       │  │        (Anthropic SDK)           │
└─────────────────────┘  └─────────────┬───────────────────┘
                                       │
                          ┌────────────▼────────────────────┐
                          │         工具执行层               │
                          │  tools/ (44+) + toolHooks.ts    │
                          │  PreToolUse → call → PostToolUse│
                          └─────────────┬───────────────────┘
                                       │
           ┌───────────────────────────┼───────────────────────┐
           │                           │                       │
┌──────────▼──────┐      ┌─────────────▼──────┐   ┌───────────▼──────┐
│   权限控制层     │      │    状态管理层        │   │    UI 渲染层      │
│ useCanUseTool   │      │  AppState (Zustand)  │   │  React + Ink     │
│ permissions/    │      │  context/            │   │  components/     │
└─────────────────┘      └────────────────────┘   └──────────────────┘
           │
┌──────────▼──────────────────────────────────────────────────────────┐
│                          支撑服务层                                   │
│  MCP集成 | OAuth认证 | LSP支持 | 上下文压缩 | 分析追踪 | 插件系统     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 目录结构

```
src/
├── entrypoints/          # 多个应用入口
│   ├── cli.tsx           # CLI 主入口，处理 --version、--dump-system-prompt 等
│   └── init.ts           # 初始化：Node 版本检查、Session 管理、Worktree 支持
├── main.tsx              # 应用主循环 (4,683 行)
├── setup.ts              # 启动配置：权限模式、Tmux 集成、日志初始化
├── query.ts              # 查询处理主逻辑 (1,729 行)
├── QueryEngine.ts        # 查询引擎核心 (1,295 行)
├── Tool.ts               # 工具类型定义和接口 (792 行)
├── commands.ts           # 命令注册表 (100+ 命令)
├── context.ts            # Git 状态和系统上下文管理
│
├── tools/                # 44+ 工具实现
│   ├── BashTool/
│   ├── FileEditTool/
│   ├── FileReadTool/
│   ├── FileWriteTool/
│   ├── GlobTool/
│   ├── GrepTool/
│   ├── WebFetchTool/
│   ├── WebSearchTool/
│   ├── MCPTool/
│   ├── AgentTool/
│   ├── SkillTool/
│   ├── TaskCreateTool/
│   ├── EnterPlanModeTool/
│   ├── NotebookEditTool/
│   └── ... (更多工具)
│
├── commands/             # 103+ 命令实现
│   ├── commit/
│   ├── review/
│   ├── config/
│   ├── mcp/
│   ├── skills/
│   ├── memory/
│   ├── tasks/
│   └── ... (更多命令)
│
├── services/             # 38+ 服务
│   ├── api/              # Anthropic API 客户端 (22 个子模块)
│   │   ├── claude.ts     # API 调用、流式处理、重试逻辑
│   │   └── errors.ts
│   ├── tools/            # 工具执行编排
│   │   ├── toolOrchestration.ts
│   │   ├── toolExecution.ts
│   │   └── StreamingToolExecutor.ts
│   ├── mcp/              # MCP 服务器管理 (25 个子模块)
│   ├── compact/          # 上下文自动压缩
│   ├── analytics/        # 分析追踪
│   ├── lsp/              # Language Server Protocol
│   ├── oauth/            # OAuth 2.0 认证
│   └── ...
│
├── state/                # 应用状态管理
│   ├── AppState.tsx      # React 状态定义
│   ├── AppStateStore.ts  # 状态存储
│   └── store.ts          # Zustand store 实现
│
├── hooks/                # 87+ React Hooks
│   ├── useCanUseTool.ts  # 工具权限检查
│   ├── useTasksV2.ts     # 任务管理
│   ├── useQueueProcessor.ts
│   ├── useVoice.ts       # 语音集成
│   └── ...
│
├── components/           # 146+ React 组件
├── ink/                  # 终端 UI 渲染层 (50 文件)
├── types/                # TypeScript 类型定义
│   ├── message.ts
│   ├── command.ts
│   ├── hooks.ts
│   └── permissions.ts
│
├── utils/                # 298+ 工具函数模块
│   ├── config.ts
│   ├── permissions/
│   ├── model/
│   ├── bash/             # Bash 解析器 (bashParser.ts 4,436 行)
│   └── ...
│
├── bootstrap/            # 启动状态初始化
├── tasks/                # 任务系统实现
├── skills/               # 技能系统
├── plugins/              # 插件系统
├── bridge/               # Bridge 通信 (33 文件)
├── coordinator/          # 多代理协调
├── memdir/               # 内存目录管理
├── migrations/           # 数据迁移
├── remote/               # 远程操作
├── schemas/              # 数据 Schema
├── vim/                  # Vim 模式支持
└── keybindings/          # 键盘快捷键配置
```

---

## 核心概念

### 查询引擎

查询引擎是整个系统的核心，负责与 Claude API 交互并编排工具调用。

**主要入口**：`src/query.ts` + `src/QueryEngine.ts`

```typescript
export type QueryEngineConfig = {
  cwd: string
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  canUseTool: CanUseToolFn
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void
  customSystemPrompt?: string
  userSpecifiedModel?: string
  thinkingConfig?: ThinkingConfig
  maxTurns?: number
  maxBudgetUsd?: number
}
```

**执行流程**：

```
用户输入
  ↓
消息正规化 (normalizeMessagesForAPI)
  ↓
系统提示准备 (getSystemPrompt)
  ↓
API 调用 (services/api/claude.ts)  — 流式响应
  ↓
工具调用处理 (runTools)
  ├── 并发安全工具 → 并发执行
  └── 破坏性工具 → 串行执行
  ↓
结果聚合 → 继续对话 or 结束
```

---

### 工具系统

工具是 Claude 与外部系统交互的基本单元。

**工具接口定义** (`src/Tool.ts`):

```typescript
export type Tool<Input, Output, P extends ToolProgressData> = {
  name: string
  aliases?: string[]
  
  // 核心执行方法
  call(args: z.infer<Input>, context: ToolUseContext, canUseTool: CanUseToolFn): Promise<ToolResult<Output>>
  
  // 并发和只读性声明
  isConcurrencySafe(input: z.infer<Input>): boolean
  isReadOnly(input: z.infer<Input>): boolean
  isDestructive?(input: z.infer<Input>): boolean
  
  // 输入验证 schema
  readonly inputSchema: Input
}
```

**内置工具列表**：

| 类别 | 工具 |
|------|------|
| 文件操作 | FileReadTool, FileWriteTool, FileEditTool, GlobTool, GrepTool, NotebookEditTool |
| 命令执行 | BashTool, PowerShellTool, REPLTool (ant-only) |
| 网络 | WebFetchTool, WebSearchTool |
| AI/代理 | AgentTool, SkillTool, SendMessageTool |
| 任务管理 | TaskCreateTool, TaskUpdateTool, TaskListTool, TaskGetTool, TaskStopTool, TaskOutputTool |
| 模式控制 | EnterPlanModeTool, ExitPlanModeTool, EnterWorktreeTool, ExitWorktreeTool |
| MCP | MCPTool, ListMcpResourcesTool, ReadMcpResourceTool, McpAuthTool |
| 调度 | ScheduleCronTool, RemoteTriggerTool |
| 其他 | TodoWriteTool, ToolSearchTool, AskUserQuestionTool, SleepTool |

---

### 命令系统

命令分为两种类型：

```typescript
// 本地命令 — 直接执行，不走 LLM
type LocalCommand = {
  type: 'local'
  supportsNonInteractive: boolean
  load: () => Promise<LocalCommandModule>
}

// 提示命令 — 生成系统提示注入查询引擎
type PromptCommand = {
  type: 'prompt'
  progressMessage: string
  allowedTools?: string[]
  model?: string
  source: 'builtin' | 'mcp' | 'plugin' | 'bundled'
  getPromptForCommand(args: string, context: ToolUseContext): Promise<ContentBlockParam[]>
}
```

**常用命令**：

| 类别 | 命令 |
|------|------|
| Git 工作流 | commit, diff, review, branch, pr_comments |
| 配置管理 | config, model, permissions, keybindings, theme |
| 会话管理 | clear, compact, resume, export, copy |
| 代码分析 | cost, context, files, stats |
| 工具管理 | mcp, skills, plugins, memory |
| 任务管理 | tasks, agent, session |
| 账户 | login, logout, usage, upgrade |
| 调试 | doctor, debug-tool-call, heapdump, perf-issue |
| 其他 | help, version, vim, voice, init |

---

### Hooks 机制

Hooks 允许在工具调用的各个生命周期节点注入自定义逻辑。

**Hook 事件类型**：

| 事件 | 触发时机 |
|------|----------|
| `PreToolUse` | 工具调用前，可修改输入参数 |
| `PostToolUse` | 工具调用成功后 |
| `PostToolUseFailure` | 工具执行失败后 |
| `UserPromptSubmit` | 用户提交消息时 |
| `SessionStart` | 会话开始时 |
| `Setup` | 应用初始化时 |
| `PermissionDenied` | 权限被拒绝时 |
| `Notification` | 通知事件 |
| `SubagentStart` | 子代理启动时 |
| `PreCompact` / `PostCompact` | 上下文压缩前后 |

**Hook 执行流程**：

```typescript
// 1. 从配置中获取匹配的 hooks
// 2. 按优先级排序
// 3. 顺序执行，支持提前终止
// 4. 每个 hook 可以返回:
//    - continue: false  → 终止后续 hooks
//    - decision: 'block' → 拒绝工具调用
//    - updatedInput      → 修改工具输入参数
```

---

### 权限系统

权限系统控制工具能否被自动执行或需要用户确认。

**权限模式**：

| 模式 | 说明 |
|------|------|
| `default` | 敏感操作提示用户确认 |
| `bypassPermissions` | 自动批准所有操作（危险模式）|
| `dontAsk` | 本次会话内不再重复询问 |
| `acceptEdits` | 仅自动批准非破坏性操作 |
| `plan` | 先展示操作计划，等待用户确认 |
| `auto` | 使用 TRANSCRIPT_CLASSIFIER 自动决策 |

**权限检查流程**：

```
工具调用请求
  ↓
1. 查找匹配的权限规则 (allow/deny)
  ↓
2. 执行 PreToolUse hooks
  ↓
3. 按权限模式判断
  ├── bypassPermissions/dontAsk → 直接批准
  ├── acceptEdits → 检查是否为破坏性操作
  ├── plan → 展示计划等待确认
  └── default → 提示用户
  ↓
4. 返回 PermissionResult { approved, reason }
```

---

### 任务系统

任务系统支持在后台并发执行多种类型的任务。

**任务类型**：

| 类型 | 说明 |
|------|------|
| `local_bash` | 本地 Bash 命令任务 |
| `local_agent` | 本地 Agent 代理任务 |
| `remote_agent` | 远程 Agent 任务 |
| `in_process_teammate` | 进程内队友任务（多代理协作）|
| `local_workflow` | 本地工作流任务 |
| `monitor_mcp` | MCP 监控任务 |

**任务状态**：`pending` → `running` → `completed` / `failed` / `killed`

---

### MCP 集成

Claude Code 深度集成 [Model Context Protocol (MCP)](https://modelcontextprotocol.io)，允许连接外部服务（数据库、浏览器、Slack 等）。

**服务实现**：`src/services/mcp/client.ts` (3,348 行)

功能包括：
- MCP 服务器连接管理（stdio / SSE / HTTP）
- 工具和资源动态发现
- OAuth 认证支持
- 结构化内容（图片、文件）支持
- MCP 服务器权限审批流程

---

## 技术栈

| 技术 | 版本/说明 |
|------|-----------|
| **运行时** | Bun |
| **语言** | TypeScript（严格模式）|
| **终端 UI** | React + [Ink](https://github.com/vadimdemedes/ink) |
| **状态管理** | Zustand |
| **Schema 验证** | Zod v4 |
| **AI SDK** | @anthropic-ai/sdk |
| **MCP SDK** | @modelcontextprotocol/sdk |
| **CLI 解析** | Commander.js |
| **文件搜索** | ripgrep |
| **工具库** | lodash-es |
| **特性标志** | bun:bundle（编译时消除）|

---

## 特性标志系统

使用 Bun 的 `bun:bundle` 进行**编译时死代码消除**，未激活的特性代码完全不会出现在产物中：

```typescript
import { feature } from 'bun:bundle'

// 编译时决定是否包含此代码
const proactiveTool = feature('PROACTIVE') ? SleepTool : null
const voiceMode = feature('VOICE_MODE') ? VoiceModule : null
```

**主要特性标志**：

| 标志 | 功能 |
|------|------|
| `KAIROS` | 高级助手模式（智能简报等）|
| `COORDINATOR_MODE` | 多代理协调模式 |
| `VOICE_MODE` | 语音输入支持 |
| `BRIDGE_MODE` | 远程执行桥接 |
| `PROACTIVE` | 主动任务模式 |
| `TRANSCRIPT_CLASSIFIER` | 自动权限分类器 |
| `BASH_CLASSIFIER` | Bash 命令分类器 |
| `TEAMMEM` | 团队记忆同步 |
| `BUDDY` | 伴随精灵 UI |
| `CONTEXT_COLLAPSE` | 上下文折叠优化 |
| `HISTORY_SNIP` | 历史记录裁剪 |
| `MONITOR_TOOL` | MCP 监控工具 |
| `BG_SESSIONS` | 后台会话支持 |

---

## 关键数据流

### 消息类型体系

```typescript
type Message =
  | UserMessage          // 用户输入
  | AssistantMessage     // Claude 响应（含工具调用）
  | ProgressMessage      // 工具执行进度
  | SystemMessage        // 系统消息
  | AttachmentMessage    // 附件消息
  | ToolUseSummaryMessage // 工具使用摘要
  | TombstoneMessage     // 已删除消息的占位符
```

### AppState 核心字段

```typescript
type AppState = {
  messages: Message[]          // 完整消息历史
  sessionId: UUID              // 会话 ID
  toolPermissionContext: ...   // 权限上下文
  backgroundTasks: Map<string, BackgroundTask>
  inProgressToolUseIDs: Set<string>
  usage: NonNullableUsage      // Token 使用统计
  fileHistoryState: ...        // 文件编辑历史
}
```

### 成本追踪

`src/cost-tracker.ts` 追踪每次 API 调用的 Token 用量和费用，支持：
- 按 session 统计
- 输入/输出/缓存 token 分类
- 实时费用显示

---

## 主要模块说明

| 文件/目录 | 规模 | 核心职责 |
|-----------|------|----------|
| `main.tsx` | 4,683 行 | 应用主循环、CLI 初始化、消息分发 |
| `screens/REPL.tsx` | 5,005 行 | 交互式 REPL 界面全部实现 |
| `utils/messages.ts` | 5,512 行 | 消息处理工具函数库 |
| `utils/sessionStorage.ts` | 5,105 行 | 会话持久化存储 |
| `utils/hooks.ts` | 5,022 行 | Hook 执行引擎 |
| `cli/print.ts` | 5,594 行 | 输出格式化渲染 |
| `query.ts` | 1,729 行 | LLM 查询处理 |
| `QueryEngine.ts` | 1,295 行 | 查询引擎核心 |
| `services/api/claude.ts` | 3,419 行 | Anthropic API 客户端 |
| `services/mcp/client.ts` | 3,348 行 | MCP 协议客户端 |
| `utils/bash/bashParser.ts` | 4,436 行 | Bash 命令解析器 |
| `utils/attachments.ts` | 3,997 行 | 附件处理 |

---

## 环境变量参考

| 变量 | 说明 |
|------|------|
| `NODE_ENV` | `development` / `test` / `production` |
| `CLAUDE_CODE_REMOTE` | `true` — 启用远程执行模式 |
| `CLAUDE_CODE_DISABLE_THINKING` | 禁用扩展思考 |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | 禁用自动内存 |
| `DISABLE_BACKGROUND_TASKS` | 禁用后台任务 |
| `DISABLE_INTERLEAVED_THINKING` | 禁用交错思考 |
| `USER_TYPE` | `ant` — 启用内部专属工具（如 REPLTool）|
| `CLAUDE_CODE_MESSAGING_SOCKET` | 消息通信 socket 路径 |
| `NODE_OPTIONS` | Node.js 堆大小配置 |

---

## 代码规模统计

| 维度 | 数量 |
|------|------|
| 总文件数 | 1,903 |
| TypeScript 文件 | 1,884 |
| 总代码行数 | ~512,000 |
| CLI 命令 | 103+ |
| 工具 | 44+ |
| 服务 | 38+ |
| React 组件 | 146+ |
| React Hooks | 87+ |
| 工具函数模块 | 298+ |
| 任务类型 | 6 |
| Hook 事件 | 12+ |
