# 第二章：Agentic AI 架构与多智能体协同范式

## 2.1 单智能体：一个 Agent 到底是怎么工作的？

在讨论多 Agent 之前，必须先搞清楚**一个 Agent 的内部运作机制**。所谓 Agent，本质上就是一个**围绕 LLM 构建的自主决策循环**——它不只是"问一次答一次"的聊天机器人，而是能够**感知环境、制定计划、调用工具、观察结果、持续迭代**直到任务完成的自治系统。

### 2.1.1 核心循环：ReAct（Reasoning + Acting）

几乎所有生产级 Agent 都基于 ReAct 模式运行。其核心是一个 `while(true)` 循环：

```
用户输入
  │
  ▼
┌─────────────────────────────┐
│  LLM 推理（Reasoning）       │ ← 分析当前状态，决定下一步
│  ↓                          │
│  选择工具 & 生成参数          │ ← tool_call: read_file("src/main.ts")
│  ↓                          │
│  执行工具（Acting）           │ ← 实际读取文件、运行命令、搜索代码
│  ↓                          │
│  观察结果（Observation）      │ ← 工具返回的内容注入上下文
│  ↓                          │
│  判断：任务完成了吗？          │
│    ├─ 否 → 回到顶部继续推理   │
│    └─ 是 → 输出最终回答       │
└─────────────────────────────┘
```

这个循环看似简单，但工程实现中有大量细节。以开源项目 **OpenCode**（一个终端 AI 编码 Agent）的 `processor.ts` 为例，它的核心处理逻辑就是这样一个循环：

```typescript
// OpenCode: packages/opencode/src/session/processor.ts（简化）
async process(streamInput) {
  while (true) {
    const stream = await LLM.stream(streamInput)  // 调用 LLM

    for await (const value of stream.fullStream) {
      switch (value.type) {
        case "tool-call":
          // LLM 决定调用某个工具，执行它
          break
        case "tool-result":
          // 工具执行完毕，结果注入上下文
          break
        case "finish-step":
          // 一轮推理结束，检查是否还有待执行的工具调用
          // 如果有 → 继续循环；如果没有 → 退出
          break
      }
    }
    // 没有更多工具调用 → 任务完成，退出循环
    if (Object.keys(toolcalls).length === 0) break
  }
}
```

**关键洞察**：Agent 的"智能"不在于单次 LLM 调用有多强，而在于这个**循环能跑多少轮、每轮能做出多好的决策**。一个写代码的 Agent 可能需要：读文件 → 理解结构 → 写代码 → 运行测试 → 看报错 → 修 bug → 再测试，经历 5-10 轮循环才能完成一个任务。

### 2.1.2 工具系统：Agent 的"手和脚"

LLM 本身只能生成文本。要让它真正"做事"，需要给它一套**工具（Tools）**。工具的本质是：LLM 输出一个结构化的函数调用请求，运行时环境执行这个函数，再把结果喂回 LLM。

典型的编码 Agent 工具集包括：

| 工具 | 功能 | 示例 |
|------|------|------|
| `read` | 读取文件内容 | `read("src/index.ts")` |
| `write` | 创建或覆盖文件 | `write("src/utils.ts", content)` |
| `edit` | 精确编辑文件片段 | `edit("src/main.ts", old, new)` |
| `exec` | 执行 Shell 命令 | `exec("npm test")` |
| `grep` | 搜索代码内容 | `grep("TODO", "*.ts")` |
| `web_search` | 搜索互联网 | `web_search("React 19 new features")` |

以 OpenClaw（一个多平台 AI Agent 平台）为例，它的 `system-prompt.ts` 中定义了核心工具摘要：

```typescript
// OpenClaw: src/agents/system-prompt.ts（简化）
const coreToolSummaries = {
  read: "Read file contents",
  write: "Create or overwrite files",
  edit: "Make precise edits to files",
  exec: "Run shell commands",
  grep: "Search file contents for patterns",
  web_search: "Search the web",
  web_fetch: "Fetch content from a URL",
  // ...
}
```

### 2.1.3 系统提示词：Agent 的"人格与规则"

系统提示词（System Prompt）决定了 Agent 的行为边界。它不是简单的"你是一个助手"，而是一份详细的**行为规范文档**，通常包括：

- **角色定义**：你是谁，擅长什么
- **工具使用规则**：什么时候该用什么工具，怎么用
- **输出格式约束**：回答的结构、代码风格要求
- **安全边界**：哪些操作需要用户确认，哪些绝对禁止

