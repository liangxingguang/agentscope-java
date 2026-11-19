# 项目概述

本文档介绍 AgentScope-Java 的整体架构、模块划分和技术选型，内容与仓库当前代码（`2.1.0-SNAPSHOT`）保持一致。

## 1. 项目定位

### 1.1 AgentScope-Java 是什么？

AgentScope-Java 是一个 **面向生产环境的 AI Agent 开发框架**（Maven 坐标 `io.agentscope:agentscope-parent`），
提供构建 LLM 应用所需的核心抽象，并内置对"长时间运行、安全受控"的 Agent 执行的支持。

**核心价值**：
- 🚀 **双层 Agent 架构**：`ReActAgent`（推理内核）+ `HarnessAgent`（工程能力层）
- 🔧 **工具生态**：注解工具、MCP、内置文件/搜索/子 Agent 工具
- 🧠 **状态与记忆**：无状态 Agent + 可插拔 `AgentStateStore`，天然支持水平扩容
- 🔄 **多 Agent 协作**：Subagent、Team、Channel（钉钉/飞书/企业微信）
- 🛡️ **生产就绪**：事件流、权限引擎、中间件、沙箱、Durable 持久化运行时

### 1.2 适用场景

| 场景 | 描述 |
|------|------|
| 智能客服 | 自动问答、订单查询、多轮会话、人工接管 |
| 编码助手 | 读写代码、执行命令、沙箱隔离（见 `agentscope-codingagent`） |
| 数据分析 | 多步取数、图表生成、报告输出（见 `agentscope-dataagent`） |
| 长任务编排 | 跨请求、跨重启、跨主机继续执行（`agentscope-durable`） |
| 企业知识问答 | 应用层 RAG + `Knowledge` 扩展 |
| IM 机器人 | Channel 模块接入钉钉、飞书、企业微信 |

## 2. 整体架构

### 2.1 分层架构

```
┌──────────────────────────────────────────────────────────────────┐
│                        应用层 (Application)                       │
│  智能客服 │ 编码助手 │ 数据分析 │ Web/IM 前端 │ 你的业务系统        │
└───────────────────────────┬──────────────────────────────────────┘
                            │  call() / streamEvents()
┌───────────────────────────┴──────────────────────────────────────┐
│                    工程层 (agentscope-harness)                    │
│  HarnessAgent: 工作区 │ 分层记忆 │ 技能 │ 子 Agent │ Plan Mode     │
│                上下文压缩 │ 文件系统 │ 沙箱 │ 团队协作             │
└───────────────────────────┬──────────────────────────────────────┘
                            │  Middleware / Toolkit / StateStore
┌───────────────────────────┴──────────────────────────────────────┐
│                     核心层 (agentscope-core)                      │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                       ReActAgent                           │  │
│  │   Reasoning(思考) ⇄ Acting(行动)  →  直到完成/达上限        │  │
│  └────────────────────────────────────────────────────────────┘  │
│  Model │ Toolkit │ Middleware │ AgentState │ AgentEvent │ Permission│
└───────────────────────────┬──────────────────────────────────────┘
                            │
┌───────────────────────────┴──────────────────────────────────────┐
│               持久化运行时 (agentscope-durable)                   │
│  DurableRuntime │ SessionBackend │ HarnessAdapter │ Executor      │
│  任务受理 │ 检查点 │ 事件重放 │ 跨副本恢复 │ 外部工具回调           │
└───────────────────────────┬──────────────────────────────────────┘
                            │
┌───────────────────────────┴──────────────────────────────────────┐
│                    扩展层 (agentscope-extensions)                 │
│  模型: dashscope │ openai │ anthropic │ gemini │ ollama           │
│  存储: Redis │ MySQL │ PostgreSQL │ MongoDB │ OSS │ COS           │
│  RAG : simple │ dify │ ragflow │ haystack │ bailian              │
│  其他: MCP │ Channel │ Sandbox │ Skill │ Studio │ Nacos │ Higress │
└──────────────────────────────────────────────────────────────────┘
```

