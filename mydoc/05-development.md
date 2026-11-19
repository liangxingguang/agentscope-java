# 开发指南

本文档介绍 AgentScope-Java（v2.x）的开发规范、扩展开发方式、测试与调试技巧。

## 1. 项目结构规范

### 1.1 核心模块结构

```
agentscope-core/src/main/java/io/agentscope/core/
├── ReActAgent.java         # 推理内核（约 7400 行，主入口）
├── Version.java
├── agent/                  # Agent 接口族、RuntimeContext、CallExecution、Event/EventType
│   ├── config/             # ReactConfig（maxIters 默认 20）、ModelConfig
│   ├── accumulator/        # 流式块累积
│   └── user/
├── model/                  # Model、ChatModelBase、ModelRegistry、GenerateOptions
│   ├── spi/                # ModelProvider（SPI 扩展点）
│   └── transport/          # JDK / OkHttp / WebSocket 传输
├── message/                # Msg 及各角色消息、ContentBlock
├── event/                  # 32 种 AgentEvent
├── tool/                   # Toolkit、Tool、ToolGroup、ToolExecutor、ToolEmitter
├── middleware/             # MiddlewareBase、MiddlewareChain
├── state/                  # AgentState、AgentStateStore、ConflictPolicy
├── memory/                 # Memory（已废弃）、LongTermMemory
├── journal/                # 执行日志
├── permission/             # PermissionEngine
├── interruption/           # InterruptControl
├── hook/                   # Hook（已废弃）+ recorder/
├── skill/                  # AgentSkill、SkillBox、SkillRegistry
├── rag/                    # Knowledge（已废弃）、RAGMode、RetrieveConfig
├── workspace/  observation/  tracing/  credential/  util/
└── formatter/  exception/
```

测试代码镜像同一包结构：`agentscope-core/src/test/java/io/agentscope/core/...`。

### 1.2 命名规范

| 类型 | 规范 | 示例 |
|------|------|------|
| 类名 | PascalCase | `ReActAgent`、`DashScopeChatModel` |
| 方法名 | camelCase | `doCallInner`、`buildGenerateOptions` |
| 常量 | SCREAMING_SNAKE_CASE | `DEFAULT_MAX_ITERS`、`UNVERSIONED` |
| 包名 | 全小写 | `io.agentscope.core.tool` |
| 工具名 | snake_case | `get_weather`、`search_web` |
| 事件类型 | SCREAMING_SNAKE_CASE | `TEXT_BLOCK_DELTA` |
| 测试方法 | `method_scenario_expected` | `call_withEmptyContext_startsNewConversation` |

### 1.3 许可证头

所有 Java 文件必须包含 Apache 2.0 头（由 `skywalking-eyes` / `.licenserc.yaml` 校验）：

```java
/*
 * Copyright 2024-2026 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
```

## 2. 代码规范

### 2.1 格式化（Spotless）

```bash
mvn spotless:check     # 检查（绑定在 compile 阶段，构建会自动执行）
mvn spotless:apply     # 自动修复
```

配置（根 `pom.xml`）：

| 项 | 值 |
|----|-----|
| 格式化器 | google-java-format |
| 版本 | 1.28.0 |
| 风格 | **AOSP（4 空格缩进）** |
| `reflowLongStrings` | true |
| `formatJavadoc` | false |
| 其他 | 移除未使用 import、禁止通配符 import、自动重排 import |

> 项目**没有** Checkstyle 插件。

### 2.2 Javadoc 规范

公共 API 必须有 Javadoc；`maven-javadoc-plugin` 的 `doclint` 配置为 `all,-missing`
（不强制每个成员都写，但写了就必须合法）。

```java
/**
 * 执行工具调用并返回结果。
 *
 * <p>此方法会验证参数、执行工具并处理异常；工具抛异常时返回包含错误信息的结果。
 *
 * @param param 工具调用信息，包含工具名与参数
 * @return 工具执行结果
 * @throws IllegalArgumentException 如果工具不存在
 */
public Mono<ToolResultBlock> callTool(ToolCallParam param) { ... }
```

