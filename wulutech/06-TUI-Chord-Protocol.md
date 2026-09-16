# 06 — TUI / Chord / Protocol / Session / Telemetry

> 源码：`packages/tui/` / `packages/chord/` / `packages/protocol/` / `packages/client/` / `packages/server/` / `packages/telemetry/` / `packages/session-backends/`

---

## 1. TUI（`packages/tui` — `pi-tui`）

### 1.1 定位

`pi-tui` 是 Pi 自研的 **差分渲染终端 UI 库**，替代 Ink/Blessed，特点是“模型化组件 + 布局引擎 + 增量渲染 + 原生集成”。与 `coding-agent` 的 `interactive` 模式深度绑定，但也可独立使用。

### 1.2 架构

```
TUI (tui.ts)               — 抽象接口：size / render / input / mouse / overlay
├── TuiMainScreen          — 主屏实现（接管 stdout，差分渲染，AltScreen 切换）
├── TuiAltScreen           — 备用屏实现（用于 fullscreen 覆盖层如 SessionSelector）
└── Components             — Box / VStack / HStack / Text / ScrollView / SelectList / Input / Editor / Markdown / Image / Loader ...

Layout (layout.ts / layout-node.ts)
  布局节点树 → measure → arrange → 产出 TuiLine[] → 差分对比 prev 帧 → 最小化 ANSI 输出

Input (keys.ts / keybindings.ts / stdin-buffer.ts)
  原始 stdin bytes → StdinBuffer (批量切分、Kitty 协议检测) → Key (id/type/modifiers) → KeybindingsManager → 动作

Native (native-*.ts / terminal.ts / terminal-colors.ts / terminal-image.ts)
  ProcessTerminal（Node 进程终端）、颜色探测（OSC 11）、图片（Kitty / iTerm2 / Sixel）、剪贴板（OSC 52 / 平台原生）
```

### 1.3 核心组件

| 组件 | 文件 | 作用 |
|---|---|---|
| `Box` | `components/box.ts` | 带边框/内边距的容器，支持 `borderStyle` |
| `VStack` / `HStack` | `components/v-stack.ts` / `h-stack.ts` | 垂直/水平栈布局 |
| `Text` / `TruncatedText` | `components/text.ts` / `truncated-text.ts` | 文本与截断文本 |
| `ScrollView` | `components/scroll-view.ts` | 可滚动视口，支持 `scrollTo` 与滚动条 |
| `SelectList` | `components/select-list.ts` | 带搜索的选项列表（模型/会话选择） |
| `SettingsList` | `components/settings-list.ts` | 设置项列表 |
| `Input` / `Editor` | `components/input.ts` / `components/editor.ts` | 单行/多行输入；`Editor` 支持 `undo-stack`/`kill-ring`/`word-navigation` |
| `Markdown` | `components/markdown.ts` | 基于 `marked` 的 Markdown 渲染（`Marked`, `Token`, `Tokens`） |
| `Image` | `components/image.ts` | 终端图片（自动选择 Kitty/iTerm2/Sixel 协议） |
| `Loader` / `CancellableLoader` | `components/loader.ts` / `cancellable-loader.ts` | Spinner 与可取消加载器 |

### 1.4 输入与快捷键

- `keys.ts`：`Key`（`id` + `type: press|repeat|release` + `modifiers`）+ `parseKey` / `matchesKey` / `decodeKittyPrintable` / `isKittyProtocolActive`。
- `keybindings.ts`：`KeybindingDefinition → Keybindings → KeybindingsManager`，默认 `TUI_KEYBINDINGS` / `DEFAULT_EDITOR_KEYBINDINGS` / `DEFAULT_APP_KEYBINDINGS`（`AGENTS.md` 要求新增快捷键必须加入默认绑定而非硬编码 `matchesKey("ctrl+x")`）。
- `stdin-buffer.ts`：`StdinBuffer` 将原始 `data` 事件按“输入帧”聚合，处理粘包/半包与 Kitty 键盘协议。

### 1.5 渲染管线

```
VStack 树（coding-agent 组装，如 ChatViewport）
  → measure (visibleWidth / wrapTextWithAnsi)
  → arrange (terminal size)
  → flatten 为 TuiLine[] (每行 cells + cursor)
  → diff(prevLines, nextLines) → 最小 ANSI 重绘序列
  → ProcessTerminal.write()
```

`tui-main-screen.ts` 负责主循环：监听 `SIGWINCH` 尺寸变更、接管 `stdin` raw mode、管理 `Overlay`（`overlayOptions` / `OverlayHandle`）的锚定与层叠。

### 1.6 原生能力

