# 示例代码说明

本文档介绍 `agentscope-examples` 下的**真实**示例结构与运行方式。

> **重要**：v1 文档中的 `agentscope-examples/quickstart` 模块、`ExampleUtils` 工具类、
> `HookExample`、`SessionExample`、`ToolGroupExample`（旧写法）、`RAGFlowRAGExample`
> 等路径/命名已变化。当前示例集中在 **`agentscope-examples/documentation`**，
> 包名统一为 `io.agentscope.examples.documentation2.*`。

## 0. 运行前准备

```bash
# 仓库根目录执行；先把所有模块装进本地仓库
mvn -B -T1 install -DskipTests

export DASHSCOPE_API_KEY=sk-xxxxxxxx      # Linux/macOS
# Windows CMD: set DASHSCOPE_API_KEY=sk-xxxxxxxx
```

统一运行方式：

```bash
mvn exec:java -pl agentscope-examples/documentation \
    -Dexec.mainClass=io.agentscope.examples.documentation2.<子包>.<类名>
```

## 1. 示例总览

| 子包 | 示例 | 讲解要点 | 难度 |
|------|------|----------|------|
| `quickstart` | `BasicChatExample` | Agent 创建、`model("dashscope:qwen-plus")`、`streamEvents` | ⭐ |
| `quickstart` | `UserIsolatedMultiTurnsExample` | `RuntimeContext` 用户/会话隔离、多轮对话 | ⭐⭐ |
| `tool` | `ToolCallingExample` | `@Tool` / `@ToolParam`、`Toolkit.registerTool` | ⭐⭐ |
| `tool` | `ToolGroupExample` | `createToolGroup` + `enableMetaTool` 自动激活工具组 | ⭐⭐⭐ |
| `tool` | `ToolBaseExample` | 实现 `ToolBase` 接口定义工具 | ⭐⭐ |
| `tool` | `ToolCallingWithConverterExample` | 自定义 `ToolResultConverter` | ⭐⭐ |
| `tool` | `ToolExecutionContextExample` | `ToolExecutionContext` 透传 | ⭐⭐ |
| `tool` | `ToolSearchExample` | 工具搜索 / 延迟加载 | ⭐⭐⭐ |
| `tool` | `RoutingByToolCallsExample` | 依据工具调用做路由 | ⭐⭐⭐ |
| `tool` | `PermissionContextExample` | 权限上下文基础用法 | ⭐⭐ |
| `structuredoutput` | `StructuredOutputExample` | `call(msg, Class)` + `getStructuredData` | ⭐⭐ |
| `structuredoutput` | `StructuredOutputFallbackExample` | 结构化输出降级策略 | ⭐⭐⭐ |
| `streaming` | `StreamingConsoleExample` | 控制台流式输出 | ⭐⭐ |
| `streaming` | `AgentEventStreamExample` | 32 种事件的完整遍历 | ⭐⭐ |
| `streaming` | `ToolStreamingExample` | 工具参数/结果增量流 | ⭐⭐⭐ |
| `streaming` | `StreamingWebExample` | SSE/Web 场景 | ⭐⭐⭐ |
| `state` | `StateExample` | `JsonFileAgentStateStore` + 会话恢复 | ⭐⭐ |
| `state` | `StateAutoSaveExample` | 状态自动保存/加载 | ⭐⭐ |
| `middleware` | `CustomizedMiddlewareExample` | `MiddlewareBase` 生命周期监控 | ⭐⭐ |
| `middleware` | `ModelCallMiddlewareExample` | 拦截单次模型调用 | ⭐⭐⭐ |
| `middleware` | `SystemPromptMiddlewareExample` | 改写系统提示词 | ⭐⭐ |
| `model` | `ModelRegistryExample` | `model("provider:id")` 解析 | ⭐⭐ |
| `model` | `ModelTransportExample` | 自定义 HTTP 传输/代理 | ⭐⭐⭐ |
| `mcp` | `McpStdioExample` | MCP stdio 子进程 | ⭐⭐⭐ |
| `mcp` | `McpSseExample` | MCP SSE 传输 | ⭐⭐⭐ |
| `mcp` | `McpStreamableHttpExample` | MCP Streamable HTTP | ⭐⭐⭐ |
| `multimodal` | `VisionExample` | 图片输入 | ⭐⭐ |
| `multimodal` | `DocumentExample` | 文档输入 | ⭐⭐ |
| `multimodal` | `MultiModalToolExample` | 多模态工具返回 | ⭐⭐⭐ |
| `hitl` | `InterruptionExample` | `agent.interrupt()` + 中间件观测 | ⭐⭐⭐ |
| `hitl` | `PermissionHITLExample` | 三态权限引擎 + 人工确认 + `DONT_ASK` 无头模式 | ⭐⭐⭐ |
| `hitl` | `HookStopAgentExample` | 在流程中主动停止 Agent | ⭐⭐⭐ |
| `skill` | `AgentSkillExample` | `FileSystemSkillRepository` + `.skillRepository(...)` | ⭐⭐ |
| `skill` | `SkillWithToolGroupExample` | 技能绑定工具组 | ⭐⭐⭐ |
| `skill` | `GitSkillRepositoryExample` | Git 仓库技能 | ⭐⭐⭐ |
| `rag` | `ApplicationLayerRagExample` | **v2 推荐**：应用层检索 + 拼 prompt | ⭐⭐ |
| `context` | `RuntimeContextExample` | `RuntimeContext` 各字段作用 | ⭐⭐ |
| `judge` | `OpenAIDecisionsExample` 等 5 个 | 文本/图像分类与路由中间件 | ⭐⭐⭐ |
| `harness/*` | 13 个示例 | 工作区、记忆压缩、Plan Mode、子 Agent、Channel、异步工具 | ⭐⭐⭐ |
| `agui` | `AguiExampleApplication` | AG-UI 协议（Spring Boot 应用） | ⭐⭐⭐ |
| `copilotkit` | `CopilotKitExampleApplication` | CopilotKit 集成（Spring Boot 应用） | ⭐⭐⭐ |

