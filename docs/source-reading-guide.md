# pi 源码阅读指南

本文档梳理 pi 项目的整体架构和运行逻辑，帮助读者快速建立对代码库的全局理解。

## 项目概览

pi 是一个自扩展的编码代理 CLI（https://pi.dev），采用 monorepo 结构，锁步版本（当前 0.75.5），包含四个包：

| 包 | npm 包名 | 职责 |
|---|---|---|
| `packages/tui` | `@earendil-works/pi-tui` | 终端 UI 库，差分渲染引擎 |
| `packages/ai` | `@earendil-works/pi-ai` | 统一多供应商 LLM API（20+ 供应商） |
| `packages/agent` | `@earendil-works/pi-agent-core` | Agent 运行时：工具调用、会话管理、上下文压缩 |
| `packages/coding-agent` | `@earendil-works/pi-coding-agent` | 交互式编码代理 CLI（主产品） |

**构建顺序**：`tui → ai → agent → coding-agent`（coding-agent 依赖其他三个包，ai 和 tui 无内部依赖）。

**关键配置**：
- TypeScript 5.9，strict 模式，仅允许可擦除语法（Node strip-only 模式）
- Tab 缩进（宽度 3），120 字符行宽（Biome 格式化）
- 目标：Node.js >=22.19.0，ES2022
- 不允许 `enum`、`namespace`、`import =`、`export =`、参数属性、内联 import

---

## 第一阶段：从启动流程开始

### 1. CLI 入口与参数解析

**核心文件**：`packages/coding-agent/src/main.ts`

启动流程：

```
main.ts
  ├─ parseArgs()                    # 解析 CLI 参数
  ├─ 确定运行模式                    # interactive / print / json / rpc
  ├─ createAgentSessionServices()   # 创建核心服务
  │   ├─ SettingsManager            # 用户设置
  │   ├─ ModelRegistry              # 可用 LLM 模型
  │   ├─ AuthStorage                # API 密钥
  │   └─ ResourceLoader             # 扩展、技能、提示模板、主题
  ├─ createAgentSessionRuntime()    # 包装会话生命周期
  └─ 执行对应模式
      ├─ InteractiveMode            # 交互式 REPL
      ├─ runPrintMode()             # 单次执行
      └─ runRpcMode()               # JSON-RPC 模式
```

**入口点**：
- `src/cli.ts` — 主 CLI 入口
- `src/main.ts` — 参数解析和服务创建
- `src/index.ts` — SDK 导出（供程序化使用）

### 2. 交互模式

**核心文件**：`packages/coding-agent/src/modes/`

交互模式实现了一个 REPL 循环：

1. **接收输入** — 用户在终端输入提示
2. **提交给 agent** — 将消息发送到 agent 循环
3. **渲染输出** — 实时流式显示 LLM 响应和工具执行
4. **等待下一轮** — 循环回到输入

其他模式（print、json、rpc）共享相同的 agent 核心逻辑，差异在于输入/输出的处理方式。

### 3. 系统提示构建

**核心文件**：`packages/coding-agent/src/core/system-prompt.ts`

系统提示由多个部分拼接而成：

```
buildSystemPrompt()
  ├─ 工具列表（可用工具及描述）
  ├─ 行为准则（简洁性、文件路径等规则）
  ├─ 文档链接（pi 文档和示例路径）
  ├─ 自定义提示（可选覆盖）
  ├─ 上下文文件（AGENTS.md / CLAUDE.md）
  ├─ 技能部分（可用技能描述）
  └─ 日期和工作目录
```

扩展可以通过 `before_agent_start` 事件修改系统提示；工具可以定义 `promptSnippet` 和 `promptGuidelines` 字段。

---

## 第二阶段：Agent 核心

### 4. Agent 主逻辑

**核心文件**：`packages/agent/src/agent.ts`

`Agent` 类是高层有状态封装，管理：
- 消息队列和事件发射
- Agent 生命周期（启动、停止、暂停）
- 与底层 agent loop 的协调

