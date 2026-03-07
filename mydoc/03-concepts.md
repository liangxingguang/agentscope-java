# 核心概念

本文档详细介绍 AgentScope-Java 的核心概念和设计原理。

## 1. Agent（智能体）

### 1.1 什么是 Agent？

Agent 是 AI 应用中的"大脑"，它能够：
- 理解用户的输入
- 决定下一步行动
- 调用工具完成任务
- 记住之前的对话

### 1.2 ReActAgent

AgentScope-Java 的核心 Agent 实现，采用 **ReAct（Reasoning + Acting）** 模式：

```
┌─────────────────────────────────────────────────────────────┐
│                      ReActAgent                              │
│                                                              │
│    用户输入 ──→ Reasoning(思考) ──→ Acting(行动) ──→ 输出   │
│                     ↑               │                       │
│                     └───────────────┘                       │
│                        (循环直到完成)                        │
└─────────────────────────────────────────────────────────────┘
```

### 1.3 创建 Agent

```java
ReActAgent agent = ReActAgent.builder()
    // 基础配置
    .name("助手")                    // Agent 名称
    .description("一个通用的 AI 助手") // 描述信息
    .sysPrompt("你是一个友好的助手")  // 系统提示词
    
    // 核心组件
    .model(model)                    // 语言模型（必须）
    .memory(memory)                  // 记忆系统（必须）
    .toolkit(toolkit)                // 工具箱（可选）
    
    // 执行配置
    .maxIters(10)                    // 最大迭代次数
    
    .build();
```

### 1.4 Agent 配置详解

| 配置项 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| name | String | 否 | Agent 名称，用于标识 |
| sysPrompt | String | 否 | 系统提示词，定义 Agent 行为 |
| model | Model | 是 | 语言模型 |
| memory | Memory | 是 | 记忆系统 |
| toolkit | Toolkit | 否 | 工具集合 |
| maxIters | int | 否 | 最大迭代次数，默认 10 |
| hooks | List&lt;Hook&gt; | 否 | 钩子列表 |

### 1.5 Agent 调用方式

```java
// 同步调用（阻塞等待结果）
Msg response = agent.call(userMsg).block();

// 异步调用（非阻塞）
Mono<Msg> responseMono = agent.call(userMsg);
responseMono.subscribe(response -> {
    System.out.println(response.getTextContent());
});

// 流式调用（实时输出）
Flux<Msg> stream = agent.stream(userMsg);
stream.subscribe(chunk -> {
    System.out.print(chunk.getTextContent());
});
```

## 2. Model（模型）

### 2.1 什么是 Model？

Model 是与大语言模型（LLM）通信的接口，负责：
- 发送消息给 AI
- 接收 AI 的响应
- 处理流式输出

### 2.2 Model 接口

```java
public interface Model {
    /**
     * 流式调用模型
     * @param messages 消息列表
     * @param tools 工具 schema 列表
     * @param options 生成选项
     * @return 响应流
     */
    Flux<ChatResponse> stream(
        List<Msg> messages, 
        List<ToolSchema> tools, 
        GenerateOptions options
    );
    
    /**
     * 获取模型名称
     */
    String getModelName();
}
```

### 2.3 支持的模型实现

#### DashScopeChatModel（阿里云）

```java
DashScopeChatModel model = DashScopeChatModel.builder()
    .apiKey("your-api-key")
    .modelName("qwen-plus")          // qwen-plus, qwen-max, qwen-turbo
    .stream(true)                     // 启用流式输出
    .enableThinking(true)             // 启用思考过程
    .formatter(new DashScopeChatFormatter()) // 消息格式化器
    .build();
```

#### OpenAIChatModel

```java
OpenAIChatModel model = OpenAIChatModel.builder()
    .apiKey("your-api-key")
    .modelName("gpt-4")              // gpt-4, gpt-3.5-turbo
    .baseUrl("https://api.openai.com/v1") // 可选，自定义 API 地址
    .build();
```

#### AnthropicChatModel（Claude）

```java
AnthropicChatModel model = AnthropicChatModel.builder()
    .apiKey("your-api-key")
    .modelName("claude-3-opus")      // claude-3-opus, claude-3-sonnet
    .build();
```

### 2.4 生成选项

```java
GenerateOptions options = GenerateOptions.builder()
    .temperature(0.7)         // 温度，控制随机性 (0-2)
    .maxTokens(4096)          // 最大输出 token 数
    .topP(0.9)                // Top-p 采样
    .thinkingBudget(1024)     // 思考 token 预算
    .build();
```

## 3. Memory（记忆）

### 3.1 什么是 Memory？

Memory 管理 Agent 的对话历史，让 AI 能够"记住"之前的对话。

### 3.2 Memory 接口

```java
public interface Memory {
    // 添加消息
    void addMessage(Msg message);
    
    // 获取所有消息
    List<Msg> getMessages();
    
    // 删除指定消息
    void deleteMessage(int index);
    
    // 清空所有消息
    void clear();
}
```

### 3.3 Memory 实现

#### InMemoryMemory（内存存储）

