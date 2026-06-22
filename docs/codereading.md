# OpenCode 代码高层总结

## 项目概述

OpenCode 是一个 AI 编码助手（AI coding assistant），提供 TUI（终端 UI）、Desktop（Electron 桌面端）、Server（HTTP API 服务器）三种交互模式。底层基于 Effect v4（响应式 Effect 系统）+ Drizzle（ORM/SQLite）+ Bun 运行时。

---

## 核心功能

1. **Session 会话管理** — 持久化的对话历史，支持多项目多 worktree 并发隔离运行
2. **System Context 系统上下文** — 动态组装可刷新的上下文源（日期、AGENTS.md、技能列表等），在安全的 provider-turn 边界注入模型
3. **Tool Registry 工具注册中心** — 统一工具定义/注册/调用/输出截断，支持 Location 级覆盖和进程级应用工具
4. **LLM Provider 多模型适配** — 通过 protocol adapter 支持多种 LLM 提供商（OpenAI、Anthropic、Google 等）
5. **Prompt 与 Conversation 流** — steer/queue 双投递语义、compaction 对话压缩、Context Epoch 上下文纪元
6. **Plugin 插件系统** — 可扩展工具、上下文源、TUI 特性等
7. **Permission 权限系统** — 工具调用级别的安全授权（wildcard 匹配 deny/allow）
8. **MCP（Model Context Protocol）** — 外部工具服务器集成
9. **LSP 集成** — 语言服务器协议支持

---

## 代码结构（Packages 架构）

```
packages/
├── core/          # 🔴 最核心：Session V2 持久化、System Context、Tool Registry、Event V2、Schema
├── opencode/      # 🔴 业务实现：Prompt loop、Compaction、Tool 内置实现、Skill、Agent、Config、Provider
├── cli/           # 🟡 CLI 入口：命令框架、Daemon 服务管理、命令路由分发
├── tui/           # 🟡 终端 UI：基于 opentui 的 TUI 渲染、路由、keymap、组件
├── llm/           # 🟡 LLM 抽象层：provider adapter、tool definition、streaming、route
├── app/           # 🟢 共享 App 层
├── desktop/       # 🟢 Electron 桌面端
├── server/        # 🟢 HTTP API Server
├── console/       # 🟢 Web Console
├── web/           # 🟢 Web UI
├── ui/            # 🟢 SolidJS 共享 UI 组件
├── plugin/        # 🟢 插件 SDK（对外暴露的 API）
├── sdk/js/        # 🟢 JavaScript SDK
├── identity/      # 🟢 身份认证
├── enterprise/    # 🟢 企业版功能
├── function/      # 🟢 函数运行时
├── slack/         # 🟢 Slack 集成
├── containers/    # 🟢 容器支持
├── effect-drizzle-sqlite/  # Drizzle + SQLite Effect 绑定
├── effect-sqlite-node/     # SQLite Node 绑定
└── docs/          # 文档
```

---

## 最重要的需要读的代码（按优先级）

### 第一优先级：理解核心架构

| 文件 | 说明 |
|------|------|
| `packages/core/src/session/execution.ts` | **SessionExecution** — 全局调度入口：resume/wake/interrupt |
| `packages/core/src/session/run-coordinator.ts` | **SessionRunCoordinator** — 并发 drain 协调器，管理同 Session 的排他执行 |
| `packages/core/src/session/runner/` | **SessionRunner** — 执行一次 provider turn 的核心循环 |
| `packages/core/src/session/input.ts` | **SessionInput** — 持久化 prompt 的 admit/promote 生命周期 |
| `packages/core/src/session/prompt.ts` | **Prompt** — 提示词、文件附件、Agent 附件的数据模型 |
| `packages/opencode/src/session/prompt.ts` | **SessionPrompt loop** — 旧版大循环（1700行），理解执行流程全貌 |

### 第二优先级：上下文与工具系统

| 文件 | 说明 |
|------|------|
| `packages/core/src/system-context/index.ts` | **SystemContext** — 代数的上下文组装/比对/刷新（310行，极精炼） |
| `packages/core/src/system-context/builtins.ts` | 内置上下文源：日期、AGENTS.md 等 |
| `packages/core/src/system-context/registry.ts` | 上下文源注册中心 |
| `packages/core/src/tool/registry.ts` | **ToolRegistry** — 工具注册/物化/结算 |
| `packages/core/src/tool/tool.ts` | **Tool.make** — 规范工具定义（输入/输出 schema + execute） |
| `packages/opencode/src/tool/registry.ts` | 内置工具注册 |
| `packages/core/src/tool-output-store.ts` | 工具输出截断与存储 |

### 第三优先级：事件、消息与会话持久化