### 5. Agent 循环（核心中的核心）

**核心文件**：`packages/agent/src/agent-loop.ts`

这是整个项目最关键的文件。理解了它，就理解了 pi 的运行机制。

**循环结构**：

```
agentLoop() / agentLoopContinue()
  │
  └─ while (true)                           # 外层循环：处理后续消息
       │
       └─ while (hasToolCalls || pending)    # 内层循环：工具调用 + 转向消息
            │
            ├─ 1. 处理待定消息（转向/steering）
            ├─ 2. 流式获取 LLM 响应
            ├─ 3. 执行工具调用（并行或顺序）
            ├─ 4. 检查 shouldStopAfterTurn
            └─ 5. 检查转向消息
```

**事件流**：

```
agent_start → turn_start → message_start → message_update* → tool_execution_start
→ tool_execution_update* → tool_execution_end → message_end → turn_end → agent_end
```

**工具执行模式**：
- **顺序执行**（`executeToolCallsSequential`）：逐个执行，每个完成后发出 `tool_execution_end`
- **并行执行**（`executeToolCallsParallel`）：预检顺序执行，实际执行并发，按完成顺序发出事件，最后按原始顺序返回结果

**工具调用生命周期**：

```
1. prepareToolCall()          # 验证参数，检查 beforeToolCall 钩子
2. executePreparedToolCall()  # 执行工具，流式更新
3. finalizeExecutedToolCall() # 应用 afterToolCall 钩子覆盖
4. 发出 tool_execution_end 事件和 toolResult 消息
```

**关键特性**：
- 每轮动态解析 API 密钥（支持过期的 OAuth token）
- 工具结果可通过 `afterToolCall` 钩子覆盖
- 工具结果可设置 `terminate: true` 提前终止
- 支持中途注入转向消息（steering messages）
- LLM 停止后仍可处理后续消息（follow-up messages）

### 6. 会话管理与持久化

**核心文件**：
- `packages/agent/src/harness/agent-harness.ts` — 高层编排
- `packages/agent/src/harness/session/session.ts` — 会话抽象
- `packages/agent/src/harness/session/jsonl-storage.ts` — JSONL 存储
- `packages/agent/src/harness/session/jsonl-repo.ts` — 多会话仓库

**JSONL 存储格式**（v3）：

```jsonl
{"type":"session","version":3,"id":"...","cwd":"...","parentSession":"..."}
{"type":"message","id":"...","parentId":"...","message":{...}}
{"type":"compaction","id":"...","summary":"...","firstKeptEntryId":"...","tokensBefore":12345}
{"type":"branch_summary","id":"...","fromId":"...","summary":"..."}
{"type":"leaf","id":"...","targetId":"..."}
```

**条目类型**：
- `message` — 用户/助手/工具结果消息
- `compaction` — 对话压缩点
- `branch_summary` — 分支导航摘要
- `leaf` — 当前活跃树位置标记
- `thinking_level_change` / `model_change` — 设置变更
- `label` / `session_info` / `custom` — 元数据和自定义数据

**会话树结构**：
- 每个条目有 `id`、`parentId`、`timestamp`
- `leaf` 条目记录当前活跃位置
- `getPathToRoot(leafId)` 通过回溯父节点重建分支
- 支持在同一文件中共存多个分支

**上下文构建**（`session.ts` → `buildContext()`）：

```
1. 扫描路径找到最新的 model/thinking-level/compaction
2. 如果存在压缩：
   - 插入 compactionSummary 消息
   - 跳过 firstKeptEntryId 之前的条目
   - 包含压缩点之后的所有条目
3. 否则：包含所有消息
```

**分支操作**：
- `moveTo(entryId, summary?)` — 导航到不同分支点
- 离开分支时生成 `branch_summary` 条目
- 公共祖先检测用于高效摘要

### 7. 上下文压缩（Compaction）

**核心文件**：`packages/agent/src/harness/compaction/compaction.ts`

