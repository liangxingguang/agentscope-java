# 核心概念

本文档介绍 AgentScope-Java（v2.x）的核心概念与设计原理。所有 API 均与仓库当前代码一致。

## 1. Agent（智能体）

### 1.1 什么是 Agent？

Agent 是 AI 应用中的"大脑"，它能够理解输入、决定下一步行动、调用工具并记住上下文。

框架提供两层 Agent：

| 类型 | 全名 | 定位 |
|------|------|------|
| 推理内核 | `io.agentscope.core.ReActAgent` | 纯粹的 ReAct 循环 + 工具 + 事件 + 中间件 |
| 工程层 | `io.agentscope.harness.agent.HarnessAgent` | 在 ReActAgent 上叠加工作区/记忆/技能/子 Agent/沙箱 |

`HarnessAgent` 内部就是一个 `ReActAgent`，通过 Middleware 与 Toolkit 通道增强，**推理内核不变**。

### 1.2 ReAct 循环

```
┌─────────────────────────────────────────────────────────────┐
│                       ReActAgent                             │
│                                                              │
│    用户输入 ──→ Reasoning(思考) ──→ Acting(行动) ──→ 输出    │
│                     ↑               │                        │
│                     └───────────────┘                        │
│                    (循环直到完成，默认上限 20 轮)             │
└─────────────────────────────────────────────────────────────┘
```

### 1.3 创建 Agent

```java
ReActAgent agent = ReActAgent.builder()
        // 基础配置
        .name("助手")
        .agentId("assistant")                 // 持久化身份
        .description("一个通用的 AI 助手")
        .sysPrompt("你是一个友好的助手")

        // 核心组件
        .model("dashscope:qwen-plus")         // 或 .model(Model instance)
        .toolkit(new Toolkit())
        .stateStore(new JsonFileAgentStateStore(path))  // 可选，默认内存
        .defaultSessionId("alice")            // 可选

        // 执行配置
        .maxIters(10)                         // 默认 20
        .middleware(new MyMiddleware())       // 替代 v1 的 .hook(...)

        .build();
```

> **v2 不再需要 `.memory(...)`**。会话历史由 `AgentState` 持有，通过 `AgentStateStore` 持久化。
> `io.agentscope.core.memory.Memory` / `InMemoryMemory` 已 `@Deprecated(forRemoval = true)`。

### 1.4 常用 Builder 配置

| 配置项 | 类型 | 说明 |
|--------|------|------|
| `name(String)` | String | Agent 名称 |
| `agentId(String)` | String | 持久化身份标识 |
| `sysPrompt(String)` | String | 系统提示词 |
| `model(Model)` / `model(String)` | Model / String | 模型实例或 `"provider:model-id"` |
| `toolkit(Toolkit)` | Toolkit | 工具集合 |
| `middleware(...)` / `middlewares(...)` | MiddlewareBase | 中间件（v2 推荐） |
| `hook(...)` / `hooks(...)` | Hook | 旧钩子（已废弃，仍兼容） |
| `maxIters(int)` | int | 最大迭代轮数，默认 `20`（`ReactConfig.DEFAULT_MAX_ITERS`） |
| `stateStore(AgentStateStore)` | AgentStateStore | 状态存储 |
| `defaultSessionId(String)` | String | 默认会话 ID |
| `conflictPolicy(ConflictPolicy)` | 枚举 | 并发写状态时的冲突策略 |
| `modelExecutionConfig(ExecutionConfig)` | ExecutionConfig | 模型调用超时/重试 |
| `toolExecutionConfig(ExecutionConfig)` | ExecutionConfig | 工具调用超时/重试 |
| `generateOptions(GenerateOptions)` | GenerateOptions | 温度、maxTokens 等 |
| `permissionContext(PermissionContextState)` | 权限状态 | 权限门控 |
| `enableMetaTool(boolean)` | boolean | 启用工具组元工具 |
| `enableTaskList(boolean)` | boolean | 启用任务清单 |
| `longTermMemory(LongTermMemory)` | 长期记忆 | 长期记忆（配合 mode） |
| `skillBox(SkillBox)` / `skillRepository(...)` | 技能 | 技能仓库 |
| `knowledge(Knowledge)` / `ragMode(...)` | RAG | 1.x 兼容的 RAG 配置 |

### 1.5 调用方式

```java
// 同步（阻塞）
Msg reply = agent.call(new UserMessage("你好")).block();

// 同步 + 运行时上下文
RuntimeContext ctx = RuntimeContext.builder()
        .sessionId("demo")
        .userId("alice")
        .build();
Msg reply2 = agent.call(new UserMessage("你好"), ctx).block();

// 结构化输出
Msg reply3 = agent.call(new UserMessage("提取产品需求"), ProductRequirements.class).block();
ProductRequirements data = reply3.getStructuredData(ProductRequirements.class);

// 事件流（v2 主要用法）
agent.streamEvents(new UserMessage("介绍一下你自己"))
        .doOnNext(event -> {
            switch (event.getType()) {
                case TEXT_BLOCK_DELTA -> System.out.print(((TextBlockDeltaEvent) event).getDelta());
                case TOOL_CALL_START -> System.out.println("\n[tool] " + ((ToolCallStartEvent) event).getToolCallName());
                default -> { }
            }
        })
        .blockLast();

// 中断
agent.interrupt();                       // 或 interrupt(new UserMessage("停一下"))
```

**主要签名（`ReActAgent`）**：

