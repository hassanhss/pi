# Agent Event 流转路径：emit 事件的消费链路

梳理 `agent-loop.ts` 中 `emit` 发出的事件如何被层层消费，最终到达 UI 层渲染给用户。

---

## 核心流程图

```
agent-loop.ts  emit(event)
       │
       ▼
 ┌─────────────────────────────────────────────────┐
 │  两类调用者传入 emit 并接收事件：                    │
 │                                                   │
 │  ① Agent 类 (agent.ts:386,402)                   │
 │     → emit = (e) => this.processEvents(e)        │
 │                                                   │
 │  ② AgentHarness 类 (agent-harness.ts:587)        │
 │     → emit = (e) => this.handleAgentEvent(e)     │
 └─────────┬───────────────────────┬────────────────┘
           │                       │
           ▼                       ▼
    Agent.processEvents()    AgentHarness.handleAgentEvent()
      │ 更新内部状态              │ 持久化 session
      │ 分发给 subscribers       │ 分发给 subscribers
      ▼                       ▼
  AgentSession              harness.subscribe() 监听者
  ._handleAgentEvent()
      │
      │ → _emitExtensionEvent() 扩展事件
      │ → _emit() 分发给 UI 层监听者
      │ → 持久化到 SessionManager
      ▼
  ┌──────────────────────────────────────────┐
  │  最终消费者 (4种模式):                      │
  │                                          │
  │  • InteractiveMode → TUI 渲染 (终端界面)  │
  │  • PrintMode       → JSON/文本输出到 stdout │
  │  • RPCMode         → RPC 客户端转发        │
  │  • SDK             → 用户自定义回调         │
  └──────────────────────────────────────────┘
```

---

## 事件类型一览

`emit` 发出的 `AgentEvent` 是一个 discriminated union（类型定义在 `packages/agent/src/types.ts:403-418`）：

| 事件类型 | 含义 | 携带数据 |
|---|---|---|
| `agent_start` | Agent 生命周期开始 | — |
| `agent_end` | Agent 生命周期结束 | `messages: AgentMessage[]` |
| `turn_start` | 一轮对话开始 | — |
| `turn_end` | 一轮对话结束 | `message`, `toolResults` |
| `message_start` | 单条消息开始 | `message: AgentMessage` |
| `message_update` | 流式消息增量更新 | `message`, `assistantMessageEvent` |
| `message_end` | 单条消息结束 | `message` |
| `tool_execution_start` | 工具开始执行 | `toolCallId`, `toolName`, `args` |
| `tool_execution_update` | 工具执行进度更新 | `toolCallId`, `toolName`, `args`, `partialResult` |
| `tool_execution_end` | 工具执行结束 | `toolCallId`, `toolName`, `result`, `isError` |

---

## 阅读顺序

### 1. emit 的源头：agent-loop

**`packages/agent/src/agent-loop.ts`**

- `runAgentLoop()` (line 95) 和 `runAgentLoopContinue()` (line 120) 接收 `emit: AgentEventSink` 参数
- `runLoop()` (line 155) 是核心循环，在各种阶段调用 `emit` 发出事件
- `streamAssistantResponse()` (line 275) 在 LLM 流式响应时发出 `message_start/update/end`
- `executeToolCalls()` (line 373) 在工具执行时发出 `tool_execution_start/update/end`

### 2. 第一层消费：Agent 类

**`packages/agent/src/agent.ts`**

- `runPromptMessages()` (line 386) 调用 `runAgentLoop()`，传入 `emit = (e) => this.processEvents(e)`
- `runContinuation()` (line 402) 调用 `runAgentLoopContinue()`，同理
- `processEvents()` (line 509) 先更新内部状态（`streamingMessage`、`pendingToolCalls` 等），然后遍历所有 subscribers 分发事件
- `subscribe(listener)` (line 231) 注册监听者

### 3. 第一层消费（备选路径）：AgentHarness 类

**`packages/agent/src/harness/agent-harness.ts`**

- `executeTurn()` (line 587) 调用 `runAgentLoop()`，传入 `emit = (e) => this.handleAgentEvent(e, signal)`
- `handleAgentEvent()` (line 510) 处理事件：
  - `message_end` → 持久化到 session，再分发
  - `turn_end` → 分发 + 刷新写入 + 发出 `save_point`
  - `agent_end` → 刷新写入 + 转为 idle + 分发 + 发出 `settled`
  - 其他事件 → 直接分发