当上下文接近模型 token 限制时自动触发压缩。

**默认设置**：

```typescript
{
  enabled: true,
  reserveTokens: 16384,     // 为摘要提示/输出预留
  keepRecentTokens: 20000,  // 保留的近期上下文
}
```

**Token 估算**：
- 优先使用实际的 provider usage（`getLastAssistantUsage`）
- 回退到基于字符的估算（1 token ≈ 4 字符）

**压缩流程**：

```
shouldCompact() → prepareCompaction() → generateSummary() → compact()
     │                    │                     │
     │              找到切割点           使用 LLM 生成摘要
     │              收集待摘要消息
     │
  contextTokens > contextWindow - reserveTokens
```

**结构化摘要格式**：

```markdown
## Goal
[用户目标]

## Constraints & Preferences
- [约束条件]

## Progress
### Done / In Progress / Blocked

## Key Decisions
- **Decision**: Rationale

## Next Steps
1. [有序的下一步]

## Critical Context
- [重要数据/引用]

<read-files>[...]</read-files>
<modified-files>[...]</modified-files>
```

如果压缩在轮次中间切割，会生成两个摘要：主历史摘要和轮次前缀摘要。

---

## 第三阶段：AI 抽象层

### 8. 统一类型系统

**核心文件**：`packages/ai/src/types.ts`

所有 provider 共享的统一接口：

**消息类型**：

```typescript
type Message = UserMessage | AssistantMessage | ToolResultMessage;

interface UserMessage {
  role: "user";
  content: string | (TextContent | ImageContent)[];
  timestamp: number;
}

interface AssistantMessage {
  role: "assistant";
  content: (TextContent | ThinkingContent | ToolCall)[];
  api: Api;              // 使用的 API 类型
  provider: Provider;     // 供应商
  model: string;          // 模型 ID
  usage: Usage;           // token 使用量
  stopReason: StopReason;
  timestamp: number;
}

interface ToolResultMessage {
  role: "toolResult";
  toolCallId: string;
  toolName: string;
  content: (TextContent | ImageContent)[];
  isError: boolean;
  timestamp: number;
}
```

**工具定义**：

```typescript
interface Tool<TParameters extends TSchema = TSchema> {
  name: string;
  description: string;
  parameters: TSchema;    // TypeBox schema
}
```

**上下文结构**：

```typescript
interface Context {
  systemPrompt?: string;
  messages: Message[];
  tools?: Tool[];
}
```

**流式事件**（`AssistantMessageEvent`）：

```
start → text_start → text_delta* → text_end → thinking_start → thinking_delta*
→ thinking_end → toolcall_start → toolcall_delta* → toolcall_end → done / error
```

每个事件都携带 `partial: AssistantMessage`，允许实时构建完整消息。

### 9. Provider 注册与延迟加载

**核心文件**：
- `packages/ai/src/api-registry.ts` — 注册表
- `packages/ai/src/providers/register-builtins.ts` — 内置注册
- `packages/ai/src/providers/` — 各 provider 实现

**延迟加载模式**：

```typescript
// 注册时只创建一个懒加载包装器
function createLazyStream<TApi, TOptions>(
  loadModule: () => Promise<ProviderModule>,
): StreamFunction {
  return (model, context, options) => {
    const outer = new AssistantMessageEventStream();
    loadModule()  // 首次调用时才真正 import
      .then(module => {
        const inner = module.stream(model, context, options);
        forwardStream(outer, inner);
      })
      .catch(error => {
        outer.push({ type: "error", reason: "error", error });
        outer.end();
      });
    return outer;
  };
}
```

**支持的 API 类型**（9 种）：
- `anthropic-messages`
- `openai-completions`（OpenAI 兼容）
- `openai-responses`
- `azure-openai-responses`
- `openai-codex-responses`
- `mistral-conversations`
- `google-generative-ai`
- `google-vertex`
- `bedrock-converse-stream`