### 2.2 核心组件

#### ReActAgent（推理内核）

`io.agentscope.core.ReActAgent` 是框架的核心，实现 ReAct 循环：

```
┌─────────────────────────────────────────────────────────┐
│                    ReAct 循环                            │
│                                                         │
│    ┌──────────┐      ┌──────────┐      ┌──────────┐    │
│    │Reasoning │ ──→  │  Acting  │ ──→  │ 检查完成 │    │
│    │ (思考)   │      │ (行动)   │      │  条件    │    │
│    └──────────┘      └──────────┘      └──────────┘    │
│         ↑                                    │          │
│         └────────────────────────────────────┘          │
│                  (循环，默认上限 20 轮)                   │
└─────────────────────────────────────────────────────────┘
```

**特点**：
- **无状态**：会话历史存放在 `AgentState`，由 `AgentStateStore` 持久化；同一 Agent 实例可服务多个会话
- **两种输出**：`call()` 返回最终 `Msg`；`streamEvents()` 返回 32 种类型化事件
- **可拦截**：`Middleware` 提供 6 个切面（agent/reasoning/acting/modelCall/systemPrompt/agentStateReady）

> `ReActAgent.java` 单文件约 7400 行，是理解整个框架的最佳入口。

#### HarnessAgent（工程能力层）

`io.agentscope.harness.agent.HarnessAgent` 在 ReActAgent 之上通过 Middleware 与 Toolkit 通道叠加能力，
推理内核保持不变：

| 能力 | 说明 |
|------|------|
| 工作区 (workspace) | 人格、记忆、技能、子 Agent 声明的落盘目录 |
| 分层记忆 | 上下文对话 + `MEMORY.md` + 磁盘事实日志，自动压缩 |
| 技能 (Skill) | Markdown 技能按需加载，可跨会话共享 |
| 子 Agent | Markdown 声明 + `agent_spawn` / `agent_send` 运行时编排 |
| 文件系统 | 本地 / 共享存储 / 沙箱，可路由（`filesystemRoute`） |
| Plan Mode | 只读规划态，计划文件持久化并驱动执行 |

#### Model（模型层）

模型以 **SPI 扩展模块** 形式提供，通过 `ServiceLoader` 注册：

```
┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐
│ DashScope  │ │   OpenAI   │ │ Anthropic  │ │ Gemini /   │
│ (阿里云)   │ │            │ │  (Claude)  │ │ Ollama     │
└─────┬──────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘
      │              │              │              │
      └──────────────┴──────┬───────┴──────────────┘
                            │  META-INF/services/...ModelProvider
                     ┌──────┴──────┐
                     │ ModelRegistry│  → model("dashscope:qwen-plus")
                     └─────────────┘
```

`agentscope-core` 只定义 `Model` 接口与 `ModelRegistry`；具体厂商实现放在
`agentscope-extensions-model-*` 中，用 `model("provider:model-id")` 字符串即可自动解析。

#### Toolkit（工具系统）

```java
// 1. 定义工具
public class WeatherTools {
    @Tool(name = "get_weather", description = "获取天气")
    public String getWeather(@ToolParam(name = "city", description = "城市") String city) {
        return "北京：晴天 25°C";
    }
}

// 2. 注册
Toolkit toolkit = new Toolkit();
toolkit.registerTool(new WeatherTools());

// 3. 交给 Agent
ReActAgent agent = ReActAgent.builder().toolkit(toolkit).build();
```

也可以按需分组（`createToolGroup` / `setActiveGroups`）、注册 MCP 客户端（`registerMcpClient`）、
注册子 Agent（`registration().subAgent(...)`）。

#### AgentState / AgentStateStore（状态与记忆）

v2 移除了"必须传 Memory"的设计。会话历史放在 `AgentState`：

