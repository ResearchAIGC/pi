# 03 — Agent 内核（pi-agent-core）

> 源码：`packages/agent/src/`（约 8k 行，不含 harness）
> 核心文件：`types.ts` / `agent.ts` / `agent-loop.ts` / `stream-fn.ts` / `harness/*`

---

## 1. 设计目标

`pi-agent-core` 的目标是提供一个 **与具体 LLM Provider、UI、存储无关** 的通用 Agent 运行时。它只做三件事：

1. **循环（Loop）**：把用户消息变成 Assistant 回复，必要时执行工具，再把工具结果喂回模型，直到模型不再调用工具。
2. **状态（State）**：持有 `systemPrompt / model / tools / messages / streamingMessage / pendingToolCalls / errorMessage`，并通过 `AgentEvent` 事件流暴露变更。
3. **可扩展性（Extensibility）**：在循环的关键节点插入 `transformContext` / `convertToLlm` / `beforeToolCall` / `afterToolCall` / `shouldStopAfterTurn` / `prepareNextTurn` / `getSteering/FollowUpMessages` 等钩子。

更重的“持久化、分支、压缩、重试 Lane”则由同包的 `AgentHarness` 子系统承担（见 §5）。

---

## 2. 类型系统（`types.ts`）

### 2.1 AgentMessage

```ts
// packages/agent/src/types.ts:326
export type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];
```

`Message` 来自 `pi-ai`（`UserMessage | AssistantMessage | ToolResultMessage`），`CustomAgentMessages` 是一个空接口，允许应用通过 declaration merging 注入自定义消息类型。`coding-agent` 即通过此机制注入 `CustomMessage`（`customType + content + display + details`）。

此设计使 **低层循环无需理解自定义消息**，而上层可自由扩展，`convertToLlm` 负责把 `AgentMessage[]` 过滤/映射为 `Message[]`。

### 2.2 ThinkingLevel

```ts
export type ThinkingLevel = "off" | "minimal" | "low" | "medium" | "high" | "xhigh" | "max";
```

`xhigh`/`max` 仅部分模型支持，需配合 `Model.thinkingLevelMap`（`pi-ai`）检测。

### 2.3 Tool 契约

```ts
export interface AgentTool<TParams extends TSchema, TDetails> extends Tool<TParams> {
  label: string;
  prepareArguments?: (args: unknown) => Static<TParams>;   // 兼容性 shim
  execute: (id, params, signal?, onUpdate?) => Promise<AgentToolResult<TDetails>>;
  replay?: "never" | "safe";                               // 恢复策略
  executionMode?: ToolExecutionMode;                       // sequential | parallel
}
export interface AgentToolResult<T> {
  content: (TextContent|ImageContent)[];
  details: T;
  usage?: Usage;
  addedToolNames?: string[];   // 动态引入的新工具
  terminate?: boolean;         // 提前终止当前批次的 hint
}
```

关键点：`execute` 抛异常即视为 `isError:true`；`addedToolNames` 用于“工具发现”场景（如 OpenAI 的 `deferredTools`）；`terminate` 仅当批次内所有工具都置 `true` 时才终止（`agent-loop.ts:589-591`）。

### 2.4 钩子上下文

- `BeforeToolCallContext` / `AfterToolCallContext`：携带 `assistantMessage + toolCall + args + context/isError/result`。
- `ShouldStopAfterTurnContext` / `PrepareNextTurnContext`：携带 `message + toolResults + context + newMessages`，供外部决定是否停、如何替换下一轮的 `context/model/thinkingLevel`。

---

## 3. Agent 类（`agent.ts` — 592 行）

### 3.1 状态管理

```ts
export class Agent {
  private _state: MutableAgentState;          // systemPrompt/model/thinkingLevel/tools/messages/isStreaming/...
  private listeners = Set<(event, signal) => void>();
  private steeringQueue: PendingMessageQueue; // mode: "all" | "one-at-a-time"
  private followUpQueue: PendingMessageQueue;
  private activeRun?: ActiveRun;              // { promise, resolve, abortController }
}
```