每个 API 类型对应一个 provider 目录（如 `providers/anthropic.ts`、`providers/openai-completions.ts`）。

**Provider 适配器模式**：
1. 接收统一的 `Context`（消息 + 工具 + 系统提示）
2. 转换为供应商特定的 API 格式
3. 发起流式请求
4. 将供应商特定的事件转换为统一的 `AssistantMessageEvent`

### 10. 公共 API

**核心文件**：`packages/ai/src/stream.ts`、`packages/ai/src/index.ts`

```typescript
// 流式调用
function stream<TApi>(model: Model<TApi>, context: Context, options?): AssistantMessageEventStream

// 完整完成
async function complete<TApi>(model: Model<TApi>, context: Context, options?): Promise<AssistantMessage>

// 简化流式（自动处理 thinking 等细节）
function streamSimple<TApi>(model, context, options?): AssistantMessageEventStream

// 简化完成
async function completeSimple<TApi>(model, context, options?): Promise<AssistantMessage>
```

### 11. 模型生成系统

**核心文件**：
- `packages/ai/scripts/generate-models.ts` — 生成脚本（~1950 行）
- `packages/ai/src/models.generated.ts` — 自动生成（421KB+，**不要手动编辑**）
- `packages/ai/src/models.ts` — 运行时模型注册表

生成脚本从多个来源获取模型数据：
- `https://models.dev/api.json`（主要来源）
- OpenRouter API（动态模型发现）
- Vercel AI Gateway 目录

运行时提供：`getModel()`、`getProviders()`、`getModels()`、`calculateCost()`

### 12. 消息转换

**核心文件**：`packages/ai/src/transform-messages.ts`

处理跨供应商差异：
- **工具调用 ID 规范化**：OpenAI ID 可达 450+ 字符，Anthropic 限制 64 字符
- **消息格式转换**：不同供应商的消息结构差异
- **思考块处理**：签名保留，编辑块处理
- **孤立的工具调用**：插入合成的工具结果
- **图像降级**：非视觉模型自动移除图像

### 13. 高级特性

**思考/推理支持**：
- 统一的 `SimpleStreamOptions`，带 `reasoning?: ThinkingLevel`
- 供应商特定实现：Anthropic（自适应/预算），OpenAI（reasoning_effort），Google（思考 token 预算）

**Prompt 缓存**：
- `cacheRetention?: "none" | "short" | "long"`
- 供应商特定实现（Anthropic cache_control，OpenAI prompt_cache_key）

**成本追踪**：
- 基于 token 使用量的内置成本计算
- 模型定义中的每供应商成本数据

**图像支持**：
- `packages/ai/src/images.ts`、`packages/ai/src/image-models.ts`
- 独立的图像模型注册表

---

## 第四阶段：工具系统

### 14. 工具架构

**核心目录**：`packages/coding-agent/src/core/tools/`

每个工具由三部分组成：
1. **TypeBox Schema** — 参数验证
2. **execute 函数** — 实际执行逻辑
3. **渲染函数** — TUI 中的展示（`renderCall`、`renderResult`）

**工具接口**（agent 包定义）：

```typescript
interface AgentTool<TParameters> extends Tool<TParameters> {
  label: string;                    // UI 显示名
  prepareArguments?: (args) => Static<TParameters>;  // 预验证转换
  execute: (toolCallId, params, signal?, onUpdate?) => Promise<AgentToolResult>;
  executionMode?: "sequential" | "parallel";  // 每个工具可覆盖
}
```

**可插拔操作**：每个核心工具都支持自定义 `Operations` 接口，允许远程委托（如 SSH 扩展将 bash 操作转发到远程机器）。

### 15. 核心工具详解

**read**（`tools/read.ts`）：
- 支持图像和自动调整大小
- 行偏移/限制的部分读取
- TUI 中的语法高亮
- 自定义 `ReadOperations` 用于远程系统