OpenCode 的做法很有代表性——它会**根据底层模型动态选择不同的系统提示词**：

```typescript
// OpenCode: packages/opencode/src/session/system.ts
export function provider(model: Provider.Model) {
  if (model.api.id.includes("gpt-5")) return [PROMPT_CODEX]
  if (model.api.id.includes("gemini-")) return [PROMPT_GEMINI]
  if (model.api.id.includes("claude")) return [PROMPT_ANTHROPIC]
  return [PROMPT_ANTHROPIC_WITHOUT_TODO]  // 默认 fallback
}
```

这是因为不同模型对提示词的响应方式不同——Claude 更擅长遵循结构化指令，GPT 系列对 few-shot 示例更敏感，Gemini 对 XML 标签有特殊优化。**好的 Agent 框架会针对模型特性做适配，而不是一套 prompt 打天下**。

### 2.1.4 防死循环：Agent 的"安全阀"

Agent 的自主循环带来一个严重风险：**死循环**。LLM 可能反复调用同一个工具、得到同样的结果、却不知道换个策略。这在生产环境中会烧掉大量 Token 和时间。

**OpenCode 的方案——Doom Loop 检测**：

```typescript
// OpenCode: packages/opencode/src/session/processor.ts
const DOOM_LOOP_THRESHOLD = 3

// 检查最近 3 次工具调用是否完全相同
const lastThree = parts.slice(-DOOM_LOOP_THRESHOLD)
if (
  lastThree.length === DOOM_LOOP_THRESHOLD &&
  lastThree.every(p =>
    p.tool === value.toolName &&
    JSON.stringify(p.state.input) === JSON.stringify(value.input)
  )
) {
  // 触发权限询问，让用户决定是否继续
  await PermissionNext.ask({ permission: "doom_loop", ... })
}
```

**OpenClaw 的方案——多层检测器**：

OpenClaw 的 `tool-loop-detection.ts` 实现了更精细的四层检测：

| 检测器 | 触发条件 | 处理方式 |
|--------|----------|----------|
| `generic_repeat` | 连续 N 次相同工具+相同参数 | 警告 → 强制中断 |
| `known_poll_no_progress` | 轮询类工具结果无变化 | 提示换策略 |
| `ping_pong` | 两个工具交替调用无进展 | 检测 A→B→A→B 模式 |
| `global_circuit_breaker` | 总工具调用次数超限（默认 30 次） | 强制终止 |

```typescript
// OpenClaw: src/agents/tool-loop-detection.ts
export const WARNING_THRESHOLD = 10
export const CRITICAL_THRESHOLD = 20
export const GLOBAL_CIRCUIT_BREAKER_THRESHOLD = 30
```

---

## 2.2 从单 Agent 到多 Agent：为什么需要"团队作战"？

单 Agent 在处理明确、短链路任务时表现优异，但面对复杂场景时会遭遇三大瓶颈：

1. **上下文超载**：一个 Prompt 塞入过多角色设定与工具定义，导致注意力分散（Lost in the Middle 问题）。
2. **能力耦合**：让同一个 Agent 既写代码、又做代码审查、还搜索文档，容易引发指令冲突。
3. **权限失控**：一个全能 Agent 拥有所有工具的访问权限，安全风险极高。

**多智能体的本质**：通过**角色解耦**与**权限隔离**，将复杂问题拆解为多个子任务，由专用 Agent 各司其职。

---

## 2.3 多智能体架构范式（速览）

业界主流的多 Agent 架构可以归纳为三种模式：

### 模式一：Supervisor（中央调度型）

```
         ┌────────────┐
         │ Supervisor  │ ← 接收任务、拆解、分发、合成结果
         └─────┬──────┘
        ┌──────┼──────┐
   ┌────┴──┐ ┌┴────┐ ┌┴────┐
   │Worker A│ │Worker B│ │Worker C│
   └───────┘ └──────┘ └──────┘
```

一个"项目经理" Agent 负责理解用户意图、拆解子任务、分发给专业 Worker、收集结果后合成最终答案。**控制力强，但 Supervisor 是单点瓶颈**。

### 模式二：Peer-to-Peer（对等协作型）

```
   Agent A ⇄ Agent B
     ↕           ↕
   Agent C ⇄ Agent D
```

Agent 之间直接通信，无中央节点。**灵活但容易失控**，适合开放探索类任务。

### 模式三：Hierarchical（层级型）