`tools`/`messages` 的 setter 会 **浅拷贝数组**（`slice()`），防止外部持有引用被意外修改（`agent.ts:69-88`）。

### 3.2 事件订阅

```ts
subscribe(listener: (event: AgentEvent, signal: AbortSignal) => void): () => void
```

`processEvents()`（`agent.ts:544-591`）先更新 `_state`（`streamingMessage`/`pendingToolCalls`/`errorMessage`/`messages`），再按订阅顺序 `await listener(event, signal)`。`agent_end` 是最后事件，但 `finishRun()` 需等所有 `agent_end` 监听器 settle 后才清 `isStreaming / pendingToolCalls`。

### 3.3 队列语义

`PendingMessageQueue`（`agent.ts:125-159`）：`mode:"all"` 时 `drain()` 一次性取空；`mode:"one-at-a-time"` 时每次只取队首，保证多条 steering 不会在同一 turn 内堆积。

`steer()` / `followUp()` / `clear*()` / `hasQueuedMessages()` 暴露给上层；`AgentSession` 在此之上维护 `string[]` 的 UI 镜像（`_steeringMessages` / `_followUpMessages`）并通过 `queue_update` 事件通知 TUI。

### 3.4 核心方法

| 方法 | 职责 |
|---|---|
| `prompt(input: string\|AgentMessage\|AgentMessage[])` | 归一化为 `AgentMessage[]` → `runPromptMessages()` → `runAgentLoop()` |
| `continue()` | 从当前 transcript 的最后一条 `user/toolResult` 继续；若最后是 `assistant` 且有队列消息，改为 `runPromptMessages(queued)` |
| `abort()` | `activeRun.abortController.abort()` |
| `waitForIdle()` | `activeRun.promise` |
| `reset()` | 清 `messages` + `streamingMessage` + `pendingToolCalls` + 队列（仅空闲时） |

`runWithLifecycle()`（`agent.ts:486-509`）负责创建 `AbortController`、置 `isStreaming:true`、捕获异常时合成 `stopReason:"error"|"aborted"` 的 `AssistantMessage` 并走正常事件路径。

---

## 4. Agent Loop（`agent-loop.ts` — 803 行）

这是 Pi 最精密的部分，分为三层函数：

```
agentLoop() / agentLoopContinue()  → 创建 EventStream，异步调用 run*()
runAgentLoop() / runAgentLoopContinue() → 准备 prompts/context，emit agent_start/turn_start，调用 runLoop()
runLoop()                          → 真正的双层循环
  streamAssistantResponse()         → 单次 LLM 调用与流式消费
  executeToolCalls()               → 单次 Tool 批次执行
  failToolCallsFromTruncatedMessage() → length 截断的容错
```

### 4.1 runLoop 双层循环（`agent-loop.ts:156-273`）

```ts
async function runLoop(initialContext, newMessages, config, signal, emit, streamFn) {
  let pendingMessages = await config.getSteeringMessages() || [];
  while (true) {                          // 外层：followUp 驱动
    let hasMoreToolCalls = true;
    while (hasMoreToolCalls || pendingMessages.length > 0) { // 内层：tool + steering 驱动
      if (lastCompletedTurn) {            // 1. prepareNextTurn（可改 context/model/thinking）
        pendingMessages ||= await config.getSteeringMessages();
        emit("turn_start");
      }
      if (pendingMessages.length) {       // 2. 注入 steering 消息
        for (m of pendingMessages) { emit(message_start/end); context.messages.push(m); }
        pendingMessages = [];
      }
      const message = await streamAssistantResponse(...); // 3. 流式 LLM
      newMessages.push(message);
      if (message.stopReason === "error"|"aborted") { emit(turn_end); emit(agent_end); return; }
      // 4. 工具执行
      if (toolCalls.length) {
        const batch = (message.stopReason==="length")
          ? await failToolCallsFromTruncatedMessage(...)
          : await executeToolCalls(...);
        toolResults.push(...batch.messages);
        hasMoreToolCalls = !batch.terminate;
        for (r of toolResults) { context.messages.push(r); newMessages.push(r); }
      }
      emit("turn_end", message, toolResults);
      if (await shouldStopAfterTurn(...)) { emit(agent_end); return; }
      pendingMessages = await getSteeringMessages();
    }
    const followUps = await getFollowUpMessages();
    if (followUps.length) { pendingMessages = followUps; continue; }
    break;
  }
  emit(agent_end);
}
```