最简单的实现，对话保存在内存中，重启后丢失。

```java
Memory memory = new InMemoryMemory();
```

#### JsonSession（文件持久化）

对话保存到 JSON 文件，重启后可以恢复。

```java
Session session = new JsonSession("sessions/my-session.json");
Memory memory = session.getMemory();

// 保存会话
session.save();

// 加载会话
session.load();
```

#### LongTermMemory（长期记忆）

使用向量数据库存储，支持语义检索。

```java
LongTermMemory memory = LongTermMemory.builder()
    .embeddingModel(embeddingModel)  // 向量化模型
    .vectorStore(vectorStore)        // 向量数据库
    .build();
```

### 3.4 记忆流程

```
用户输入
    │
    ▼
┌─────────────────┐
│     Memory      │ ← 保存用户消息
│  [历史消息列表]  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│     Model       │ ← 发送历史消息 + 新消息
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│     Memory      │ ← 保存 AI 响应
└─────────────────┘
```

## 4. Tool（工具）

### 4.1 什么是 Tool？

Tool 让 AI 能够执行实际操作，如：
- 查询天气
- 搜索网络
- 计算数学
- 操作文件

### 4.2 定义工具

使用 `@Tool` 注解定义工具：

```java
public class WeatherTools {
    
    @Tool(
        name = "get_weather",              // 工具名称
        description = "获取指定城市的天气"   // 工具描述
    )
    public String getWeather(
        @ToolParam(
            name = "city",                 // 参数名称
            description = "城市名称，如：北京、上海"  // 参数描述
        ) String city,
        @ToolParam(
            name = "unit",
            description = "温度单位：celsius 或 fahrenheit"
        ) String unit
    ) {
        // 实际实现
        return String.format("%s 今天晴天，温度 25°%s", city, unit);
    }
}
```

### 4.3 注册工具

```java
Toolkit toolkit = new Toolkit();

// 方式一：注册对象（自动发现所有 @Tool 方法）
toolkit.registerTool(new WeatherTools());

// 方式二：注册单个方法
toolkit.registerTool(new WeatherTools(), "getWeather");
```

### 4.4 工具执行流程

```
AI 决定调用工具
        │
        ▼
┌───────────────────┐
│   ToolExecutor    │ ← 解析参数
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│   执行工具方法    │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│   返回结果        │ → 发送给 AI
└───────────────────┘
```

### 4.5 工具返回类型

```java
// 返回字符串（最简单）
@Tool(name = "get_time")
public String getTime() {
    return LocalTime.now().toString();
}

// 返回 Mono<String>（异步）
@Tool(name = "fetch_url")
public Mono<String> fetchUrl(String url) {
    return webClient.get(url);
}

// 使用自定义转换器
@Tool(
    name = "get_data",
    converter = JsonConverter.class
)
public Data getData(String id) {
    return dataService.findById(id);
}
```

### 4.6 ToolGroup（工具组）

将工具分组，按需激活：

```java
// 创建工具组
ToolGroup mathGroup = new ToolGroup("math_ops", "数学运算");
mathGroup.addTool(calculatorTool);

ToolGroup fileGroup = new ToolGroup("file_ops", "文件操作");
fileGroup.addTool(readFileTool);

// 注册到 Toolkit
toolkit.registerGroup(mathGroup);
toolkit.registerGroup(fileGroup);

// 激活特定组
toolkit.setActiveGroups(Set.of("math_ops"));
```

## 5. Message（消息）

### 5.1 什么是 Message？

Message 是 Agent 之间、Agent 与用户之间通信的基本单位。

### 5.2 消息结构

```java
Msg msg = Msg.builder()
    .id("uuid")                    // 唯一标识
    .name("用户名")                 // 发送者名称
    .role(MsgRole.USER)            // 角色：USER, ASSISTANT, SYSTEM, TOOL
    .content(contentBlocks)        // 内容块列表
    .metadata(metadata)            // 元数据
    .timestamp("2024-01-15 10:00:00") // 时间戳
    .build();
```

### 5.3 内容块类型

| 类型 | 说明 | 示例 |
|------|------|------|
| TextBlock | 文本内容 | "你好" |
| ImageBlock | 图片内容 | base64 或 URL |
| ToolUseBlock | 工具调用 | {"name": "get_weather", "args": {...}} |
| ToolResultBlock | 工具结果 | {"result": "晴天 25°C"} |
| ThinkingBlock | 思考过程 | AI 的推理过程 |

### 5.4 创建消息

```java
// 纯文本消息
Msg textMsg = Msg.builder()
    .role(MsgRole.USER)
    .textContent("你好，今天天气怎么样？")
    .build();

// 包含图片的消息
Msg imageMsg = Msg.builder()
    .role(MsgRole.USER)
    .content(
        TextBlock.builder().text("这张图片是什么？").build(),
        ImageBlock.builder().url("https://example.com/image.jpg").build()
    )
    .build();

// 从响应中提取信息
String text = response.getTextContent();
List<ToolUseBlock> tools = response.getContentBlocks(ToolUseBlock.class);
```

## 6. Hook（钩子）

