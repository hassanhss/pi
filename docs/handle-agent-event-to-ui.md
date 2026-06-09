# _handleAgentEvent 到 UI 渲染的完整链路

深入分析 `AgentSession._handleAgentEvent` 的内部逻辑，以及它如何通过事件广播最终驱动终端 UI 渲染。

> 关联阅读：[agent-event-flow.md](./agent-event-flow.md)（事件整体流转路径）

---

## 定位

```
packages/coding-agent/src/core/agent-session.ts:476
```

```typescript
private _handleAgentEvent = async (event: AgentEvent): Promise<void> => { ... }
```

在构造函数 (L343) 和重连 (L710) 时注册：

```typescript
this._unsubscribeAgent = this.agent.subscribe(this._handleAgentEvent);
```

---

## 方法内部四个阶段

### 阶段一：队列管理（L477-497）

当用户消息开始发送时，**在 emit 之前**先从队列中移除，确保 UI 看到最新状态。

```
message_start (role=user)
    │
    ├── 检查 steeringMessages 队列 → 找到则移除 + _emitQueueUpdate()
    │
    └── 检查 followUpMessages 队列 → 找到则移除 + _emitQueueUpdate()
```

`_emitQueueUpdate()` 发出 `queue_update` 事件：

```typescript
private _emitQueueUpdate(): void {
    this._emit({
        type: "queue_update",
        steering: [...this._steeringMessages],
        followUp: [...this._followUpMessages],
    });
}
```

### 阶段二：扩展事件发射（L500）

```typescript
await this._emitExtensionEvent(event);
```

`_emitExtensionEvent`（L601-673）将 AgentEvent 逐类型转换为扩展系统事件，关键行为：

| Agent 事件 | 扩展事件 | 特殊行为 |
|---|---|---|
| `agent_start` | `agent_start` | 重置 `_turnIndex` |
| `message_end` | `message_end` | **可替换消息内容**（`emitMessageEnd` 返回 replacement 后调用 `_replaceMessageInPlace`） |
| `turn_end` | `turn_end` | 追加 `turnIndex`，然后递增 |
| 其他事件 | 直接映射 | 无额外处理 |

> **注意**：扩展系统在 `_emit()` 之前运行，因此扩展对消息的修改会反映到 UI。

### 阶段三：广播到监听器（L503）

```typescript
this._emit(event.type === "agent_end"
    ? { ...event, willRetry: this._willRetryAfterAgentEnd(event) }
    : event);
```

- 遍历 `_eventListeners[]`，逐一调用回调
- `agent_end` 附加 `willRetry` 标记（由重试设置 + 最后 assistant 消息状态决定）
- **这是连接 UI 层的桥梁**

### 阶段四：会话持久化（L506-546）

仅处理 `message_end` 事件：

```
message_end
    │
    ├── role=custom       → sessionManager.appendCustomMessageEntry(...)
    │
    ├── role=user|assistant|toolResult → sessionManager.appendMessage(...)
    │
    └── role=assistant
         ├── 记录 _lastAssistantMessage（用于自动压缩判断）
         ├── 重置 _overflowRecoveryAttempted
         └── 成功时发出 auto_retry_end 事件 + 重置重试计数器
```

---

## 监听器注册链

```
AgentSession.subscribe(listener)    ← agent-session.ts:680
    │
    │  将 listener 推入 _eventListeners[]
    │  返回 unsubscribe 函数
    │
    ▼
InteractiveMode.subscribeToAgent()  ← interactive-mode.ts:2681
```

```typescript
private subscribeToAgent(): void {
    this.unsubscribe = this.session.subscribe(async (event) => {
        await this.handleEvent(event);
    });
}
```

---

## handleEvent：事件到 UI 组件的映射

`InteractiveMode.handleEvent`（L2686-3022）通过 switch 将事件路由到具体 UI 组件：

### agent_start（L2694-2718）

- 清空 `pendingTools`
- 设置终端进度条
- 创建 `Loader` 动画加入 `statusContainer`
- 清理残留的重试 UI

### queue_update（L2721-2724）

- `updatePendingMessagesDisplay()` 更新待处理消息列表

### message_start（L2737-2757）

| 消息角色 | 渲染行为 |
|---|---|
| `custom` | `addMessageToChat()` |
| `user` | `addMessageToChat()` + 更新待处理消息 |
| `assistant` | 创建 `AssistantMessageComponent`，加入 `chatContainer`，开始流式渲染 |

```typescript
// assistant 消息的流式渲染起点
this.streamingComponent = new AssistantMessageComponent(
    undefined,
    this.hideThinkingBlock,
    this.getMarkdownThemeWithSettings(),
    this.hiddenThinkingLabel,
);
this.streamingMessage = event.message;
this.chatContainer.addChild(this.streamingComponent);
this.streamingComponent.updateContent(this.streamingMessage);
```

