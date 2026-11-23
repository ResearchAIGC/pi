# 06 — CodingAgent 编排：`AgentSession` 如何把一切粘成可用的 `pi`

> 源码锚点：`packages/coding-agent/src/core/agent-session.ts:2652` / `packages/coding-agent/src/core/session-manager.ts:1746` / `packages/coding-agent/src/core/tools/` / `packages/coding-agent/src/core/compaction/`

---

## 1. 为什么需要 `AgentSession`

`Agent`（`packages/agent/src/agent.ts:592`）只解决“单次运行的循环与事件”，但 `pi` 命令要解决的是：

1. **多关注点协同**：模型选择、鉴权、工具集、压缩、重试、分支、Skills、扩展、持久化、界面事件，全部要在一次 `prompt()` 中协调。
2. **多运行模式复用**：`pi`（交互式终端）、`pi --print`（一次性执行）、`pi --rpc`（JSONL 服务端）、SDK（库调用）四种模式应复用同一编排，否则逻辑会分叉。
3. **内存与文件的双写一致**：`Agent.state.messages`（内存中的历史）与 `SessionManager`（文件中的历史）必须在每条消息结束后保持一致。如果扩展在 `message_end` 时改写了消息，两处都要同步修正。

`AgentSession` 就是这个粘合层（`core/agent-session.ts:2652`）。所有运行模式共享它，各模式只在输入输出层不同。

---

## 2. 模块地图

```
packages/coding-agent/src/
├── cli/args.ts + main.ts                    # 参数解析与模式分发
├── core/
│   ├── agent-session.ts  (2.6k)             # 编排核心（本章主角）
│   ├── session-manager.ts (1.7k)            # JSONL 图持久化（03 已讲）
│   ├── model-runtime.ts  (795)              # 模型与鉴权的桥梁
│   ├── tools/  read/bash/edit/write/grep/find/ls/powershell
│   ├── compaction/  compact/branch-summarization/utils
│   ├── extensions/  loader/runner/wrapper/types
│   ├── skills.ts / prompt-templates.ts / resource-loader.ts / system-prompt.ts
│   ├── settings-manager.ts / trust-manager.ts
│   └── sdk.ts                               # createAgentSession 等
└── modes/
    ├── interactive/ (35+ 组件，终端界面)
    ├── print-mode.ts (一次性)
    └── rpc/ (JSONL 服务端)
```

---

## 3. AgentSession 的状态与事件

### 3.1 内部状态（按职责分组）

```ts
class AgentSession {
  readonly agent: Agent; sessionManager: SessionManager;

  // 队列：中途纠偏与结束后追问的界面镜像
  private _steeringMessages: string[];
  private _followUpMessages: string[];
  // 上下文：伴随下一次输入的附加消息
  private _pendingNextTurnMessages: CustomMessage[];
  private _pendingCustomMessages: CustomMessage[]; // 轮次内暂存，turn_end 时才插入

  // 异步任务的控制器
  private _compactionAbortController / _autoCompactionAbortController / _branchSummaryAbortController;
  private _retryAttempt; _retryAbortController?: AbortController;

  // 工具：已注册与当前生效的两套表
  private _toolRegistry: Map<string, AgentTool>;
  private _toolDefinitions: Map<string, ToolDefinitionEntry>;

  // 提示词：基础版本与运行中被扩展改写后的快照
  private _baseSystemPromptOptions: NormalizedBuildSystemPromptOptions;
  private _runSystemPromptOptions?: NormalizedBuildSystemPromptOptions;
}
```

### 3.2 事件与双写（`_handleAgentEvent:639-761`）

每当 `Agent` 产生事件，`AgentSession` 按固定顺序处理：

```ts
_handleAgentEvent = async (event: AgentEvent) => {
  if (event.message_start && role==="user") { /* 从队列移除已投递的消息，通知界面 */ }
  await _emitExtensionEvent(event);          // 1. 先通知扩展
  _emit(sessionEvent);                        // 2. 再通知外部订阅者（如 TUI）
  if (event.message_end) {
    // 3. 落盘：根据消息角色写入不同类型的记录
    if (role==="custom") sessionManager.appendCustomMessageEntry(...);
    else sessionManager.appendMessage(message); // user/assistant/toolResult/system
  }
  if (event.turn_end) _flushPendingCustomMessages(); // 避免插入到工具调用与结果之间
}
```

