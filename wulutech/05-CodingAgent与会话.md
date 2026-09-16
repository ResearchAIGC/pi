# 05 — Coding Agent 与会话（pi-coding-agent）

> 源码：`packages/coding-agent/src/`（约 60k 行）
> 核心：`core/agent-session.ts` / `core/session-manager.ts` / `core/model-runtime.ts` / `core/tools/*` / `modes/*`

---

## 1. 模块地图

```
packages/coding-agent/src/
├── main.ts / cli.ts / cli/args.ts        # 进程入口与参数解析
├── config.ts                             # 路径与版本常量
├── core/
│   ├── agent-session.ts  (2.5k 行)       # 会话编排核心（所有模式共享）
│   ├── session-manager.ts (1.7k 行)      # 会话持久化（JSONL DAG）
│   ├── model-runtime.ts  (795 行)        # 模型/AUTH 运行时
│   ├── model-registry.ts / model-resolver.ts  # 模型发现与作用域解析
│   ├── extensions/       (loader/runner/wrapper/types)
│   ├── tools/            (read/bash/edit/write/grep/find/ls/powershell)
│   ├── compaction/       (compact/branch-summarization/utils)
│   ├── resource-loader.ts / skills.ts / system-prompt.ts / prompt-templates.ts
│   ├── session-export.ts / export-html/  # JSONL/HTML 导出
│   ├── settings-manager.ts / trust-manager.ts / source-info.ts
│   ├── slash-commands.ts / bash-executor.ts / exec.ts
│   └── sdk.ts            # 程序化 SDK（createAgentSession 等）
├── modes/
│   ├── interactive/      (TUI 模式，含 35+ 组件)
│   ├── rpc/              (JSONL RPC 模式)
│   └── print-mode.ts     (一次性输出模式)
├── extensions/           (内置扩展，如 llama)
├── client/ / utils/ / experimental/ / bun/
└── index.ts              (公共导出)
```

---

## 2. AgentSession — 会话编排核心（`core/agent-session.ts`）

### 2.1 职责

`AgentSession` 是 **所有运行模式共享** 的编排层，封装：

- 对 `Agent` 的事件订阅与会话持久化（`_handleAgentEvent`）
- 模型/思考等级管理（`model` / `thinkingLevel` + `scopedModels` 轮转）
- 压缩（`compact` / `_runAutoCompaction` / `_compactBeforeNextAssistantResponse`）
- Bash 执行（`executeBashWithOperations`）
- 分支/导航/队列（`steer` / `followUp` / `navigateTree`）
- 扩展系统绑定（`ExtensionRunner` + 工具钩子）

### 2.2 关键内部状态

```ts
class AgentSession {
  agent: Agent;
  sessionManager: SessionManager;
  settingsManager: SettingsManager;
  private _steeringMessages/_followUpMessages: string[]; // UI 镜像
  private _pendingNextTurnMessages: CustomMessage[];     // 伴随下一次 prompt 的上下文
  private _pendingCustomMessages: CustomMessage[];       // turn 内排队的 custom 消息
  private _compactionAbortController / _autoCompactionAbortController / _branchSummaryAbortController;
  private _retryAttempt: number; _retryAbortController?: AbortController;
  private _toolRegistry: Map<string, AgentTool>; _toolDefinitions: Map<string, ToolDefinitionEntry>;
  private _baseSystemPrompt: string; _systemPromptOverride?: string;
}
```

### 2.3 事件持久化（`_handleAgentEvent` — `agent-session.ts:639-719`）

```ts
_handleAgentEvent = async (event: AgentEvent) => {
  if (event.message_start && role==="user") { /* 从队列移除已投递消息，emit queue_update */ }
  await _emitExtensionEvent(event);          // 先发扩展
  _emit(sessionEvent);                        // 再发外部监听器
  if (event.message_end) {
    if (role==="custom") sessionManager.appendCustomMessageEntry(...);
    else if (user|assistant|toolResult) sessionManager.appendMessage(message);
    if (role==="assistant") { _lastAssistantMessage = msg; /* 重置 overflow/重试计数 */ }
  }
  if (event.turn_end) _flushPendingCustomMessages(); // turn 末尾才可插入 custom，避免夹在 tool_call/result 间
}
```

### 2.4 工具钩子（`_installAgentToolHooks` — `agent-session.ts:482-536`）

- `beforeToolCall`：委派给 `ExtensionRunner.emitToolCall()`，支持 `block` + `reason` + `terminate`。
- `afterToolCall`：委派给 `emitToolResult()`，并对 `content` 做图片归一化（`normalizeToolResultImages`，受 `settings.imageAutoResize` 控制）。