```
┌────────────────────────────────────────────────┐
│                  AgentState                     │
├────────────────────────────────────────────────┤
│  context        : List<Msg>   对话消息           │
│  summary        : String      压缩摘要           │
│  tasksContext   : 任务列表                       │
│  toolContext    : 工具上下文                     │
│  permissionContext : 权限状态                    │
│  planModeContext   : 计划模式状态                │
└────────────────────────────────────────────────┘
              ↕ 持久化
┌────────────────────────────────────────────────┐
│              AgentStateStore                    │
├────────────────────────────────────────────────┤
│  InMemoryAgentStateStore   - 内存（默认）        │
│  JsonFileAgentStateStore   - JSON 文件           │
│  Redis / MySQL / PostgreSQL / Mongo 扩展        │
└────────────────────────────────────────────────┘
```

> `io.agentscope.core.memory.Memory` 与 `InMemoryMemory` 在 2.0.0 起已标记
> `@Deprecated(forRemoval = true)`，仅为兼容 1.0.x 用户代码保留。

#### Middleware（中间件）

v1 的 `Hook` 已废弃（`@Deprecated(forRemoval = true, since = "2.0.0")`），
v2 用 `MiddlewareBase` 的六个切面替代：

```
onAgent          → 整个 Agent 调用前后
onReasoning      → 思考阶段前后（可 tap 文本/思考增量事件）
onActing         → 工具执行阶段前后（可 tap ToolProgressEvent）
onModelCall      → 单次模型调用前后
onSystemPrompt   → 改写系统提示词
onAgentStateReady→ AgentState 就绪时
```

通过 `activePoints()` 声明参与哪些切面，`order()` 控制顺序。

#### AgentEvent（事件流）

`streamEvents()` 产出 `AgentEvent`，`AgentEventType` 共 **32 种**：

```
生命周期   AGENT_START, AGENT_END, AGENT_RESULT, CUSTOM
模型调用   MODEL_CALL_START, MODEL_CALL_END
文本       TEXT_BLOCK_START, TEXT_BLOCK_DELTA, TEXT_BLOCK_END
思考       THINKING_BLOCK_START, THINKING_BLOCK_DELTA, THINKING_BLOCK_END
数据块     DATA_BLOCK_START, DATA_BLOCK_DELTA, DATA_BLOCK_END
工具调用   TOOL_CALL_START, TOOL_CALL_DELTA, TOOL_CALL_END
工具结果   TOOL_RESULT_START, TOOL_PROGRESS, TOOL_RESULT_TEXT_DELTA,
           TOOL_RESULT_DATA_DELTA, TOOL_RESULT_END
人机交互   REQUIRE_USER_CONFIRM, USER_CONFIRM_RESULT,
           REQUIRE_EXTERNAL_EXECUTION, EXTERNAL_EXECUTION_RESULT
控制       EXCEED_MAX_ITERS, REQUEST_STOP, ALL_TOOLS_DENIED
其他       SUBAGENT_EXPOSED, HINT_BLOCK
```

#### Permission（权限系统）

`io.agentscope.core.permission` 提供三态工具门控：

| 模式 | 行为 |
|------|------|
| `PermissionBehavior.ALLOW` | 直接执行 |
| `PermissionBehavior.ASK` | 暂停，返回 `GenerateReason.PERMISSION_ASKING`，等人工确认 |
| `PermissionBehavior.DENY` | 拒绝执行 |

`PermissionMode.DONT_ASK` 用于无人值守场景（CI、自动化测试），把 ASK 直接降级为拒绝。

## 3. 模块详解

### 3.1 agentscope-core（核心模块）

`io.agentscope:agentscope-core` — 651 个 Java 文件，约 16.6 万行。

顶层只有两个类：`ReActAgent`、`Version`。子包：