其他子模块：

| 模块 | 内容 |
|------|------|
| `agentscope-examples/jev` | JEV 评估/判定示例与基准程序 |
| `agentscope-examples/durable` | Durable 集成（含 Spring Boot） |
| `agentscope-examples/agents/agentscope-chat` | HarnessAgent + Durable 可恢复聊天应用，含前端 |
| `agentscope-examples/agents/agentscope-codingagent` | 编码 Agent |
| `agentscope-examples/agents/agentscope-dataagent` | 数据分析 Agent |
| `agentscope-examples/agents/agentscope-paw` | 示例 Agent 应用 |

---

## 2. BasicChatExample（基础对话）

### 功能
最简单的 Agent：创建 → 流式对话 → 增量打印文本。

### 核心代码（真实源码）

```java
ReActAgent agent =
        ReActAgent.builder()
                .name("Assistant")
                .sysPrompt("You are a helpful AI assistant. Be friendly and concise.")
                .model("dashscope:qwen-plus")
                .toolkit(new Toolkit())
                .build();

Msg userMsg = new UserMessage(input.trim());
agent.streamEvents(userMsg)
        .doOnNext(event -> {
            if (event instanceof TextBlockDeltaEvent e) {
                System.out.print(e.getDelta());
            }
        })
        .blockLast();
```

### 运行

```bash
mvn exec:java -pl agentscope-examples/documentation \
    -Dexec.mainClass=io.agentscope.examples.documentation2.quickstart.BasicChatExample
```

### 学习要点
1. `model("dashscope:qwen-plus")` 由 `ModelRegistry` 自动解析 provider
2. **不需要** `.memory(...)`
3. 流式输出用 `streamEvents()` + `TextBlockDeltaEvent`

### 调用链

```
main()
  └── ReActAgent.builder()...build()
  └── agent.streamEvents(UserMessage)
        └── ReActAgent.CallExecution.doCallInner()
              └── reasoning()  → Model.stream() → Flux<ChatResponse>
              └── acting()     → Toolkit.callTool()（如有工具调用）
        └── Flux<AgentEvent> → TextBlockDeltaEvent.getDelta()
```