```java
Mono<Msg>  call(List<Msg> msgs)
Mono<Msg>  call(List<Msg> msgs, RuntimeContext ctx)
Mono<Msg>  call(List<Msg> msgs, Class<?> structuredOutputClass, RuntimeContext ctx)
Mono<Msg>  call(String text, RuntimeContext ctx)
Flux<AgentEvent> streamEvents(List<Msg> msgs)
Flux<AgentEvent> streamEvents(Msg msg)
Flux<AgentEvent> streamEvents(Msg msg, RuntimeContext ctx)
Flux<AgentEvent> streamEvents(String text)
void interrupt()
void interrupt(Msg msg)
```

> `agent.stream(msg)` / `agent.stream(msgs, StreamOptions)` 等旧接口已
> `@Deprecated(forRemoval = true, since = "2.0.0")`，请改用 `streamEvents(...)`。

## 2. Model（模型）

### 2.1 什么是 Model？

Model 是与大语言模型通信的抽象，负责把消息与工具 Schema 发给模型、把流式响应转成 `ChatResponse`。

### 2.2 Model 接口

```java
public interface Model {
    Flux<ChatResponse> stream(List<Msg> messages, List<ToolSchema> tools, GenerateOptions options);

    String getModelName();

    // 以下均为 default 能力声明
    default ToolSearchMode getToolSearchMode() { ... }
    default boolean supportsNativeStructuredOutput() { return false; }
    default boolean supportsNativeStructuredOutputWithTools() { ... }
    default boolean supportsToolChoiceSpecific() { return false; }
    default int getContextWindowSize() { return 0; }
}
```

核心实现类都在扩展模块中：

| 类 | 模块 |
|----|------|
| `DashScopeChatModel` | `agentscope-extensions-model-dashscope` |
| `OpenAIChatModel` | `agentscope-extensions-model-openai` |
| `OpenAIResponsesChatModel` | `agentscope-extensions-model-openai-official` |
| `AnthropicChatModel` | `agentscope-extensions-model-anthropic` |
| `GeminiChatModel` | `agentscope-extensions-model-gemini` |
| `OllamaChatModel` | `agentscope-extensions-model-ollama` |

### 2.3 两种用法

**方式一：模型标识符（推荐）**

```java
.model("dashscope:qwen-plus")
.model("openai:gpt-4o")
.model("anthropic:claude-3-5-sonnet-latest")
```

`ModelRegistry` 通过 `ServiceLoader` 加载 `ModelProvider`（SPI 文件位于各扩展模块的
`META-INF/services/io.agentscope.core.model.spi.ModelProvider`），自动解析 provider 并读取对应环境变量。

**方式二：显式构造**

```java
import io.agentscope.extensions.model.dashscope.DashScopeChatModel;
import io.agentscope.extensions.model.dashscope.formatter.DashScopeChatFormatter;

DashScopeChatModel model = DashScopeChatModel.builder()
        .apiKey(System.getenv("DASHSCOPE_API_KEY"))
        .modelName("qwen-plus")
        .stream(true)
        .enableThinking(true)                  // 输出 ThinkingBlock
        .formatter(new DashScopeChatFormatter())
        .build();
```

### 2.4 生成选项

```java
GenerateOptions options = GenerateOptions.builder()
        .temperature(0.7)
        .topP(0.9)
        .topK(50)
        .maxTokens(4096)
        .thinkingBudget(1024)
        .includeThoughts(true)
        .reasoningEffort("medium")
        .seed(42L)
        .parallelToolCalls(true)
        .toolChoice(new ToolChoice.Auto())   // Auto / None / Required / Specific(name)
        .cacheControl(true)
        .executionConfig(ExecutionConfig.builder()
                .timeout(Duration.ofSeconds(60))
                .maxAttempts(3)
                .build())
        .build();

ReActAgent.builder().generateOptions(options)...
```

`GenerateOptions` 还支持 `additionalHeader(...)` / `additionalBodyParam(...)` /
`additionalQueryParam(...)` 等透传配置。

### 2.5 传输层

`io.agentscope.core.model.transport` 提供可替换的 HTTP 传输：

- `JdkHttpTransport`（默认）、`OkHttpTransport`、`WebSocketTransport`
- `HttpTransportConfig` / `ProxyConfig` / `ProxyType`：超时、代理（HTTP / SOCKS）

## 3. State（状态与记忆）

### 3.1 为什么不再有 Memory？

v1 把对话历史放在 `Memory` 对象里，Agent 实例因此有状态，难以水平扩容。
v2 把状态抽成 **`AgentState`（数据）** + **`AgentStateStore`（存储）**，Agent 变成无状态的：

```
Agent 实例（无状态，可复用/可共享）
        │  每次调用
        ▼
AgentStateStore.load(userId, sessionId) → AgentState
        │
        ▼（ReAct 循环读写 context）
AgentStateStore.save(userId, sessionId, "agent_state", state)
```

### 3.2 AgentState 内容

| 字段 | 说明 |
|------|------|
| `context` | 对话消息列表（List&lt;Msg&gt;） |
| `summary` | 上下文压缩后的摘要 |
| `curIter` | 当前迭代轮次 |
| `replyId` | 当前回复标识 |
| `shutdownInterrupted` | 是否因停机中断 |
| `tasksContext` | 任务清单状态 |
| `toolContext` | 工具上下文 |
| `permissionContext` | 权限状态 |
| `planModeContext` | 计划模式状态 |

```java
AgentState state = agent.getAgentState();
int size = state.getContext().size();      // 历史消息数（替代 v1 的 memory.getMessages()）
```

### 3.3 AgentStateStore 实现

| 实现 | 位置 | 适用场景 |
|------|------|----------|
| `InMemoryAgentStateStore` | core | 默认；单进程、临时对话 |
| `JsonFileAgentStateStore` | core | 单机持久化 |
| Redis / MySQL / PostgreSQL / MongoDB | `agentscope-extensions-*` | 分布式部署、多副本 |
| Durable `SessionBackend` | `agentscope-durable` | 长任务、检查点、跨主机恢复 |

