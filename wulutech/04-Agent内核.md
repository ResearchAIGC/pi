# 04 — Agent 内核：循环、并发与 Harness

> 源码锚点：`packages/agent/src/types.ts:463` / `packages/agent/src/agent.ts:592` / `packages/agent/src/agent-loop.ts:803` / `packages/agent/src/harness/runtime/drive/` / `packages/agent/src/harness/pico3/`

---

## 1. 一个循环要同时处理什么

`Agent Loop` 表面上是 `while (有工具调用) { 调模型; 执行工具 }`，但实际要同时处理：

1. **流式增量**：模型边生成边吐 `text_delta`、`thinking_delta`，界面需要逐块渲染。
2. **截断容错**：模型因长度截断（`stopReason:"length"`）时，工具参数可能不完整，执行它们是危险的。
3. **并发与确定性**：`read` 三次可以并发跑以提速，但落盘的 `ToolResultMessage` 必须与模型输出的源码顺序一致，否则重放会不一致。
4. **可插拔**：在不改循环的前提下，允许外部注入剪裁、转换、拦截等逻辑。
5. **可持久化**：循环本身只在内存中跑一次，崩溃、分支、压缩、重试由上层的 `Harness` 承担。

为此，Pi 用三级分层，每层职责单一：

```
最内层：agent-loop.ts 的纯函数循环
         职责：发请求、收流、跑工具。无状态，可单独测试。
    ↑ 被调用
中间层：Agent 类
         职责：持有状态、队列、事件订阅。有状态，但不持久化。
    ↑ 被包装
最外层：AgentHarness / pico3
         职责：持久化 Lane（分支）与 Drive（状态机），可恢复。
```

---

## 2. 类型契约（`packages/agent/src/types.ts:463`）

### 2.1 消息类型与 Transcript

```ts
// types.ts:333
type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];
// CustomAgentMessages 是一个空接口，上层可通过 TypeScript 的声明合并注入自定义类型
// 例如 coding-agent 注入了 CustomMessage { customType, content, display, details }
```

也就是说，底层循环不理解自定义消息，上层可以自由扩展。`convertToLlm` 在每次调模型前负责把 `AgentMessage[]` 过滤为模型能理解的 `Message[]`。

`TranscriptContext`（`0.86` 的变化）把提示词与工具声明收敛到首条 `system` 消息中：

- **读**：`agent.state.systemPrompt` 是只读的，通过重放首条 `system` 消息得到；`agent.state.tools` 是当前可执行工具集，若与 `system` 消息声明的不一致，会在下次请求前以 `system` 消息向模型宣告。
- **写**：改提示词只需发送一条 `system` 消息，例如 `prompt([{role:"system", content:"", sections:{...}}, {role:"user"...}])`。

`convertToLlm` 在每次调模型前执行，保证自定义消息永不泄露；`transformContext` 可选地在它之前对消息做剪裁。

### 2.2 钩子

以下四类钩子分别在循环的关键节点被调用，外部可据此决定是否拦截、覆盖或停止：

- `beforeToolCall` / `afterToolCall`：工具执行前后，携带 `assistantMessage`、`toolCall`、`args` 等。
- `shouldStopAfterTurn` / `prepareNextTurn`：每轮结束后，决定是否停止或替换下一轮的上下文与模型。

### 2.3 工具

```ts
// types.ts:406-431
interface AgentTool<TParams extends TSchema, TDetails> extends Tool<TParams> {
  label: string; // 界面展示用
  prepareArguments?: (args: unknown) => Static<TParams>; // 兼容性垫片
  execute: (id, params, signal?, onUpdate?) => Promise<AgentToolResult<TDetails>>;
  replay?: "never" | "safe"; // 恢复策略
  executionMode?: ToolExecutionMode; // 该工具是否必须串行
}
interface AgentToolResult<T> {
  content: (TextContent|ImageContent)[]; // 返回给模型的内容
  details: T; // 结构化细节，供界面渲染
  terminate?: boolean; // 需整批工具都为 true 才会提前结束
}
```

约定：`execute` 抛异常即视为失败（`isError:true`）；`onUpdate` 可推送中间进度，触发 `tool_execution_update` 事件。

---

## 3. Agent 类（`packages/agent/src/agent.ts:592`）

