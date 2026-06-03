# 用户输入到磁盘落盘：源码阅读路径

梳理「用户在终端打出一句话 → 磁盘上出现一条 session message」的完整代码路径，给出按序阅读的文件和关键函数。

---

## 阅读顺序

### 1. CLI 入口

**`packages/coding-agent/src/cli.ts`**

- 设置进程标题、环境变量、HTTP dispatcher
- 调用 `main(process.argv.slice(2))`

### 2. 主函数：创建运行时

**`packages/coding-agent/src/main.ts`**

- `main()` 解析 CLI 参数，创建核心服务（SettingsManager、ModelRegistry 等）
- 创建 `InteractiveMode` 实例，调用 `interactiveMode.run()` 进入交互循环

### 3. 交互循环：捕获用户输入

**`packages/coding-agent/src/modes/interactive/interactive-mode.ts`**

- `run()` 方法的核心循环：

  ```typescript
  while (true) {
    const userInput = await this.getUserInput()  // 阻塞等待用户输入
    await this.session.prompt(userInput)          // 把输入交给 agent session
  }
  ```

- `getUserInput()` 底层通过 TUI Editor 组件接收键盘输入

### 4. TUI 编辑器组件（可选深入了解）

**`packages/tui/src/components/editor.ts`**

- `Editor` 类负责渲染输入框、捕获键盘事件
- 用户按回车时触发 `onSubmit(text)` 回调，文本回到上层

### 5. Agent Session：把用户文本变成 message 对象

**`packages/coding-agent/src/core/agent-session.ts`**

- `prompt(text)` 方法：
  - 把用户文本构造成 `{ role: "user", content: [...] }` 消息对象
  - 调用 `this._runAgentPrompt(messages)` 发给 agent loop

### 6. Agent Loop：驱动 LLM 调用

**`packages/agent/src/agent-loop.ts`**

- `runAgentLoop()` 启动循环
- `streamAssistantResponse()` 调用 LLM API 流式获取回复
- 过程中通过事件（event）向上层汇报每条消息的状态

### 7. 事件处理：触发持久化

**`packages/coding-agent/src/core/agent-session.ts`**（回到这个文件）

- `_handleAgentEvent()` 监听 agent 事件
- 收到 `message_end` 事件时：
  - 判断 `role` 是 `user` / `assistant` / `toolResult`
  - 调用 `this.sessionManager.appendMessage(message)`

### 8. Session Manager：写入磁盘

**`packages/coding-agent/src/core/session-manager.ts`**

- `appendMessage()`：构造一条 `SessionMessageEntry`：

  ```json
  {
    "type": "message",
    "id": "abc123",
    "parentId": "上一个消息的id",
    "timestamp": "...",
    "message": { "role": "user", "content": [...] }
  }
  ```

- `_appendEntry()` → `_persist()`：用 `appendFileSync()` 把 JSON 字符串追加写入 `.jsonl` 文件

---

## 流程图

```
用户在终端打字
    ↓
cli.ts → main()
    ↓
main.ts → InteractiveMode.run()
    ↓
interactive-mode.ts → getUserInput()            [等待用户输入]
    ↓                       ↑
    │             editor.ts [TUI 编辑器捕获键盘]
    ↓
agent-session.ts → prompt(text)                 [构造 user message 对象]
    ↓
agent-session.ts → _runAgentPrompt()            [发给 agent loop]
    ↓
agent-loop.ts → runAgentLoop()                  [调用 LLM，流式返回]
    ↓                 [event: message_end]
    ↓
agent-session.ts → _handleAgentEvent()          [监听事件]
    ↓
session-manager.ts → appendMessage()            [构造 entry]
    ↓
session-manager.ts → _persist()                 [appendFileSync]
    ↓
磁盘上的 .jsonl 文件  ✅
```

---

## 阅读优先级

| 优先级 | 文件 | 关注点 |
|--------|------|--------|
| **必读** | `agent-session.ts` | `prompt()` 和 `_handleAgentEvent()` — 消息从文本变成对象、事件触发写入 |
| **必读** | `session-manager.ts` | `appendMessage()` → `_appendEntry()` → `_persist()` — JSONL 格式和文件写入 |
| **必读** | `interactive-mode.ts` | `run()` 循环 — 用户输入如何进入系统 |
| 选读 | `agent-loop.ts` | LLM 调用和事件流的细节 |
| 选读 | `editor.ts` | TUI 渲染和键盘捕获 |

核心链路是 **interactive-mode → agent-session → session-manager**，这三层各司其职：输入捕获、消息处理与事件驱动、磁盘持久化。