```java
import io.agentscope.core.state.AgentStateStore;
import io.agentscope.core.state.JsonFileAgentStateStore;
import java.nio.file.Paths;

AgentStateStore store = new JsonFileAgentStateStore(
        Paths.get(System.getProperty("user.home"), ".agentscope", "sessions"));

ReActAgent agent = ReActAgent.builder()
        .stateStore(store)
        .defaultSessionId("alice")     // 同一 sessionId 会复用历史
        .build();
```

`AgentStateStore` 接口按 `(userId, sessionId, key)` 组织数据，支持：
`save` / `save(List)` / `get` / `getList` / `exists` / `delete` / `listSessionIds`，
以及可选的**乐观锁版本控制**（`saveIfVersion` / `getVersioned`）。

并发写同一 `AgentState` 时可用 `ConflictPolicy` 指定策略
（如 `ConcurrentSessionModificationException` 抛错）。

### 3.4 长期记忆（⚠️ 已废弃，正在重写）

```java
.longTermMemory(myLongTermMemory)
.longTermMemoryMode(LongTermMemoryMode.AGENT_CONTROL)  // 或 STATIC_CONTROL
.longTermMemoryAsyncRecord(true)       // 异步写入，降低延迟
```

> **`LongTermMemory` / `LongTermMemoryMode` / `LongTermMemoryTools` / `StaticLongTermMemoryHook`
> 全部是 `@Deprecated(forRemoval = true, since = "2.0.0")`**，对应的 Builder 方法
> `.longTermMemory(...)` / `.longTermMemoryMode(...)` / `.longTermMemoryAsyncRecord(...)`
> 同样是 `forRemoval`。官方迁移指南写明「will be removed in the next minor」，
> 新代码不要依赖。

`io.agentscope.core.memory` 下还有 `LongTermMemoryTools`（把长期记忆暴露为工具）、
`StaticLongTermMemoryHook`、`StateBackedMemory`（同样已废弃）。
扩展层提供 `agentscope-extensions-mem0`、`-reme`、`-memory-bailian` 等实现。