### 2.5 下一轮刷新（`_installAgentNextTurnRefresh` — `agent-session.ts:557-579`）

在 `prepareNextTurnWithContext` 中：

1. 若 `shouldCompact()` 为 true → `_runAutoCompaction("threshold", false)`。
2. 用最新 `agent.state.messages` 重建 `context`，以 `systemPrompt` / `tools` / `model` / `thinkingLevel` 的最新值为准。

保证每 turn 开始前上下文是“压缩后 + 工具集最新”。

### 2.6 Prompt 路径（`prompt()` — `agent-session.ts:1175-1326`）

```
prompt(text, {expandPromptTemplates=true, images, streamingBehavior, source})
  1. 若 text 以 "/" 起：_tryExecuteExtensionCommand() → 直接执行并 return（不进入 LLM）
  2. 校验：compaction 是否进行中 → 抛错
  3. _runInputHandlers(text, images, source) → 扩展可 transform/handled 输入
  4. _expandSkillCommand(text)       → /skill:name → <skill> 块
  5. expandPromptTemplate(text)       → /template → 文件内容
  6. 若 isStreaming：按 streamingBehavior → _queueSteer() / _queueFollowUp()
  7. 校验：model 是否选定、auth 是否就绪
  8. 处理 aborted 后的溢出压缩：_checkCompaction(lastAssistant, false)
  9. 构造 messages[]：pendingNextTurnMessages + UserMessage(+images)
 10. runner.emitBeforeAgentStart(prompt, images, baseSystemPrompt, options) → 可注入 CustomMessage / 覆盖 systemPrompt
 11. _runAgentPrompt(messages) → agent.prompt() → loop → _handlePostAgentRun()
     while (handlePostAgentRun()) await agent.continue();
```

`_handlePostAgentRun()`（`agent-session.ts:1116-1144`）在每轮结束后检查：可重试错误→ `_prepareRetry()`、溢出/阈值压缩→ `_checkCompaction()`、扩展在 `agent_end` 后排入的队列→ `hasQueuedMessages()`，任一成立则 `continue()`。

---

## 3. SessionManager — 会话持久化（`core/session-manager.ts` — 1746 行）

### 3.1 数据模型

```ts
SessionHeader      { version: CURRENT_SESSION_VERSION, id: uuidv7, model?, thinkingLevel?, createdAt }
SessionEntry =     SessionMessageEntry | CustomMessageEntry | CustomEntry 
                 | CompactionEntry | BranchSummaryEntry | FileEntry 
                 | ModelChangeEntry | ThinkingLevelChangeEntry | SessionInfoEntry
SessionContext     { header, entries: SessionEntry[] }  // 内存中的完整会话
SessionTreeNode    { id, parentId, type, label?, children: SessionTreeNode[] } // 树视图
```

每条 `SessionEntry` 含 `id` (= `uuidv7`)、`parentId`（指向父 Entry，形成 DAG）、`timestamp`。顶层 `SessionHeader` 独占文件首行，其余 `SessionEntry` 每行一 JSON（JSONL）。

### 3.2 文件格式（`docs/session-format.md`）

```
~/.pi/agent/sessions/<uuid>.jsonl   (或项目 .pi/sessions/)
line1: SessionHeader (JSON)
line2: SessionEntry  (JSON) — UserMessage
line3: SessionEntry  (JSON) — AssistantMessage
...
```

`migrateSessionEntries()` 负责跨版本迁移；`getLatestCompactionEntry()` 定位最近压缩点；`buildSessionContext()` / `buildContextEntries()` / `sessionEntryToContextMessages()` 将 Entry 列表还原为 `AgentContext`。

### 3.3 关键方法

| 方法 | 作用 |
|---|---|
| `appendMessage(message)` | 追加 `SessionMessageEntry`，更新 `leafId` |
| `appendCustomMessageEntry(type, content, display, details)` | 追加 `CustomMessageEntry` |
| `appendCustomEntry(type, data)` | 追加 `CustomEntry`（不入 LLM 上下文） |
| `appendCompactionEntry(summary, oldLeafId, newLeafId)` | 追加 `CompactionEntry` |
| `fork(entryId, position)` | 从指定 Entry fork 新会话文件 |
| `navigateTree(targetId, summarize?, label?)` | 切换 `leafId`，可选生成 `BranchSummaryEntry` |
| `getEntries() / findEntries(query)` | 扫描 Entry（支持 `type`/`customType`/`limit`/`order`/`cursor`） |
| `exportSessionToJsonl / exportSessionToHtml` | 导出 |

