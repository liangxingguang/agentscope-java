# 开发指南

本文档介绍 AgentScope-Java 的开发规范、调试技巧和最佳实践。

## 1. 项目结构规范

### 1.1 核心模块结构

```
agentscope-core/
└── src/
    ├── main/
    │   └── java/io/agentscope/core/
    │       ├── agent/        # Agent 基类和接口
    │       ├── model/        # 模型层
    │       ├── memory/       # 记忆系统
    │       ├── tool/         # 工具系统
    │       ├── message/      # 消息结构
    │       ├── hook/         # 钩子系统
    │       ├── session/      # 会话管理
    │       └── util/         # 工具类
    └── test/
        └── java/io/agentscope/core/
            └── ... (测试类)
```

### 1.2 命名规范

| 类型 | 规范 | 示例 |
|------|------|------|
| 类名 | PascalCase | `ReActAgent`, `DashScopeChatModel` |
| 方法名 | camelCase | `executeIteration`, `buildGenerateOptions` |
| 常量 | SCREAMING_SNAKE_CASE | `MAX_ITERATIONS`, `DEFAULT_TIMEOUT` |
| 包名 | 全小写 | `io.agentscope.core.tool` |
| 工具名 | snake_case | `get_weather`, `search_web` |

### 1.3 文件头部

所有 Java 文件必须包含 Apache 2.0 许可证头：

```java
/*
 * Copyright 2024-2026 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
```

## 2. 代码规范

### 2.1 格式化

使用 Spotless 自动格式化：

```bash
# 检查格式
mvn spotless:check

# 自动修复
mvn spotless:apply
```

格式化规则：
- AOSP 风格（4 空格缩进）
- 自动移除未使用的导入
- 禁止通配符导入
- 自动重排导入顺序

### 2.2 Javadoc 规范

所有公共方法必须有 Javadoc：

```java
/**
 * 执行工具调用并返回结果。
 *
 * <p>此方法会验证工具参数，执行工具，并处理异常情况。
 * 如果工具抛出异常，会返回包含错误信息的结果。
 *
 * @param toolUse 工具调用信息，包含工具名和参数
 * @param context 执行上下文，提供运行时信息
 * @return 工具执行结果
 * @throws IllegalArgumentException 如果工具不存在
 */
public ToolResultBlock execute(ToolUseBlock toolUse, ToolExecutionContext context) {
    // ...
}
```

### 2.3 导入顺序

```
// 1. java.*
import java.util.List;
import java.util.Map;

// 2. javax.*
import javax.annotation.Nullable;

// 3. 第三方库
import com.fasterxml.jackson.databind.ObjectMapper;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import reactor.core.publisher.Mono;

// 4. io.agentscope.*
import io.agentscope.core.message.Msg;
import io.agentscope.core.tool.Tool;
```

### 2.4 日志规范

使用 SLF4J：

```java
private static final Logger log = LoggerFactory.getLogger(YourClass.class);

// 不同级别
log.trace("详细信息，调试用");
log.debug("调试信息");
log.info("重要信息");
log.warn("警告信息");
log.error("错误信息", exception);

// 参数化日志
log.debug("处理消息: {}, 工具: {}", msg.getId(), toolName);

// 不要这样写（性能差）
log.debug("处理消息: " + msg.getId());
```

## 3. 开发自定义组件

### 3.1 开发自定义 Model

实现 `Model` 接口：