```
io.agentscope.core
├── agent/            Agent 接口族、RuntimeContext、执行控制、Event/EventType
│   ├── config/       ReactConfig（maxIters 默认 20）、ModelConfig
│   ├── accumulator/  流式块累积器
│   └── user/         用户相关抽象
├── model/            Model 接口、ModelRegistry、GenerateOptions、ExecutionConfig
│   ├── spi/          ModelProvider（SPI 扩展点）
│   └── transport/    JDK / OkHttp / WebSocket 传输、代理配置
├── message/          Msg、UserMessage、AssistantMessage、SystemMessage、ToolResultMessage
│                     TextBlock/ImageBlock/AudioBlock/VideoBlock/File/DataBlock…
├── event/            32 种 AgentEvent 类型
├── tool/             Toolkit、Tool、ToolGroup、ToolExecutor、ToolEmitter
│   ├── builtin/      TodoTools、RequirementTools
│   ├── coding/       编码类工具
│   ├── file/         文件类工具
│   ├── mcp/          MCP 客户端封装
│   ├── search/       搜索工具
│   └── subagent/     子 Agent 工具
├── middleware/       MiddlewareBase、MiddlewareChain、各类 Input 记录
├── state/            AgentState、AgentStateStore、ConflictPolicy、Task*
│   └── legacy/       v1 状态迁移支持
├── memory/           Memory（已废弃）、LongTermMemory、StateBackedMemory
├── journal/          执行日志（可回放）
├── permission/       PermissionEngine、PermissionRule、PermissionMode
├── interruption/     InterruptControl、InterruptSource
├── hook/             Hook（已废弃）+ recorder/ 追踪导出
├── skill/            AgentSkill、SkillBox、SkillRegistry、SkillHook
├── rag/              Knowledge、RAGMode、RetrieveConfig、GenericRAGHook
├── workspace/        shutdown/  observation/  tracing/  credential/  util/
└── formatter/  exception/
```

### 3.2 agentscope-harness（工程层）

`io.agentscope:agentscope-harness` — 504 个 Java 文件，约 11.5 万行。

```
io.agentscope.harness.agent
├── HarnessAgent              双 Builder 支持（含 fromAgent）
├── workspace/                工作区布局与初始化
├── memory/                   分层记忆与压缩
├── skill/                    技能仓库与加载
├── subagent/                 子 Agent 声明与运行
├── team/                     团队协作
├── filesystem/               本地 / 共享 / 沙箱文件系统
├── sandbox/                  沙箱规格
├── gateway/  bus/            网关与事件总线
├── context/                  上下文构建与选择策略
├── middleware/               工程层中间件
├── journal/  observation/  artifact/  verification/  coordination/
└── tool/  tools/
```

### 3.3 agentscope-durable（持久化运行时）

`io.agentscope:agentscope-durable` — 81 个 Java 文件，约 2.7 万行，4 个子模块：

| 子模块 | 作用 |
|--------|------|
| `durable-runtime` | `DurableRuntime`、`DurableSession/Turn/Thread`、`SessionBackend`、`ExecutionRecovery` |
| `durable-adapters` | `HarnessAdapter` 等与 Core/Harness 的桥接实现 |
| `durable-worker` | 独立执行工具/外部作业的 Worker |
| `protocol` | 公开交互契约与共享 fixture（Schema 的权威来源） |

解决的问题：**任务跨越客户端断线、应用重启、执行主机替换**后仍可继续。
详细说明见 `agentscope-durable/README.md` 与 `docs/dev/*/docs/harness/durable.md`。

### 3.4 agentscope-extensions（扩展层）

27 个顶层子模块，另有 `agentscope-spring-boot-starters` 下 14 个 starter。