注意 `prepareNextTurn` 之前会 **条件性地再 poll 一次 steering**（`agent-loop.ts:190-201`），避免 `one-at-a-time` 模式在同一 turn 内吞两条消息。

### 4.2 streamAssistantResponse（`agent-loop.ts:279-370`）

```ts
async function streamAssistantResponse(context, config, signal, emit, streamFn) {
  let messages = context.messages;
  if (config.transformContext) messages = await config.transformContext(messages, signal);
  const llmMessages = await config.convertToLlm(messages);
  const llmContext = { systemPrompt: context.systemPrompt, messages: llmMessages, tools: context.tools };
  const apiKey = (await config.getApiKey?.(model.provider)) || config.apiKey;
  const response = await streamFn(model, llmContext, { ...config, apiKey, signal });
  for await (event of response) {
    switch(event.type) {
      case "start":          emit(message_start); break;
      case "text_*"/"thinking_*"/"toolcall_*": emit(message_update); break;
      case "done"/"error":   return response.result();
    }
  }
}
```

### 4.3 工具执行（`agent-loop.ts:406-803`）

- **truncation 保护**：`stopReason==="length"` 时所有 tool_calls 因“可能截断”被直接失败（`failToolCallsFromTruncatedMessage`），提示模型重发。
- **分流**：若任一 `tool.executionMode==="sequential"` 或全局 `toolExecution==="sequential"` → `executeToolCallsSequential`，否则 `executeToolCallsParallel`。
- **prepare 阶段**：`prepareToolCall()` 查找 `Tool` → `prepareArguments` shim → `validateToolArguments`（TypeBox） → `beforeToolCall`（可 `block`）。任意失败即 `ImmediateToolCallOutcome`。
- **execute 阶段**：`executePreparedToolCall()` 调用 `tool.execute(id, args, signal, onUpdate)`，`onUpdate` 触发 `tool_execution_update`。
- **finalize 阶段**：`finalizeExecutedToolCall()` → `afterToolCall` 可覆盖 `content/details/isError/usage/terminate`。
- **parallel 的特殊处理**：`prepare` 串行，`execute` 并发，但 `tool_execution_end` 按完成顺序发射，`ToolResultMessage` 按源码顺序追加（`agent-loop.ts:547-555`）。

### 4.4 事件契约

`EventStream<AgentEvent, AgentMessage[]>`（`utils/event-stream.ts`）以 `agent_end` 为终止信号，`response.result()` 解析最终 `AssistantMessage`。

`StreamFn` 必须满足（`types.ts:21-32` + `agent-loop.ts:306-311`）：

- 永不 throw / reject（请求/模型/运行时失败必须编码进返回的流）。
- 失败流以 `error` 事件 + `stopReason:"error"|"aborted"` 的 `AssistantMessage` 结束。

---

## 5. AgentHarness 持久化子系统（`harness/` — 约 15k 行）

### 5.1 定位

`AgentHarness` 是对 `Agent` 的**持久化 wrapper**：`Agent` 只在内存里跑一个 Run，`AgentHarness` 则把每个 Run 的输入/输出/工具结果以 `Entry` 形式追加到 `Session`（`Storage` 后端），并提供 `Lane` 抽象实现并发域隔离、重试、压缩、分支。

### 5.2 Lane（`harness/runtime/lane.ts`）