---

## 3. ToolCallingExample（工具调用）

### 核心代码

```java
Toolkit toolkit = new Toolkit();
toolkit.registerTool(new SimpleTools());

ReActAgent agent =
        ReActAgent.builder()
                .name("ToolAgent")
                .sysPrompt("You are a helpful assistant with access to tools. "
                        + "Use tools when needed to answer questions accurately.")
                .model("dashscope:qwen-max")
                .toolkit(toolkit)
                .middleware(new AgentTraceMiddleware())
                .build();
```

工具定义：

```java
public static class SimpleTools {

    @Tool(name = "get_current_time", description = "获取指定时区当前时间")
    public String getCurrentTime(
            @ToolParam(name = "timezone", description = "时区，如 Asia/Tokyo")
            String timezone) {
        return ZonedDateTime.now(ZoneId.of(timezone))
                .format(DateTimeFormatter.ISO_ZONED_DATE_TIME);
    }

    @Tool(name = "calculate", description = "计算简单数学表达式")
    public String calculate(@ToolParam(name = "expression") String expression) { ... }

    @Tool(name = "search", description = "模拟搜索")
    public String search(@ToolParam(name = "query") String query) { ... }
}
```

### 运行

```bash
mvn exec:java -pl agentscope-examples/documentation \
    -Dexec.mainClass=io.agentscope.examples.documentation2.tool.ToolCallingExample
```

### 学习要点
- 一个 `@Tool` 方法 = 一个工具；`description` 是模型选择工具的唯一依据，务必写清"何时使用"
- 支持 `Mono<String>` 异步返回、`ToolEmitter` 进度流、`converter` 自定义转换

---

## 4. ToolGroupExample（工具组 + 元工具）

### 核心代码

```java
Toolkit toolkit = new Toolkit();

toolkit.createToolGroup("file_ops", "File system operations (read, write, list)", false);
toolkit.registration().tool(new FileTools()).group("file_ops").apply();

toolkit.createToolGroup("math_ops", "Mathematical calculations (factorial, is_prime)", false);
toolkit.registration().tool(new MathTools()).group("math_ops").apply();

toolkit.createToolGroup("network_ops", "Network operations (ping, dns_lookup)", false);
toolkit.registration().tool(new NetworkTools()).group("network_ops").apply();

ReActAgent agent = ReActAgent.builder()
        .name("GroupAgent")
        .model("dashscope:qwen-plus")
        .toolkit(toolkit)
        .enableMetaTool(true)     // 让模型自己开关工具组
        .build();
```

### 学习要点
1. 组默认 `active=false`，初始对模型不可见
2. `enableMetaTool(true)` 后，模型通过 `reset_equipped_tools` 元工具激活需要的组
3. 也可用 `toolkit.setActiveGroups(List.of("math_ops"))` 由代码激活

### 运行

```bash
mvn exec:java -pl agentscope-examples/documentation \
    -Dexec.mainClass=io.agentscope.examples.documentation2.tool.ToolGroupExample
```

---

## 5. StructuredOutputExample（结构化输出）

### 核心代码（真实源码）

```java
Msg msg = agent.call(userMsg, ProductRequirements.class).block();
ProductRequirements result = msg.getStructuredData(ProductRequirements.class);

System.out.println(result.productType);
System.out.println(result.minRam);
```

`ProductRequirements` 可以是普通类或 record，框架自动生成 JSON Schema 并做校验/重试。

### 运行

```bash
mvn exec:java -pl agentscope-examples/documentation \
    -Dexec.mainClass=io.agentscope.examples.documentation2.structuredoutput.StructuredOutputExample
```

### 学习要点
- `call(msgs, Class)` → `Msg.getStructuredData(Class)`
- 也支持 `call(msgs, JsonNode schema)` 传自定义 Schema
- 模型不支持原生结构化输出时，框架会走提示词提醒 + 校验重试（见 `StructuredOutputReminder`）

