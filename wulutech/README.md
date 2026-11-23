# Pi Agent 解决方案 — 全面分析文档集

> 项目：`earendil-works/pi`（`pi` / `Pi Agent Harness`）  
> 版本：`0.86.1`（`packages/*/package.json` 同步） | 分析时间：`2026-09-21` | 落盘：`wulutech/`  
> 方法：全量读源码 + 运行期校验（`npm run check`、模型目录生成链、事件/类型契约）

**这套文档为从未接触过 Pi 的读者而写。** 不假设你了解 Agent Harness 或 Pi 的历史版本。阅读顺序按“一条用户请求从输入到落盘”的因果展开，每章先用人话讲清“要解决什么问题”，再给代码与文件锚点。

---

## 一句话定位

**Pi 是一套把“编码 Agent”从一次性聊天做成可持久化、可分支、可扩展的终端操作系统的 Harness（底座）。**

- **Harness 是什么**：你可以把它理解为“Agent 的操作系统”。它不直接写提示词，而是提供会话、分支、压缩、重试、队列、持久化这些可被复用的底座能力。
- **产品与底座的关系**：对外你用到的命令是 `pi`（`@earendil-works/pi-coding-agent`）；对内它由底座（`@earendil-works/pi-agent-core` + `@earendil-works/pi-durable`）、能力（`@earendil-works/pi-ai`，统一多模型）、交互（`@earendil-works/pi-tui` + `@earendil-works/chord` + `pi-protocol`）三部分粘合而成。
- **与套壳 Chatbot 的区别**：Pi 的竞争力不在更聪明的提示词，而在四件套——**会话 DAG**（会话是一张可分支的图，而非一条直线）、**Lane 并发域**（一个分支一个隔离域）、**扩展即一等公民**（不改内核就能加能力）、**Provider 中立**（一套代码对接 30+ 模型）。

---

## 文档导航（按顺序读）

| 序号 | 文档 | 它回答什么 | 适合谁 |
|---|---|---|---|
| 01 | [项目概览](./01-项目概览.md) | Pi 解决什么问题？12 个包为什么要这样拆？ | 所有人（8 分钟，先读此篇） |
| 02 | [架构总览](./02-架构总览.md) | 一条 `prompt("修复 bug")` 如何穿透 5 层到达 LLM 并返回？ | 想做技术选型的架构师 |
| 03 | [会话与持久化](./03-会话与持久化.md) | 为什么说“会话是一张图（DAG）”？新旧两代存储如何衔接？ | 关心存储、分支、压缩的开发者 |
| 04 | [Agent 内核](./04-Agent内核.md) | `Agent` 的双层循环与 `Harness.Drive` 状态机如何分工？ | 想改循环、工具并发的内核开发者 |
| 05 | [AI 抽象层](./05-AI抽象层.md) | 如何用 `Api` 组织 30+ 模型？“类 OpenAI 但有差异”的网关如何被消化？ | 要对接新模型或自建网关的开发者 |
| 06 | [CodingAgent 编排](./06-CodingAgent编排.md) | `AgentSession` 如何把模型、工具、压缩、Skills 粘成可用的 `pi`？ | 想理解 `pi` 产品逻辑或写 SDK 的开发者 |
| 07 | [交互与传输](./07-交互与传输.md) | `pi-tui` 为何自研渲染？`chord` 与 `pi-protocol` 如何支撑远程 Pi？ | 基础设施、TUI、远程部署相关 |
| 08 | [扩展与工程化](./08-扩展与工程化.md) | 不改内核如何加工具与 UI？供应链如何硬化？ | 扩展作者、工程化负责人 |

---

## 10 分钟速读路径

- **只想知道 Pi 是什么**：本 README + `01` 全文。
- **要评估是否采用**：`01` → `02` 的链路图 → `03` 的 fork/compaction 例子。
- **要写扩展或接模型**：`02` 的链路图 → `05` 或 `08`。
- **要定制 TUI 或远程**：`07` 全文。

---

## 关键结论（先说结论）

1. **Harness 大于 Chatbot**：`AgentHarness` 把会话、分支、压缩、重试、队列、持久化做成可复用底座，`coding-agent` 只是其上的一层产品化。`0.86` 新增的 `pi-durable` 是下一代底座的契约层。
2. **会话是一张图（DAG）是最大亮点**：追加日志的 `Entry { id, head, edits }` 天然支持 `fork` / `分支摘要` / `压缩`，远比线性历史适合长程任务。
3. **Provider 中立做得深**：以 `Api`（协议）而非 `Provider`（厂商）组织能力，用 20+ 个兼容开关系统化适配“类 OpenAI 但有差异”的网关。
4. **自扩展是一等公民**：30+ 个生命周期事件 + TypeBox 工具 + UI 覆盖，使容器化、本地模型、项目专属命令都无需改内核。
5. **工程化克制**：从 `erasable TS` 到 `exact pin` / `shrinkwrap` / `--ignore-scripts` 的全链路硬化，在同类 Agent 中少见。

> 版本差异说明：`0.86.0 → 0.86.1` 新增了 `pi-durable` 包、`pico3` 参考内核、`prompt cache warming`、`Meta Muse` 等。`README` 末尾有增量小结，各章正文已同步到 `0.86.1`。

---

## 版本增量（`0.85.1 → 0.86.1` 摘要）

- **新增包**：`@earendil-works/pi-durable`（`packages/durable/src/types.ts:362`），构建顺序变为 `ai → durable → agent`。
- **Agent**：`harness/pico3` 参考内核（实验性），`TranscriptContext` 取代旧 `Context`，工具参数收紧为 `JsonValue`。
- **CodingAgent**：`prompt cache warming`、`/bug` 上报、`per-model compaction`、`Radius` 离线目录、`Meta` Provider。
- **Chord**：`delta: tracker → operation log` 重构；`protocol` 的 `PROTOCOL_VERSION=8` 未变。

---

*本轮基于静态分析与局部运行校验，未执行需真实 API Key 的端到端测试；时序与错误路径以源码契约（`types.ts` 注释与 `agent-loop.ts:156-273` 双层循环）为准。*