```
   主 Agent
    ├── 子 Agent 组 1（规划）
    ├── 子 Agent 组 2（执行）
    └── 子 Agent 组 3（验证）
```

模仿组织架构，高层负责战略，基层负责执行。**可扩展性最强，但架构复杂度高**。

> **工程实践结论**：绝大多数生产系统采用 **Supervisor + 有限层级** 的混合架构。纯对等模式难以管控成本，纯中心化缺乏弹性。下面我们通过两个真实开源项目来看具体是怎么做的。

---

## 2.4 实战案例一：OpenCode 的多 Agent 体系

[OpenCode](https://github.com/anomalyco/opencode) 是一个开源的终端 AI 编码 Agent，采用了**主 Agent + 专业子 Agent** 的 Supervisor 架构。

### 2.4.1 Agent 角色定义

OpenCode 在 `agent.ts` 中定义了 7 个 Agent，按 `mode` 分为两类：

```typescript
// OpenCode: packages/opencode/src/agent/agent.ts（简化）
const agents = {
  // ===== 主 Agent（mode: "primary"）=====
  build: {
    name: "build",
    description: "默认全功能 Agent，可执行所有工具",
    mode: "primary",
  },
  plan: {
    name: "plan",
    description: "只读模式，禁止编辑工具，用于分析和规划",
    mode: "primary",
    permission: { edit: { "*": "deny" } },  // 关键：禁止所有编辑操作
  },

  // ===== 子 Agent（mode: "subagent"）=====
  general: {
    name: "general",
    description: "通用子 Agent，用于复杂搜索和多步骤任务",
    mode: "subagent",
  },
  explore: {
    name: "explore",
    description: "代码库探索专用，只允许只读工具",
    mode: "subagent",
    permission: {
      "*": "deny",           // 默认禁止所有
      grep: "allow",         // 只开放搜索类工具
      glob: "allow",
      read: "allow",
      bash: "allow",
    },
  },
}
```

**架构图**：

```
用户 ──→ build Agent（默认）──→ 直接处理大部分任务
           │
           ├── @general ──→ general Agent（并行处理复杂子任务）
           │
           └── @explore ──→ explore Agent（只读探索代码库）

用户 ──→ plan Agent（Tab 切换）──→ 只读分析，不修改任何文件
```

### 2.4.2 权限隔离：每个 Agent 有自己的"能力边界"

OpenCode 最精妙的设计之一是**基于权限规则集的能力隔离**。每个 Agent 不是简单地"能用/不能用"某个工具，而是通过 glob 模式精细控制：

```typescript
// plan Agent 的权限：可以读，但只能写到特定目录
plan: {
  permission: {
    edit: {
      "*": "deny",                                    // 禁止编辑所有文件
      ".opencode/plans/*.md": "allow",                // 但允许写规划文档
    },
    question: "allow",   // 允许向用户提问
    plan_exit: "allow",  // 允许退出 plan 模式
  },
}

// explore Agent 的权限：极度收窄，只保留搜索能力
explore: {
  permission: {
    "*": "deny",
    grep: "allow",
    glob: "allow",
    list: "allow",
    read: "allow",
    bash: "allow",        // 允许执行命令（用于 find、cat 等）
    codesearch: "allow",
    websearch: "allow",
  },
}
```

**为什么这样设计？**
- `build` 是"全栈工程师"，什么都能干
- `plan` 是"架构师"，只看不动手，避免误操作
- `explore` 是"代码审计员"，只搜索不修改，适合快速了解陌生代码库
- `general` 是"研究助理"，可以并行处理多个搜索任务

### 2.4.3 子 Agent 调用机制

用户在消息中使用 `@general` 即可调用子 Agent。主 Agent 会将任务委派给子 Agent，子 Agent 独立运行自己的 ReAct 循环，完成后将结果返回主 Agent。

```
build Agent 收到用户消息: "重构 auth 模块，先了解现有实现"
  │
  ├── 判断：需要先探索代码库
  │
  ├── 调用 @explore: "搜索所有 auth 相关的文件和函数"
  │     └── explore Agent 独立运行：
  │           grep("auth", "*.ts") → 找到 5 个文件
  │           read("src/auth/index.ts") → 理解入口
  │           read("src/auth/jwt.ts") → 理解 JWT 逻辑
  │           返回结构化摘要给 build Agent
  │
  └── build Agent 基于探索结果，开始编写重构代码
```

---

## 2.5 实战案例二：OpenClaw 的子 Agent 生态

[OpenClaw](https://github.com/openclaw/openclaw) 是一个更复杂的多平台 AI Agent 平台，支持 CLI、Web、Discord、Telegram、Slack 等多种渠道。它实现了一套**完整的子 Agent 生命周期管理系统**，是目前开源社区中多 Agent 工程化做得最深入的项目之一。

### 2.5.1 子 Agent 的完整生命周期

OpenClaw 的子 Agent 系统由以下核心模块组成：

```
src/agents/
  ├── subagent-spawn.ts          # 创建子 Agent
  ├── subagent-registry.ts       # 注册表：跟踪所有活跃的子 Agent
  ├── subagent-announce.ts       # 完成通知：子 Agent 完成后通知父 Agent
  ├── subagent-depth.ts          # 嵌套深度控制
  ├── subagent-lifecycle-events.ts  # 生命周期事件
  └── tool-loop-detection.ts     # 防死循环
```

**生命周期流程**：

```
父 Agent 调用 spawn_subagent(task, agentId)
  │
  ▼
① 创建（Spawn）
  ├── 验证 agentId 合法性
  ├── 检查嵌套深度是否超限
  ├── 选择运行模式（run / session）
  ├── 构建子 Agent 专用的 System Prompt（minimal 模式）
  └── 注册到 subagent-registry
  │
  ▼
② 执行（Run）
  ├── 子 Agent 独立运行 ReAct 循环
  ├── 拥有自己的工具集和权限边界
  └── 可以继续 spawn 孙子 Agent（受深度限制）
  │
  ▼
③ 通知（Announce）
  ├── 子 Agent 完成后，通过 announce 队列通知父 Agent
  ├── 支持重试机制（瞬态错误自动重试 3 次）
  └── 父 Agent 收到结果后继续自己的推理循环
  │
  ▼
④ 清理（Cleanup）
  ├── 从 registry 中注销
  └── 根据配置决定是否删除会话记录
```

### 2.5.2 子 Agent 创建：spawn 机制

```typescript
// OpenClaw: src/agents/subagent-spawn.ts（简化）
export async function spawnSubagentDirect(
  params: SpawnSubagentParams,
  ctx: SpawnSubagentContext,
): Promise<SpawnSubagentResult> {
  const task = params.task
  const requestedAgentId = params.agentId?.trim()

  // 1. 验证 agentId 格式（防止注入攻击）
  if (requestedAgentId && !isValidAgentId(requestedAgentId)) {
    return { status: "error", error: "Invalid agent ID format" }
  }

  // 2. 检查嵌套深度
  const currentDepth = getSubagentDepthFromSessionStore(ctx.agentSessionKey)
  if (currentDepth >= DEFAULT_SUBAGENT_MAX_SPAWN_DEPTH) {
    return { status: "error", error: "Max spawn depth exceeded" }
  }

  // 3. 选择运行模式
  const mode = resolveSpawnMode({
    requestedMode: params.mode,
    threadRequested: params.thread ?? false,
  })

  // 4. 注册到 registry 并启动
  registerSubagentRun(childSessionKey, runId, ...)
  return { status: "accepted", childSessionKey, runId, mode }
}
```

两种运行模式的区别：
- **`run` 模式**：一次性任务，完成后自动清理。适合"帮我搜索 X"这类短任务。
- **`session` 模式**：持久会话，完成后保持活跃，可以继续对话。适合需要多轮交互的复杂任务。

### 2.5.3 系统提示词分级：full vs minimal

OpenClaw 为不同层级的 Agent 使用不同详细程度的系统提示词：

```typescript
// OpenClaw: src/agents/system-prompt.ts
export type PromptMode = "full" | "minimal" | "none"
// - "full": 所有段落（主 Agent 使用）
// - "minimal": 精简版（子 Agent 使用，只保留工具、工作区、运行时信息）
// - "none": 仅基础身份行
```

**为什么子 Agent 用 minimal？** 因为子 Agent 只需要完成一个具体的子任务，不需要知道整个系统的消息路由规则、频道配置、语音 TTS 设置等。**精简 prompt = 更少 Token = 更快响应 = 更低成本**。

### 2.5.4 完成通知：announce 机制

子 Agent 完成任务后，需要把结果"汇报"给父 Agent。OpenClaw 实现了一套带重试的异步通知机制：

```typescript
// OpenClaw: src/agents/subagent-announce.ts（简化）
// 瞬态错误自动重试，间隔递增
const RETRY_DELAYS_MS = [5_000, 10_000, 20_000]  // 5s → 10s → 20s

// 区分瞬态错误和永久错误
const TRANSIENT_PATTERNS = [
  /gateway not connected/i,
  /gateway timeout/i,
  /\b(econnreset|econnrefused|etimedout)\b/i,
]
const PERMANENT_PATTERNS = [
  /unsupported channel/i,
  /chat not found/i,
  /user not found/i,
]
```

**设计亮点**：
- 父 Agent 在 spawn 子 Agent 后**不会轮询等待**，而是继续处理其他事务
- 子 Agent 完成后通过 announce 队列**推送**结果
- 如果通知失败（网络抖动等），自动重试最多 3 次，间隔递增
- 区分"可重试错误"（网关超时）和"不可重试错误"（频道不存在），避免无意义重试

### 2.5.5 多 Agent 并发安全

当多个 Agent 同时操作同一个代码仓库时，会产生严重的并发冲突。OpenClaw 在 `AGENTS.md` 中制定了一套详细的安全规范：

```markdown
## Multi-agent safety rules（摘自 OpenClaw AGENTS.md）

- 不要创建/应用/删除 git stash（其他 Agent 可能正在工作）
- 不要切换分支（除非用户明确要求）
- git pull --rebase 时不要丢弃其他 Agent 的工作
- commit 时只提交自己的更改，不要 commit all
- 看到不认识的文件时，忽略它，专注自己的任务
```

这些规则不是"建议"，而是写进系统提示词的**硬性约束**。它们解决的是多 Agent 共享文件系统时最常见的问题：一个 Agent 的 `git stash` 把另一个 Agent 的工作进度搞丢了。

---

## 2.6 多 Agent 系统的关键工程挑战

### 2.6.1 Token 爆炸与成本控制

Agent 间每次通信都消耗 Token。一个 5 层嵌套的子 Agent 调用链，每层传递完整上下文，Token 消耗会呈指数增长。

**实际解法**：
- **Prompt 分级**：如 OpenClaw 的 full/minimal/none 三级，子 Agent 只拿必要信息
- **结果压缩**：子 Agent 返回结构化摘要而非完整对话历史
- **深度限制**：OpenClaw 的 `DEFAULT_SUBAGENT_MAX_SPAWN_DEPTH` 硬性限制嵌套层数

### 2.6.2 防死循环与熔断

| 策略 | OpenCode 实现 | OpenClaw 实现 |
|------|--------------|--------------|
| 重复检测 | 连续 3 次相同调用触发询问 | 4 种检测器，阈值分级（10/20/30） |
| 全局限制 | 依赖模型的 max steps | 全局断路器（30 次工具调用强制终止） |
| 用户介入 | 弹出权限确认 | 警告 → 严重 → 强制中断三级响应 |

### 2.6.3 权限隔离

**核心原则：最小权限**。每个 Agent 只拥有完成其任务所需的最少工具和文件访问权限。

OpenCode 的实现尤其优雅——它使用 **glob 模式** 做细粒度权限控制：

```typescript
// 允许读取所有文件，但 .env 文件需要用户确认
read: {
  "*": "allow",
  "*.env": "ask",
  "*.env.*": "ask",
  "*.env.example": "allow",  // .env.example 是安全的
}
```

这意味着 Agent 读 `src/index.ts` 会直接执行，但读 `.env.production` 时会暂停并询问用户——因为环境变量文件可能包含密钥。

---

## 本章小结

- **单 Agent 的核心是 ReAct 循环**：LLM 推理 → 工具调用 → 观察结果 → 继续推理，直到任务完成。工程重点在于工具系统、系统提示词和防死循环机制。
- **多 Agent 的本质是"角色解耦 + 权限隔离"**：不是简单地多跑几个模型，而是让每个 Agent 专注于自己擅长的事，通过结构化通信协作。
- **OpenCode 展示了轻量级多 Agent**：build/plan/explore 三种模式通过权限规则集实现能力分化，用 `@general` 调用子 Agent 并行处理复杂任务。
- **OpenClaw 展示了重量级多 Agent**：完整的子 Agent 生命周期管理（spawn → run → announce → cleanup），带重试的异步通知，多 Agent 并发安全规范。
- **生产落地三大挑战**：Token 成本控制（prompt 分级 + 结果压缩）、防死循环（多层检测器 + 熔断）、权限隔离（glob 模式细粒度控制）。

---