| 分组 | 模块 |
|------|------|
| 模型 | `agentscope-extensions-model-{dashscope,openai,openai-official,anthropic,gemini,ollama}` |
| RAG | `-rag`（其下 `-rag-simple`、`-rag-dify`、`-rag-ragflow`、`-rag-haystack`、`-rag-bailian`） |
| 长期记忆 | `-mem`（其下 `-mem0`、`-reme`、`-memory-bailian`） |
| 存储 | `-redis`、`-mysql`、`-postgresql`、`-mongodb`、`-jdbc`、`-oss`、`-cos` |
| 能力 | `-protocol`、`-channel`、`-sandbox`、`-skills`（其下 git / mysql / postgresql 仓库）、`-scheduler`、`-training`、`-judge`（其下 `-jev`、`-openai-decisions`） |
| 集成 | `-nacos`、`-higress`、`-studio` |
| Durable 适配 | `-durable-jdbc`、`-durable-http`、`-durable-sandbox`、`-durable-rocketmq`、`-durable-objects`、`-durable-mcp` |
| Spring Boot | `agentscope-spring-boot-starter`、`-dashscope-`、`-openai-`、`-openai-official-`、`-openai-decisions-`、`-anthropic-`、`-gemini-`、`-ollama-`、`-a2a-`、`-agui-`、`-admin-`、`-nacos-`、`-jev-`、`-chat-completions-web-` |

> **注意**：`agentscope-extensions` 下没有 Quarkus / Micronaut 集成。

### 3.5 agentscope-examples（示例模块）

441 个 Java 文件，约 8.2 万行，5 个子模块：

| 子模块 | 内容 |
|--------|------|
| `documentation` | 主线示例（61 个 `*Example` 类），包名 `io.agentscope.examples.documentation2.*` |
| `jev` | JEV 评估/判定示例与基准程序 |
| `durable` | Durable 集成与 Spring Boot 示例 |
| `agents/agentscope-chat` | 基于 HarnessAgent + Durable 的可恢复聊天应用（含前端） |
| `agents/agentscope-codingagent` | 编码 Agent |
| `agents/agentscope-dataagent` | 数据分析 Agent |
| `agents/agentscope-paw` | 示例 Agent 应用 |

`documentation` 下的示例分类（与 `03-concepts.md` 对应）：

```
quickstart/ │ tool/ │ structuredoutput/ │ streaming/ │ state/ │ middleware/
model/ │ mcp/ │ multimodal/ │ skill/ │ rag/ │ hitl/ │ harness/ │ judge/
agui/ │ copilotkit/ │ context/
```

### 3.6 依赖与发布

| 模块 | 作用 |
|------|------|
| `agentscope-dependencies-bom` | 第三方依赖版本统一（Reactor / Jackson / JUnit / Mockito / SLF4J …） |
| `agentscope-distribution/agentscope-bom` | 对外发布的 `io.agentscope` 各模块版本 BOM |
| `agentscope-distribution/agentscope-all` | 聚合分发（一站式依赖） |

## 4. 技术选型

### 4.1 为什么选择这些技术？

| 技术 | 选择原因 |
|------|----------|
| Java 17 | record / sealed / 模式匹配；企业环境普及度高 |
| Project Reactor | 原生支持流式响应与背压，事件流天然适配 |
| Jackson | 消息、JSON Schema、状态快照的序列化标准 |
| Maven + `${revision}` | 多模块统一版本，`flatten-maven-plugin` 处理发布 |
| Spotless（AOSP 4 空格） | 强制统一风格，`compile` 阶段即检查 |
| JaCoCo | 覆盖率门槛与报告 |

### 4.2 支持的模型

| Provider ID | 扩展模块 | 典型模型 |
|-------------|----------|----------|
| `dashscope` | `agentscope-extensions-model-dashscope` | `qwen-plus`、`qwen-max`、`qwen-turbo` |
| `openai` | `agentscope-extensions-model-openai` | `gpt-4o`、`gpt-4` |
| `openai-official` | `agentscope-extensions-model-openai-official` | OpenAI Responses API |
| `anthropic` | `agentscope-extensions-model-anthropic` | `claude-3-5-sonnet` 等 |
| `gemini` | `agentscope-extensions-model-gemini` | Gemini 系列 |
| `ollama` | `agentscope-extensions-model-ollama` | 本地开源模型 |