```java
public class CustomChatModel implements Model {
    
    private final String apiKey;
    private final String modelName;
    
    public CustomChatModel(String apiKey, String modelName) {
        this.apiKey = apiKey;
        this.modelName = modelName;
    }
    
    @Override
    public Flux<ChatResponse> stream(
            List<Msg> messages, 
            List<ToolSchema> tools, 
            GenerateOptions options) {
        
        // 1. 转换消息格式
        List<ApiMessage> apiMessages = convertMessages(messages);
        
        // 2. 构建请求
        ApiRequest request = buildRequest(apiMessages, tools, options);
        
        // 3. 调用 API
        return callApi(request)
            .map(this::convertResponse);
    }
    
    @Override
    public String getModelName() {
        return modelName;
    }
    
    // 使用 Builder 模式
    public static Builder builder() {
        return new Builder();
    }
    
    public static class Builder {
        private String apiKey;
        private String modelName = "default-model";
        
        public Builder apiKey(String apiKey) {
            this.apiKey = apiKey;
            return this;
        }
        
        public Builder modelName(String modelName) {
            this.modelName = modelName;
            return this;
        }
        
        public CustomChatModel build() {
            Objects.requireNonNull(apiKey, "apiKey is required");
            return new CustomChatModel(apiKey, modelName);
        }
    }
}
```

### 3.2 开发自定义 Tool

使用 `@Tool` 注解：

```java
public class MyTools {
    
    /**
     * 简单工具示例。
     */
    @Tool(
        name = "my_tool",
        description = "执行某个操作，当需要...时使用此工具"
    )
    public String myTool(
            @ToolParam(
                name = "param1",
                description = "参数1的描述"
            ) String param1,
            @ToolParam(
                name = "param2",
                description = "参数2的描述",
                required = false
            ) String param2) {
        
        // 参数验证
        if (param1 == null || param1.isEmpty()) {
            return "错误: param1 不能为空";
        }
        
        // 执行操作
        String result = doSomething(param1, param2);
        
        // 返回结果（字符串格式）
        return result;
    }
    
    /**
     * 异步工具示例。
     */
    @Tool(name = "async_tool", description = "异步操作")
    public Mono<String> asyncTool(String input) {
        return Mono.fromCallable(() -> {
            // 异步执行
            Thread.sleep(1000);
            return "处理完成: " + input;
        });
    }
    
    /**
     * 带自定义转换器的工具。
     */
    @Tool(
        name = "complex_tool",
        description = "返回复杂对象",
        converter = JsonConverter.class
    )
    public ComplexResult complexTool(String input) {
        return new ComplexResult(input, "processed");
    }
}
```

### 3.3 开发自定义 Hook

实现 `Hook` 接口：

```java
public class MyHook implements Hook {
    
    private static final Logger log = LoggerFactory.getLogger(MyHook.class);
    
    /**
     * 思考前事件处理。
     */
    @Override
    public Mono<PreReasoningEvent> onEvent(PreReasoningEvent event) {
        // 可以修改输入消息
        List<Msg> messages = event.getInputMessages();
        
        // 记录日志
        log.info("开始思考，消息数: {}", messages.size());
        
        // 必须返回事件（可以修改后返回）
        return Mono.just(event);
    }
    
    /**
     * 思考后事件处理。
     */
    @Override
    public Mono<PostReasoningEvent> onEvent(PostReasoningEvent event) {
        Msg msg = event.getReasoningMessage();
        
        // 检查是否需要停止
        if (shouldStop(msg)) {
            event.requestStop();
        }
        
        return Mono.just(event);
    }
    
    /**
     * 流式输出事件处理。
     */
    @Override
    public Mono<ReasoningChunkEvent> onEvent(ReasoningChunkEvent event) {
        // 实时处理每个输出块
        Msg chunk = event.getChunkMessage();
        Msg accumulated = event.getAccumulatedMessage();
        
        // 可以在这里实现实时显示
        System.out.print(chunk.getTextContent());
        
        return Mono.just(event);
    }
}
```

### 3.4 开发自定义 Memory

实现 `Memory` 接口：