- **剪贴板**：`native-platform.ts` → `getNativeClipboard()`（优先平台原生，fallback OSC 52）。
- **图片**：`terminal-image.ts` → `detectCapabilities()` / `renderImage()`（支持 PNG/JPEG/WebP/GIF，自动 Kitty/iTerm2 编码，`encodeKitty` / `encodeITerm2`）。
- **颜色**：`terminal-colors.ts` → `parseTerminalColorSchemeReport()` / `parseOsc11BackgroundColor()`。
- **预编译**：`native/darwin` / `native/linux` / `native/win32` 的 `build.sh` 产出 `.node` 预编译件，通过 `native-module-path.ts` 解析。

---

## 2. Chord（`packages/chord` — `chord`）

### 2.1 定位

Chord 是 **应用组合运行时（Application Composition Runtime）**，提供 `Facet`（插件）+ `replicatedState`（复制状态）+ `RemoteService`（RPC）+ `ServiceProvider`（发布/订阅）四类原语。Pi 用它来组装复杂的多服务应用，但其本身是独立通用库。

### 2.2 导出（`src/index.ts:4-79`）

```ts
// Facet 系统
createFacetHost, createStaticFacetLoader, combineFacetLoaders, defineFacet, defineService

// 复制状态
replicatedState

// 远程服务
createRemoteServiceBinding, createRemoteServiceEndpoint, RemoteServiceProvider,
RemoteServiceError, parseServiceCall, decodeServiceControlCall, ...

// 工具
isJsonValue, JsonValue, JsonRepresentation
```

### 2.3 模型

- **Facet**（`facets/`）：`Facet` 为声明式模块，`FacetHost` 为宿主，`FacetLoader` 负责发现/加载（静态或远程）。`defineFacet()` 定义 facet 的 `name` + `services` + `load()`。
- **Service**（`services/`）：`defineService()` 定义服务的 `name`/`mode`/`methods`，`RemoteServiceProvider` 发布服务状态，客户端通过 `createRemoteServiceBinding` 绑定并调用（`ServiceCall` → `WireServiceProviderUpdate` → `ServiceSubscriptionSnapshot`）。
- **ReplicatedState**（`delta/`）：基于操作日志（delta）的可复制状态，支持 `MutableReplicatedState` 与订阅。
- **Context**（`context/`）：`Context` / `ContextKey` 依赖注入容器，贯穿 Facet/Service。

### 2.4 目录

```
packages/chord/src/
├── api.ts / types.ts / json.ts          # 公共 API 与 JsonValue 类型
├── node.ts / node/                      # Node 宿主实现
├── context/                             # 依赖注入
├── delta/                               # 复制状态 delta 协议
├── facets/                              # Facet 加载与宿主
└── services/                            # RemoteService 协议与编解码
```

---

## 3. Protocol（`packages/protocol` — `pi-protocol`）

### 3.1 定位

`pi-protocol` 是 **传输中立** 的 CBOR 协议，用于 `pi-client` ↔ `pi-server` 间的远程会话。传输层无关（可走 stdout/stdin、Unix Socket、TCP 等），仅定义帧与信封。

### 3.2 版本与握手（`src/protocol.ts:5`）

```ts
export const PROTOCOL_VERSION = 8 as const;
```

握手：客户端首帧必为 `{ type:"hello", version }`（`ClientHello`），服务端回 `{ type:"hello", version:8, serverId: UUIDv4 }`（`ServerHello`）或 `{ type:"hello_error", error }`。

### 3.3 信封（`src/protocol.ts:28-110`）

```ts
RpcTarget = ServerTarget { serverId } | SessionTarget { serverId, sessionId, attachmentId }

ClientMessage = ClientHello | RequestEnvelope { type:"request", id, target, call: JsonValue }
              | CancelEnvelope  { type:"cancel",  id, target }

ServerMessage = ServerHello | ServerHelloError
              | ResponseEnvelope { type:"response", id, ok: true, result? } | { ok:false, error }
              | ServiceEventEnvelope { type:"service_update", subscriptionId, update }
              | AttachmentEnvelope   { type:"attachment", attachment: SessionTarget | null }
```

`call: OpaqueJsonValue`（`JsonValue`）使协议可承载任意 Chord Service 调用；`cancel` 支持取消进行中的 RPC。

### 3.4 编解码（`src/cbor/` / `codec.ts` / `framing.ts`）

- `cbor/`：CBOR 编码器/解码器（紧凑二进制，支持 `JsonValue` 全集）。
- `framing.ts`：帧封装（length-prefix framing），保证流式传输中的消息边界。
- `codec.ts`：`encode(ClientMessage) → Uint8Array` / `decode(bytes) → ServerMessage`。

---

## 4. Client / Server

### 4.1 Client（`packages/client` — `pi-client`）

```
client.ts       — PiClient：连接管理、Request/Response 关联
connection.ts   — Connection：帧收发、重连、心跳
transport.ts    — Transport 抽象（framed CBOR bytes）
unix.ts         — Unix Socket 传输（UnixTransport）
promise.ts / errors.ts / types.ts
```