- `emitAny()` (line 239) 遍历所有 subscriber 调用
- `emitOwn()` 发出 harness 特有事件（`queue_update`、`save_point`、`abort`、`settled` 等）

### 4. 中间层：AgentSession

**`packages/coding-agent/src/core/agent-session.ts`**

- 构造时 (line 343) 订阅 Agent：`this.agent.subscribe(this._handleAgentEvent)`
- `_handleAgentEvent()` (line 476) 是核心事件处理器：
  - 跟踪 steering/follow-up 队列状态
  - `_emitExtensionEvent()` 转发到扩展运行器
  - `_emit()` 分发给用户注册的监听者（`_eventListeners[]`）
  - 在 `message_end` 时持久化到 `SessionManager`
  - 跟踪最后一条 assistant 消息用于自动压缩
- `AgentSessionEvent` 在 `AgentEvent` 基础上扩展了 `queue_update`、`compaction_start/end`、`session_info_changed`、`thinking_level_changed`、`auto_retry_start/end` 等会话级事件

### 5. 最终消费者：UI 模式

#### InteractiveMode（终端 TUI）

**`packages/coding-agent/src/modes/interactive/interactive-mode.ts`**

- line 2674 订阅 session：`session.subscribe(async (event) => this.handleEvent(event))`
- `handleEvent()` (line 2679) 根据事件类型渲染终端 UI：
  - `agent_start` → 清除待处理工具，启动加载动画
  - `message_start` → 创建 UI 组件（如 `AssistantMessageComponent`）
  - `message_update` → 更新流式内容
  - `message_end` → 完成渲染，触发工具执行展示
  - `tool_execution_start/update/end` → 管理工具执行 UI
  - `turn_end` → 完成轮次展示
  - `agent_end` → 停止加载器，处理重试逻辑

#### PrintMode（标准输出）

**`packages/coding-agent/src/modes/print-mode.ts`**

- line 104 订阅 session
- JSON 模式：每个事件序列化为 JSON 写入 stdout
- 文本模式：只输出最终 assistant 文本

#### RPCMode（RPC 客户端）

**`packages/coding-agent/src/modes/rpc/rpc-mode.ts`**

- line 354 订阅 session：`session.subscribe((event) => output(event))`
- 所有事件通过 `output()` 转发给 RPC 客户端
- line 357 额外订阅 raw agent 做背压控制

#### SDK（用户自定义）

**`packages/coding-agent/src/core/sdk.ts`**

- line 293 创建 Agent 实例
- 用户通过 `session.subscribe((event) => { ... })` 自行处理事件

---

## EventStream 机制

**`packages/ai/src/utils/event-stream.ts`**

`EventStream<T, R>` 实现了 `AsyncIterable<T>`：

- **`push(event)`** — 追加事件。如果满足完成条件（`agent_end`），解析最终结果并标记流完成
- **`end(result)`** — 标记流完成，唤醒所有等待中的消费者
- **`[Symbol.asyncIterator]()`** — 异步生成器，yield 队列中的事件或等待新事件
- **`result()`** — 返回最终结果的 Promise

`agentLoop()` 和 `agentLoopContinue()` 返回 `EventStream<AgentEvent, AgentMessage[]>`，在内部将 `emit` 回调包装为 `stream.push(event)`（见 agent-loop.ts:44）。

---

## 关键文件索引

| 角色 | 文件路径 | 关键行 |
|---|---|---|
| emit 源头 | `packages/agent/src/agent-loop.ts` | `runLoop()` :155, `streamAssistantResponse()` :275 |
| AgentEvent 类型 | `packages/agent/src/types.ts` | :403-418 |
| AgentEventSink 类型 | `packages/agent/src/agent-loop.ts` | :25 |
| Agent 类 | `packages/agent/src/agent.ts` | `processEvents()` :509, `subscribe()` :231 |
| AgentHarness 类 | `packages/agent/src/harness/agent-harness.ts` | `handleAgentEvent()` :510 |
| EventStream | `packages/ai/src/utils/event-stream.ts` | — |
| AgentSession | `packages/coding-agent/src/core/agent-session.ts` | `_handleAgentEvent()` :476 |
| InteractiveMode | `packages/coding-agent/src/modes/interactive/interactive-mode.ts` | `handleEvent()` :2679 |
| PrintMode | `packages/coding-agent/src/modes/print-mode.ts` | 订阅 :104 |
| RPCMode | `packages/coding-agent/src/modes/rpc/rpc-mode.ts` | 订阅 :354 |
| SDK | `packages/coding-agent/src/core/sdk.ts` | Agent 创建 :293 |