---

## 6. Middleware（中间件）示例

### 三个示例对比

| 示例 | 切片 | 用途 |
|------|------|------|
| `CustomizedMiddlewareExample` | `onAgent` / `onReasoning` / `onActing` | 全生命周期日志 |
| `ModelCallMiddlewareExample` | `onModelCall` | 拦截模型请求/响应 |
| `SystemPromptMiddlewareExample` | `onSystemPrompt` | 动态改写系统提示词 |

### 核心代码

```java
static class MonitoringMiddleware implements MiddlewareBase {

    @Override
    public Set<ExtensionPoint> activePoints() {
        return EnumSet.of(
                ExtensionPoint.ON_AGENT, ExtensionPoint.ON_REASONING, ExtensionPoint.ON_ACTING);
    }

    @Override
    public Flux<AgentEvent> onAgent(
            Agent agent, RuntimeContext ctx, AgentInput input,
            Function<AgentInput, Flux<AgentEvent>> next) {
        System.out.println("[MW] onAgent START, messages=" + input.msgs().size());
        return next.apply(input).doOnComplete(() -> System.out.println("[MW] onAgent END"));
    }
}
```

### 运行

```bash
mvn exec:java -pl agentscope-examples/documentation \
    -Dexec.mainClass=io.agentscope.examples.documentation2.middleware.CustomizedMiddlewareExample
```

> **迁移提示**：v1 的 `HookExample` 已删除，等价能力见
> `CustomizedMiddlewareExample`（其 Javadoc 里列出了 Hook → Middleware 的逐条映射）。

---

## 7. StateExample（会话持久化）

### 核心代码（真实源码）

```java
Path sessionPath =
        Paths.get(System.getProperty("user.home"), ".agentscope", "examples", "sessions");
AgentStateStore stateStore = new JsonFileAgentStateStore(sessionPath);

ReActAgent agent =
        ReActAgent.builder()
                .name("Assistant")
                .sysPrompt("You are a helpful AI assistant with persistent memory.")
                .toolkit(new Toolkit())
                .stateStore(stateStore)          // 自动加载/保存
                .defaultSessionId(sessionId)
                .model("dashscope:qwen-max")
                .build();

// 读取已恢复的历史
int ctxSize = agent.getAgentState().getContext().size();
```

### 与 v1 的差异（示例 Javadoc 原文）

| v1 | v2 |
|---|---|
| `legacy.session.JsonFileAgentStateStore` | `io.agentscope.core.state.JsonFileAgentStateStore` |
| `legacy.memory.InMemoryMemory` + `.memory()` | 完全移除，历史在 `AgentState` |
| `agent.loadFrom(session, id)` / `saveTo(...)` | Builder 上 `.stateStore(store)` + `.defaultSessionId(id)`，自动读写 |
| `agent.getMemory().getMessages()` | `agent.getAgentState().getContext()` |

### 运行

```bash
mvn exec:java -pl agentscope-examples/documentation \
    -Dexec.mainClass=io.agentscope.examples.documentation2.state.StateExample
```

---

## 8. Streaming（事件流）示例

### 核心代码

```java
agent.streamEvents(userMsg)
        .doOnNext(event -> {
            switch (event.getType()) {
                case TEXT_BLOCK_DELTA ->
                        System.out.print(((TextBlockDeltaEvent) event).getDelta());
                case THINKING_BLOCK_DELTA ->
                        System.out.print(((ThinkingBlockDeltaEvent) event).getDelta());
                case TOOL_CALL_START ->
                        System.out.printf("%n[TOOL] %s%n",
                                ((ToolCallStartEvent) event).getToolCallName());
                case TOOL_PROGRESS -> System.out.printf("%n[PROGRESS] %s%n", event);
                case AGENT_RESULT ->
                        System.out.printf("%n[DONE] %s%n",
                                ((AgentResultEvent) event).getResult().getTextContent());
                default -> { }
            }
        })
        .blockLast();
```

### 运行