### 2.3 导入顺序

google-java-format 按 ASCII 排序，静态导入单独一组：

```java
import static org.junit.jupiter.api.Assertions.assertEquals;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.agentscope.core.message.Msg;
import io.agentscope.core.tool.Tool;
import java.util.List;
import java.util.Map;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import reactor.core.publisher.Mono;
```

### 2.4 日志规范

```java
private static final Logger log = LoggerFactory.getLogger(YourClass.class);

log.debug("处理消息: {}, 工具: {}", msg.getId(), toolName);   // ✅ 参数化
// log.debug("处理消息: " + msg.getId());                    // ❌ 字符串拼接
```

## 3. 开发自定义组件

### 3.1 自定义 Model

推荐继承 `ChatModelBase`（`stream()` 是 `final`，负责接入 tracing；你只需实现 `doStream()`）：

```java
import io.agentscope.core.message.Msg;
import io.agentscope.core.model.ChatModelBase;
import io.agentscope.core.model.ChatResponse;
import io.agentscope.core.model.GenerateOptions;
import io.agentscope.core.model.ToolSchema;
import java.util.List;
import reactor.core.publisher.Flux;

public class MyChatModel extends ChatModelBase {

    private final String modelName;
    private final String baseUrl;

    private MyChatModel(Builder b) {
        this.modelName = b.modelName;
        this.baseUrl = b.baseUrl;
    }

    @Override
    public String getModelName() {
        return modelName;
    }

    @Override
    protected Flux<ChatResponse> doStream(
            List<Msg> messages, List<ToolSchema> tools, GenerateOptions options) {
        // 1. 把 AgentScope Msg 转成厂商格式
        // 2. 发起流式 HTTP 请求（可用 model.transport 中的 HttpTransport）
        // 3. 把厂商分片转成 ChatResponse
        return callApi(messages, tools, options).map(this::toChatResponse);
    }

    public static Builder builder() { return new Builder(); }

    public static final class Builder {
        private String apiKey;
        private String modelName = "default-model";
        private String baseUrl = "https://api.example.com/v1";

        public Builder apiKey(String v) { this.apiKey = v; return this; }
        public Builder modelName(String v) { this.modelName = v; return this; }
        public Builder baseUrl(String v) { this.baseUrl = v; return this; }
        public MyChatModel build() { return new MyChatModel(this); }
    }
}
```

若要支持 `model("myprovider:xxx")` 字符串解析，实现 SPI：

```java
// io.agentscope.core.model.spi.ModelProvider
public class MyModelProvider implements ModelProvider {
    @Override public String providerId() { return "myprovider"; }
    @Override public boolean supports(String modelId) { return true; }
    @Override public Model create(String modelId) {
        return MyChatModel.builder().modelName(modelId).build();
    }
}
```

并在 `src/main/resources/META-INF/services/io.agentscope.core.model.spi.ModelProvider`
中写入实现类全名。

### 3.2 自定义 Tool