用法：`model("dashscope:qwen-plus")`（`ModelRegistry` 通过 SPI 自动解析）。

## 5. 数据流

### 5.1 一次 `call()` 的处理流程

```
用户输入 (UserMessage)
    │
    ▼
RuntimeContext（sessionId / userId / threadId）
    │
    ▼
AgentStateStore.load()  →  AgentState（历史 context）
    │
    ▼
Middleware.onAgent（前）
    │
    ▼
┌─── ReAct 循环（≤ maxIters，默认 20）───────────────┐
│  Middleware.onReasoning                            │
│      → Model.stream(messages, tools, options)      │
│      → 产出 TextBlock / ThinkingBlock / ToolUseBlock│
│  Middleware.onActing（若有工具调用）                │
│      → PermissionEngine 门控                        │
│      → Toolkit.callTool()                          │
│      → ToolResultBlock 回灌上下文                   │
└────────────────────────────────────────────────────┘
    │
    ▼
AgentStateStore.save(AgentState)
    │
    ▼
Middleware.onAgent（后） → 返回 Msg
```

### 5.2 消息结构

```
Msg（抽象基类，按角色分具体类型）
├── role: USER      → UserMessage
├── role: ASSISTANT → AssistantMessage
├── role: SYSTEM    → SystemMessage
└── role: TOOL      → ToolResultMessage

content: List<ContentBlock>
├── TextBlock       文本
├── ThinkingBlock   思考过程
├── ImageBlock / AudioBlock / VideoBlock / DocumentBlock / DataBlock  多模态
├── ToolUseBlock    工具调用
└── ToolResultBlock 工具结果
```

## 6. 与其他框架对比

| 特性 | AgentScope-Java | LangChain4j | Spring AI |
|------|-----------------|-------------|-----------|
| ReAct 循环 | ✅ 内置（`ReActAgent`） | ⚠️ 需组合 | ⚠️ 需组合 |
| 事件流 | ✅ 32 种类型化事件 | ⚠️ 有限 | ⚠️ 有限 |
| 中间件/切面 | ✅ 6 个切面 | ❌ | ⚠️ Advisor |
| 无状态 + 可插拔状态存储 | ✅ 原生 | ⚠️ 需自建 | ⚠️ 需自建 |
| 权限门控 / HITL | ✅ 三态引擎 | ❌ | ⚠️ 有限 |
| 沙箱执行 | ✅ 本地/Docker/K8s/云沙箱 | ❌ | ❌ |
| 多 Agent 编排 | ✅ Subagent / Team / Channel | ⚠️ 有限 | ⚠️ 有限 |
| 长任务持久化 | ✅ `agentscope-durable` | ❌ | ❌ |
| 国内模型 | ✅ DashScope 一等公民 | ⚠️ 社区 | ⚠️ 社区 |

## 7. 设计原则

- **单一职责**：`Model` 只管模型调用，`Toolkit` 只管工具，`AgentStateStore` 只管状态持久化
- **开闭原则**：新增模型 → 实现 `ModelProvider` SPI；新增工具 → 加 `@Tool` 方法；新增切面 → 实现 `MiddlewareBase`
- **里氏替换**：任何 `Model`、`AgentStateStore`、`MiddlewareBase` 实现可互换
- **接口隔离**：`Model` 核心只有 `stream()` + `getModelName()`，其余为 default 能力声明
- **依赖倒置**：`ReActAgent` 依赖 `Model` / `Toolkit` / `AgentStateStore` 抽象
- **组合优于继承**：`HarnessAgent` 通过 Middleware + Toolkit 通道叠加能力，不改推理内核

## 8. 下一步

- [快速开始](./02-quickstart.md)：搭建环境，运行第一个程序
- [核心概念](./03-concepts.md)：深入理解各个组件