**双写不变量**：内存与文件在 `message_end` 后保持一致。若扩展通过 `message_end` 改写了消息，`_replaceMessageInPlace()`（`agent-session.ts:791-805`）会原地修改事件对象，使两处同时修正。

### 3.3 工具钩子（`_installAgentToolHooks:508-564`）

- `beforeToolCall` 转发给扩展的 `tool_call` 事件，支持拦截（`block`）与改写。
- `afterToolCall` 转发给 `tool_result` 事件，并对返回的图片做归一化（受 `settings.imageAutoResize` 与 `model.inputLimits.images.resize` 控制，`0.86.1` 新增）。

### 3.4 下一轮刷新（`_installAgentNextTurnRefresh:585-619`）

在每轮结束、下一轮开始前的间隙：

1. 检查是否需要压缩，若需要则先执行 `_runAutoCompaction`。
2. 用最新的 `agent.state.messages` 重建上下文，并通过 `SystemMessage.sections` 的差异持久化提示词与工具的变更（`0.86` 的持久化机制）。

---

## 4. 一次 `prompt()` 的过程

为便于理解，将 `agent-session.ts:1321-1432` 的 11 步按四个阶段归纳：

**阶段一：分流与拦截（步骤 1-3）**

```
1. 若输入以 "/" 开头，尝试作为扩展命令执行 → 命中则直接返回，不进模型
2. 若压缩正在进行，拒绝新输入
3. 触发扩展的 input 事件，可改写或接管输入
```

**阶段二：内容展开（步骤 4-6）**

```
4. 展开 /skill:name → 技能内容块
5. 展开 /template → 模板文件内容
6. 若正处于流式中，按策略将输入加入 steering（中途纠偏）或 followUp（收尾追问）队列
```

**阶段三：校验与准备（步骤 7-10）**

```
7. 校验模型是否已选、鉴权是否就绪（现场刷新短生命周期 token）
8. 检查上次中断后是否需要压缩
9. 触发 before_agent_start，扩展可注入上下文或改写提示词（0.86 起会持久化）
10. 归一化图片（缩放 + 提示拼接到文本）
```

**阶段四：执行与循环（步骤 11）**

```
11. 组装最终消息列表 → 调 agent.prompt() 进入循环 → 每轮结束后检查：
    是否需要重试？是否需要压缩？是否有扩展排入的后续消息？
    任一为是则 continue，否则结束
```

---

## 5. ModelRuntime（`core/model-runtime.ts:795`）

`AgentSession` 与 `pi-ai` 模型目录之间的桥梁：

- 聚合 `ModelRegistry`（模型发现）、`AuthStorage`（凭证存取）、`ModelsStore`（持久化目录）。
- `getAuth(model)`：按厂商解析 `apiKey` / `headers` / `baseUrl` / `env`，支持短生命周期凭证的同步。
- `resolveCliModel` / `resolveModelScope`：解析命令行 `--model` / `--models` 参数，产出可通过 `Ctrl+P` 轮转的模型列表。
- `hasConfiguredAuth` / `checkAuth` / `isUsingOAuth`：供 `prompt()` 前校验与错误提示。
- `0.86.1` 新增 `meta` Provider 的 Muse OAuth（`/login meta` + `META_API_KEY` 自动刷新）。

---

## 6. 八个工具的契约（`core/tools/*`）

| 工具 | 输入 | 输出 | 特点 |
|---|---|---|---|
| `read` | `{ path }` | 含 `content` 与是否截断 | 大文件自动截断，支持头尾截断 |
| `bash` | `{ command, timeout?, workdir? }` | 含 `stdout`、`stderr`、`exitCode` | 通过 `BashOperations` 抽象，本地或远程（如 Gondolin）执行 |
| `powershell` | `{ command }` | 同上 | 同 `bash`，Windows 侧 |
| `edit` | `{ path, oldText, newText }` | 含 `diff` | 精确替换，生成 diff 供界面着色 |
| `write` | `{ path, content }` | — | 创建或覆盖，受文件级队列保护 |
| `grep` | `{ pattern, path }` | 含匹配列表 | 封装 `rg` |
| `find` | `{ pattern, path }` | 含文件列表 | glob 搜索 |
| `ls` | `{ path }` | 含目录项 | 列目录 |