```java
import io.agentscope.core.tool.Tool;
import io.agentscope.core.tool.ToolParam;

public class MyTools {

    /** 简单工具。 */
    @Tool(name = "my_tool", description = "执行某个操作，当需要...时使用此工具")
    public String myTool(
            @ToolParam(name = "param1", description = "参数1描述") String param1,
            @ToolParam(name = "param2", description = "参数2描述", required = false) String param2) {

        if (param1 == null || param1.isEmpty()) {
            return "错误: param1 不能为空";       // 返回错误字符串比抛异常更友好
        }
        return doSomething(param1, param2);
    }

    /** 异步工具。 */
    @Tool(name = "async_tool", description = "异步操作")
    public Mono<String> asyncTool(@ToolParam(name = "input") String input) {
        return Mono.fromCallable(() -> heavyWork(input));
    }

    /** 进度流工具：声明 ToolEmitter 参数，框架自动注入。 */
    @Tool(name = "long_task", description = "长任务并汇报进度")
    public String longTask(
            @ToolParam(name = "total") int total,
            ToolEmitter emitter) {
        for (int i = 1; i <= total; i++) {
            emitter.emit(ToolResultBlock.text("进度 " + i + "/" + total));
        }
        return "完成";
    }

    /** 自定义结果转换器。 */
    @Tool(name = "complex_tool", description = "返回复杂对象",
          converter = MyJsonConverter.class)
    public ComplexResult complexTool(@ToolParam(name = "input") String input) {
        return new ComplexResult(input, "processed");
    }
}
```