### 6.1 什么是 Hook？

Hook 允许你在 Agent 执行过程中插入自定义逻辑，如：
- 监控执行过程
- 修改输入输出
- 记录日志

### 6.2 Hook 类型

| 事件 | 触发时机 | 用途 |
|------|----------|------|
| PreReasoningEvent | 思考前 | 修改输入消息 |
| ReasoningChunkEvent | 思考流式输出 | 实时显示思考过程 |
| PostReasoningEvent | 思考后 | 处理思考结果 |
| PreActingEvent | 行动前 | 修改工具调用 |
| ActingChunkEvent | 行动流式输出 | 实时显示工具执行 |
| PostActingEvent | 行动后 | 处理工具结果 |

### 6.3 实现自定义 Hook

```java
public class LoggingHook implements Hook {
    
    private static final Logger log = LoggerFactory.getLogger(LoggingHook.class);
    
    @Override
    public Mono<PreReasoningEvent> onEvent(PreReasoningEvent event) {
        log.info("开始思考，输入消息数: {}", event.getInputMessages().size());
        return Mono.just(event);
    }
    
    @Override
    public Mono<PostReasoningEvent> onEvent(PostReasoningEvent event) {
        log.info("思考完成，生成消息: {}", event.getReasoningMessage().getTextContent());
        return Mono.just(event);
    }
}

// 使用 Hook
ReActAgent agent = ReActAgent.builder()
    .hook(new LoggingHook())
    .build();
```

### 6.4 Hook 执行顺序

```
┌─────────────────────────────────────────────────────────┐
│                    Agent 执行流程                        │
│                                                         │
│  PreReasoningEvent                                      │
│        │                                                │
│        ▼                                                │
│  ReasoningChunkEvent (多次)                             │
│        │                                                │
│        ▼                                                │
│  PostReasoningEvent                                     │
│        │                                                │
│        ├─── 需要工具？──→ PreActingEvent                │
│        │                      │                         │
│        │                      ▼                         │
│        │               ActingChunkEvent                 │
│        │                      │                         │
│        │                      ▼                         │
│        │               PostActingEvent                  │
│        │                      │                         │
│        ◄──────────────────────┘                         │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## 7. Session（会话）

### 7.1 什么是 Session？

Session 管理 Agent 的状态持久化，支持：
- 保存/加载对话历史
- 管理会话 ID
- 跨进程共享状态

### 7.2 Session 使用

```java
// 创建会话
Session session = new JsonSession("sessions/chat.json");

// 加载历史
session.load();

// 创建 Agent（使用会话的 Memory）
ReActAgent agent = ReActAgent.builder()
    .memory(session.getMemory())
    .build();

// 对话...
agent.call(userMsg).block();

// 保存会话
session.save();
```

### 7.3 Session 实现

| 实现 | 存储位置 | 适用场景 |
|------|----------|----------|
| InMemorySession | 内存 | 临时对话 |
| JsonSession | JSON 文件 | 单机持久化 |
| RedisSession | Redis | 分布式部署 |
| MySQLSession | MySQL | 企业级应用 |

## 8. RAG（检索增强生成）

### 8.1 什么是 RAG？

RAG 让 AI 能够访问外部知识库，提高回答的准确性和时效性：

```
用户问题
    │
    ▼
┌─────────────────┐
│   向量检索      │ ← 在知识库中查找相关内容
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   上下文组装    │ ← 将检索结果加入提示词
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   模型生成      │ ← AI 基于上下文回答
└─────────────────┘
```

### 8.2 使用 RAG

```java
// 创建知识库
Knowledge knowledge = Knowledge.builder()
    .name("产品文档")
    .documents(documents)       // 文档列表
    .embeddingModel(embedding)  // 向量化模型
    .vectorStore(vectorStore)   // 向量存储
    .build();

// 配置 Agent
ReActAgent agent = ReActAgent.builder()
    .knowledge(knowledge)
    .ragMode(RAGMode.GENERIC)   // RAG 模式
    .build();
```

## 9. 响应式编程

### 9.1 Project Reactor

AgentScope-Java 使用 Project Reactor 进行响应式编程：

```java
// Mono: 0 或 1 个元素
Mono<Msg> singleResponse = agent.call(userMsg);

// Flux: 0 到 N 个元素
Flux<Msg> streamingResponse = agent.stream(userMsg);
```

### 9.2 常用操作

```java
// 阻塞获取结果
Msg response = agent.call(userMsg).block();

// 异步回调
agent.call(userMsg).subscribe(response -> {
    System.out.println(response.getTextContent());
});

// 链式处理
agent.call(userMsg)
    .map(Msg::getTextContent)
    .filter(text -> text.length() > 100)
    .subscribe(System.out::println);

// 错误处理
agent.call(userMsg)
    .onErrorResume(e -> {
        log.error("调用失败", e);
        return Mono.just(fallbackResponse);
    })
    .subscribe(System.out::println);
```

## 10. 下一步

- [示例代码](./04-examples.md)：查看更多完整示例
- [开发指南](./05-development.md)：学习开发规范