```bash
mvn exec:java -pl agentscope-examples/documentation \
    -Dexec.mainClass=io.agentscope.examples.documentation2.streaming.AgentEventStreamExample
```

---

## 9. MCP 集成（3 种传输）

| 示例 | 传输 | 依赖 |
|------|------|------|
| `McpStdioExample` | 子进程 stdin/stdout | `npx @modelcontextprotocol/server-filesystem` |
| `McpSseExample` | SSE | 远程 MCP 服务 |
| `McpStreamableHttpExample` | Streamable HTTP | 远程 MCP 服务 |

### 核心代码（真实源码）

```java
McpClientWrapper mcpClient =
        McpClientBuilder.create("filesystem")
                .stdioTransport("npx", "-y", "@modelcontextprotocol/server-filesystem", "/tmp")
                .buildAsync()
                .block();

Toolkit toolkit = new Toolkit();
toolkit.registerMcpClient(mcpClient).block();

ReActAgent agent = ReActAgent.builder()
        .name("FilesystemAgent")
        .sysPrompt("You are a filesystem assistant. Use the available MCP tools.")
        .model("dashscope:qwen-max")
        .toolkit(toolkit)
        .build();
```

### 运行

```bash
npm install -g @modelcontextprotocol/server-filesystem

mvn exec:java -pl agentscope-examples/documentation \
    -Dexec.mainClass=io.agentscope.examples.documentation2.mcp.McpStdioExample
```

> **注意**：v1 文档里的 `McpClient.builder().command(...)` + `toolkit.registerMcpTools(client)`
> 已不存在；现在是 `McpClientBuilder.create(name).stdioTransport(...)` +
> `toolkit.registerMcpClient(wrapper)`（返回 `Mono<Void>`）。

---

## 10. HITL（人机协同）示例

### PermissionHITLExample

```java
PermissionContextState permCtx =
        PermissionContextState.builder()
                .mode(PermissionMode.DEFAULT)
                .addAllowRule("safe_read",
                        new PermissionRule("safe_read", null, PermissionBehavior.ALLOW, "policy"))
                .addAskRule("dangerous_delete",
                        new PermissionRule("dangerous_delete", null, PermissionBehavior.ASK, "policy"))
                .build();

ReActAgent agent = ReActAgent.builder()
        .permissionContext(permCtx)
        .build();
```

流程：
1. 用户消息触发需要确认的工具 → Agent 提前返回，`getGenerateReason() == PERMISSION_ASKING`
2. 应用展示待确认的工具调用并收集用户决定
3. 应用发送后续消息恢复或取消

```bash
# 交互模式（会询问确认）
mvn exec:java -pl agentscope-examples/documentation \
    -Dexec.mainClass=io.agentscope.examples.documentation2.hitl.PermissionHITLExample

# 无头模式（ASK 自动降级为 DENY）
mvn exec:java -pl agentscope-examples/documentation \
    -Dexec.mainClass=io.agentscope.examples.documentation2.hitl.PermissionHITLExample \
    -Dexec.args="--headless"
```

### InterruptionExample

```java
// 长任务执行中主动中断
agent.interrupt();
```

---

## 11. Skill（技能）示例

| 示例 | 说明 |
|------|------|
| `AgentSkillExample` | 用 `FileSystemSkillRepository` + `.skillRepository(...)` 加载技能（示例注释明确写着这是 `SkillBox` 的非 legacy 替代） |
| `SkillWithToolGroupExample` | 技能与工具组绑定，激活技能时按需启用工具 |
| `GitSkillRepositoryExample` | 从 Git 仓库拉取 `SKILL.md` 技能集 |

> `SkillBox` 属于旧路径：`Builder.skillBox(...)` 已 `@Deprecated(since = "2.0.0")`，
> 类 Javadoc 声明「intended for internal use only」。新代码统一用 `skillRepository(...)`。

技能以 Markdown 形式承载（含 front-matter 的 `name` / `description`），
加载方式：