**bash**（`tools/bash.ts`）：
- 最复杂的工具
- 支持自定义 `BashOperations` 用于远程执行
- 超时支持、环境变量控制
- `BashSpawnHook` 用于预执行修改

**edit**（`tools/edit.ts`）：
- 精确文本编辑，单次调用多个替换
- Diff 生成和显示
- 行尾规范化

**write**（`tools/write.ts`）：
- 文件创建，自动创建目录

**grep**（`tools/grep.ts`）：
- 内容搜索，尊重 .gitignore
- 大小写敏感控制

**find**（`tools/find.ts`）：
- 文件发现，.gitignore 感知

**ls**（`tools/ls.ts`）：
- 目录列表

### 16. 工具执行流程

```
LLM 请求工具调用（带参数）
  │
  ├─ prepareArguments()     # 可选的预验证转换
  ├─ validateToolArguments() # TypeBox schema 验证
  ├─ beforeToolCall() 钩子  # 扩展可阻止或修改
  ├─ tool.execute()         # 实际执行（可调用 onUpdate 流式更新）
  ├─ afterToolCall() 钩子   # 扩展可覆盖结果/错误/终止
  └─ 发出 tool_execution_end 事件
```

---

## 第五阶段：扩展与定制机制

### 17. 扩展系统

**核心文件**：
- `packages/coding-agent/src/core/extensions/loader.ts` — 发现和加载
- `packages/coding-agent/src/core/extensions/runner.ts` — 生命周期管理
- `packages/coding-agent/src/core/extensions/types.ts` — 类型定义

**扩展发现位置**：
- 项目本地：`cwd/.pi/extensions/`
- 全局：`~/.pi/agent/extensions/`
- CLI 指定：`--extensions path`
- SDK 内联工厂

**扩展 API（`ExtensionAPI`）**：

| 能力 | 方法 |
|---|---|
| 事件订阅 | `on(event, handler)` |
| 工具注册 | `registerTool(tool)` |
| 命令注册 | `registerCommand(name, options)` |
| 快捷键注册 | `registerShortcut(keyId, options)` |
| 标志注册 | `registerFlag(name, options)` |
| 消息渲染 | `registerMessageRenderer(customType, renderer)` |
| 发送消息 | `sendMessage()`, `sendUserMessage()` |
| 会话操作 | `appendEntry()`, `setSessionName()` |
| Provider 注册 | `registerProvider()` |

**扩展生命周期**：
1. 使用 `jiti` 动态加载 TypeScript/JavaScript 模块
2. Runner 绑定核心服务（替换加载时的占位 stub）
3. 扩展获得真实操作能力，可开始交互
4. 会话关闭时使旧上下文失效

### 18. 事件钩子

钩子不是独立模块，而是集成在扩展事件系统中：

| 事件 | 触发时机 | 用途 |
|---|---|---|
| `tool_call` | 工具执行前 | 可阻止或修改参数 |
| `tool_result` | 工具执行后 | 可修改结果 |
| `user_bash` | 用户 `!` 前缀命令 | 可替换为远程执行 |
| `before_agent_start` | LLM 调用前 | 可修改系统提示 |
| `context` | 发送给 LLM 前 | 可修改消息 |
| `before_provider_request` | 网络请求前 | 网络拦截 |
| `after_provider_response` | 网络响应后 | 网络拦截 |

### 19. 技能系统（Skills）

**核心文件**：`packages/coding-agent/src/core/skills.ts`

- Markdown 文件，带 frontmatter（name、description）
- 自动发现位置：`~/.pi/agent/skills/` 和 `cwd/.pi/skills/`
- 验证规则：名称最长 64 字符，描述最长 1024 字符
- 注入到系统提示，供 LLM 发现和使用

### 20. 提示模板

**核心文件**：`packages/coding-agent/src/core/prompt-templates.ts`

- Markdown 文件，带可选 frontmatter
- 支持参数替换：`$1`、`$@`、`${@:2:3}`
- 通过 `/template-name` 语法访问

### 21. 主题系统