`PiClient` 为传输中立：构造时传入 `Transport`（如 `UnixTransport`），后续所有 `request(target, call)` / `subscribe(service)` 均走该传输。

### 4.2 Server（`packages/server` — `pi-server`，experimental）

```
server.ts            — PiServer：监听、握手、路由
session-router.ts    — SessionRouter：按 sessionId 路由到对应 Session 附件
listener.ts          — Listener 抽象
connection.ts        — 服务端 Connection
transports/unix/     — Unix Socket 监听
types.ts / errors.ts
```

`SessionRouter` 负责多会话多附件的 `attachmentId` 管理；`AttachmentEnvelope` 带外更新当前 presentation 的选中 Session 路径。

---

## 5. Telemetry（`packages/telemetry` — `pi-telemetry`）

### 5.1 定位

厂商中立的 **Telemetry 契约与 Schema**，定义 Span/Event 的强类型 Schema，供 `agent`/`ai` 产出一致的可观测数据，适配器层再桥接到 OTel / 自定义后端。

### 5.2 核心 API（`src/index.ts`）

```ts
defineTelemetrySchema({ spans: { mySpan: { startAttributes, endAttributes, events } } })
  → TelemetrySchemaDefinition (含 SchemaTelemetrySpan 等类型)

createTypedSpanStarter(schema, telemetryContext)
  → TypedSpanStarter<Schema> （类型安全的 startSpan）

InMemoryTelemetryContext / NOOP_TELEMETRY_CONTEXT
  → TelemetryContext 实现（测试用内存记录 / 生产用空操作）

RecordedTelemetryEvent / RecordedTelemetrySpan / SpanAttributes
```

### 5.3 在 Pi 中的使用

- `agent` 的 `agent-loop` / `Harness.Drive` 在每次 `stream` / `tool execution` / `compaction` 时 `startHarnessSpan` / `startAiSpan`（`agent/src/harness/telemetry.ts`），产出 `HarnessSpan` / `AiSpan`。
- `ai` 的每个 `api/*.ts` 在 HTTP 调用时产出 `AiSpan`（含 `provider`/`model`/`usage`/`error` 属性）。
- `TelemetrySchema` 保证所有 Span 的 `start/end/event` 属性经 `TypeBox` 校验，跨包一致；`testing/` 提供一致性测试套件。

---

## 6. Session Backends

### 6.1 sqlite-node（`packages/session-backends/sqlite-node`）

Node 原生的 SQLite 后端，实现 `agent/src/harness/session/types.ts` 的 `Storage` 接口：

```ts
interface Storage {
  commit(writes: Write[], context): Promise<CommitResult>;
  getEntries(ids, context): Promise<Map<id, Entry>>;
  getValue<T>(address: Value<T>, context): Promise<StoredValue<T> | undefined>;
  scanBranch(query, context): Promise<Entry[]>;
  getStats(context): Promise<SessionStats>;
  close(context): Promise<void>;
}
```

`Write` 含 `entry` / `value` / `list` 三类；`Value<T>` 为类型化地址（`branchTip`/`laneConfig` 等）。SQLite 的事务保证 `insertEntry + setValue(branchTip)` 的原子性。

### 6.2 JSONL（`packages/agent/src/harness/session/jsonl/`）

文件系统后端：

```
jsonl/
├── codec.ts       — Entry ↔ JSON 行编解码
├── io.ts          — 追加写与流式读
├── repo.ts        — 目录级仓库（sessions/ 下每 session 一 JSONL）
├── storage.ts     — Storage 实现
├── fork.ts        — fork 复制逻辑
├── legacy-v3.ts   — v3 迁移
└── types.ts
```

`StorageBackedSession`（`agent/src/harness/session/session.ts`）在此之上封装 `MutationLine`（串行化并发 `mutate` 回调）与 `IdGenerator`（`uuidv7`）。

---

## 7. 协同示例：一次远程 Pi 调用

```
用户（本地）                PiClient (client/unix)              PiServer (server/unix)               AgentHarness (agent)
   │  pi --remote ...          │                                      │                                       │
   ├── hello(ver=8) ──────────→│── hello(ver=8) ─────────────────────→│                                       │
   │                           │←─ hello(ver=8, serverId) ───────────│                                       │
   ├── request(id=1, target=SessionTarget, call=prompt("fix bug")) → │── Lane.prompt() ─────────────────────→ │
   │                           │                                      │  Drive: Generation → Tools → Response │
   │←─ response(id=1, ok) ─────│←─ service_update (stream events) ───│←─ HarnessEventBus.emitBatch(...) ─────│
   │                           │←─ response(id=1, ok, result) ──────│                                       │
```

`protocol` 的 CBOR 帧封装与 `Chord` 的 `ServiceProvider` / `RemoteServiceBinding` 共同完成跨进程 Service 订阅与调用。