```java
// 方式一：工作区技能（HarnessAgent 自动扫描 workspace/skills/<name>/SKILL.md）
HarnessAgent.builder()
        .workspace(workspace)
        .build();

// 方式二：显式仓库（推荐）
HarnessAgent.builder()
        .skillRepository(new ClasspathSkillRepository("skills"))
        .build();
```

---

## 12. RAG 示例

> RAG 有三种编排位置：**A. 框架层 Generic（自动 hook）**、**B. 框架层 Agentic（模型调工具）**、
> **C. 应用层（v2 推荐）**。三者的架构图、对照表与选型理由见
> [核心概念 - RAG](./03-concepts.md#12-rag检索增强生成)。本节只讲示例代码。

### ApplicationLayerRagExample（v2 推荐，即方案 C）

v2 把检索留在应用层，示例用三步显式表达：

```java
List<KnowledgeDocument> matches = retriever.retrieve(question, 2);

String context = matches.stream()
        .map(d -> d.id() + ": " + d.content())
        .collect(Collectors.joining("\n"));

ReActAgent agent = ReActAgent.builder()
        .name("SupportAssistant")
        .sysPrompt("You are a customer-support assistant. Answer only with the retrieved context.")
        .model("dashscope:qwen-plus")
        .build();

agent.streamEvents(new UserMessage("已知资料：\n" + context + "\n\n问题：" + question))
        .doOnNext(...)
        .blockLast();
```

示例用内存关键字检索，**不需要向量库或 embedding 服务**；生产环境替换
`KeywordRetriever.retrieve(...)` 即可，prompt 组装与 Agent 调用不变。

### 1.x 兼容：RAG 扩展模块

| 模块 | Knowledge 实现 |
|------|----------------|
| `agentscope-extensions-rag-simple` | `SimpleKnowledge`（自带 reader/splitter/embedding/向量库） |
| `agentscope-extensions-rag-dify` | `DifyKnowledge` |
| `agentscope-extensions-rag-ragflow` | `RAGFlowKnowledge` |
| `agentscope-extensions-rag-haystack` | `HayStackKnowledge` |
| `agentscope-extensions-rag-bailian` | `BailianKnowledge` |

```java
ReActAgent.builder()
        .knowledge(knowledge)        // Knowledge 接口在 2.0.0 起已 @Deprecated
        .ragMode(RAGMode.GENERIC)
        .build();
```

向量库支持：`InMemoryStore`、`PgVectorStore`、`MilvusStore`、`QdrantStore`、`ElasticsearchStore`；
Embedding 支持：DashScope（文本/多模态）、OpenAI、Ollama。

---

## 13. Harness 示例（13 个）

| 示例 | 演示内容 |
|------|----------|
| `harness/workspace/WorkspaceSetupExample` | 工作区目录初始化（人格/记忆/技能） |
| `harness/workspace/WorkspaceLocalShellExample` | 本地子进程 shell 工具 |
| `harness/workspace/WorkspaceSandboxExample` | 沙箱执行 |
| `harness/workspace/WorkspaceSharedStoreExample` | 共享存储文件系统 |
| `harness/memory/MemoryCompactionExample` | 分层记忆与上下文压缩 |
| `harness/context/ContextStatePersistenceExample` | 上下文状态持久化 |
| `harness/planmode/PlanModeManualExample` | 手动进入 Plan Mode |
| `harness/planmode/PlanModeAutoExample` | 自动 Plan Mode |
| `harness/subagent/SubagentStreamingExample` | 子 Agent 事件转发到父流 |
| `harness/subagent/SubagentSendDirectlyExample` | `agent_send` 直接投递 |
| `harness/async/AsyncToolExample` | 异步工具 |
| `harness/async/AsyncSubagentExample` | 异步子 Agent |
| `harness/async/AsyncWaitModeExample` | 异步等待模式 |
| `harness/channel/ChannelSendExample` | Channel 发送消息 |
| `harness/channel/GatewayMultiAgentExample` | 网关多 Agent 路由 |
| `harness/skill/SkillCompositionExample` | 多层技能组合（代码/classpath/workspace） |

### 子 Agent 声明（真实源码）

```java
HarnessAgent.builder()
        .name("orchestrator")
        .sysPrompt("You are an orchestrator. Spawn the researcher subagent to investigate.")
        .model("dashscope:qwen-plus")
        .subagent(
                SubagentDeclaration.builder()
                        .name("researcher")
                        .description("Research specialist. Use when in-depth investigation is needed.")
                        .inlineAgentsBody("You are a research assistant...")
                        .persistSession(true)
                        .build())
        .stateStore(stateStore)
        .build();
```

---

## 14. 完整应用示例

### agentscope-chat（HarnessAgent + Durable）

可恢复聊天应用：持久化任务、排队、暂停/恢复、交互应答、SSE 重放。

```bash
mvn -B -T1 -pl agentscope-examples/agents/agentscope-chat -am package -DskipTests
java -jar agentscope-examples/agents/agentscope-chat/target/agentscope-chat.jar
# http://127.0.0.1:8087
```

- 默认离线模型，**无需 API Key / 数据库 / Node.js**
- 默认数据目录 `~/.agentscope/session-chat`（`durable/durable.db` + `durable/objects/`）
- 可用 `CHAT_WORKSPACE` / `CHAT_PORT` 环境变量覆盖

### jev（评估/判定）

```bash
mvn -pl agentscope-dependencies-bom,agentscope-distribution/agentscope-bom,agentscope-examples/jev \
    -am install -DskipTests
mvn -pl agentscope-examples/jev dependency:build-classpath \
    -Dmdep.outputFile=jev-cp.txt
java -cp "agentscope-examples/jev/target/classes:$(cat jev-cp.txt)" \
     io.agentscope.examples.jev.JevHarnessExample
```

默认使用合成响应，不需要密钥；真实模型调用需显式 `--live-jev` / `--live-qwen` / `--compare`。

---

## 15. 已移除的示例（迁移对照）

| v1 示例 | 现状 |
|---------|------|
| `quickstart.BasicChatExample` | → `documentation2.quickstart.BasicChatExample` |
| `quickstart.ToolCallingExample` | → `documentation2.tool.ToolCallingExample` |
| `quickstart.StructuredOutputExample` | → `documentation2.structuredoutput.StructuredOutputExample` |
| `quickstart.HookExample` | → `documentation2.middleware.CustomizedMiddlewareExample` |
| `quickstart.SessionExample` | → `documentation2.state.StateExample` |
| `quickstart.ToolGroupExample` | → `documentation2.tool.ToolGroupExample` |
| `quickstart.McpToolExample` | → `documentation2.mcp.McpStdioExample` |
| `quickstart.InterruptionExample` | → `documentation2.hitl.InterruptionExample` |
| `PgVectorRAGExample` / `RAGFlowRAGExample` | → `documentation2.rag.ApplicationLayerRagExample` + RAG 扩展模块 |
| `SequentialPipelineExample` / `FanoutPipelineExample` | **不存在**；用 Subagent / Harness 编排替代 |
| `ExampleUtils` 工具类 | **已移除**，各示例自带输入输出逻辑 |

---

## 16. 调试技巧

### 16.1 打印事件流

```java
agent.streamEvents(msg)
        .doOnNext(e -> System.out.println(e.getType() + " -> " + e))
        .blockLast();
```

### 16.2 关键断点

```java
// ReActAgent（内部类 CallExecution）
doCallInner(List<Msg> msgs)                    // 整体调用入口
reasoning(int iter, boolean ignoreMaxIters)    // 每轮思考
acting(int iter)                               // 工具执行
```

```java
// Toolkit
callTool(ToolCallParam param)                  // 工具执行入口
```

### 16.3 查看消息

```java
Msg msg = agent.call(userMsg).block();
System.out.println("id=" + msg.getId());
System.out.println("role=" + msg.getRole());
System.out.println("text=" + msg.getTextContent());
System.out.println("blocks=" + msg.getContent().size());
System.out.println("reason=" + msg.getGenerateReason());
```

---

## 下一步

- [开发指南](./05-development.md)：开发规范与扩展开发
- [常见问题](./06-faq.md)：问题排查