所有工具通过 `create*ToolDefinition()` 定义（含参数校验、提示词片段、渲染函数），再由 `wrapRegisteredTools()` 注入 `cwd`、`signal` 等运行时信息。默认生效的是 `[read, bash, edit, write]`（`agent-session.ts:405`），可通过配置定制；`0.86.0` 起扩展可动态切换，且默认工具启用了严格的参数约束。

---

## 7. 压缩与分支摘要

### 7.1 何时压缩

`shouldCompact(tokens, contextWindow, settings)` 判断是否接近上下文上限（阈值约 75-80%）。三类触发原因：

- `threshold`：下轮将溢出，自动后台压缩。
- `overflow`：模型返回 `context_overflow` 错误，强制压缩后重试。
- `manual`：用户执行 `/compact`。

`0.86.0` 支持按模型覆盖：`compaction.modelOverrides: { [modelId]: { reserveTokens, keepRecentTokens } }`。

### 7.2 压缩流程

```
1. prepareCompaction → 找到切割点（turn 边界）、待摘要的记录、需保留的 token 数
2. generateSummary → 调模型生成摘要
3. compact → 保留切割点后的消息 + 单条摘要，头部插入压缩记录
```

扩展可通过 `session_before_compact` 取消或覆盖压缩结果。

### 7.3 分支摘要

切换分支时，为被切掉的分支生成摘要记录，使导航不丢上下文。

---

## 8. Skills 与 Prompt Templates

- **Skills**（`core/skills.ts`）：可复用的 Markdown 指令块，位于 `~/.pi/agent/skills/<name>/SKILL.md` 或项目内 `.pi/skills/`。`/skill:name [args]` 会展开为技能内容。
- **Prompt Templates**（`core/prompt-templates.ts`）：文件式斜杠命令，`~/.pi/agent/prompts/<name>.md`，`/name args` 展开为文件内容拼参数。
- **ResourceLoader**（`core/resource-loader.ts`）：扫描 Skills、模板、主题、上下文文件（`AGENTS.md` 等），聚合后注入提示词。`TrustManager`（`~/.pi/agent/trust.json`）在发现项目级资源时决定是否弹确认。

---

## 9. 运行模式与 SDK

### 9.1 三种模式

- **Interactive**（`modes/interactive/`）：终端交互模式，`ChatViewport` 将 `AgentSessionEvent` 转为界面更新，含 35+ 组件。
- **Print**（`modes/print-mode.ts`）：一次性执行，`echo "fix" | pi --print`，执行完输出到标准输出。
- **RPC**（`modes/rpc/`）：JSONL 服务端，每行一个 JSON，通过 `pi-protocol` 的 CBOR 帧通信；`experimental/micro` 为 `0.86.0` 新增。

### 9.2 SDK（`core/sdk.ts`）

```ts
import { createAgentSession, createAgentSessionServices } from "@earendil-works/pi-coding-agent";
const { session } = await createAgentSession({ cwd, model, thinkingLevel, customTools: [defineTool({...})] });
await session.prompt("修复 bug");
```

`0.86.0` 起扩展可通过 `ctx.modelRegistry.stream()` 直接调模型（走已配置的厂商与鉴权）。

---

## 10. 0.86 的增量小结

- 扩展对提示词与工具的修改会持久化到会话，跨重启与分支保留。
- 缓存预热：长工具运行与空闲时按 `Model.promptCache` 的寿命，成本感知地刷新提示词缓存。
- `/bug` 上报：环境、模型、扩展等信息脱敏后打包，上传或导出为 ZIP。
- 其他：`Radius` 离线目录、`ToolSearch` 的延迟加载、`--resume` 的渐进式选择器等。