**核心文件**：`packages/coding-agent/src/modes/interactive/theme/`

- JSON 主题文件（dark.json、light.json）
- 可自定义 TUI 色彩方案
- 支持热重载
- 扩展可通过 `resources_discover` 事件注册主题

---

## 第六阶段：TUI 渲染引擎

### 22. 三策略差分渲染

**核心文件**：`packages/tui/src/tui.ts`

**策略一：首次渲染**
- 触发条件：`previousLines.length === 0` 且无宽高变化
- 行为：输出所有行，不清除回滚缓冲区

**策略二：全屏重绘**
- 触发条件：终端宽度变化、高度变化（Termux 键盘切换除外）、内容缩减到工作区以下、首个变化行在视口之上
- 行为：清屏 + 完整重绘

**策略三：差分更新**
- 触发条件：视口内的正常内容变化
- 行为：
  1. 与上次渲染比较，找到首尾变化行
  2. 移动光标到首个变化行
  3. 清除到屏幕末尾
  4. 只渲染变化的行
  5. 追加优化：检测内容是否只是追加

所有渲染包裹在 CSI 2026 同步输出序列中，实现原子无闪烁更新。

### 23. 组件模型

**组件接口**：

```typescript
interface Component {
  render(width: number): string[];       // 必须：渲染为行数组
  handleInput?(data: string): void;      // 可选：键盘输入
  wantsKeyRelease?: boolean;             // 可选：接收 Kitty 释放事件
  invalidate(): void;                    // 清除缓存状态
}
```

**Focusable 接口**（用于 IME 支持）：
- 组件在光标位置发出 `CURSOR_MARKER`（`\x1b_pi:c\x07`）
- TUI 扫描渲染输出，找到标记，定位硬件光标
- 确保中文输入法候选窗口正确定位

**内置组件**：

| 组件 | 文件 | 用途 |
|---|---|---|
| Container | — | 基础组合容器 |
| Box | `box.ts` | 带内边距和背景的容器 |
| Text | `text.ts` | 多行文本，ANSI 感知的自动换行 |
| Input | `input.ts` | 单行输入，Emacs 风格编辑，撤销/重做 |
| Editor | `editor.ts` | 多行编辑器，自动补全，斜杠命令 |
| Markdown | `markdown.ts` | Markdown 渲染，语法高亮 |
| SelectList | `select-list.ts` | 交互式选择，模糊过滤 |
| Loader | `loader.ts` | 动画加载指示器 |
| Image | `image.ts` | 终端内联图像（Kitty/iTerm2 协议） |

### 24. 原生模块

**macOS**（`native/darwin/src/darwin-modifiers.c`）：
- 使用 CoreGraphics 检测修饰键状态
- 解决 Apple Terminal 无法区分 Shift+Enter 的问题

**Windows**（`native/win32/src/win32-console-mode.c`）：
- 启用虚拟终端输入
- 让 Windows 终端发送 VT 转义序列

两个模块都通过动态加载，从多个可能路径查找。

### 25. 终端抽象与能力检测

**终端接口**（`terminal.ts`）：

```typescript
interface Terminal {
  start(onInput, onResize): void;
  write(data: string): void;
  get columns(): number;
  get rows(): number;
  moveBy(lines: number): void;
  hideCursor() / showCursor(): void;
  clearLine() / clearFromCursor() / clearScreen(): void;
  setTitle(title: string): void;
  setProgress(active: boolean): void;
}
```

**键盘协议**：
- Kitty 键盘协议（增强模式，区分按下/重复/释放）
- xterm modifyOtherKeys（回退）
- 标准 CSI 序列（方向键、功能键）
- 平台特定处理（Apple Terminal、Windows Terminal）

**图像支持检测**：Kitty/Ghostty/WezTerm 协议、iTerm2 协议，tmux/screen 下禁用。

**宽度计算**：使用 `Intl.Segmenter` 进行字形聚类，支持东亚宽字符和 emoji。

---

## 附录 A：关键文件索引

