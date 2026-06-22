# OpenCode 代码学习方法

## 总体策略：从概念到代码，从核心到边缘

不要试图从 `main()` 线性阅读。OpenCode 是一个分层系统，先理解概念模型，再进入代码细节。

---

## 第一阶段：理解概念模型（约 1 小时）

### 必读文档
1. **`CONTEXT.md`** — 整个系统的领域术语字典。理解 System Context、Context Source、Context Epoch、Session History、Safe Provider-Turn Boundary 等核心概念。
2. **`AGENTS.md`** — 代码风格规范、Effect 使用规则、V2 Session Core 设计原则。
3. **`specs/v2/`** 下的各个 spec — session、tools、provider 等模块的设计文档。

### 核心概念速记
- **Session** = 一个对话的持久化载体，有 ID、Location、agent、model
- **Prompt** = 用户输入的文本 + 附件 + agent 引用
- **Steer vs Queue** = 两种投递语义：steer 合并到当前 activity，queue 打开新的 FIFO activity
- **Safe Provider-Turn Boundary** = provider 调用前的安全点，在此处 admit context 变更
- **Context Epoch** = 两段不变 baseline system context 之间的区间，由 compaction 或 model switch 触发

---

## 第二阶段：理解执行主循环（约 2-3 小时）

**阅读顺序：**

```
SessionExecution.resume(sessionID)
  → SessionRunCoordinator.run(sessionID)       # 协调并发、防止重入
    → SessionRunner.run({ sessionID })          # 加载历史 → 组装上下文 → 调用 LLM → 处理 tool calls
```

### 关键文件阅读路线

1. `packages/core/src/session/execution.ts` — 3 个 API：resume / wake / interrupt
2. `packages/core/src/session/run-coordinator.ts` — 单个 Session 的排他 drain chain
3. `packages/core/src/session/runner/index.ts` — `SessionRunner.Interface.run()` 的定义
4. `packages/opencode/src/session/prompt.ts` — 旧版大循环（1700行），是整个执行流程的地图

### 学习技巧
- 跟踪一次完整的 prompt → tool settlement → next turn 循环
- 在 `SessionPrompt` 中找 `loop` 函数的定义（在旧版代码中）或跟踪新版 `SessionRunner` 的实现
- 标注每个 `yield*` 在等待什么服务

---

## 第三阶段：理解 System Context（约 1-2 小时）

### 核心文件
- `packages/core/src/system-context/index.ts` — 310 行，整个代数的实现
- `packages/core/src/system-context/builtins.ts` — 内置源（日期、指令等）
- `packages/core/src/system-context/registry.ts` — 注册中心

### 学习技巧
- 画图理解：`Source<A>.make()` → opaque `SystemContext` → `combine()` → `initialize()` / `reconcile()`
- `initialize()` 产出 immutable baseline，`reconcile()` 比对变化产出 update text
- 理解 `unavailable` 语义：临时不可用 ≠ 移除

---

## 第四阶段：理解 Tool 系统（约 1-2 小时）

### 核心文件
- `packages/core/src/tool/tool.ts` — `Tool.make()` 的定义模式
- `packages/core/src/tool/registry.ts` — 注册 + 物化 + 结算
- `packages/core/src/tool/builtins.ts` — 内置工具声明
- `packages/opencode/src/tool/registry.ts` — 所有工具的实现注册

### 学习技巧
- 跟踪一个具体工具的完整流程：Bash / Read / Edit / Grep
- 理解 `definition`（给 LLM）、`settle`（执行）、`ToolOutput.bound`（截断）三阶段

---

## 第五阶段：理解数据持久化（约 1 小时）

### 核心文件
- `packages/core/src/session/sql.ts` — Drizzle schema
- `packages/core/src/session/input.ts` — Input admit/promote 的持久化
- `packages/core/src/session/event.ts` — Event 类型定义
- `packages/core/src/session/message.ts` — Message 类型定义

### 学习技巧
- SQLite 是唯一数据库，所有持久化通过 Drizzle
- V2 使用 event sourcing 模式：先写 event 再 project 到表
- `SessionInputTable` 的 `promoted_seq` 为 null 表示尚未 promote

---

## 读书的工具方法

### 1. 用 Grep 定位入口点

```bash
# 找某个概念的首次使用
grep -r "SessionExecution" --include="*.ts" packages/

# 找服务注册点
grep -r "Layer.effect" --include="*.ts" packages/core/src/
```

### 2. 跟踪 Import 链

从外部接入点反向追踪 import 关系，理解依赖方向：
- `packages/opencode/src/` 依赖 `packages/core/src/`
- `packages/cli/src/` 依赖 `packages/opencode/src/` + `packages/core/src/`
- `packages/core/src/` 不依赖其他 package（仅依赖 `@opencode-ai/llm`）

### 3. 关注 Effect 服务（Context.Service）

搜索 `class Service extends Context.Service` 是找到所有核心服务的捷径：
```
SessionExecution.Service
SessionRunner.Service
SessionRunCoordinator.Service
ToolRegistry.Service
Tools.Service
ApplicationTools.Service
ToolOutputStore.Service
PermissionV2.Service
SystemContext.Service
...
```

### 4. 阅读 Self-Export 模块的方式

文件顶部 `export * as Foo from "."` 意味着整个文件的导出都通过 `Foo.xxx` 访问。阅读时：
- 先看 `export` 开头的公共 API
- 内部 helper 都是非导出的函数/类型
- 消费代码写 `import { Foo } from "@/foo"` 然后用 `Foo.Service`

### 5. 跟踪一次完整的执行

用 debugger 或加 log 跟踪一次 `opencode dev` 的执行路径：
1. CLI 入口 → 命令分发
2. TUI 初始化
3. 用户输入 prompt
4. Prompt admit → promote
5. SessionRunner.run()
6. LLM stream → tool call → settle → output → next turn
7. Context reconcile at safe boundary

---

## 重要 Spec 文件

| 文件 | 用途 |
|------|------|
| `specs/v2/session.md` | Session V2 设计 |
| `specs/v2/tools.md` | 工具系统设计 |
| `specs/v2/provider-model.md` | Provider/Model 设计 |
| `specs/v2/config.md` | 配置系统设计 |
| `specs/v2/catalog-config-plugin-lifecycle.md` | Catalog/Config/Plugin 生命周期 |
| `specs/project.md` | 多项目/多 worktree 设计 |
| `specs/v2/schema-changelog.md` | Schema 版本变更 |
| `CONTEXT.md` | 领域术语字典 |
| `AGENTS.md` | 代码规范 + 贡献指南 |

---

## 常见问题

### V2 和 V1 的区别？
V2 是新的 event-sourced Session 核心，V1 是旧版。`packages/core/src/v1/` 和 `packages/core/src/v2-schema.ts` 是遗留/兼容层。新代码应该只看非 v1 路径。

### 为什么 core 和 opencode 要分两个包？
- **core** — 纯架构/持久化：Session Schema、System Context algebra、Tool Registry、EventV2
- **opencode** — 具体业务实现：Prompt loop、Tool 实现、Agent、Config、Compaction logic

这种分离使得架构层可以独立演进，不受具体工具/业务变化的影响。