### 3.1 状态与队列

```ts
class Agent {
  private _state: MutableAgentState; // 模型、思考等级、工具、历史、流式状态等
  private listeners = Set<(event, signal) => void>(); // 事件订阅者
  private steeringQueue: PendingMessageQueue; // 运行中途纠偏的队列
  private followUpQueue: PendingMessageQueue; // 结束后追问的队列
  private activeRun?: { promise, resolve, abortController }; // 当前运行
}
```

对 `tools` 与 `messages` 的赋值会浅拷贝数组，避免外部持有引用被意外修改（`agent.ts:69-88`）。

### 3.2 事件订阅

```ts
subscribe(listener: (event, signal) => void): () => void
```

`processEvents()`（`agent.ts:544-591`）会先更新内部状态（`streamingMessage`、`pendingToolCalls` 等），再按订阅顺序 `await` 每个监听器。`agent_end` 是最后一个事件，但 `finishRun()` 会等所有 `agent_end` 监听器都完成后才清理流式状态。`waitForIdle()` 本质上是等待 `activeRun.promise`。

### 3.3 队列语义

`PendingMessageQueue`（`agent.ts:125-159`）有两种模式：`all`（一次取空）与 `one-at-a-time`（一次取队首）。前者适合“把积压全部注入”，后者保证多条中途消息不会在同一轮堆积。`AgentSession` 在此之上维护字符串的界面镜像，并通过 `queue_update` 事件通知界面。

### 3.4 核心方法

| 方法 | 作用 |
|---|---|
| `prompt(input)` | 把输入归一化为 `AgentMessage[]`，启动一次运行 |
| `continue()` | 从当前历史的最后一条 `user` 或 `toolResult` 继续 |
| `abort()` | 中止当前运行 |
| `waitForIdle()` | 等待当前运行结束 |
| `reset()` | 清空历史与队列（仅空闲时可用） |

`runWithLifecycle()`（`agent.ts:486-509`）负责创建 `AbortController`、标记流式状态，并在异常时合成一条 `stopReason:"error"` 的消息走正常事件路径，避免抛异常中断。

---

## 4. 双层循环（`packages/agent/src/agent-loop.ts:803`）

整体分三层函数：`agentLoop()` 负责创建事件流 → `runAgentLoop()` 负责准备上下文 → `runLoop()` 负责真正的循环。

### 4.1 循环骨架

为便于理解，此处用伪代码展示 `runLoop`（`agent-loop.ts:156-273`）的双层结构，省略了错误与重试细节：

```ts
// 外层由“结束后追问”驱动，内层由“工具调用与中途纠偏”驱动
let pendingMessages = await getSteeringMessages() || [];
while (true) {                          // 外层：检查是否有 followUp
  let hasMoreToolCalls = true;
  while (hasMoreToolCalls || pendingMessages.length > 0) { // 内层：有工具或有纠偏就继续
    if (pendingMessages.length > 0) {
      注入 steering 消息到上下文;
    }
    const message = await 调模型并流式接收(); // 见 4.2
    if (message.stopReason === "error") 结束整个运行;

    if (有工具调用) {
      if (message.stopReason === "length") 全部标记失败; // 截断保护
      else 执行工具批次;
      hasMoreToolCalls = !整批都要求提前结束;
    }
    检查 shouldStopAfterTurn，决定是否退出;
    pendingMessages = await getSteeringMessages();
  }
  const followUps = await getFollowUpMessages();
  if (followUps.length > 0) { pendingMessages = followUps; continue; }
  break;
}
```

`prepareNextTurn` 在每轮结束后、检查 `steering` 之前会额外轮询一次，避免 `one-at-a-time` 模式在同一轮吞掉两条消息（`agent-loop.ts:190-201`）。

### 4.2 流式接收

```ts
async function streamAssistantResponse(context, config, signal, emit, streamFn) {
  let messages = context.messages;
  if (config.transformContext) messages = await config.transformContext(messages, signal);
  const llmMessages = await config.convertToLlm(messages);
  const response = await streamFn(model, { messages: llmMessages }, { ...config, signal });
  for await (event of response) {
    // start → 多次 text/thinking/toolcall 的 start/delta/end → done/error
  }
  return response.result(); // 最终的 AssistantMessage
}
```