### 快速定位入口

| 想了解... | 看这个文件 |
|---|---|
| CLI 怎么启动 | `packages/coding-agent/src/main.ts` |
| Agent 循环逻辑 | `packages/agent/src/agent-loop.ts` |
| LLM 类型定义 | `packages/ai/src/types.ts` |
| 某个 Provider 实现 | `packages/ai/src/providers/<name>.ts` |
| 工具怎么定义 | `packages/agent/src/types.ts`（AgentTool 接口） |
| 工具具体实现 | `packages/coding-agent/src/core/tools/<name>.ts` |
| 扩展怎么加载 | `packages/coding-agent/src/core/extensions/loader.ts` |
| 会话怎么存储 | `packages/agent/src/harness/session/jsonl-storage.ts` |
| 上下文怎么压缩 | `packages/agent/src/harness/compaction/compaction.ts` |
| 渲染怎么工作 | `packages/tui/src/tui.ts` |
| 模型列表怎么生成 | `packages/ai/scripts/generate-models.ts` |
| 系统提示怎么构建 | `packages/coding-agent/src/core/system-prompt.ts` |

### 按包组织的文件树

```
packages/ai/src/
├── index.ts                    # 主导出
├── api-registry.ts             # Provider 注册表
├── stream.ts                   # stream/complete/streamSimple/completeSimple
├── types.ts                    # 核心类型（Message, Tool, Context, Events）
├── models.ts                   # 运行时模型注册表
├── models.generated.ts         # 自动生成的模型目录（不要编辑！）
├── transform-messages.ts       # 跨供应商消息转换
├── env-api-keys.ts             # 环境变量处理
├── images.ts / image-models.ts # 图像支持
├── providers/
│   ├── register-builtins.ts    # 延迟注册所有内置 provider
│   ├── anthropic.ts            # Anthropic Messages API
│   ├── openai-completions.ts   # OpenAI Chat Completions
│   ├── openai-responses.ts     # OpenAI Responses API
│   └── ...                     # 其他 provider
└── utils/
    ├── event-stream.ts         # 异步可迭代事件流
    └── json-parse.ts           # 流式 JSON 解析

packages/agent/src/
├── index.ts                    # 主导出
├── types.ts                    # AgentTool, AgentEvent, AgentMessage 等
├── agent.ts                    # Agent 类（高层封装）
├── agent-loop.ts               # 核心循环（最关键的文件）
├── harness/
│   ├── agent-harness.ts        # 高层编排（会话、技能、提示）
│   ├── system-prompt.ts        # 系统提示格式化
│   ├── skills.ts               # 技能加载
│   ├── messages.ts             # 消息类型转换
│   ├── compaction/
│   │   ├── compaction.ts       # 上下文压缩
│   │   └── branch-summarization.ts
│   ├── session/
│   │   ├── session.ts          # 会话抽象
│   │   ├── jsonl-storage.ts    # JSONL 持久化
│   │   └── jsonl-repo.ts       # 多会话仓库
│   └── utils/
│       ├── shell-output.ts     # Shell 输出格式化
│       └── truncate.ts         # 文本截断
└── proxy.ts                    # 浏览器代理

packages/coding-agent/src/
├── cli.ts                      # CLI 入口
├── main.ts                     # 参数解析 + 服务创建
├── index.ts                    # SDK 导出
├── core/
│   ├── system-prompt.ts        # 系统提示构建
│   ├── skills.ts               # 技能发现和验证
│   ├── prompt-templates.ts     # 提示模板
│   ├── extensions/
│   │   ├── loader.ts           # 扩展加载（jiti）
│   │   ├── runner.ts           # 扩展生命周期
│   │   └── types.ts            # ExtensionAPI 类型
│   └── tools/
│       ├── read.ts             # 文件读取
│       ├── bash.ts             # 命令执行
│       ├── edit.ts             # 文本编辑
│       ├── write.ts            # 文件写入
│       ├── grep.ts             # 内容搜索
│       ├── find.ts             # 文件发现
│       └── ls.ts               # 目录列表
├── modes/
│   ├── interactive/            # 交互模式
│   └── print.ts / rpc.ts       # 其他模式
└── utils/                      # 工具函数

packages/tui/src/
├── index.ts                    # 主导出
├── tui.ts                      # 核心渲染引擎
├── terminal.ts                 # 终端抽象
├── keys.ts                     # 键盘输入解析
├── keybindings.ts              # 快捷键管理
├── stdin-buffer.ts             # 输入批处理
├── autocomplete.ts             # 自动补全
├── components/
│   ├── box.ts / text.ts / input.ts / editor.ts
│   ├── markdown.ts / select-list.ts / loader.ts
│   └── image.ts
├── native/
│   ├── darwin/                 # macOS 原生模块
│   └── win32/                  # Windows 原生模块
└── utils.ts                    # ANSI 处理、宽度计算
```