```java
public class CustomMemory implements Memory {
    
    private final List<Msg> messages = new ArrayList<>();
    private final int maxSize;
    
    public CustomMemory(int maxSize) {
        this.maxSize = maxSize;
    }
    
    @Override
    public void addMessage(Msg message) {
        Objects.requireNonNull(message, "message cannot be null");
        
        messages.add(message);
        
        // 超过最大数量时移除最早的消息
        while (messages.size() > maxSize) {
            messages.remove(0);
        }
    }
    
    @Override
    public List<Msg> getMessages() {
        return Collections.unmodifiableList(messages);
    }
    
    @Override
    public void deleteMessage(int index) {
        if (index >= 0 && index < messages.size()) {
            messages.remove(index);
        }
    }
    
    @Override
    public void clear() {
        messages.clear();
    }
    
    // 实现状态持久化
    @Override
    public void saveTo(Session session, SessionKey key) {
        session.save(key, "memory_messages", messages);
    }
    
    @Override
    public void loadFrom(Session session, SessionKey key) {
        session.get(key, "memory_messages", new TypeReference<List<Msg>>() {})
            .ifPresent(msgs -> {
                messages.clear();
                messages.addAll(msgs);
            });
    }
}
```

## 4. 测试规范

### 4.1 单元测试

使用 JUnit 5 + Mockito：

```java
class MyToolTest {
    
    @Test
    void testMyTool_withValidInput_returnsResult() {
        // Given
        MyTools tools = new MyTools();
        String input = "test";
        
        // When
        String result = tools.myTool(input);
        
        // Then
        assertNotNull(result);
        assertTrue(result.contains("test"));
    }
    
    @Test
    void testMyTool_withNullInput_returnsError() {
        MyTools tools = new MyTools();
        
        String result = tools.myTool(null);
        
        assertTrue(result.contains("错误"));
    }
}

// 使用 Mockito
class AgentTest {
    
    @Mock
    private Model model;
    
    @Mock
    private Memory memory;
    
    @InjectMocks
    private ReActAgent agent;
    
    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
    }
    
    @Test
    void testCall_withValidMessage_returnsResponse() {
        // Given
        Msg input = Msg.builder().textContent("hello").build();
        Msg expected = Msg.builder().textContent("hi").build();
        
        when(model.stream(any(), any(), any()))
            .thenReturn(Flux.just(new ChatResponse(expected)));
        when(memory.getMessages()).thenReturn(List.of());
        
        // When
        Msg result = agent.call(input).block();
        
        // Then
        assertNotNull(result);
        assertEquals("hi", result.getTextContent());
    }
}
```

### 4.2 测试命名规范

```java
// 格式: methodName_scenario_expectedBehavior
@Test
void testExecute_withValidTool_returnsSuccess() { }

@Test
void testExecute_withUnknownTool_throwsException() { }

@Test
void testCall_withEmptyMemory_startsNewConversation() { }
```

### 4.3 运行测试

```bash
# 运行所有测试
mvn test

# 运行单个测试类
mvn test -Dtest=MyToolTest

# 运行单个测试方法
mvn test -Dtest=MyToolTest#testMyTool_withValidInput_returnsResult

# 运行特定模块的测试
mvn test -pl agentscope-core
```

## 5. 调试技巧

### 5.1 IDEA 调试配置

**远程调试**：
```
# 启动应用时添加参数
java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005 -jar app.jar

# IDEA 配置 Remote JVM Debug
Host: localhost
Port: 5005
```

### 5.2 关键断点位置

```java
// 1. ReActAgent - 理解主循环
public class ReActAgent {
    // 思考阶段
    private Mono<Msg> reasoning(int iter, boolean ignoreMaxIters) {
        // 断点：观察思考输入
    }
    
    // 行动阶段
    private Mono<Msg> acting(int iter) {
        // 断点：观察工具调用
    }
}

// 2. ToolExecutor - 理解工具执行
public class ToolExecutor {
    public Mono<ToolResultBlock> execute(...) {
        // 断点：观察工具参数
    }
}

// 3. Msg - 理解消息结构
public class Msg {
    // 断点：观察消息内容
}
```

### 5.3 日志配置

```xml
<!-- logback.xml -->
<configuration>
    <!-- 开发环境 -->
    <springProfile name="dev">
        <logger name="io.agentscope" level="DEBUG"/>
        <logger name="reactor" level="DEBUG"/>
    </springProfile>
    
    <!-- 生产环境 -->
    <springProfile name="prod">
        <logger name="io.agentscope" level="INFO"/>
    </springProfile>
</configuration>
```