### 4.3 工具执行

分三个阶段，每个阶段都有对应的钩子：

- **准备**：查找工具定义 → 兼容性垫片 → 校验参数（TypeBox）→ `beforeToolCall`（可拦截）。任一步失败则直接产出失败结果。
- **执行**：调 `tool.execute(id, args, signal, onUpdate)`，`onUpdate` 触发 `tool_execution_update`。
- **收尾**：`afterToolCall` 可覆盖 `content`、`details`、`isError` 等。

并发策略：若任一工具要求串行或全局为串行，则整批串行；否则准备阶段串行、执行阶段并发，但结束事件按完成顺序发射、落盘顺序按源码顺序，保证重放一致性（`agent-loop.ts:487-561`）。

### 4.4 StreamFn 约束

`StreamFn`（`types.ts:28-37`）永不抛异常。所有失败必须编码为事件流中的 `error` 事件与 `stopReason:"error"` 的消息。

---

## 5. Harness 持久化子系统（`packages/agent/src/harness/`）

### 5.1 定位

`Agent` 只在内存中跑一次运行，`AgentHarness` 则把每次运行的输入输出以 `Entry` 追加到 `Session`，并提供 `Lane`（分支隔离域）与重试、压缩、分支能力。现役 Harness 基于 `Session/Storage`，下一代 `pico3` 基于 `pi-durable`。

### 5.2 Lane（分支）

一个 `Lane` 对应一个对话分支的线性历史，类似 Git 分支。对外提供 20 多个方法（`prompt`、`compact`、`navigateTree`、`steer` 等，`harness/agent-harness.ts:538-580`），内部通过 `Drive` 状态机驱动操作（`run` / `compaction` / `navigation` 三种）。

### 5.3 Drive 状态机

`Drive` 是驱动单个操作前进的状态机，位于 `harness/runtime/drive/`：

```
Boundary（边界校验）
  → Checkpoint（检查点）
    → Generation（构造请求，处理延迟句柄）
      → ToolPlacement（放置工具结果）
        → Tools（执行工具）
          → Response（校验响应）
            → Reconcile（归并状态）
              → 失败分支：Retry / Recovery / Deferred
```

每个状态对应一个文件（`generation.ts`、`tools.ts` 等），职责单一。

### 5.4 Session 与 Storage

- `Session`（`session/session.ts`）：面向业务的接口，通过 `MutationLine` 串行化并发写入，保证安全。
- `Storage`（`session/types.ts`）：底层存储抽象（`commit` / `getEntries` / `scanBranch` 等），由 `session-backends/sqlite-node` 或 `session/jsonl/` 实现。
- `Values`（`session/values.ts`）：类型化的地址系统，如 `branchTip(name)` 指向分支末端。
- `Hooks` / `Events`：钩子注册与事件总线，支持 `watch()` 的订阅。

### 5.5 压缩与分支摘要

`harness/compaction/compaction.ts` 的 `shouldCompact` 判定是否需要压缩，找到切割点后调模型生成摘要，再写入 `CompactionEntry`。

### 5.6 Pico3 参考内核（实验性）

`harness/pico3/`（24 个文件）是下一代 Harness 在 `agent` 内的参考实现，不参与生产：

```
pico3/
├── harness.ts / session.ts / types.ts  — 核心
├── memory.ts / scheduler.ts / chord.ts  — 调度与状态
└── kinds/{ generation, tool, job, ... } — 任务种类，每种对应一个 durable Task
```

它展示了如何把 `Generation → Tools → Response` 重写为 `durable` 上的任务调度。

---

## 6. 关键取舍

| 取舍 | 做法 | 代价 |
|---|---|---|
| 每次调模型前都过滤自定义消息 | 保证自定义消息永不泄露 | 每轮一次映射 |
| 准备串行、执行并发 | 兼顾确定性与吞吐 | 并发时仍需一次串行准备 |
| 提前结束需整批工具一致同意 | 防止单个工具误终止 | 需协调多工具意图 |
| 截断时整批工具失败 | 避免执行参数不完整的工具 | 可能增加重试开销 |
| 现役与下一代契约并存 | 保证可用性的同时可试验新设计 | 需维护两套对照 |