| 文件 | 说明 |
|------|------|
| `packages/core/src/session/event.ts` | SessionEvent — 所有事件类型定义 |
| `packages/core/src/session/message.ts` | SessionMessage — 消息类型定义 |
| `packages/core/src/session/schema.ts` | SessionSchema — Session 数据结构 |
| `packages/core/src/session/context-epoch.ts` | ContextEpoch — 上下文纪元管理 |
| `packages/core/src/session/history.ts` | 会话历史投影 |
| `packages/core/src/session/compaction.ts` | Compaction — 对话压缩 |

### 第四优先级：LLM、Provider、权限

| 文件 | 说明 |
|------|------|
| `packages/llm/src/llm.ts` | LLM 抽象接口 |
| `packages/llm/src/providers/` | 各 Provider 适配器 |
| `packages/core/src/permission/` | 权限系统 |
| `packages/core/src/agent.ts` | Agent 定义 |
| `packages/core/src/catalog.ts` | 模型目录 |
| `packages/core/src/location.ts` | Location 抽象（目录 + workspace 定位） |

### 第五优先级：入口与配置

| 文件 | 说明 |
|------|------|
| `packages/cli/src/index.ts` | CLI 入口 |
| `packages/cli/src/commands/` | 命令定义与路由 |
| `packages/opencode/src/index.ts` | opencode 主入口（yargs CLI） |
| `packages/core/src/config/` | 配置系统 |
| `packages/opencode/src/config/` | 业务配置 |

---

## 设计哲学