工具属性（`readOnly`、`concurrencySafe`、`dangerousFiles`、`returnDirect` 等）见
[核心概念 - Tool](./03-concepts.md#4-tool工具)。

### 3.3 自定义 Middleware

```java
import io.agentscope.core.agent.Agent;
import io.agentscope.core.agent.RuntimeContext;
import io.agentscope.core.event.AgentEvent;
import io.agentscope.core.middleware.*;
import java.util.EnumSet;
import java.util.Set;
import java.util.function.Function;
import reactor.core.publisher.Flux;

public class MyMiddleware implements MiddlewareBase {

    @Override
    public Set<ExtensionPoint> activePoints() {
        return EnumSet.of(ExtensionPoint.ON_REASONING, ExtensionPoint.ON_ACTING);
    }

    @Override
    public int order() {
        return 50;      // 越小越靠外层
    }

    @Override
    public Flux<AgentEvent> onReasoning(
            Agent agent,
            RuntimeContext ctx,
            ReasoningInput input,
            Function<ReasoningInput, Flux<AgentEvent>> next) {
        log.info("开始思考，上下文消息数: {}", input.messages().size());
        return next.apply(input)
                .doOnNext(e -> {
                    if (e.getType() == AgentEventType.TEXT_BLOCK_DELTA) {
                        log.debug("增量: {}", ((TextBlockDeltaEvent) e).getDelta());
                    }
                });
    }

    @Override
    public Flux<AgentEvent> onActing(
            Agent agent,
            RuntimeContext ctx,
            ActingInput input,
            Function<ActingInput, Flux<AgentEvent>> next) {
        log.info("执行工具: {}", input.toolCalls());
        return next.apply(input).doOnComplete(() -> log.info("工具执行结束"));
    }
}
```

**注意**：只有 `activePoints()` 声明过的切面才会被调用，未声明的切面即使重写也不会执行。

### 3.4 自定义 AgentStateStore

```java
import io.agentscope.core.state.AgentStateStore;
import io.agentscope.core.state.State;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

public class MyStateStore implements AgentStateStore {

    // key: userId + "/" + sessionId + "/" + key
    private final Map<String, Object> store = new ConcurrentHashMap<>();

    private static String id(String userId, String sessionId, String key) {
        return userId + "/" + sessionId + "/" + key;
    }

    @Override
    public void save(String userId, String sessionId, String key, State value) {
        store.put(id(userId, sessionId, key), value);
    }

    @Override
    public void save(String userId, String sessionId, String key, List<? extends State> values) {
        store.put(id(userId, sessionId, key), new ArrayList<>(values));
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T extends State> Optional<T> get(
            String userId, String sessionId, String key, Class<T> type) {
        return Optional.ofNullable((T) store.get(id(userId, sessionId, key)));
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T extends State> List<T> getList(
            String userId, String sessionId, String key, Class<T> itemType) {
        Object v = store.get(id(userId, sessionId, key));
        return v == null ? List.of() : (List<T>) v;
    }

    @Override
    public boolean exists(String userId, String sessionId) {
        String prefix = userId + "/" + sessionId + "/";
        return store.keySet().stream().anyMatch(k -> k.startsWith(prefix));
    }

    @Override
    public void delete(String userId, String sessionId) {
        String prefix = userId + "/" + sessionId + "/";
        store.keySet().removeIf(k -> k.startsWith(prefix));
    }

    @Override
    public Set<String> listSessionIds(String userId) {
        String prefix = userId + "/";
        Set<String> ids = new HashSet<>();
        store.keySet().stream()
                .filter(k -> k.startsWith(prefix))
                .forEach(k -> ids.add(k.split("/")[1]));
        return ids;
    }
}
```

生产实现建议额外支持版本控制（重写 `supportsVersioning()`、`getVersioned()`、`saveIfVersion()`），
以便多副本并发写时用乐观锁避免覆盖。

### 3.5 自定义结果转换器

```java
import io.agentscope.core.tool.ToolResultConverter;
import java.lang.reflect.Type;

public class MyJsonConverter implements ToolResultConverter {
    @Override
    public ToolResultBlock convert(Object result, Type returnType) {
        // 把任意对象转成 ToolResultBlock（文本 + DataBlock）
        ...
    }
}
```

## 4. 测试规范

### 4.1 技术栈

| 项 | 版本 |
|----|------|
| JUnit Jupiter | 6.0.2（BOM `junit-bom`） |
| Mockito | 5.23.0 |
| JaCoCo | 0.8.14 |
| Surefire | 3.5.5 |

### 4.2 单元测试示例

```java
class MyToolTest {

    @Test
    void myTool_withValidInput_returnsResult() {
        MyTools tools = new MyTools();
        String result = tools.myTool("test", null);
        assertNotNull(result);
        assertTrue(result.contains("test"));
    }

    @Test
    void myTool_withNullInput_returnsError() {
        assertEquals("错误: param1 不能为空", new MyTools().myTool(null, null));
    }
}
```

### 4.3 用 Mockito 打桩 Model

`ReActAgent` 依赖很多协作者，通常**不要**直接 `@InjectMocks`；用构造函数外的方式
（Builder + 真实组件 + mock 的 `Model`）更稳：

```java
class AgentTest {

    @Test
    void call_withMockedModel_returnsChunkText() {
        Model model = mock(Model.class);
        when(model.getModelName()).thenReturn("mock-model");
        when(model.stream(anyList(), anyList(), any()))
                .thenReturn(Flux.just(
                        ChatResponse.builder()
                                .content(List.of(TextBlock.builder().text("hi").build()))
                                .build()));

        ReActAgent agent = ReActAgent.builder()
                .name("test-agent")
                .model(model)
                .toolkit(new Toolkit())
                .build();

        Msg result = agent.call(new UserMessage("hello")).block();
        assertEquals("hi", result.getTextContent());
    }
}
```

> 真实测试里更常用的是"假的 Model 实现"（返回固定 `Flux<ChatResponse>`），
> 因为 `stream()` 的参数与流式语义较复杂。参考 `agentscope-core/src/test` 中现有做法。

### 4.4 测试命名

```java
@Test void execute_withValidTool_returnsSuccess() { }
@Test void execute_withUnknownTool_throwsException() { }
@Test void call_withEmptyContext_startsNewConversation() { }
```

### 4.5 运行测试

```bash
mvn test                                          # 全部
mvn test -pl agentscope-core                      # 单模块
mvn test -Dtest=MyToolTest                        # 单类
mvn test -Dtest=MyToolTest#myTool_withValidInput_returnsResult   # 单方法
mvn verify                                        # 含覆盖率报告
```

## 5. 调试技巧

### 5.1 关键断点

```java
// io.agentscope.core.ReActAgent（内部类 CallExecution，约 line 2372 起）
doCallInner(List<Msg> msgs)                     // 整体调用入口
reasoning(int iter, boolean ignoreMaxIters)     // 每轮思考（约 line 3207）
acting(int iter)                                // 工具执行（约 line 3900）
```

```java
// io.agentscope.core.tool.Toolkit
callTool(ToolCallParam param)                   // 工具执行入口
```

### 5.2 日志配置

`src/test/resources/logback-test.xml`：

```xml
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder><pattern>%d{HH:mm:ss.SSS} %-5level %logger{40} - %msg%n</pattern></encoder>
    </appender>
    <logger name="io.agentscope" level="DEBUG"/>
    <logger name="reactor" level="INFO"/>
    <root level="INFO"><appender-ref ref="STDOUT"/></root>
</configuration>
```

### 5.3 常见问题排查

**工具没被调用**

```java
toolkit.getToolNames().forEach(System.out::println);      // 是否注册成功
toolkit.getToolSchemas().forEach(s -> System.out.println(s.getName()));  // Schema 是否正确
toolkit.getActiveGroups();                                // 工具组是否激活
```

**响应为空**

```java
Msg msg = agent.call(userMsg).block();
System.out.println("blocks=" + msg.getContent().size());
System.out.println("reason=" + msg.getGenerateReason());
```

**状态没恢复**

```java
System.out.println("context size=" + agent.getAgentState().getContext().size());
// 确认 stateStore / defaultSessionId / RuntimeContext.sessionId 三者一致
```

**上下文无限增长**

使用 Harness 的压缩能力：`.compaction(CompactionConfig.builder()...build())`，
或自定义 `AgentStateStore` 在 `save` 时截断 `AgentState.context`。

## 6. 性能与最佳实践

### 6.1 响应式

```java
// ❌ 在响应式链里阻塞
public Mono<Msg> process(Msg input) {
    Msg result = blockingCall(input);
    return Mono.just(result);
}

// ✅ 放到 boundedElastic
public Mono<Msg> process(Msg input) {
    return Mono.fromCallable(() -> blockingCall(input))
            .subscribeOn(Schedulers.boundedElastic());
}

// ✅ 并行
return Flux.fromIterable(inputs).flatMap(this::process, 4).collectList();
```

### 6.2 复用 Agent 实例

Agent 是**无状态**的，推荐启动时构造一次 Builder，每次请求只 `build()` / 复用实例：

```java
private static final ReActAgent.Builder BUILDER = ReActAgent.builder()
        .name("assistant")
        .model("dashscope:qwen-plus")
        .toolkit(toolkit);

ReActAgent agent = BUILDER.build();      // 轻量
```

`HarnessAgent` 实现了 `AutoCloseable`，建议 try-with-resources：

```java
try (HarnessAgent agent = HARNESS_BUILDER.build()) {
    agent.call(new UserMessage("Hello!"), ctx).block();
}
```

### 6.3 工具与模型

- 只读且幂等的工具标 `readOnly = true`、`concurrencySafe = true`，便于并发调度
- 慢工具用异步返回 + `ExecutionConfig.timeout(...)`
- 大结果用 `toolResultEviction` 卸载到磁盘，避免撑爆上下文
- 模型调用设 `modelExecutionConfig(ExecutionConfig)` 的重试与退避

## 7. 提交规范

### 7.1 Commit Message

```
<type>(<scope>): <subject>

[optional body]
```

类型：`feat` / `fix` / `docs` / `style` / `refactor` / `test` / `chore`

示例：
```
feat(tool): 添加文件操作工具

- 新增 read_file 工具
- 新增 write_file 工具
- 支持 UTF-8 编码
```

### 7.2 提交前检查

```bash
mvn spotless:apply        # 格式化
mvn -B test               # 单测
mvn -B verify             # 含覆盖率
```

> 项目中没有 `mvn checkstyle:check`；代码风格由 Spotless 保证。

## 8. 下一步

- [常见问题](./06-faq.md)：问题排查与解决方案