### 5.4 常见问题排查

**问题 1：工具没有被调用**

```java
// 检查工具是否正确注册
toolkit.getTools().forEach(tool -> {
    System.out.println("工具: " + tool.getName());
});

// 检查工具 schema 是否正确生成
toolkit.getToolSchemas().forEach(schema -> {
    System.out.println("Schema: " + schema.getName());
});
```

**问题 2：响应为空**

```java
// 添加调试 Hook
agent = ReActAgent.builder()
    .hook(new Hook() {
        @Override
        public Mono<PostReasoningEvent> onEvent(PostReasoningEvent event) {
            Msg msg = event.getReasoningMessage();
            System.out.println("响应消息: " + msg);
            System.out.println("内容块: " + msg.getContent());
            return Mono.just(event);
        }
    })
    .build();
```

**问题 3：内存泄漏**

```java
// 检查 Memory 大小
System.out.println("消息数: " + memory.getMessages().size());

// 定期清理
if (memory.getMessages().size() > 100) {
    // 只保留最近的 50 条
    for (int i = 0; i < 50; i++) {
        memory.deleteMessage(0);
    }
}
```

## 6. 性能优化

### 6.1 响应式编程最佳实践

```java
// 不要在响应式链中阻塞
// 错误 ❌
public Mono<Msg> process(Msg input) {
    Msg result = someBlockingCall(input);  // 阻塞！
    return Mono.just(result);
}

// 正确 ✅
public Mono<Msg> process(Msg input) {
    return Mono.fromCallable(() -> someBlockingCall(input))
        .subscribeOn(Schedulers.boundedElastic());  // 在独立线程执行
}

// 并行处理多个任务
public Mono<List<Msg>> processAll(List<Msg> inputs) {
    return Flux.fromIterable(inputs)
        .flatMap(this::process)  // 并行处理
        .collectList();
}
```

### 6.2 内存优化

```java
// 使用窗口记忆，避免无限增长
public class WindowMemory implements Memory {
    private final int windowSize;
    
    @Override
    public void addMessage(Msg message) {
        messages.add(message);
        
        // 保持窗口大小
        while (messages.size() > windowSize) {
            messages.remove(0);
        }
    }
}

// 使用摘要压缩历史
public class SummaryMemory implements Memory {
    private void summarizeIfNeeded() {
        if (messages.size() > threshold) {
            String summary = summarizeMessages(messages.subList(0, half));
            messages.add(0, Msg.builder()
                .textContent("历史摘要: " + summary)
                .build());
        }
    }
}
```

### 6.3 工具优化

```java
// 使用缓存
@Tool(name = "cached_tool")
public String cachedTool(String input) {
    return cache.computeIfAbsent(input, this::expensiveOperation);
}

// 使用异步
@Tool(name = "async_tool")
public Mono<String> asyncTool(String input) {
    return Mono.fromCallable(() -> process(input))
        .timeout(Duration.ofSeconds(30))
        .onErrorReturn("处理超时");
}
```

## 7. 提交规范

### 7.1 Commit Message 格式

```
<type>(<scope>): <subject>

[optional body]
```

类型：
- `feat`: 新功能
- `fix`: Bug 修复
- `docs`: 文档更新
- `style`: 代码格式（不影响功能）
- `refactor`: 重构
- `test`: 测试相关
- `chore`: 构建/工具相关

示例：
```
feat(tool): 添加文件操作工具

- 新增 read_file 工具
- 新增 write_file 工具
- 支持 UTF-8 编码
```

### 7.2 提交前检查

```bash
# 1. 格式化代码
mvn spotless:apply

# 2. 运行测试
mvn test

# 3. 检查代码风格
mvn checkstyle:check
```

## 8. 下一步

- [常见问题](./06-faq.md)：问题排查和解决方案