### message_update（L2759-2791）

- 更新 `streamingComponent` 的内容
- **检测 toolCall**：遍历 `streamingMessage.content`，每个 `toolCall` 创建一个 `ToolExecutionComponent`：

```typescript
for (const content of this.streamingMessage.content) {
    if (content.type === "toolCall") {
        if (!this.pendingTools.has(content.id)) {
            const component = new ToolExecutionComponent(
                content.name, content.id, content.arguments,
                { showImages: ..., imageWidthCells: ... },
                this.getRegisteredToolDefinition(content.name),
                this.ui,
                this.sessionManager.getCwd(),
            );
            component.setExpanded(this.toolOutputExpanded);
            this.chatContainer.addChild(component);
            this.pendingTools.set(content.id, component);
        } else {
            component.updateArgs(content.arguments); // 参数增量更新
        }
    }
}
```

### message_end（L2794-2831）

- 最终更新 assistant 消息内容
- 处理 `aborted` / `error` 状态：给所有 pending 工具设置错误结果
- 成功时：对所有 pending 工具调用 `setArgsComplete()`（触发 diff 计算）
- 清空 `streamingComponent` 和 `streamingMessage`

### tool_execution_start（L2833-2855）

- 查找或创建 `ToolExecutionComponent`
- 调用 `markExecutionStarted()` 显示执行状态

### tool_execution_update（L2857-2863）

```typescript
component.updateResult({ ...event.partialResult, isError: false }, true);
//                                                            ↑ isPartial
```

流式更新工具执行结果（部分结果）。

### tool_execution_end（L2866-2874）

```typescript
component.updateResult({ ...event.result, isError: event.isError });
this.pendingTools.delete(event.toolCallId);
```

写入最终结果，从 pendingTools 中移除。

### agent_end（L2876-2895）

- 关闭终端进度条
- 停止并清理 Loader
- 清理残留的 streamingComponent（如果有）
- 清空 pendingTools
- 检查 shutdown 请求

### compaction_start / compaction_end（L2897-2963）

- `compaction_start`：创建 Loader + cancel 提示，拦截 Escape 键
- `compaction_end`：
  - 成功时：`rebuildChatFromMessages()` 重建整个聊天 + 显示压缩摘要
  - 失败/取消：显示对应提示

### auto_retry_start / auto_retry_end（L2966-3020）

- `auto_retry_start`：创建 Loader + CountdownTimer，拦截 Escape 键用于取消重试
- `auto_retry_end`：清除倒计时和 Loader，失败时显示错误信息

---

## UI 组件职责

| 组件 | 文件 | 职责 |
|---|---|---|
| `AssistantMessageComponent` | `components/assistant-message.ts` | 渲染 assistant 回复（Markdown 渲染、thinking blocks 折叠、tool call 占位） |
| `ToolExecutionComponent` | `components/tool-execution.ts` | 渲染工具调用（参数展示、执行状态、结果渲染、自定义 renderer 支持） |
| `Loader` | — | 旋转动画 + 文字标签 |
| `CountdownTimer` | — | 定时器，每秒回调更新文字 |

所有组件实现 `Component` 接口：

```typescript
interface Component {
    render(width: number): string[];
}
```

---

## 渲染触发

每次事件处理后都调用：

```typescript
this.ui.requestRender();
```

TUI 系统执行差量渲染——只更新终端中发生变化的行，避免全屏重绘。

---

## 事件流总图

```
Agent 核心引擎
      │
      │ AgentEvent
      ▼
┌──────────────────────────────────────────┐
│ _handleAgentEvent (agent-session.ts:476) │
│                                          │
│  ① 队列管理 → _emitQueueUpdate()        │
│  ② _emitExtensionEvent() → 扩展系统     │  ← 可修改消息
│  ③ _emit() → 广播到 _eventListeners[]   │  ← 连接 UI
│  ④ sessionManager.appendMessage()        │  ← 持久化
└─────────────┬────────────────────────────┘
              │
              │ AgentSessionEvent
              ▼
┌──────────────────────────────────────────┐
│ InteractiveMode.handleEvent (L2686)      │
│                                          │
│  switch(event.type)                      │
│    ├── agent_start     → Loader          │
│    ├── message_start   → AsstMsgComp     │
│    ├── message_update  → 流式更新 + Tool │
│    ├── message_end     → 最终化          │
│    ├── tool_exec_*     → ToolExecComp    │
│    ├── compaction_*    → Loader/rebuild  │
│    ├── auto_retry_*    → Timer/Loader    │
│    └── agent_end       → 清理            │
│                                          │
│  → ui.requestRender()                    │
└─────────────┬────────────────────────────┘
              │
              ▼
         TUI 差量渲染到终端
```