### 1. Effect First
整个系统构建在 **Effect v4**（`effect-smol`）之上。几乎所有副作用都用 `Effect` 表达，服务通过 `Layer` 组合注入，错误通过 `Schema.TaggedErrorClass` 类型化。不使用 `try/catch`。详见 [关键概念 → Effect](#effect大白话)。

### 2. Event Sourcing（V2 Session Core）
Session 状态通过不可变事件流持久化（`EventV2`）。输入先 admit（录入 inbox），再在安全边界 promote（变为可见消息）。这是 `SessionInput.admit()` → `SessionInput.promoteSteers()` 的流程。

### 3. Location-Scoped Services
服务按 Location（目录 + workspaceID）作用域隔离。`ToolRegistry`、`SessionRunner`、权限等是 Location 级，`SessionExecution` 是全局级但通过 Location 发现对应的 runner。

### 4. Opaque Types + Self-Export Pattern
模块使用 `export * as Foo from "."` 自导出模式，消费方用 `import { Foo } from "@/foo"` 然后 `Foo.Service`、`Foo.layer`。内部 helper 保持非导出。

### 5. System Context Algebra
上下文源定义为独立的类型化源（`Source<A>`），通过代数组合（`make`/`combine`）变成统一 opaque context。观察一次后与快照比较，自动产出 model-visible 的 baseline/update/removal 文本。

### 6. Tool = Schema + Execute
工具通过 `Tool.make({ description, input, output, execute })` 定义。输入/输出 schema 自动转 JSON Schema 给 LLM，错误通过 `ToolFailure` 传递，输出自动截断。

### 7. Run Coordinator
`SessionRunCoordinator` 保证同 Session 只有一个 drain chain 在跑。`run`（显式）和 `wake`（建议）可 coalesce，`interrupt` 停止当前 chain。

### 8. 术语精确性
项目使用精确的领域术语（见 `CONTEXT.md`），如 System Context（非 system prompt）、Session History（非 session context）、Context Epoch、Mid-Conversation System Message 等。

---

## 关键概念

### Drain Chain（排空链）

一次完整的 provider-turn 执行周期：从 Session inbox 中取出已 admit 的输入 → promote 成可见消息 → 调用 LLM → 处理工具调用 → 完成本轮。

- `SessionRunCoordinator`（`run-coordinator.ts:13-28`）保证同一个 Session **最多只有一个** drain chain 在跑，不同 Session 可并发 drain
- 状态机：`idle → draining → draining + 一次 coalesced 重跑 → idle`
- `run` 是显式触发 drain（至少跑一轮 provider turn），`wake` 是建议性唤醒（有新 inbox 数据时可用）
- 多个 wake/resume 会 coalesce：`run` 优先级高于 `wake`，重复 `wake` 只保留最新的 sequence number
- `interrupt` 直接停止当前 drain chain，中断前已登记的 wake 被抑制，中断后的 wake 正常处理

### Session Inbox（会话收件箱）

用户输入不是立刻对 LLM 可见，而是先持久化到 `SessionInputTable`（inbox）中，在安全的 provider-turn 边界被 promote 后才成为可见消息。完整生命周期：

1. **Admit** — `SessionInput.admit()`（`input.ts:54`）：用户消息写入 inbox，此时 `promoted_seq = null`，消息对 LLM 不可见
2. **Promote** — `SessionInput.promoteSteers()`（`input.ts:300`）：将 inbox 中符合条件的输入批量提升为 `SessionMessage.User`，设置 `promoted_seq`

**两种投递语义（`Delivery`）**：

| 类型 | 含义 | Promote 时机 |
|------|------|-------------|
| `steer` | 转向当前对话 | 下次 provider-turn 边界批量 promote |
| `queue` | 排队开新 activity | 当前轮结束后 FIFO 逐个 promote |

所以 drain chain 的本质是：**从 inbox 取出待处理输入 → promote → 调 LLM → 处理工具调用 → 完成本轮**。

### Effect（大白话）

Effect 是一个 TypeScript 库，可以理解为三个东西的组合：

1. **可重试的 Promise** — 普通 Promise 一旦失败就完了，但 Effect 描述的是一段「可能失败的计算」，可以重试、超时、降级，像积木一样组合
2. **内置依赖注入** — 不用到处 `new Service()` 传参，Effect 自动把需要的服务（数据库、文件系统等）注入到函数里，函数只管声明「我需要什么」
3. **全类型安全** — 函数签名直接告诉你「成功返回什么、可能失败什么、需要什么环境」，编译器检查所有分支，不用靠注释或约定

打个比方：手写代码像自己做菜（买菜、切菜、管火候、处理烧锅），Effect 像中央厨房——只描述要做什么菜，厨房帮你处理基础设施和环境，出错了有标准补救流程。OpenCode 整个系统全用 Effect 表达副作用，所以代码里没有 `try/catch`，全是 `yield*` 和 `Effect.gen`。

### lastAssistant

`lastAssistant(sessionID)`（`prompt.ts:1126`）查找一个 Session 中**最近一条 assistant 角色消息**的 Effect。

- **RunLoop 退出判断**（`prompt.ts:1149`）— 通过 `MessageV2.latest(msgs)` 提取 `lastAssistant`，检查其 `finish` 状态和是否有待执行的 tool calls，决定是否结束当前 runLoop
- **Resume/Shell 上下文传递**（`prompt.ts:1395,1402`）— `ensureRunning` 和 `startShell` 接收 `lastAssistant(sessionID)` 的结果，知道上次模型产出了什么，用于恢复会话或启动 shell 时传递上下文

查找逻辑：先找任意非 user 角色消息（assistant/system/tool），找不到则取最近一条消息。

---

## V1 Prompt Loop 详解（`packages/opencode/src/session/prompt.ts`）

这篇 1704 行的文件是 **一次对话的完整生命周期管理器**，涵盖用户输入到 LLM 返回最终答案的整条流水线。

### 对外暴露的 6 个方法（第 86-93 行 Interface）

| 方法 | 行号 | 作用 |
|------|------|------|
| `cancel(sessionID)` | 136 | 中断当前对话，委托给 `SessionRunState.cancel()` |
| `prompt(input)` | 1105 | **主入口**：接收用户消息 → `createUserMessage` → `loop` |
| `loop(input)` | 1392 | 纯执行循环，不创建新消息，从当前状态继续跑 |
| `shell(input)` | 1398 | 执行一次性 shell 命令（不走 AI，直接 spawn 进程） |
| `command(input)` | 1405 | 执行具名命令（如 `/pr`），模板展开后调 `prompt` |
| `resolvePromptParts(template)` | 141 | 解析模板中的 `@filename` 引用，展开成 file/agent parts |

### 对外暴露的类型

- **`PromptInput`** (1576行) — `prompt()` 入参：sessionID + parts（文本/文件/agent/子任务）+ model + agent + format + tools + noReply 等
- **`LoopInput`** (1600行) — `loop()` 入参，只有一个 sessionID
- **`ShellInput`** (1604行) — `shell()` 入参：command + agent + model
- **`CommandInput`** (1613行) — `command()` 入参：command 名称 + arguments + agent + model
- **`Service`** (95行) — Effect 服务标记类 `@opencode/SessionPrompt`
- **`layer`** (97行) / **`defaultLayer`** (1537行) — Effect 依赖注入层，`defaultLayer` 注入所有下游服务
- **`node`** (1675行) — LayerNode 依赖图声明

### 内部核心函数

#### 1. `createUserMessage` (636行) — 把用户输入变成数据库记录

```
输入: PromptInput { sessionID, parts, agent?, model?, ... }
输出: { info: UserMessage, parts: Part[] }
```

流程：
1. **解析 agent** — agent 不存在就报错并列出可用列表
2. **确定 model** — 优先级：用户指定 > agent 默认 > session 当前 model > 系统默认
3. **检查 agent/model 切换** — 与 session 当前值不同则发布 `AgentSwitched` / `ModelSwitched` 事件
4. **遍历 part 解析**（`resolveUserPart`, 712行）：
   - **MCP resource** → 调 MCP 读取 URI 内容，插入 synthetic text
   - **data: URL（text/plain）** → 解码内联文本
   - **file: URL（磁盘文件）** → 调 Read tool 读取文件/目录内容，LSP 支持符号级 range
   - **directory** → 调 Read tool 列出目录
   - **image** → Base64 编码后调用 `image.normalize()` 压缩
   - **agent attachment** → 生成合成文本指导 LLM 调 task tool
5. **持久化** — `sessions.updateMessage(info)` + 逐个 `sessions.updatePart(part)`
6. **V2 双写** — 如果 `experimentalEventSystem` flag 开，发布 `SessionEvent.Prompted` 事件
7. 如果 `input.noReply === true` → 只创建消息不启动 loop（admit-only）

#### 2. `runLoop` (1134行) — 核心 while(true) 循环

```
输入: sessionID
输出: SessionV1.WithParts (最终的 assistant 消息)
```

每轮循环的步骤：

```
while (true):
  1. status.set(sessionID, "busy")
  2. 从 DB 加载消息历史（MessageV2.filterCompactedEffect）
  3. 用 MessageV2.latest() 找 lastUser / lastAssistant / lastFinished / tasks
  
  4. [退出条件] lastAssistant.finish 存在且非 "tool-calls" 且无待处理 tool call
     → 检查孤儿 interrupted tool，打日志后 break
  
  5. step++
  6. [step==1] 异步生成 title
  
  7. 检查 tasks 栈顶：
     - subtask → handleSubtask() → continue
     - compaction → compaction.process() → 可能 stop
  
  8. [overflow 检查] 最后一条 finished 消息的 token 数溢出 → 自动创建 compaction → continue
  
  9. 解析 agent，检查 maxSteps 限制
  10. 应用 SessionReminders（将 code context / plan-reminder 等注入 messages）
  11. 创建 assistant message → sessions.updateMessage(msg)
  12. processor.create({ assistantMessage: msg, model }) → 流处理器
  13. SessionTools.resolve() → 组装当前可用的 tools
  14. 组装 system prompt（env + instructions + skills）
  15. handle.process({ system, messages, tools, model, ... }) → 调 LLM.stream()
  
  16. [structured output 模式] 收到 structured 结果 → msg.structured = output → break
  17. [result=="stop"] → break
  18. [result=="compact"] → 创建 compaction → continue
  19. 否则 → continue（下一轮把 tool result 送回 LLM）
```

关键点：**每轮 while 只做一次 LLM 调用**。LLM 返回 tool call → tool 执行 → 下一轮循环把 tool result 送回 LLM。

中断处理：
- `Effect.onInterrupt(() => finalizeInterruptedAssistant)` (1272,1381行) — fiber 被中断时，标记 assistant message 为 `error: "Aborted"` 并设 `time.completed`

#### 3. `handleSubtask` (239行) — 执行子任务

创建一个新的 assistant message，在其中嵌入一个 tool part（类型为 TaskTool），然后调用 `taskTool.execute()` 真正执行。子任务内部会**递归调 `prompt()`** 开新的对话（子 agent 模式）。

执行结果：
- 成功 → tool part 状态变为 `completed`，output/attachments 回写
- 失败 → tool part 状态变为 `error`
- 中断 → tool part 标记 `"Cancelled"`，assistant message 标记 `finish: "tool-calls"`

如果子任务带有 command，会在完成后插入一条合成 user message（"Summarize the task tool output above and continue"）让主 LLM 继续。

#### 4. `shellImpl` (435行) — 执行 shell 命令

不走 AI 模型，直接在 session 中注入一条 shell 执行记录：
1. 创建 user message + synthetic text part（"The following tool was executed by the user"）
2. 创建 assistant message + tool part（`ShellID.ToolID`, status=running）
3. `ChildProcess.spawn` 执行命令，流式收集 stdout/stderr 到 `output` 变量
4. 被中断时在 output 末尾追加 `<metadata>User aborted the command</metadata>`
5. 完成后更新 tool part 状态为 `completed`

#### 5. `title` (177行) — 自动生成对话标题

仅在 session 标题仍是默认 `"New session - ..."` 格式时触发，仅当对话中有且仅有一条真实用户消息时生成。

用小模型（title agent 或 small model）流式生成标题，清洗 `<think>` 标签，取第一个非空行，截断到 100 字符。

#### 6. `resolvePromptParts` (141行) — 解析 @文件引用

扫描模板字符串中的 `@filename` 模式：
- `~/xxx` → 展开为 home 目录路径
- 相对路径 → 基于 worktree 解析
- 检查路径是否存在：文件 → file part，目录 → 标记为 `application/x-directory`
- 路径不存在 → 尝试匹配 agent 名 → agent part
- 支持 `@` 后跟数字范围（如 `@file.ts:10-20`）

#### 7. `command` (1405行) — 执行具名命令

1. 从 `Command.Service` 查找命令定义
2. 解析 `$1` `$2` 占位符替换参数，`$ARGUMENTS` 替换全部参数
3. 解析内联 shell（\`command\`）并执行，结果替换回模板
4. 确定 task model（命令指定 > agent 默认 > 用户指定 > session 当前）
5. 如果是 subagent 模式 → 包装成 subtask part；否则展开为文本 parts
6. 调 `prompt()` 发起对话
7. 发布 `Command.Event.Executed` 事件

#### 8. `finalizeInterruptedAssistant` (1256行) — 中断收尾

被 `Effect.onInterrupt` 包裹（1272, 1381行）。如果 assistant message 尚未 `time.completed`，标记：
- `error: AbortError("Aborted")`
- `aborted: true`
- `time.completed = Date.now()`

### 完整调用链

```
用户输入
  → command(input)                      // /xxx 命令
    → prompt(input)
      ┆
  → prompt(input)                       // 普通对话
    → createUserMessage(input)          // 解析 parts，持久化到 DB
    → loop({ sessionID })
      → state.ensureRunning(sessionID, onInterrupt, runLoop)
        → runLoop(sessionID)            // 核心 while(true)
          → 加载消息历史
          → 检查退出条件
          → handleSubtask / compaction  // 处理任务栈
          → processor.create()          // 创建流处理器
          → SessionTools.resolve()      // 组装工具列表
          → handle.process()            // 调 LLM.stream() + 执行 tool calls
          → break / compact / continue
```

### 当前状态

这是 **V1 生产代码**，仍在所有 TUI/CLI/Server 路径中使用。`SessionV2.prompt`（`packages/core/src/session.ts`）是新的 event-sourcing 架构，仅测试中使用。两者并行，V2 在逐步迁移中。

---

## V2 SessionRunner 详解（`packages/core/src/session/runner/`）

这是 V2 的"一次 LLM 对话轮次"引擎，5 个文件各司其职：

### 文件总览

| 文件 | 行数 | 角色 |
|------|------|------|
| `index.ts` | 39 | 对外接口：`run({ sessionID, force? })` + `RunError` 类型 + `Service` 定义 |
| `model.ts` | 147 | 模型选择器：Session model ref → LLM 包的 `Model` 对象 |
| `to-llm-message.ts` | 149 | 消息翻译器：V2 `SessionMessage` → LLM 包的 `Message[]` |
| `publish-llm-event.ts` | 411 | 流事件持久化器：LLM 流式事件 → `SessionEvent` 写入 EventV2 |
| `llm.ts` | 404 | 核心编排器：`run()` → `runTurn()` → `runTurnAttempt()` 三层结构 |

### 1. `model.ts` — 模型选择器

把 Session 的 model 引用解析成 LLM 包能用的 `Model` 对象：
- 从 Catalog 查元信息（API 类型、endpoint、apiKey）
- 按 `api.package` 选协议路由：`@ai-sdk/openai` → `OpenAIResponses`，`@ai-sdk/anthropic` → `AnthropicMessages`，`@ai-sdk/openai-compatible` → `OpenAICompatibleChat`
- 应用 variant（如 `o3-mini` → `o3-mini-high`）
- 组装 auth（环境变量或 token）、headers、generation controls、context limits
- Session 没选模型时 → 取 default → 取第一个 supported

### 2. `to-llm-message.ts` — 消息翻译器

将 V2 的 8 种消息类型转为 LLM 协议的 `Message[]`：

| V2 类型 | 输出 |
|---------|------|
| `user` | user role：text + files 转 media content + agents 转 metadata |
| `assistant` | assistant role（text + reasoning + tool-call parts）+ N 个 tool role（tool results） |
| `system` | system role |
| `synthetic` | user role |
| `shell` | user role：`"Shell command: xxx\n\noutput"` |
| `compaction` | user role：`<conversation-checkpoint>` 包裹 summary + recent context |
| `agent-switched` / `model-switched` | **丢弃**（切换事件不进 LLM 上下文） |

关键点：assistant 消息中的 tool call 和 tool result 被拆成两份——call 留在 assistant content，result 作为独立 tool role 消息。

### 3. `publish-llm-event.ts` — 流事件持久化器

将 LLM 流式事件逐条翻译成 `SessionEvent` 并发布到 EventV2。内部维护一个 tool 状态表（`Map<callID, tool状态>`）跟踪生命周期。

核心事件映射：

| LLMEvent | SessionEvent | 说明 |
|----------|-------------|------|
| `text-start/delta/end` | `Text.Started/Delta/Ended` | 文本增量持久化 |
| `reasoning-start/delta/end` | `Reasoning.Started/Delta/Ended` | 推理过程持久化 |
| `tool-input-start/delta/end` | `Tool.Input.Started/Delta/Ended` | tool 参数流式收集 |
| `tool-call` | `Tool.Called` | tool 调用触发 |
| `tool-result` | `Tool.Success` / `Tool.Failed` | tool 结果（成功/失败） |
| `step-finish` | `Step.Ended` | 本轮完成 + token 统计 |
| `provider-error` | `Step.Failed` | provider 错误标记 |

`fragments()` 工具函数封装了流式数据的 start→delta→end 三段式生命周期管理。`failUnsettledTools()` 在中断时批量标记未完成的 tool 为失败。

### 4. `llm.ts` — 核心编排器

三层调用结构：

```
run({ sessionID, force? })
  │
  ├─ 检查 inbox 是否有 steer/queue 输入
  ├─ failInterruptedTools(sessionID)        // 先清理上次中断遗留的 tool
  │
  └─ while (有新 activity):
       for step < 25:                        // 最多 25 步
         runTurn(sessionID, promotion)
           │
           └─ runTurnAttempt(sessionID, promotion, compaction)
               ├─ 加载 System Context（ContextEpoch.initialize 或 prepare）
               ├─ promote inbox 输入 (steer / queue)
               ├─ 组装 LLM request（system baseline + messages + tool definitions）
               ├─ 检查 overflow → 需要则 compaction
               ├─ llm.stream(request)       // 流式调用 LLM
               ├─ 边收流边 publish event
               ├─ 收到 tool-call → 立即 settle（执行 tool 并发布结果）
               ├─ 流结束 → 处理中断/失败/成功
               └─ 返回 needsContinuation     // 是否还要继续
```

关键设计：

- **`TurnTransitionError`** — 一种"控制流 Defect"。agent/model 被并发修改时抛 `RebuildPreparedTurn` 让外层重试；overflow compaction 完成后抛 `ContinueAfterOverflowCompaction`。通过 `catchDefect` 捕获，不污染正常错误通道。
- **`retryAgentMismatch`** — ContextEpoch 检测到 agent 不匹配 → 转成 `RebuildPreparedTurn` 重试。
- **每轮 step 后重新检查 steer inbox** — 有新的 steer 输入直接进下一轮，不结束 activity。
- **overflow 恢复** — provider 因 context 溢出失败（且尚未产出内容）时，自动 compaction 后重试。

---

## 上下文系统详解：上下文是怎么变化的

OpenCode 的上下文分为**三层**，每层有自己的生命周期和更新机制。

### 三层上下文的架构

```
┌─────────────────────────────────────────────────────┐
│ 1. System Context（系统上下文）                        │
│    date / environment / AGENTS.md / skills / plugins  │
│    刷新策略：每个 provider-turn 边界 reconcile          │
├─────────────────────────────────────────────────────┤
│ 2. Ephemeral Context（对话上下文）                     │
│    user / assistant / tool results / system messages  │
│    刷新策略：每轮对话累加，compaction 时压缩旧消息       │
├─────────────────────────────────────────────────────┤
│ 3. Compaction Summary（压缩摘要）                      │
│    旧消息的结构化摘要（Goal / Progress / Decisions）     │
│    刷新策略：溢出时增量更新                             │
└─────────────────────────────────────────────────────┘
```

### 第一层：System Context（系统上下文）

**定义位置**：`packages/core/src/system-context/index.ts`

这是**独立可刷新的类型化上下文源**的代数组合系统。每个 source 定义三件事：

```ts
interface Source<A> {
  key: Key              // 唯一标识，如 "core/environment"
  load: Effect<A>       // 观察当前值
  baseline: (A) => string    // 首次注入模型的文本
  update: (prev, curr) => string  // 值变化后的更新文本
  removed?: (prev) => string     // 源被移除时的通知文本
}
```

**内置源**（`builtins.ts`）：
| Source Key | 内容 |
|------------|------|
| `core/environment` | 工作目录、workspace 根、git 状态、操作系统 |
| `core/date` | 当天日期 |

**扩展源**（通过 plugin 注册到 `SystemContextRegistry`）：
- `InstructionContext`（`instruction-context`）— 从 `AGENTS.md` / `CONTEXT.md` 加载的项目指引
- `SkillGuidance` — 匹配当前 agent 的 skill 说明
- `ReferenceGuidance` — 项目引用的额外文档

**刷新流程**（`context-epoch.ts:67-110`）：

```
每个 provider-turn:
  1. SystemContextRegistry.load()   → 并发加载所有 source
  2. SystemContext.combine()        → 合并为一个统一 context
  3. ContextEpoch.prepare()         → 对照数据库中的上次快照:
  
     情况 A: 首次初始化 (initialize)
       └─ observe → baseline 文本 → 存入 ContextEpochTable
  
     情况 B: agent 没变且无 pending replacement (reconcile)
       ├─ 逐个 source 与上次快照 compare
       ├─ 没变的 → 跳过
       ├─ 变了的 → render update 文本 → 发布 SessionEvent.ContextUpdated（作为 system message 注入对话）
       └─ 移除的 → render removed 文本 → 同上
  
     情况 C: agent 变了 或 数据格式不兼容 (replace)
       └─ 全部重建 → 新的 baseline 文本 → 新 baseline_seq
  
     情况 D: 有 source unavailable (block)
       └─ 等待，不生成不完整的 baseline
```

**关键设计**：
- **首次注入 baseline**（全量）→ 后续只注入 **update 增量** → 节省 token
- **revision 乐观锁** — 检查 agent/model 是否被并发修改，被改了退避重试
- **baseline 位置** — 在 LLM request 中作为 `system` 部分（`llm.ts:222`）

### 第二层：Ephemeral Context（对话上下文）

**即 Session 消息历史**，每次对话累加。消息类型（`message.ts`）：

| 类型 | 来源 | LLM 可见？ |
|------|------|-----------|
| `user` | 用户输入 → inbox → promote | ✓ |
| `assistant` | LLM 返回 | ✓ |
| `system` | ContextUpdated 事件（System Context 更新） | ✓ |
| `synthetic` | 插件注入的合成上下文 | ✓ |
| `shell` | 用户执行 shell 命令 | ✓ |
| `compaction` | 压缩摘要 | ✓（作为 checkpoint） |
| `agent-switched` | agent 切换事件 | ✗（不进 LLM） |
| `model-switched` | model 切换事件 | ✗（不进 LLM） |

**消息加载**（`history.ts`）：
```
SessionHistory.load(sessionID):
  1. 查最新 compaction 消息
  2. 查 ContextEpoch.baseline_seq
  3. 从 SessionMessageTable 取消息:
     - 如果 compaction 存在 → 只取 compaction 之后的
     - baseline_seq 前的 system 消息被排除（属于旧 epoch）
```

### 第三层：Compaction Summary（压缩摘要）

**触发条件**（`compaction.ts:230-238`）：
```
当前 token 数 > 模型 context 窗口 - max(output_tokens, 20000 buffer)
```

**压缩流程**：
1. **split** — 消息列表从后往前扫描，保留最后 `8000 tokens` 的消息作为 `recent`（原样保留）
2. **summarize** — 旧消息 `head` → 调 LLM 生成结构化摘要（Goal / Progress / Key Decisions / Next Steps / Critical Context / Relevant Files）
3. **persist** — 摘要 + recent 作为 `type: "compaction"` 消息持久化
4. **replace** — 后续对话中，旧消息不再计入 token，只有摘要 + recent 保留

**增量更新**：如果已有旧摘要，传给它 `<previous-summary>` 让它更新合并，而非从零重写。

### 一个 provider-turn 的完整上下文变化流程

```
1. System Context 刷新
   ┌──────────────────────────────────────────┐
   │ Registry.load() → combine → ContextEpoch │
   │   ↓                                      │
   │ reconcile(上次快照, 当前值)               │
   │   ├─ 有变化 → publish ContextUpdated     │
   │   │          → 作为 system message 注入  │
   │   └─ 无变化 → skip                       │
   └──────────────────────────────────────────┘

2. Inbox Promote（第193-199行）
   ┌──────────────────────────────────────────┐
   │ SessionInput.promoteSteers(sessionID)    │
   │   → admit 的 user 输入提升为可见消息       │
   └──────────────────────────────────────────┘

3. 加载历史 + 组装 Request
   ┌──────────────────────────────────────────┐
   │ SessionHistory.entriesForRunner(         │
   │   sessionID, baselineSeq                 │
   │ )                                        │
   │   → 过滤掉旧 epoch + compaction 前消息    │
   │   → toLLMMessages() 翻译成 LLM 格式       │
   │                                          │
   │ system = [agent.system, epoch.baseline]  │
   │ messages = 翻译后的历史消息               │
   │ tools = ToolRegistry 物化后的定义          │
   └──────────────────────────────────────────┘

4. Overflow Check
   ┌──────────────────────────────────────────┐
   │ 如果 request tokens > context - buffer   │
   │   → compactAfterOverflow()               │
   │   → 压缩后 rebuild 整个 request          │
   └──────────────────────────────────────────┘

5. LLM 调用 + 工具执行
   ┌──────────────────────────────────────────┐
   │ llm.stream(request)                      │
   │   ├─ 流式文本 → publish Text.Delta       │
   │   ├─ tool-call → settle() 立即执行       │
   │   └─ tool result → publish Tool.Success  │
   │                                          │
   │ 有 tool call → needsContinuation = true  │
   │   → 下轮 while 继续                      │
   │ 无 tool call 且 finish → 本轮结束         │
   └──────────────────────────────────────────┘
```

---

## Agent 系统详解

**定义位置**：`packages/core/src/agent.ts`

### Agent 是什么

一个 Agent 是一个**对话配置模板**，包含：

```ts
class Info {
  id: ID                    // 唯一标识，如 "build", "plan", "compaction"
  model: ModelRef?          // 指定模型（可选，不指定则用 session 当前）
  request: Request          // provider 级别的额外参数（headers, body）
  system: string?           // 自定义 system prompt（追加在 baseline 之后）
  description: string?      // 描述文本
  mode: "subagent" | "primary" | "all"  // 运行模式
  hidden: boolean           // 是否对用户隐藏
  color: string?            // TUI 显示颜色
  steps: number?            // 最大对话步数限制
  permissions: Rule[]       // 工具权限规则（deny/allow）
}
```

### 三种运行模式

| mode | 含义 | 用途 |
|------|------|------|
| `primary` | 仅作为顶层对话 agent | 用户直接对话 |
| `subagent` | 仅作为子任务 agent | 被 `task` tool 内部调用 |
| `all` | 两种场景都可用 | 默认 `build` agent |

### Agent 选择流程

```
SessionPrompt.prompt(input) 或 SessionRunner.run(...)
  ↓
AgentV2.select(agentID?)
  ├─ 有指定 → 按 ID 查找
  └─ 无指定 → selectedDefault():
       ├─ 用户配的 default agent（且 mode !== "subagent" && !hidden）
       ├─ "build" agent（内置默认）
       └─ 第一个可用的 agent
```

### Agent 对上下文的影响

```
loadSystemContext(agent):
  SystemContextRegistry.load()      // 通用上下文源
  + SkillGuidance.load(agent)       // agent 专属 skill 说明
  + ReferenceGuidance.load()        // 项目引用文档
  → SystemContext.combine()         // 合并
```

不同 agent 可能触发不同的 skill guidance，所以**切换 agent 会触发 System Context 的 replace**（context-epoch.ts:85-93）——旧 epoch 的 snapshot 与新 agent 可能不兼容，需要全部重建 baseline。

### Subagent（子任务）执行机制

```ts
// V1 路径 (prompt.ts:239-290)
handleSubtask({ task, model, ... })
  1. 创建新的 assistant message（嵌入 tool part）
  2. tool part 类型为 TaskTool
  3. taskTool.execute() 递归调用 prompt() → 开新对话（子 agent）
  4. 结果回写 tool part（completed / error / cancelled）
```

子 agent 有独立的对话上下文、独立的消息历史、独立的 step 计数限制。主对话在子 agent 完成后继续。

### Agent 与权限

每个 agent 可配 `permissions: Rule[]`，格式为 `{ permission: string, action: "deny" | "allow" }`。在 `ToolRegistry.materialize()` 时应用规则过滤，决定哪些 tool 对当前 agent 可见。

### Agent 与 Compaction

`compaction` agent 是一个特殊的隐藏 agent（`mode: "subagent"`, `hidden: true`），它有独立的权限规则（deny 所有 tool），用于执行对话压缩的 LLM 调用（`compaction.ts:338`）。可以给它配置一个便宜的模型来省钱。

---

### 与 V1 `runLoop` 对比

| | V1 `runLoop` (prompt.ts) | V2 `runner/` |
|---|---|---|
| 结构 | 一个巨大 while(true) 内联所有逻辑 | 分层：`run`→`runTurn`→`runTurnAttempt` + 独立 collaborator |
| 消息加载 | 直接读 SessionMessageTable | `SessionHistory.entriesForRunner()` 通过 event projection |
| 上下文 | 直接调 `sys.skills()` / `sys.environment()` | `SystemContextRegistry.load()` + `SkillGuidance` + `ReferenceGuidance` |
| tool 执行 | `SessionTools.resolve()` 组装 AI SDK tool 对象 | `ToolRegistry.materialize()` → `settle()` 在事件流中执行 |
| 中断 | `Effect.onInterrupt` + `finalizeInterruptedAssistant` | `FiberSet.clear` + `failUnsettledTools` |
| 并发保护 | 无（V1 同一 Session 只有一个 runLoop） | `TurnTransitionError` 检测变更后主动重试 |
| 状态持久化 | 直接写 `sessions.updateMessage/updatePart` | 全部通过 `events.publish(SessionEvent.xxx)` 事件溯源 |