一个 `Lane` 对应一个“对话分支”的线性历史（类似 git branch）。对外暴露 20+ 方法（`prompt/skill/compact/navigateTree/resume/abort/steer/followUp/recordUsage/watch` 等，`agent-harness.ts:538-580`）。内部通过 `Drive` 状态机驱动 Operation。

### 5.3 Drive 状态机（`harness/runtime/drive/`）

```
Boundary ─→ Checkpoint ─→ Generation ─→ ToolPlacement ─→ Tools ─→ Response
    ↑            │            │               │              │         │
    └────────────┴─── Retry / Deferred / Terminal / Structural ───────┘
              Reconcile ←───┴──── Recovery ←─┴─── (失败分支)
```

- `generation.ts`：构造 LLM 请求，处理 `deferred` 句柄。
- `tools.ts` / `tool-placement.ts`：工具执行与结果放置（含 `deferred` 工具的延迟加载）。
- `response.ts` / `reconcile.ts`：响应校验与状态归并。
- `retry.ts` / `recovery.ts`：重试与恢复。
- `boundary.ts` / `checkpoint.ts` / `structural.ts`：边界校验与检查点。

### 5.4 Session 与 Storage（`harness/session/`）

- `Session`（`session/session.ts`）：面向业务的接口（`branch/createBranch/append/*Value/*List/scanBranch`），通过 `MutationLine`（串行化 `mutate` 回调）保证并发安全，`StorageBackedSession` 为其默认实现。
- `Storage`（`session/types.ts`）：底层存储抽象（`commit/getEntries/getValue/scanBranch/close` 等），由 `session-backends/sqlite-node` 或 `jsonl`（`session/jsonl/`）实现。
- `Values`（`session/values.ts`）：类型化的 `Value<T>` / `ValueList<T>` 地址系统（`branchTip(name)` / `laneConfig(name)` / `laneState(name)` / `sessionName` / `entryLabel(id)`），通过 `setValue/deleteValue/appendList` 写入。
- `Commit`（`session/commit.ts`）：`insertEntry` 构造 `Entry` 写入。

### 5.5 Hooks 与 Events（`harness/hooks.ts` / `harness/events.ts`）

- `HookRegistry`：注册 `before_run / before_drive / before_run_end / transform_context / before_request / before_payload / after_response / before_tool / after_tool / before_compaction / before_navigation` 等 12 类钩子（`agent-harness.ts:430-509`）。
- `HarnessEventBus`：发布 `run_start/end`, `turn_start/end`, `message_start/update/end`, `tool_start/update/end`, `retry_*`, `compaction_*`, `navigation_*`, `usage`, `config_update` 等事件，并支持 `watch()` 的 reducer 订阅（`runtime/reducer.ts`）。

### 5.6 Compaction 与 Branch Summarization（`harness/compaction/`）

- `compaction.ts`：`shouldCompact(tokens, window, settings)` 阈值判定 → `prepareCompaction()` 找 `cutPoint` → `generateSummary()` 调用 LLM 生成摘要 → `compact()` 写入 `CompactionEntry`。
- `branch-summarization.ts`：分支切换时对 `entriesToSummarize` 生成 `BranchSummaryEntry`。

---

## 6. Proxy 与 Search

- `proxy.ts`：为 `Agent` 提供透明代理（用于测试与工具拦截）。
- `search/`：Harness 的会话搜索索引（支持按 `ToolResultMessage` 等过滤）。

---

## 7. 关键设计取舍

| 取舍 | 做法 | 代价 |
|---|---|---|
| `convertToLlm` 在每次 LLM 调用前执行 | 保证 `CustomMessage` 等自定义类型永不泄露到 Provider | 每次 turn 额外一次映射 |
| `prepare` 串行 + `execute` 并行 | 兼顾 determinism 与吞吐 | parallel 模式下仍需一次串行准备 |
| `terminate` 需批次全体一致 | 防止单个工具误终止整个批次 | 需要协调多个工具的 terminate 意图 |
| `length` 截断时全部失败 | 避免执行参数被截断的危险工具调用 | 可能造成重试开销 |

