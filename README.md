# claude-code-sourcemap

> [!WARNING]  
> 本仓库基于 Anthropic 公开发布的 npm 包 `@anthropic-ai/claude-code` v2.1.88 中的 source map 还原，**仅供技术研究与学习**。
> 源码版权归 [Anthropic](https://www.anthropic.com) 所有，请勿用于商业用途。

## 技术概览

| 项目 | 详情 |
|------|------|
| 版本 | 2.1.88 |
| 运行时 | Bun |
| UI 框架 | React + Ink（终端渲染） |
| 源文件 | 1,884 个 `.ts` / `.tsx` |
| 总代码量 | ~51 万行 |
| 包大小 | cli.js.map 约 57 MB |
| 还原方式 | 提取 sourcemap 中 `sourcesContent` 字段 |

## 系统架构

```
                          ┌─────────────┐
                          │  main.tsx   │  CLI 入口 (Commander.js)
                          └──────┬──────┘
                                 │
                    ┌────────────┼────────────┐
                    ▼            ▼            ▼
              ┌──────────┐ ┌──────────┐ ┌──────────┐
              │ entrypoints│ │  cli/    │ │ bootstrap│
              │ cli.tsx   │ │handlers/ │ │          │
              └─────┬────┘ └──────────┘ └──────────┘
                    │
         ┌──────────┼──────────────────────┐
         ▼          ▼                      ▼
   ┌──────────┐ ┌──────────┐        ┌──────────┐
   │  query/  │ │  state/  │        │  repl    │
   │  配置引擎 │ │ AppState │        │ REPL循环 │
   └────┬─────┘ └────┬─────┘        └──────────┘
        │            │
        ▼            ▼
   ┌─────────────────────────┐
   │     services/           │  核心服务层
   │  ├─ api/    (Anthropic API)│
   │  ├─ mcp/    (MCP 协议)    │
   │  ├─ analytics/           │
   │  ├─ compact/             │
   │  ├─ plugins/             │
   │  └─ ...                  │
   └─────────────────────────┘
```

## 目录结构与模块说明

### 核心层

| 目录 | 说明 |
|------|------|
| `main.tsx` | CLI 入口，使用 Commander.js 解析命令，启动性能分析器和 REPL |
| `entrypoints/` | 启动入口，`cli.tsx` 负责快速路径（`--version`）和全量初始化，`init.ts` 负责初始化流程 |
| `query/` | 查询引擎，管理 token 预算、配置和停止钩子 |
| `state/` | 全局状态管理，`AppState.tsx` + `AppStateStore.ts`，React Context 驱动 |
| `context/` | 系统上下文和用户上下文构建 |

### 工具系统 (`tools/`)

39 个工具实现，是 Claude Code 执行能力的基础：

**文件操作：** `FileReadTool`、`FileEditTool`、`FileWriteTool`、`GlobTool`、`GrepTool`、`NotebookEditTool`

**执行环境：** `BashTool`、`PowerShellTool`、`REPLTool`

**搜索与网络：** `WebSearchTool`、`WebFetchTool`、`ToolSearchTool`

**Agent 协调：** `AgentTool`（派生 Worker）、`SendMessageTool`（向 Worker 发消息）、`TaskStopTool`（停止 Worker）、`TeamCreateTool`/`TeamDeleteTool`

**任务管理：** `TaskCreateTool`、`TaskGetTool`、`TaskListTool`、`TaskUpdateTool`、`TaskOutputTool`、`TodoWriteTool`

**计划模式：** `EnterPlanModeTool`、`ExitPlanModeTool`、`EnterWorktreeModeTool`、`ExitWorktreeModeTool`

**MCP 集成：** `MCPTool`（动态 MCP 调用）、`ListMcpResourcesTool`、`ReadMcpResourceTool`、`McpAuthTool`

**其他：** `SkillTool`、`ScheduleCronTool`、`BriefTool`、`SleepTool`、`ConfigTool`、`AskUserQuestionTool`、`LSPTool`

### 命令系统 (`commands/`)

50+ 个斜杠命令，每个命令一个目录：

**常用命令：** `/commit`、`/review`、`/diff`、`/compact`、`/model`、`/config`、`/cost`、`/clear`

**Git 与 PR：** `/branch`、`/issue`、`/pr_comments`、`/autofix-pr`、`/bughunter`

**开发工具：** `/doctor`、`/mcp`、`/plugin`、`/skills`、`/hooks`、`/permissions`、`/debug-tool-call`

**会话管理：** `/resume`、`/session`、`/rewind`、`/export`、`/share`、`/memory`

**远程与协作：** `/remote-setup`、`/remote-env`、`/bridge`、`/desktop`、`/ide`、`/chrome`

**高级功能：** `/plan`、`/effort`、`/context`、`/fast`、`/sandbox-toggle`、`/env`、`/login`、`/logout`

### 服务层 (`services/`)

| 模块 | 说明 |
|------|------|
| `api/` | Anthropic API 客户端，请求构建、重试、用量追踪、Bootstrap 数据 |
| `mcp/` | MCP 协议完整实现：连接管理、OAuth、认证、官方 Registry、VSCode SDK 集成 |
| `analytics/` | GrowthBook Feature Flags、事件埋点 |
| `compact/` | 上下文压缩，长对话自动摘要 |
| `plugins/` | 插件生命周期管理 |
| `SessionMemory/` | 会话记忆系统 |
| `MagicDocs/` | 智能文档生成 |
| `PromptSuggestion/` | 提示建议 |
| `AgentSummary/` | Agent 执行摘要 |
| `tools/` | 工具注册和调度 |
| `voice/` | 语音交互服务 |
| `rateLimitMessages/` | 限流消息处理 |
| `remoteManagedSettings/` | 远程配置管理 |
| `settingsSync/` | 设置同步 |

### 多 Agent 协调 (`coordinator/`)

单文件 `coordinatorMode.ts` 实现完整的多 Agent 编排：

- **角色**：Coordinator 不直接执行，只调度 Worker
- **工作流**：Research → Synthesis → Implementation → Verification
- **并发控制**：只读任务并行，写操作按文件集串行
- **Worker 生命周期**：通过 `AgentTool` 创建、`SendMessageTool` 续接、`TaskStopTool` 停止
- **关键设计**：Worker 完全隔离，prompt 必须自包含；Synthesis 由 Coordinator 亲自执行，防止懒代理
- **Scratchpad**：跨 Worker 共享的持久化文件区域

### 任务系统 (`tasks/`)

| 模块 | 说明 |
|------|------|
| `LocalAgentTask` | 本地 Agent 任务执行 |
| `RemoteAgentTask` | 远程 Agent 任务 |
| `LocalShellTask` | 本地 Shell 任务 |
| `InProcessTeammateTask` | 进程内队友任务 |
| `DreamTask` | 后台自动任务 |

### 技能系统 (`skills/`)

**内置技能（17 个）：**

| 技能 | 说明 |
|------|------|
| `verify` | 代码验证 |
| `review` | 代码审查（/review 命令） |
| `commit` | Git 提交 |
| `debug` | 调试辅助 |
| `stuck` | 卡住时的建议 |
| `remember` | 记忆管理 |
| `batch` | 批量操作 |
| `loop` | 循环执行 |
| `simplify` | 代码简化 |
| `claudeApi` | Claude API 调用 |
| `claudeInChrome` | Chrome 浏览器集成 |
| `scheduleRemoteAgents` | 远程 Agent 调度 |
| `skillify` | 技能生成 |
| `updateConfig` | 配置更新 |
| `keybindings` | 快捷键管理 |

### UI 层

| 目录 | 说明 |
|------|------|
| `ink/` | 自定义终端渲染引擎（Ink 框架深度定制，含 Yoga 布局、BIDI 支持、ANSI 处理） |
| `components/` | 100+ React 组件：消息展示、Diff 视图、MCP 对话框、权限提示等 |
| `screens/` | REPL 主界面、Doctor 诊断、会话恢复 |
| `hooks/` | 100+ React Hooks：输入处理、IDE 集成、语音、Vim 模式等 |
| `outputStyles/` | 输出样式系统 |

### 集成层

| 目录 | 说明 |
|------|------|
| `bridge/` | IDE 桥接（VS Code / JetBrains），WebSocket 通信、权限回调 |
| `remote/` | 远程会话管理，WebSocket 连接、权限桥接、SDK 消息适配 |
| `server/` | Direct Connect 服务器，用于远程开发环境 |
| `upstreamproxy/` | 上游代理和请求转发 |
| `vim/` | 完整 Vim 模式：motion、operator、text object |
| `voice/` | 语音输入/输出模式 |

### 其他

| 目录 | 说明 |
|------|------|
| `utils/` | 80+ 工具函数：Git、Shell、认证、API、文件系统、终端、ANSI 转换等 |
| `schemas/` | JSON Schema 定义（hooks 等） |
| `constants/` | 全局常量 |
| `types/` | TypeScript 类型定义 |
| `native-ts/` | 原生 TypeScript 模块 |
| `migrations/` | 数据迁移脚本 |
| `moreright/` | 扩展功能 |

## 关键技术决策

1. **Bun 运行时**：利用 `bun:bundle` 的 `feature()` 实现编译时特性门控和死代码消除
2. **React + Ink**：用 React 组件模型构建终端 UI，状态管理清晰
3. **MCP 协议**：深度集成的 Model Context Protocol，支持 OAuth、官方 Registry、VSCode SDK
4. **Source Map 还原**：所有源码从 npm 包的 sourcemap `sourcesContent` 字段提取，保留完整原始注释
5. **性能优化**：启动时并行预热（Keychain、MDM、Bootstrap）、快速路径（`--version` 零加载）、`--dump-system-prompt` 懒加载

## 声明

- 源码版权归 [Anthropic](https://www.anthropic.com) 所有
- 本仓库仅用于技术研究与学习，请勿用于商业用途
- 如有侵权，请联系删除
