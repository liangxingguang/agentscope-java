# 常见问题 (FAQ)

本文档收集 AgentScope-Java（v2.x）使用过程中的常见问题与解决方案。

---

## 环境配置

### Q1: JDK 版本不对怎么办？

**现象**
```
[ERROR] invalid target release: 17
```

**解决**
1. 确认当前 JDK：
```bash
java -version      # 必须是 17
mvn -version       # 看 "Java version: 17"
```
2. 安装 JDK 17：
- Windows：[Eclipse Temurin](https://adoptium.net/) 或 Oracle JDK
- macOS：`brew install openjdk@17`
- Linux：`sudo apt install openjdk-17-jdk`
3. 配置环境变量：
```bash
export JAVA_HOME=/path/to/jdk-17
export PATH=$JAVA_HOME/bin:$PATH
```
Windows CMD：
```bat
set JAVA_HOME=C:\Program Files\Eclipse Adoptium\jdk-17
set PATH=%JAVA_HOME%\bin;%PATH%
```

> 项目编译目标固定为 17，**不能**用 JDK 11/8 构建；JDK 21 运行也需保持 `--release 17` 语义。

---

### Q2: Maven 下载依赖很慢怎么办？

编辑 `~/.m2/settings.xml`：

```xml
<settings>
    <mirrors>
        <mirror>
            <id>aliyun</id>
            <mirrorOf>central</mirrorOf>
            <name>Aliyun Maven</name>
            <url>https://maven.aliyun.com/repository/public</url>
        </mirror>
    </mirrors>
</settings>
```

---

### Q3: 编译时报 Spotless 错误怎么办？

**现象**
```
[ERROR] Failed to execute goal com.diffplug.spotless:spotless-maven-plugin ...
        spotless:check (default) on project agentscope-core
```

**解决**
```bash
mvn spotless:apply
```
Spotless 绑定在 `compile` 阶段，用 **AOSP 风格（4 空格）**。临时跳过：
```bash
mvn -Dspotless.check.skip=true compile
```

---

## API Key

### Q4: API Key 配置了还是报错？

1. 检查环境变量（**必须在启动进程的环境中**）：
```bash
# Linux/macOS
echo $DASHSCOPE_API_KEY
# Windows CMD
echo %DASHSCOPE_API_KEY%
# Windows PowerShell
echo $env:DASHSCOPE_API_KEY
```
2. 永久设置：
```bash
echo 'export DASHSCOPE_API_KEY=sk-xxxxxxxx' >> ~/.bashrc
source ~/.bashrc
```
3. IDEA 中：`Run` → `Edit Configurations` → `Environment variables` 添加
   `DASHSCOPE_API_KEY=sk-xxxxxxxx`
4. 确认引入对应模型扩展模块（`model("dashscope:...")` 需要
   `agentscope-extensions-model-dashscope` 在 classpath 上，否则会提示找不到 provider）。

---

### Q5: 如何获取 DashScope API Key？

1. 访问 [阿里云 DashScope 控制台](https://dashscope.console.aliyun.com/)
2. 登录/注册
3. 左侧 `API-KEY 管理` → `创建新的 API-KEY`
4. 复制保存（只显示一次）

---

## Agent 运行

### Q6: Agent 没有调用工具？

按顺序排查：

**1. 工具真的注册了吗？**
```java
toolkit.getToolNames().forEach(System.out::println);
```

**2. 工具组是不是没激活？**
```java
System.out.println(toolkit.getActiveGroups());
```
若工具加入了组且组默认 `active=false`，需要：
```java
toolkit.setActiveGroups(List.of("my_group"));
// 或 .enableMetaTool(true) 让模型自己开
```

**3. 工具描述是否说清"何时使用"？**
```java
@Tool(
    name = "get_weather",
    description = "获取指定城市的当前天气。当用户询问天气、气温、降雨时使用此工具。")
```

**4. 系统提示词是否提到工具有用？**
```java
.sysPrompt("你是一个助手，可以使用提供的工具来回答问题。")
```

**5. 模型是否支持 function calling？**
`qwen-plus`、`qwen-max`、`gpt-4o`、`claude-3-5-sonnet` 等都支持；小模型/本地模型可能不支持。

---

### Q7: Agent 响应为空？

```java
Msg response = agent.call(userMsg).block();

System.out.println("blocks = " + response.getContent().size());
System.out.println("text   = [" + response.getTextContent() + "]");
System.out.println("reason = " + response.getGenerateReason());
```

`getGenerateReason()` 的常见值：

| 值 | 含义 |
|----|------|
| `MODEL_STOP` | 正常结束 |
| `TOOL_CALLS` | 本轮以工具调用结束（还会继续） |
| `STRUCTURED_OUTPUT` | 结构化输出完成 |
| `TOOL_SUSPENDED` | 工具挂起，等待外部结果 |
| `PERMISSION_ASKING` | 等待人工确认（见 Q10） |
| `ALL_TOOLS_DENIED` | 所有工具被拒绝 |
| `INTERRUPTED` | 被中断 |
| `MAX_ITERATIONS` | 达到 `maxIters` 上限 |

> 历史文档里的 `MODEL_STOP / TOOL_SUSPENDED / MAX_ITERATIONS` 仍然存在，
> 但**完整枚举**以上表（`io.agentscope.core.message.GenerateReason`）为准。

---

### Q8: Agent 进入无限循环？

**1. 设置最大迭代次数**（`ReActConfig.DEFAULT_MAX_ITERS = 20`）：
```java
ReActAgent.builder().maxIters(10).build();
```

**2. 用中间件主动停止**

在 `onActing` 中发出 `RequestStopEvent`：
```java
@Override
public Flux<AgentEvent> onActing(
        Agent agent, RuntimeContext ctx, ActingInput input,
        Function<ActingInput, Flux<AgentEvent>> next) {

    boolean dangerous = input.toolCalls().stream()
            .anyMatch(t -> Set.of("delete_file", "send_email").contains(t.getName()));
    if (dangerous) {
        return Flux.just(new RequestStopEvent(
                "危险工具需要人工确认", GenerateReason.PERMISSION_ASKING));
    }
    return next.apply(input);
}
```

**3. 检查系统提示词**：明确"什么时候应该结束回答"。

**4. `stopOnReject(true)`**：工具被拒绝时直接停止，避免模型反复重试。

---

### Q9: 内存/上下文越来越大？

**1. Harness 自动压缩（推荐）**
```java
HarnessAgent.builder()
        .compaction(CompactionConfig.builder()
                .triggerMessages(80)
                .keepMessages(20)
                .build())
        .build();
```

**2. 工具结果卸载到磁盘**
```java
HarnessAgent.builder()
        .toolResultEviction(ToolResultEvictionConfig.builder()...build())
        .build();
```

**3. 自定义 `AgentStateStore` 在 `save` 时截断**
```java
@Override
public void save(String userId, String sessionId, String key, State value) {
    if (value instanceof AgentState s && s.getContext().size() > 100) {
        // 只保留最近 50 条
        ...
    }
    backend.save(userId, sessionId, key, value);
}
```

**4. 关掉不必要的历史**：用 `RuntimeContext` 分不同 `sessionId`，避免单个会话无限增长。

> v1 的 `memory.clear()` / `deleteMessage()` 属于已废弃的 `Memory` API，
> v2 请直接操作 `AgentState.context` 或使用压缩配置。

---

## 工具

### Q10: 工具参数解析错误？

```
错误: 无法解析参数 xxx
```

1. 所有参数都要有 `@ToolParam(name = ...)`：
```java
// ✅
public String calculate(
        @ToolParam(name = "a", description = "第一个数字") int a,
        @ToolParam(name = "b", description = "第二个数字") int b) { ... }

// ❌ 缺少注解，模型不知道参数名
public String calculate(int a, int b) { ... }
```

2. 参数类型用简单类型或可 JSON 反序列化的 POJO；避免 `Object`、泛型嵌套。
3. 在方法内做兜底校验：
```java
if (b == 0) {
    return "错误: 除数不能为 0";
}
```

---

### Q11: 工具执行超时 / 卡住？

**1. 配置超时与重试**
```java
ExecutionConfig toolConfig = ExecutionConfig.builder()
        .timeout(Duration.ofSeconds(30))
        .maxAttempts(2)
        .initialBackoff(Duration.ofMillis(200))
        .build();

ReActAgent.builder()
        .toolExecutionConfig(toolConfig)
        .build();
```

**2. 工具内自己也加超时**
```java
@Tool(name = "long_running", description = "长任务")
public Mono<String> longRunning(@ToolParam(name = "input") String input) {
    return Mono.fromCallable(() -> doWork(input))
            .timeout(Duration.ofSeconds(30))
            .onErrorReturn("操作超时");
}
```

**3. 标记工具属性**：`timeoutManaged = true` 表示工具自己管理超时。

---

## 模型

### Q12: 如何切换到 OpenAI GPT-4？

**1. 加依赖**
```xml
<dependency>
    <groupId>io.agentscope</groupId>
    <artifactId>agentscope-extensions-model-openai</artifactId>
    <version>2.0.3</version>
</dependency>
```

**2. 二选一**

模型标识符（推荐）：
```java
ReActAgent.builder().model("openai:gpt-4o").build();
```

显式构造：
```java
import io.agentscope.extensions.model.openai.OpenAIChatModel;

ReActAgent.builder()
        .model(OpenAIChatModel.builder()
                .apiKey(System.getenv("OPENAI_API_KEY"))
                .modelName("gpt-4o")
                .build())
        .build();
```

> 旧文档中的 `agentscope-openai` 坐标已不存在，正确坐标是
> `agentscope-extensions-model-openai`。

---

### Q13: 流式输出不显示？

`stream(true)` 只是让模型侧流式返回；**要在应用侧实时看到内容，必须消费事件流**：

```java
agent.streamEvents(new UserMessage("介绍一下你自己"))
        .doOnNext(event -> {
            if (event instanceof TextBlockDeltaEvent e) {
                System.out.print(e.getDelta());
            }
        })
        .blockLast();
```

> `agent.stream(msg)`（返回 `Flux<Msg>`）已在 2.0.0 标记 `@Deprecated(forRemoval = true)`。

---

### Q14: 如何查看 Agent 的思考过程？

**1. 打开 thinking（DashScope）**
```java
DashScopeChatModel.builder()
        .apiKey(apiKey)
        .modelName("qwen-plus")
        .enableThinking(true)
        .formatter(new DashScopeChatFormatter())
        .build();
```

**2. 订阅思考增量事件**
```java
agent.streamEvents(msg)
        .doOnNext(event -> {
            if (event instanceof ThinkingBlockDeltaEvent e) {
                System.out.print(e.getDelta());
            }
        })
        .blockLast();
```

**3. 用中间件记录**
```java
@Override
public Flux<AgentEvent> onReasoning(Agent agent, RuntimeContext ctx,
        ReasoningInput input, Function<ReasoningInput, Flux<AgentEvent>> next) {
    System.out.println("上下文消息数: " + input.messages().size());
    return next.apply(input)
            .doOnComplete(() -> System.out.println("思考结束"));
}
```

---

### Q15: 如何调试工具调用？

**方式一：事件流**
```java
agent.streamEvents(msg).doOnNext(event -> {
    switch (event.getType()) {
        case TOOL_CALL_START -> System.out.println("[调用] "
                + ((ToolCallStartEvent) event).getToolCallName());
        case TOOL_CALL_DELTA -> System.out.println("[参数增量] " + event);
        case TOOL_PROGRESS -> System.out.println("[进度] " + event);
        case TOOL_RESULT_END -> System.out.println("[结果] " + event);
        default -> { }
    }
}).blockLast();
```

**方式二：中间件**
```java
@Override
public Flux<AgentEvent> onActing(Agent agent, RuntimeContext ctx,
        ActingInput input, Function<ActingInput, Flux<AgentEvent>> next) {
    System.out.println("[工具调用] " + input.toolCalls());
    return next.apply(input);
}
```

**方式三：工具内日志 + 断点**（断在 `Toolkit#callTool`）

---

## 状态与会话

### Q16: 如何处理敏感信息？

1. 用环境变量：
```java
String apiKey = System.getenv("DASHSCOPE_API_KEY");
```
2. 不要硬编码：
```java
// ❌ .apiKey("sk-xxxxxxxx")
```
3. `.gitignore` 排除敏感文件：
```
.env
secrets.properties
*-local.properties
.agentscope/
```

---

### Q17: 如何实现多轮对话？

Agent 默认无状态，需要**会话标识 + 状态存储**：

**方式一：Builder 上配置（最简单）**
```java
ReActAgent agent = ReActAgent.builder()
        .model("dashscope:qwen-plus")
        .stateStore(new JsonFileAgentStateStore(Paths.get(".agentscope/sessions")))
        .defaultSessionId("alice")
        .build();

agent.call(new UserMessage("我叫小明")).block();
agent.call(new UserMessage("我叫什么？")).block();   // AI 记得"小明"
```

**方式二：`RuntimeContext`（多会话共享同一个 Agent 实例）**
```java
ReActAgent agent = ReActAgent.builder().model("dashscope:qwen-plus").build();

RuntimeContext alice = RuntimeContext.builder().userId("alice").sessionId("s1").build();
RuntimeContext bob   = RuntimeContext.builder().userId("bob").sessionId("s1").build();

agent.call(new UserMessage("我叫小明"), alice).block();
agent.call(new UserMessage("我叫什么？"), alice).block();   // "小明"
agent.call(new UserMessage("我叫什么？"), bob).block();     // 不知道
```

**隔离维度**：`userId` → `sessionId` → `threadId`。

---

### Q18: 如何在 Spring Boot 中使用？

**1. 加依赖**
```xml
<dependency>
    <groupId>io.agentscope</groupId>
    <artifactId>agentscope-spring-boot-starter</artifactId>
    <version>2.0.3</version>
</dependency>
<dependency>
    <groupId>io.agentscope</groupId>
    <artifactId>agentscope-dashscope-spring-boot-starter</artifactId>
    <version>2.0.3</version>
</dependency>
```

**2. 配置 `application.yml`**
```yaml
agentscope:
  agent:
    enabled: true
    name: Assistant
    sys-prompt: 你是一个友好的助手
    max-iters: 10
  dashscope:
    api-key: ${DASHSCOPE_API_KEY}
    enable-thinking: false
```

**3. 注入使用**

`AgentscopeAutoConfiguration` 在存在 `Model` bean 时会自动创建 `ReActAgent` bean：

```java
@Service
public class AgentService {

    private final ReActAgent agent;

    public AgentService(ReActAgent agent) {     // 自动注入
        this.agent = agent;
    }

    public String chat(String input) {
        Msg response = agent.call(new UserMessage(input)).block();
        return response.getTextContent();
    }
}
```

配置项说明：

| 属性 | 说明 |
|------|------|
| `agentscope.agent.enabled` | 是否启用自动装配（默认 `true`） |
| `agentscope.agent.name` | Agent 名称（默认 `Assistant`） |
| `agentscope.agent.sys-prompt` | 系统提示词 |
| `agentscope.agent.max-iters` | 最大迭代（默认 `10`） |
| `agentscope.model.provider` | 指定模型 provider |
| `agentscope.dashscope.api-key` / `base-url` / `enable-thinking` | DashScope 配置 |

> 自动装配产出的 `Memory` / `Toolkit` bean 是 **prototype** 作用域（有状态，非线程安全）。
> 生产环境建议自己配置 `AgentStateStore` + `RuntimeContext`，让 Agent 保持无状态。

---

### Q19: 项目可以商用吗？

AgentScope-Java 采用 **Apache 2.0 许可证**，可免费商用。

- ✅ 商用 / 修改 / 分发
- ✅ 闭源集成
- ⚠️ 需保留版权与许可声明
- ⚠️ 需说明对原始代码的修改

---

### Q20: `model("dashscope:qwen-plus")` 报错找不到 provider？

**原因**：`ModelRegistry` 通过 `ServiceLoader` 加载 `ModelProvider`，
对应扩展模块不在 classpath 上。

**解决**：引入模型扩展模块

```xml
<!-- DashScope -->
<dependency>
    <groupId>io.agentscope</groupId>
    <artifactId>agentscope-extensions-model-dashscope</artifactId>
    <version>2.0.3</version>
</dependency>
```

可用的 provider id 与模块：

| provider id | 模块 |
|-------------|------|
| `dashscope` | `agentscope-extensions-model-dashscope` |
| `openai` | `agentscope-extensions-model-openai` |
| `openai-official` | `agentscope-extensions-model-openai-official` |
| `anthropic` | `agentscope-extensions-model-anthropic` |
| `gemini` | `agentscope-extensions-model-gemini` |
| `ollama` | `agentscope-extensions-model-ollama` |

---

### Q21: 从 v1 升级到 v2，主要改哪些地方？

| v1 写法 | v2 写法 |
|---------|---------|
| `.memory(new InMemoryMemory())` | 删除；需要持久化用 `.stateStore(...)` + `.defaultSessionId(...)` |
| `new JsonSession("x.json")` | `new JsonFileAgentStateStore(path)` |
| `session.save()` / `session.load()` | 自动，由 `AgentStateStore` 负责 |
| `agent.getMemory().getMessages()` | `agent.getAgentState().getContext()` |
| `.hook(...)` / `.hooks(...)` | `.middleware(...)` / `.middlewares(...)` |
| `Hook` + `PreReasoningEvent` 等 | `MiddlewareBase` 的 6 个切面 |
| `agent.stream(msg)` → `Flux<Msg>` | `agent.streamEvents(msg)` → `Flux<AgentEvent>` |
| `ReasoningChunkEvent` | `TEXT_BLOCK_DELTA` / `THINKING_BLOCK_DELTA` 事件 |
| `PostReasoningEvent.stopAgent()` | 在 `onActing` 发出 `RequestStopEvent` |
| `DashScopeChatModel`（core 内） | 移到 `io.agentscope.extensions.model.dashscope` |
| 单一 `agentscope-core` 依赖 | `agentscope-harness` + 模型扩展模块 |
| `McpClient.builder().command(...)` | `McpClientBuilder.create(name).stdioTransport(...)` |
| `toolkit.registerMcpTools(client)` | `toolkit.registerMcpClient(wrapper)`（返回 `Mono<Void>`） |
| `enableMetaTool(true)` 旧工具组 API | `toolkit.createToolGroup(name, desc, false)` + `registration().tool(x).group(name).apply()` |

示例模块的 Javadoc 里保留了逐条迁移说明，可直接对照：
`agentscope-examples/documentation/src/main/java/io/agentscope/examples/documentation2/**`。

---

### Q22: 如何获取帮助？

| 资源 | 链接 |
|------|------|
| 官方文档 | https://java.agentscope.io/ |
| GitHub 仓库 | https://github.com/agentscope-ai/agentscope-java |
| 问题反馈 | https://github.com/agentscope-ai/agentscope-java/issues |
| Discord | https://discord.gg/eYMpfnkG8h |

**提问建议**
1. 先搜 [Issues](https://github.com/agentscope-ai/agentscope-java/issues)
2. 提供环境信息（JDK / Maven / 操作系统 / 提交 hash）
3. 提供最小可复现代码
4. 提供完整错误日志（含 `getGenerateReason()` 输出）

---

## 问题未解决？

1. 查阅 [官方文档](https://java.agentscope.io/)
2. 搜索 [GitHub Issues](https://github.com/agentscope-ai/agentscope-java/issues)
3. 在 [Discord](https://discord.gg/eYMpfnkG8h) 提问
4. 提交 [新 Issue](https://github.com/agentscope-ai/agentscope-java/issues/new)