会话树导航通过 `BranchPreparation`（`compaction/branch-summarization.ts`）收集 `entriesToSummarize`，必要时调用 LLM 生成摘要。

### 3.4 与 AgentState 的同步

`Agent.state.messages` 与 `SessionManager.entries` 是 **双写**：`_handleAgentEvent` 在 `message_end` 时同时更新二者，且 `AgentSession` 通过 `_replaceMessageInPlace()`（`agent-session.ts:748-762`）保证 `message_end` 扩展替换后的消息在两处一致。

---

## 4. ModelRuntime（`core/model-runtime.ts` — 795 行）

`ModelRuntime` 是 `AgentSession` 与 `pi-ai` 模型目录的桥梁：

- 聚合 `ModelRegistry`（模型发现） + `AuthStorage`（credential 存取） + `ModelsStore`（持久化目录）。
- `getAuth(model)`：按 `provider` 解析 `apiKey/headers/baseUrl/env`（支持 `CredentialSynchronizationError` 表示需同步凭证）。
- `resolveCliModel(input)` / `resolveModelScopeWithDiagnostics()`：解析 `--model` / `--models` / `enabledModels` 的模型选择，产出 `ScopedModel[]`，供 `AgentSession` 轮转（`Ctrl+P`）。
- `hasConfiguredAuth(provider)` / `checkAuth(provider)` / `isUsingOAuth(provider)`：供 `prompt()` 前校验与错误提示（`formatNoApiKeyFoundMessage` / `formatNoModelSelectedMessage`）。

---

## 5. 工具系统（`core/tools/*` — 8 工具）

| 工具 | 输入 | 输出 `details` | 特点 |
|---|---|---|---|
| `read` | `{ path }` | `ReadToolDetails { content, truncated }` | 截断策略：`DEFAULT_MAX_LINES`/`DEFAULT_MAX_BYTES`，`truncateHead/Line/Tail` |
| `bash` | `{ command, timeout? }` | `BashToolDetails { stdout, stderr, exitCode }` | 通过 `BashOperations` 抽象（`createLocalBashOperations` / Gondolin 远程），`file-mutation-queue` 串行化写操作 |
| `powershell` | `{ command }` | `PowerShellToolDetails` | 同 bash，`PowerShellOperations` |
| `edit` | `{ path, oldText, newText }` | `EditToolDetails { diff }` | 原子替换，`generateDiffString` 产出 diff 供 UI 着色 |
| `write` | `{ path, content }` | `—` | 创建/覆盖，二者均受 `withFileMutationQueue` 保护 |
| `grep` | `{ pattern, path }` | `GrepToolDetails { matches }` | 对应 `rg` 封装 |
| `find` | `{ pattern, path }` | `FindToolDetails` | glob 搜索 |
| `ls` | `{ path }` | `LsToolDetails { entries }` | 目录列举 |

所有工具通过 `create*ToolDefinition()` 产出 `ToolDefinition<TParams, TDetails>`（含 `parameters: TypeBox` / `promptSnippet` / `promptGuidelines` / `renderCall/renderResult`），再由 `wrapRegisteredTools()` 包装为 `AgentTool`（注入 `signal` / `onUpdate` / `cwd`）。

默认工具集为 `[read, bash, edit, write]`（`agent-session.ts:405`），可通过 `initialActiveToolNames` / `allowedToolNames` / `excludedToolNames` / `baseToolsOverride` 定制；`SettingsManager` 也持久化用户的 `activeTools` 选择。

---

## 6. Compaction（`core/compaction/`）

### 6.1 触发条件

`shouldCompact(tokens, contextWindow, settings)`（阈值默认约 75-80% 窗口，具体见 `DEFAULT_COMPACTION_SETTINGS`）。三类 `reason`：

- `threshold`：`prepareNextTurn` 中检测到将溢出，自动后台压缩。
- `overflow`：LLM 返回 `context_overflow` 可恢复错误，强制压缩后重试被中断 turn。
- `manual`：用户 `/compact`。

### 6.2 流程

```
prepareCompaction(messages, settings) → { cutPoint, entriesToSummarize, tokensToKeep }
  findCutPoint() / findTurnStartIndex() 找“turn 边界”处的切割点
  serializeConversation(entriesToSummarize) 序列化为待摘要文本
generateSummary(model, serialized, customInstructions?) → LLM 调用
  generateSummaryWithUsage() 带 usage 统计
compact(messages, summary, oldLeafId, newLeafId) → CompactionEntry
  保留 cutPoint 后消息 + 单条 summarization 消息，开头插入 CompactionEntry
```