**当前替代做法**：把记忆当成外部存储自己管——检索后拼进 prompt（与
[12. RAG](#12-rag检索增强生成) 的应用层思路一致），或用 `AgentState.summary` +
Harness 的 `CompactionConfig` 做上下文压缩。

## 4. Tool（工具）

### 4.1 定义工具

```java
public class WeatherTools {

    @Tool(
        name = "get_weather",
        description = "获取指定城市的天气，当用户询问天气时使用",
        readOnly = true,            // 只读（可被并发调度）
        concurrencySafe = true
    )
    public String getWeather(
        @ToolParam(name = "city", description = "城市名称，如：北京")
        String city,
        @ToolParam(name = "unit", description = "celsius 或 fahrenheit", required = false)
        String unit) {
        return String.format("%s 今天晴天，温度 25°%s", city, unit);
    }
}
```

**`@Tool` 可用属性**：

| 属性 | 默认 | 说明 |
|------|------|------|
| `name` / `description` | `""` | 工具名与描述（喂给模型） |
| `strict` | `false` | 严格 Schema 校验 |
| `deferLoading` | `false` | 延迟加载（配合工具搜索） |
| `readOnly` | `false` | 只读，无副作用 |
| `concurrencySafe` | `true` | 可并发执行 |
| `timeoutManaged` | `false` | 由工具自己管理超时 |
| `externalTool` | `false` | 外部执行（由宿主回调） |
| `system` | `false` | 系统工具 |
| `stateInjected` | `false` | 注入 `AgentState` |
| `returnDirect` | `false` | 结果直接返回用户，不再进模型 |
| `dangerousFiles` / `dangerousDirectories` | `{}` | 危险路径声明（触发权限检查） |
| `converter` | `DefaultToolResultConverter` | 自定义结果转换器 |

**`@ToolParam` 属性**：`name`、`description`、`required`（默认 `true`）。

### 4.2 注册工具

```java
Toolkit toolkit = new Toolkit();

// 方式一：注册对象，自动发现所有 @Tool 方法
toolkit.registerTool(new WeatherTools());

// 方式二：链式注册（推荐，可顺带配置组、MCP、子 Agent）
toolkit.registration()
        .tool(new WeatherTools())
        .group("weather")
        .apply();

// 方式三：注册已实现 AgentTool 的工具
toolkit.registerAgentTool(myAgentTool);

// 查询
toolkit.getToolNames();
toolkit.getToolSchemas();
```

### 4.3 工具组（ToolGroup）

```java
Toolkit toolkit = new Toolkit();

// 创建组（默认 META 作用域，可被 meta tool 管理）
toolkit.createToolGroup("weather", "天气相关工具", false);
toolkit.addToolToGroup("weather", "get_weather");

// 或用 Builder
import io.agentscope.core.tool.ToolGroup;
toolkit.registerToolGroup(
        ToolGroup.builder()
                .name("math")
                .description("数学运算")
                .active(false)
                .tools(Set.of("calculate"))
                .build());

// 手动激活
toolkit.setActiveGroups(List.of("weather"));
toolkit.updateToolGroups(List.of("math"), true);   // 批量启用/停用
toolkit.getActiveGroups();
```

`enableMetaTool(true)` 时，模型可通过元工具自行管理工具组开关。

### 4.4 工具返回类型

```java
// 字符串
@Tool(name = "get_time")
public String getTime() { return LocalTime.now().toString(); }

// 响应式（异步）
@Tool(name = "fetch_url")
public Mono<String> fetchUrl(@ToolParam(name = "url") String url) {
    return webClient.get(url);
}

// 自定义结果转换
@Tool(name = "get_data", converter = JsonConverter.class)
public Data getData(@ToolParam(name = "id") String id) { ... }

// 流式进度：声明 ToolEmitter 参数，框架自动注入
@Tool(name = "process_data", description = "处理数据集并汇报进度")
public String processData(
        @ToolParam(name = "dataset_name", description = "数据集名") String datasetName,
        ToolEmitter emitter) {
    for (int i = 1; i <= 5; i++) {
        int pct = i * 20;
        emitter.emit(ToolResultBlock.text("已处理 " + pct + "%"));   // → TOOL_PROGRESS 事件
    }
    return datasetName + " 处理完成";
}
```

### 4.5 工具执行流程

```
模型返回 ToolUseBlock
        │
        ▼
PermissionEngine 门控（ALLOW / ASK / DENY）
        │
        ▼
Toolkit.callTool(ToolCallParam)
   ├── 参数反序列化与校验
   ├── 执行方法（可并发）
   └── 结果转换（ToolResultConverter）
        │
        ▼
ToolResultBlock → 回灌上下文 → 下一轮 reasoning
```

## 5. Message（消息）

### 5.1 按角色分类型

| 角色 | 类 | 构造 |
|------|-----|------|
| USER | `UserMessage` | `new UserMessage("你好")` |
| ASSISTANT | `AssistantMessage` | `new AssistantMessage("回复")` |
| SYSTEM | `SystemMessage` | `new SystemMessage("你是助手")` |
| TOOL | `ToolResultMessage` | 由框架生成 |

`MsgRole` 只有四个值：`USER`、`ASSISTANT`、`SYSTEM`、`TOOL`。

### 5.2 Msg 结构与 Builder

```java
Msg msg = Msg.builder()
        .id("uuid")
        .name("用户")
        .role(MsgRole.USER)
        .content(List.of(TextBlock.builder().text("你好").build()))
        .metadata(Map.of("traceId", "abc"))
        .timestamp("2024-01-15 10:00:00")
        .build();

// 常用读取
msg.getTextContent();       // 拼接所有 TextBlock
msg.getContent();           // List<ContentBlock>
msg.getRole();
msg.getMetadata();
msg.getGenerateReason();    // 结束原因
```

### 5.3 内容块类型

| 类型 | 说明 |
|------|------|
| `TextBlock` | 纯文本 |
| `ThinkingBlock` | 模型思考过程 |
| `ImageBlock` / `AudioBlock` / `VideoBlock` | 多模态（支持 `URLSource` / `Base64Source` / `FileIdSource`） |
| `DocumentBlock` | 文档 |
| `DataBlock` | 结构化数据块（v2 统一） |
| `ToolUseBlock` | 工具调用 |
| `ToolResultBlock` | 工具结果 |
| `HintBlock` | 提示块 |

### 5.4 多模态消息

```java
Msg imgMsg = new UserMessage(
        TextBlock.builder().text("这张图是什么？").build(),
        ImageBlock.builder()
                .source(new URLSource("https://example.com/a.jpg"))
                .build());
```

### 5.5 GenerateReason（结束原因）

`MODEL_STOP`、`TOOL_CALLS`、`STRUCTURED_OUTPUT`、`TOOL_SUSPENDED`、`HOST_PAUSED`、
`REASONING_STOP_REQUESTED`、`ACTING_STOP_REQUESTED`、`PERMISSION_ASKING`、
`MIDDLEWARE_STOP_REQUESTED`、`ALL_TOOLS_DENIED`、`INTERRUPTED`、`MAX_ITERATIONS`、
`TOOL_RETURN_DIRECT`。

> v1 文档里说的 `TOOL_SUSPENDED` / `MAX_ITERATIONS` 仍然存在，但完整枚举以上述为准。

## 6. Middleware（中间件）

### 6.1 为什么用 Middleware 替代 Hook？

v1 的 `Hook` 是扁平的"事件回调"，难以组合。v2 引入 **洋葱模型** 中间件：
每个切面拿到"后续链路"（`next`），可以在前、后做处理，甚至短路。

`io.agentscope.core.hook.Hook` 现为 `@Deprecated(forRemoval = true, since = "2.0.0")`，
仅为兼容保留。

### 6.2 六个切面

| 方法 | 时机 | 参数 |
|------|------|------|
| `onAgent(...)` | 整个 Agent 调用前后 | `AgentInput` |
| `onReasoning(...)` | 思考阶段前后 | `ReasoningInput` |
| `onActing(...)` | 行动（工具）阶段前后 | `ActingInput` |
| `onModelCall(...)` | 单次模型调用前后 | `ModelCallInput` |
| `onSystemPrompt(...)` | 生成系统提示词时 | `Mono<String>` |
| `onAgentStateReady(...)` | `AgentState` 就绪 | `void` |

另有 `onAgentFinish(...)`。使用 `activePoints()` 声明参与哪些切面，`order()` 控制顺序。

### 6.3 实现示例

```java
import io.agentscope.core.ReActAgent;
import io.agentscope.core.agent.Agent;
import io.agentscope.core.agent.RuntimeContext;
import io.agentscope.core.event.AgentEvent;
import io.agentscope.core.event.AgentEventType;
import io.agentscope.core.event.ToolCallStartEvent;
import io.agentscope.core.middleware.*;
import java.util.EnumSet;
import java.util.Set;
import java.util.function.Function;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

public class MonitoringMiddleware implements MiddlewareBase {

    /** 只在这几个切面参与（未声明的切面即使重写了也不会被调用） */
    @Override
    public Set<ExtensionPoint> activePoints() {
        return EnumSet.of(
                ExtensionPoint.ON_AGENT,
                ExtensionPoint.ON_REASONING,
                ExtensionPoint.ON_SYSTEM_PROMPT);
    }

    @Override
    public int order() {
        return 10;              // 越小越靠外层
    }

    @Override
    public Flux<AgentEvent> onAgent(
            Agent agent,
            RuntimeContext ctx,
            AgentInput input,
            Function<AgentInput, Flux<AgentEvent>> next) {
        long start = System.currentTimeMillis();
        return next.apply(input)
                .doOnComplete(
                        () -> System.out.println(
                                "agent 耗时 " + (System.currentTimeMillis() - start) + "ms"));
    }

    @Override
    public Flux<AgentEvent> onReasoning(
            Agent agent,
            RuntimeContext ctx,
            ReasoningInput input,
            Function<ReasoningInput, Flux<AgentEvent>> next) {
        return next.apply(input)
                .doOnNext(
                        event -> {
                            if (event.getType() == AgentEventType.TOOL_CALL_START) {
                                System.out.println("[tool] " + ((ToolCallStartEvent) event).getToolCallName());
                            }
                        });
    }

    @Override
    public Mono<String> onSystemPrompt(Agent agent, RuntimeContext ctx, String currentPrompt) {
        return Mono.just(currentPrompt + "\n今天日期：" + java.time.LocalDate.now());
    }
}

ReActAgent agent = ReActAgent.builder()
        .middleware(new MonitoringMiddleware())
        .build();
```

`ExtensionPoint` 的完整值：`ON_AGENT`、`ON_REASONING`、`ON_ACTING`、`ON_MODEL_CALL`、
`ON_SYSTEM_PROMPT`、`ON_AGENT_STATE_READY`、`ON_AGENT_FINISH`。

> 完整示例见 `documentation2/middleware/CustomizedMiddlewareExample.java`
> 与 `documentation2/hitl/InterruptionExample.java`。

## 7. Event（事件流）

`streamEvents()` 产出 `AgentEvent`，`AgentEventType` 共 **32 种**：

| 分组 | 事件 |
|------|------|
| 生命周期 | `AGENT_START`、`AGENT_END`、`AGENT_RESULT`、`CUSTOM` |
| 模型调用 | `MODEL_CALL_START`、`MODEL_CALL_END` |
| 文本 | `TEXT_BLOCK_START`、`TEXT_BLOCK_DELTA`、`TEXT_BLOCK_END` |
| 思考 | `THINKING_BLOCK_START`、`THINKING_BLOCK_DELTA`、`THINKING_BLOCK_END` |
| 数据块 | `DATA_BLOCK_START`、`DATA_BLOCK_DELTA`、`DATA_BLOCK_END` |
| 工具调用 | `TOOL_CALL_START`、`TOOL_CALL_DELTA`、`TOOL_CALL_END` |
| 工具结果 | `TOOL_RESULT_START`、`TOOL_PROGRESS`、`TOOL_RESULT_TEXT_DELTA`、`TOOL_RESULT_DATA_DELTA`、`TOOL_RESULT_END` |
| 人机交互 | `REQUIRE_USER_CONFIRM`、`USER_CONFIRM_RESULT`、`REQUIRE_EXTERNAL_EXECUTION`、`EXTERNAL_EXECUTION_RESULT` |
| 控制 | `EXCEED_MAX_ITERS`、`REQUEST_STOP`、`ALL_TOOLS_DENIED` |
| 其他 | `SUBAGENT_EXPOSED`、`HINT_BLOCK` |

```java
agent.streamEvents(new UserMessage("总结今天的新闻"))
        .doOnNext(event -> {
            switch (event.getType()) {
                case TEXT_BLOCK_DELTA -> System.out.print(((TextBlockDeltaEvent) event).getDelta());
                case THINKING_BLOCK_DELTA -> System.out.print(((ThinkingBlockDeltaEvent) event).getDelta());
                case TOOL_CALL_START -> System.out.println("\n[tool] " + ((ToolCallStartEvent) event).getToolCallName());
                case TOOL_PROGRESS -> System.out.println("\n[progress] " + event);
                case AGENT_RESULT -> System.out.println("\n[done] " + ((AgentResultEvent) event).getResult().getTextContent());
                default -> { }
            }
        })
        .blockLast();
```

## 8. Session / RuntimeContext（会话与运行时上下文）

### 8.1 RuntimeContext

会话标识与跨层传递的上下文，键包括：

```java
RuntimeContext ctx = RuntimeContext.builder()
        .sessionId("demo")
        .userId("alice")
        .threadId("thread-1")
        .turnId("turn-1")
        .runId("run-1")
        .put("tenant", "acme")               // 自定义属性
        .build();

agent.call(new UserMessage("你好"), ctx).block();
```

| 字段 | 作用 |
|------|------|
| `sessionId` | 会话隔离维度（状态存储主键之一） |
| `userId` | 用户隔离维度 |
| `threadId` / `turnId` | 线程/轮次标识（Durable、子 Agent 使用） |
| `runId` | 单次运行标识 |
| `agentState` | 直接挂载 `AgentState` |

### 8.2 会话隔离的三层维度

```
org / tenant  →  userId  →  sessionId  →  threadId
```

多租户场景下，`RuntimeContext` 的键会贯穿工作区路径、KV 命名空间与沙箱状态槽位。

## 9. Permission（权限）

```java
import io.agentscope.core.permission.*;

// PermissionRule 是 record：
// (toolName, ruleContent, behavior, source)
PermissionContextState permCtx =
        PermissionContextState.builder()
                .mode(PermissionMode.DEFAULT)          // 无人值守时用 DONT_ASK
                .addAllowRule(
                        "safe_read",
                        new PermissionRule(
                                "safe_read", null, PermissionBehavior.ALLOW, "policy"))
                .addAskRule(
                        "dangerous_delete",
                        new PermissionRule(
                                "dangerous_delete", null, PermissionBehavior.ASK, "policy"))
                .build();

ReActAgent agent = ReActAgent.builder()
        .permissionContext(permCtx)
        .build();
```

| 行为 | 结果 |
|------|------|
| `PermissionBehavior.ALLOW` | 直接执行 |
| `PermissionBehavior.ASK` | 暂停，`generateReason == PERMISSION_ASKING`，等人工确认 |
| `PermissionBehavior.DENY` | 拒绝执行 |

`PermissionMode` 取值：`DEFAULT`、`ACCEPT_EDITS`、`EXPLORE`、`BYPASS`、`DONT_ASK`。

完整流程见 `documentation2/hitl/PermissionHITLExample.java`。

## 10. Skill（技能）

技能是「按需加载的行为说明书」（Markdown），可在多个会话与 Agent 间共享。
推荐路径是**技能仓库（`AgentSkillRepository`）**，不是 `SkillBox`。

### 10.1 技能文件长什么样

```markdown
---
name: code-reviewer
description: Use when the user asks for code review, style feedback, or PR audits.
---

# Code Reviewer

Steps:
1. Read `references/style-guide.md` for project conventions.
2. Run `scripts/run-checks.sh <target-path>` and summarize the output.
```

front-matter 的 `name` / `description` 决定模型何时选中它；正文是执行步骤。
技能可携带 `references/`、`scripts/` 等资源文件。

### 10.2 推荐用法：`skillRepository(...)`

```java
// 本地目录
HarnessAgent agent = HarnessAgent.builder()
        .name("assistant")
        .model("dashscope:qwen-plus")
        .workspace(workspace)
        .skillRepository(new FileSystemSkillRepository(skillsDir, false))
        .build();

// Git 仓库（agentscope-extensions-skill-git-repository）
HarnessAgent.builder()
        .skillRepository(new GitSkillRepository("https://github.com/your-org/team-skills.git"))
        .build();

// 多个仓库按优先级叠加（后加的优先级更高，同名技能覆盖前面）
HarnessAgent.builder()
        .skillRepositories(List.of(classpathRepo, gitRepo))
        .skillFilter(myFilter)          // 可选：细粒度过滤
        .build();
```

注册至少一个仓库后，`DynamicSkillMiddleware` 会自动装配，每次 `call()` 重建技能提示词。
推理过程中模型通过 `load_skill_through_path` 按需加载具体技能。

**内置仓库实现**：`FileSystemSkillRepository`、`ClasspathSkillRepository`、
`RuntimeContextSkillRepository`；扩展模块提供 `GitSkillRepository`（Git）、
`MysqlSkillRepository` / `PostgresqlSkillRepository`（数据库）、`NacosSkillRepository`（Nacos）。

### 10.3 旧路径：`SkillBox`（不推荐新代码使用）

> `SkillBox` 类的 Javadoc 已声明「Since 2.0.0, SkillBox is intended for internal use only;
> prefer skill repositories」，`Builder.skillBox(...)` 标注了 `@Deprecated(since = "2.0.0")`
> （软废弃，暂无 `forRemoval`）。示例源码注释里直接写：`skillRepository(...)`
> 「replaces `.skillBox(skillBox)`」。

```java
// 仅存量代码会这样写
SkillBox skillBox = new SkillBox(toolkit);
skillBox.registration().skill(skill).apply();

ReActAgent agent = ReActAgent.builder()
        .skillBox(skillBox)          // @Deprecated(since = "2.0.0")
        .build();
```

### 10.4 相关类

`AgentSkill`（技能数据）、`SkillRegistry`、`SkillFilter`、`SkillFileFilter`、
`DynamicSkillMiddleware`（动态重建提示词）、`SkillToolFactory`、
`SkillHook`（`@Deprecated(since = "2.0.0")`）。

## 11. Workspace / Sandbox（工作区与沙箱）

属于 `HarnessAgent` 能力：

```java
HarnessAgent agent = HarnessAgent.builder()
        .name("assistant")
        .agentId("assistant")
        .model("dashscope:qwen-plus")
        .workspace(Paths.get(".agentscope/workspace"))   // 人格/记忆/技能/子 Agent 落盘
        .build();
```

- **文件系统**：`LocalFilesystemSpec`、`RemoteFilesystemSpec`、`SandboxFilesystemSpec`；
  `filesystemRoute(prefix, fs)` 可按路径前缀路由
- **沙箱**：本地子进程 / Docker / Kubernetes / 云沙箱，支持快照与恢复
- **压缩**：`compaction(CompactionConfig)`、`disableCompaction()`
- **工具结果卸载**：`toolResultEviction(ToolResultEvictionConfig)`

## 12. RAG（检索增强生成）

### 12.1 RAG 的三个步骤

RAG 要跑通，三步都得有人做。架构上的差异在于**第 ① ② 步由谁执行**：

```
① 检索 Retrieve   问题 → embedding / 关键词 → 向量库或检索服务 → 召回 Top-K 文档
② 增强 Augment    召回内容 + 用户问题 → 拼成 prompt
③ 生成 Generate   prompt → LLM → 答案
```

### 12.2 三种编排位置

```
方案 A：框架层 · Generic（v1 主流）
┌──────────────────────────────────────────────────────┐
│  ReActAgent                                          │
│    PreReasoningEvent ──► GenericRAGHook              │
│                          ├── ① 检索 Knowledge        │
│                          └── ② 注入 user message     │
│    → Model → ③ 生成                                  │
└──────────────────────────────────────────────────────┘
业务只给一个 Knowledge 对象，检索对业务不可见

方案 B：框架层 · Agentic（模型自主检索）
┌──────────────────────────────────────────────────────┐
│  ReActAgent                                          │
│    Model ──"我想查一下"──► @Tool retrieve_knowledge  │
│           ◄── 召回结果 ──                            │
│    → Model → ③ 生成                                  │
└──────────────────────────────────────────────────────┘
是否检索由模型决定，多一轮工具调用往返

方案 C：应用层（v2 推荐）
┌────────────────────────────┐   ┌────────────────────┐
│ 你的应用代码                │   │   ReActAgent       │
│  ① retriever.retrieve(q)    │   │   （不知道 RAG 存在）│
│  ② buildPrompt(q, hits)  ───┼──►│ 用户消息 = prompt  │
│                             │   │  ③ Model → 答案    │
└────────────────────────────┘   └────────────────────┘
```

### 12.3 三种方案对照

| | A. Generic Hook | B. Agentic Tool | **C. 应用层（v2 推荐）** |
|---|---|---|---|
| 谁检索 | 框架（每轮自动） | 模型决策 | **你的代码** |
| 谁拼 prompt | 框架 | 框架 | **你的代码** |
| Agent 是否感知 RAG | 是（内部 hook） | 是（多一个工具） | **完全不知情** |
| 检索时机可控 | ❌ 每轮都查 | ⚠️ 模型决定 | ✅ 想查才查 |
| 可单测 | ⚠️ 难 | ⚠️ 难 | ✅ 检索是纯函数 |
| 换向量库 / 检索策略 | 要动依赖 | 要动依赖 | ✅ 只改自己的 retriever |
| 多租户、权限过滤 | 夹在框架里 | 夹在框架里 | ✅ 天然在业务层 |
| 额外 token 开销 | 每轮都注入 | 工具往返 | ✅ 只在需要时注入一次 |
| 起步成本 | 最低 | 低 | 略高（需自写检索） |

**v2 推荐应用层的理由**：

1. **检索质量是业务问题**——chunk 策略、rerank、元数据过滤、租户隔离都与数据强相关，框架猜不准
2. **可观测**——"到底召回了什么"必须是你能直接打印出来的东西，而不是框架内部行为
3. **可测试**——`retriever.retrieve(q, 2)` 是纯函数，能写单测；隐式 hook 只能端到端测
4. **不锁死**——检索可能在 ES、PgVector、RAGFlow 服务、甚至一条 SQL 里，框架只需接受一个字符串
5. **与无状态设计一致**——Agent "输入什么处理什么"，注入逻辑放在外面才干净

> 注意：「应用层」指的是**编排位置**，不是拒绝框架组件。你完全可以在应用层调用
> `SimpleKnowledge` / `PgVectorStore` / `DashScopeTextEmbedding`，只是调用时机和 prompt
> 组装由你掌控。

### 12.4 方案 C 代码（应用层，v2 推荐）

```java
// ① 检索（你自己的 retriever：embedding + 向量库，或关键词检索）
List<KnowledgeDocument> matches = retriever.retrieve(question, 2);

// ② 增强（你自己的 prompt 模板）
String context = matches.stream()
        .map(d -> "[" + d.id() + "]\n" + d.content())
        .collect(Collectors.joining("\n\n"));
String prompt = """
        Retrieved context:
        %s

        User question:
        %s
        """.formatted(context, question);

// ③ 生成（Agent 是普通 ReActAgent，没有任何 RAG 配置）
ReActAgent agent =
        ReActAgent.builder()
                .name("SupportAssistant")
                .sysPrompt("Answer only with the retrieved context. "
                        + "If it does not contain the answer, say that you do not know.")
                .model("dashscope:qwen-plus")
                .toolkit(new Toolkit())
                .build();

agent.streamEvents(new UserMessage(prompt))       // ← 传的是拼好的 prompt
        .doOnNext(event -> {
            if (event instanceof TextBlockDeltaEvent e) {
                System.out.print(e.getDelta());
            }
        })
        .blockLast();
```

完整可运行示例（含内存关键词检索器，无需向量库）：
`documentation2/rag/ApplicationLayerRagExample.java`。

### 12.5 方案 A / B 代码（框架层，1.x 兼容路径）

**A. Generic：自动检索并注入**

```java
import io.agentscope.core.rag.Knowledge;
import io.agentscope.core.rag.RAGMode;

ReActAgent agent = ReActAgent.builder()
        .knowledge(knowledge)       // DifyKnowledge / RAGFlowKnowledge / ...
        .ragMode(RAGMode.GENERIC)   // GenericRAGHook 拦截 PreReasoningEvent
        .build();
```

`io.agentscope.core.rag.GenericRAGHook` 的行为：拦截 `PreReasoningEvent` →
从用户消息提取 query → 检索 → 把召回内容作为一条 user message 注入 → 继续推理。

**B. Agentic：检索变成工具，由模型决定何时调用**

```java
import io.agentscope.core.rag.KnowledgeRetrievalTools;

Toolkit toolkit = new Toolkit();
toolkit.registerTool(new KnowledgeRetrievalTools(knowledge));  // 提供 @Tool retrieve_knowledge
```

工具签名：`retrieve_knowledge(query, limit=5)`，返回格式化后的文档与分数。

> `Knowledge` 与 `RAGMode` 均标记为 `@Deprecated(forRemoval = true, since = "2.0.0")`，
> 代表 A / B 属于兼容路径；扩展模块
> `agentscope-extensions-rag-{simple,dify,ragflow,haystack,bailian}` 中的
> `SimpleKnowledge` / `DifyKnowledge` / `RAGFlowKnowledge` / `HayStackKnowledge` /
> `BailianKnowledge` 仍实现该接口。

## 13. 响应式编程

框架使用 Project Reactor：`Mono`（0/1 个元素）与 `Flux`（0..N 个元素）。

```java
// 阻塞
Msg reply = agent.call(new UserMessage("hi")).block();

// 异步回调
agent.call(new UserMessage("hi")).subscribe(m -> System.out.println(m.getTextContent()));

// 链式
agent.call(new UserMessage("hi"))
     .map(Msg::getTextContent)
     .filter(t -> t.length() > 10)
     .subscribe(System.out::println);

// 错误处理
agent.call(new UserMessage("hi"))
     .onErrorResume(e -> Mono.just(new AssistantMessage("抱歉，出错了")))
     .subscribe();

// 事件流
agent.streamEvents(new UserMessage("hi"))
     .take(Duration.ofSeconds(30))
     .blockLast();
```

**不要阻塞响应式链**：阻塞调用要包在 `Mono.fromCallable(...).subscribeOn(Schedulers.boundedElastic())` 里。

## 14. 已废弃 API 清单（将被移除）

官方迁移指南（`docs/dev/en/docs/change-log.md`）把 v1→v2 的变化分成两档：

- **Part A · Required** —— 不迁移会编译失败或运行时抛错
- **Part B · Recommended** —— 仍可用但 `@Deprecated(forRemoval = true)`，
  原文措辞是 **"will be removed in the next minor"**

### 14.1 `forRemoval = true`（确定会删）

| 范围 | 已废弃 | 用这个替代 |
|------|--------|-----------|
| Hook | `io.agentscope.core.hook` **整个包**：`Hook`、`HookEvent`、`HookEventType`、全部 `*Event` | `MiddlewareBase` 的 7 个 `ExtensionPoint` |
| Builder | `.hook(...)` / `.hooks(...)` | `.middleware(...)` / `.middlewares(...)` |
| Memory | `Memory`、`InMemoryMemory`、`StateBackedMemory`、`AgentStateMemoryView` | `AgentState` + `AgentStateStore` |
| Builder | `.memory(Memory)` | `.stateStore(...)` + `.defaultSessionId(...)` |
| 长期记忆 | `LongTermMemory`、`LongTermMemoryMode`、`LongTermMemoryTools`、`StaticLongTermMemoryHook` | 外部存储 + 应用层注入；或 `AgentState.summary` + 压缩 |
| Builder | `.longTermMemory(...)` / `.longTermMemoryMode(...)` / `.longTermMemoryAsyncRecord(...)` | 同上 |
| RAG | `Knowledge`、`KnowledgeRetrievalTools`、`RAGMode`、`GenericRAGHook`、`Document`、`DocumentMetadata`、`RetrieveConfig` | 应用层 RAG（见第 12 节） |
| Builder | `.knowledge(...)` / `.knowledges(...)` / `.ragMode(...)` / `.retrieveConfig(...)` | 同上 |
| 流式 | `StreamableAgent#stream(...)` 全部重载、`agent.Event` / `EventType` | `streamEvents(...)` → `Flux<AgentEvent>` |
| 追踪 | `io.agentscope.core.tracing` 下的 `Tracer` / `TracerRegistry` 等 | `io.agentscope.extensions.studio` 中间件 |

统计：`agentscope-core` + `agentscope-harness` 主源码共 **88 处** `forRemoval = true`
（`agent` 23、`hook` 20、`memory` 8、`rag` 7、`tracing` 3，其余为 `ReActAgent` 上的 Builder 方法）。

### 14.2 软废弃（暂无 `forRemoval`）

| 范围 | 标注 |
|------|------|
| `Builder.skillBox(SkillBox)` | `@Deprecated(since = "2.0.0")` → 用 `.skillRepository(...)` |
| `SkillHook` | `@Deprecated(since = "2.0.0")` |
| `McpTool` 部分方法 | `@Deprecated(since = "2.0.0")` |
| `agent.Event` / `EventType` / `EventSource` | `@Deprecated(since = "2.0.0")`（无 `forRemoval`） |
| `Builder.checkRunning(boolean)` | `@Deprecated`，参数被忽略 |

> **注意**：迁移指南里写 `SkillBox` 类本身是 `forRemoval`，但**当前源码里类上没有该注解**，
> 只有 Javadoc 提示「intended for internal use only」。以源码为准，按「不推荐新代码使用」处理。

### 14.3 已彻底移除（无过渡）

- `io.agentscope.core.plan` 整个包（`PlanNotebook`、`Plan`、`SubTask`、`PlanStorage`、`PlanToHint`）
- `ReActAgent.Builder.memory(...)` / `.statePersistence(...)` / `.structuredOutputReminder(...)`

### 14.4 怎么自查

```bash
# 编译时打开 lint，会打印所有 deprecated 用法
mvn -Dmaven.compiler.showDeprecation=true -Dmaven.compiler.showWarnings=true compile

# 或直接搜索
grep -rn "forRemoval = true" agentscope-core/src/main --include=*.java
```

升级前建议先把 `forRemoval` 的用法全部清零，避免下一个小版本直接编译失败。

## 15. 下一步

- [示例代码](./04-examples.md)：全部示例清单
- [开发指南](./05-development.md)：扩展开发与规范