---

## 附录 B：数据流总览

### 一次完整的交互流程

```
用户输入 "帮我修复这个 bug"
  │
  ▼
[交互模式] Input 组件接收输入
  │
  ▼
[Agent Harness] 创建 UserMessage，追加到 Session
  │
  ▼
[Agent Loop] 检查是否需要压缩上下文
  │── 需要 → [Compaction] 生成摘要，替换旧消息
  │
  ▼
[Agent Loop] 构建完整上下文（systemPrompt + 消息 + 工具）
  │
  ▼
[AI 层] transformMessages() → 转换为供应商格式
  │
  ▼
[Provider] 延迟加载对应 provider → 发起流式请求
  │
  ▼
[AI 层] 接收 AssistantMessageEvent 流
  │── text_delta → TUI 实时渲染文本
  │── toolcall_end → 准备执行工具
  │
  ▼
[Agent Loop] 执行工具调用
  │── beforeToolCall 钩子
  │── tool.execute()（可能并行）
  │── afterToolCall 钩子
  │
  ▼
[Agent Loop] 将 ToolResultMessage 追加到上下文，继续循环
  │
  ▼
[Provider] 再次调用 LLM（带工具结果）
  │
  ▼
... 循环直到 LLM 返回 stop（无工具调用）
  │
  ▼
[Session] 持久化完整对话到 JSONL
  │
  ▼
[交互模式] 等待下一轮输入
```

---

## 附录 C：测试指南

### 测试结构

```
packages/ai/test/           # Provider 测试（60+ 文件）
packages/agent/test/        # 核心 agent 测试
packages/coding-agent/test/
  └── suite/                # harness 测试套件
      ├── harness.ts        # 主测试 harness（使用 faux provider）
      └── regressions/      # 回归测试：<issue-number>-<slug>.test.ts
```

### 运行测试

```bash
./test.sh                    # 所有非 e2e 测试（自动清理 API 密钥）
node ../../node_modules/vitest/dist/cli.js --run test/specific.test.ts  # 单个测试
```

**关键规则**：
- 不要运行 `npm test`（当环境变量存在时会触发 e2e 测试）
- `packages/coding-agent/test/suite/` 使用 harness + faux provider，不需要真实 API 密钥
- 回归测试放在 `regressions/<issue-number>-<slug>.test.ts`

---

## 阅读建议

1. **从启动流程开始**（第一阶段），跟踪一次完整的交互，建立整体感觉
2. **重点读 `agent-loop.ts`**（第二阶段第 5 步）—— 读懂这个文件，整个运行机制就通了
3. **挑一个 provider 看**（第三阶段第 9 步），理解适配器模式
4. **挑一个简单工具看**（第四阶段），如 `read` 或 `write`，理解工具架构
5. **TUI 最后看**（第六阶段），它相对独立
6. 每一步边读边在 `./pi-test.sh` 里跑起来验证理解
7. 遇到不理解的类型，用 LSP `goToDefinition` 追溯
8. 工具系统可以跳着看，不需要每个工具都读