扩展可通过 `session_before_compact` 事件 **取消**或**覆盖** `compaction` 结果（`SessionBeforeCompactResult: { cancel?, compaction? }`）。

### 6.3 Branch Summarization

`collectEntriesForBranchSummary(tree, targetId)` 与 `generateBranchSummary(model, entries, label?)` 在 `navigateTree` 时为被切掉的分支生成 `BranchSummaryEntry`，使导航不丢失上下文。

---

## 7. Settings / Trust / Resources

- **SettingsManager**（`settings-manager.ts`）：读写 `~/.pi/agent/settings.json` 与项目级 `.pi/settings.json`，管理 `model` / `thinkingLevel` / `compaction` / `retry` / `imageSettings` / `activeTools` 等；`CompactionSettings` / `RetrySettings` 有默认值与校验（`validateCompactionSettings` / `validateRetryPolicy`）。
- **ProjectTrustStore**（`trust-manager.ts`）：`~/.pi/agent/trust.json`，记录每个 `cwd` 的信任状态 `yes|no|undecided`，扩展可通过 `project_trust` 事件介入；`hasTrustRequiringProjectResources()` 判定是否需弹 trust 确认。
- **ResourceLoader**（`resource-loader.ts`）：扫描 Skills / Prompts / Themes / Context Files（`AGENTS.md` / `README.md` 等），`loadProjectContextFiles()` 聚合项目上下文注入 `buildSystemPrompt()`。

---

## 8. 运行模式（`modes/*`）

### 8.1 Interactive（`modes/interactive/interactive-mode.ts`）

- 基于 `TuiMainScreen` + `TuiAltScreen`，主区域为 `ChatViewport`（`chat-viewport.ts`），底部为输入 `Editor`，覆盖层为 `SelectList`（模型/会话/命令选择）、`LoginDialog`、`ConfigSelector` 等 35+ 组件（`modes/interactive/components/`）。
- `InteractiveMode` 持有 `AgentSession` + `TuiRenderer`，将 `AgentSessionEvent` 转换为 TUI 组件更新（`assistant-message` 增量渲染、`tool-execution` 的 `renderCall/renderResult`、`diff` 着色、`mermaid` 图、`thinking` 折叠等）。
- 键盘：`keybindings.ts` 将 `KeyId`（如 `ctrl+p`）映射到动作；通过 `TUI_KEYBINDINGS` 默认绑定，并暴露 `setKeybindings` 供扩展修改。

### 8.2 Print（`modes/print-mode.ts`）

一次性执行：`runPrintMode(input, options)` → 创建 `AgentSession` → `session.prompt(input)` → 等待 `isIdle` → 输出结果到 stdout。用于 `echo "fix bug" | pi --print` 场景；无需 TUI。

### 8.3 RPC（`modes/rpc/*`）

- `rpc-mode.ts`：启动 JSONL 协议的 RPC 服务端，stdin/stdout 每行一 JSON（`json-event.ts`）。
- `rpc-client.ts`：`RpcClient` 通过 framed CBOR / JSONL 与服务端通信，提供 `prompt`/`abort`/`setModel` 等方法。
- `RpcEntry`（`rpc-entry.ts`）：打包产物入口，供 `client`/`server` 的远程部署复用。

---

## 9. SDK（`core/sdk.ts`）

供外部 TypeScript 程序以库方式使用 Pi：

```ts
import { createAgentSession, createAgentSessionServices } from "@earendil-works/pi-coding-agent";

const { session } = await createAgentSession({
  cwd, model, thinkingLevel,
  customTools: [defineTool({...})],
  allowedToolNames: ["read","bash","edit","write"],
});
await session.prompt("修复 bug");
```

`createAgentSessionServices` / `createAgentSessionFromServices` 支持复用已创建的服务（Registry/Loader 等）；`AgentSessionRuntime` / `ModelRuntime` 等可单独实例化。

---

## 10. CLI 与 Main（`main.ts` / `cli.ts` / `cli/args.ts`）

`parseArgs(argv)` 解析 `--model` / `--models` / `--thinking` / `--no-session` / `--print` / `--rpc` / 扩展注册的 `--flag` 等，`main(options)` 据此选择 `InteractiveMode` / `runPrintMode` / `runRpcMode`，并处理单例锁、更新检查、错误码等。

