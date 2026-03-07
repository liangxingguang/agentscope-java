# 示例代码说明

本文档详细介绍 agentscope-examples 中的各个示例，帮助你理解如何使用 AgentScope-Java。

## 示例概览

| 示例 | 难度 | 主要功能 | 核心知识点 |
|------|------|----------|------------|
| BasicChatExample | ⭐ | 基础对话 | Agent 创建、Model 配置 |
| ToolCallingExample | ⭐⭐ | 工具调用 | @Tool 注解、Toolkit |
| StructuredOutputExample | ⭐⭐ | 结构化输出 | Schema 定义、类型转换 |
| HookExample | ⭐⭐ | 钩子系统 | Hook 接口、事件处理 |
| SessionExample | ⭐⭐ | 会话管理 | JsonSession、持久化 |
| ToolGroupExample | ⭐⭐⭐ | 工具组 | ToolGroup、动态激活 |
| McpToolExample | ⭐⭐⭐ | MCP 集成 | MCP 协议、外部工具 |
| InterruptionExample | ⭐⭐⭐ | 中断机制 | InterruptedException |

---

## 1. BasicChatExample（基础对话）

### 功能说明
最简单的示例，展示如何创建一个能对话的 AI Agent。

### 核心代码

```java
public class BasicChatExample {
    public static void main(String[] args) {
        // 获取 API Key
        String apiKey = ExampleUtils.getDashScopeApiKey();

        // 创建 Agent
        ReActAgent agent = ReActAgent.builder()
                .name("Assistant")
                .sysPrompt("你是一个友好的 AI 助手。")
                .model(DashScopeChatModel.builder()
                        .apiKey(apiKey)
                        .modelName("qwen-plus")
                        .stream(true)              // 启用流式输出
                        .enableThinking(true)      // 启用思考过程
                        .formatter(new DashScopeChatFormatter())
                        .build())
                .memory(new InMemoryMemory())
                .toolkit(new Toolkit())
                .build();

        // 开始对话
        ExampleUtils.startChat(agent);
    }
}
```

### 运行方式

```bash
mvn exec:java -Dexec.mainClass="io.agentscope.examples.quickstart.BasicChatExample"
```

### 学习要点

1. **Agent 构建器模式**：使用 `builder()` 链式配置
2. **Model 配置**：DashScope 模型的基本配置
3. **流式输出**：`stream(true)` 让响应逐字显示
4. **思考过程**：`enableThinking(true)` 显示 AI 推理过程

### 代码调用链

```
main()
  └── ExampleUtils.getDashScopeApiKey()  // 获取 API Key
  └── ReActAgent.builder()               // 创建 Builder
        ├── .name()
        ├── .sysPrompt()
        ├── .model()
        │     └── DashScopeChatModel.builder()
        ├── .memory()
        └── .build()                     // 构建 Agent
  └── ExampleUtils.startChat(agent)      // 开始交互
        └── Scanner.nextLine()           // 读取用户输入
        └── agent.call(msg)              // 调用 Agent
              └── ReActAgent.doCall()    // 内部实现
                    └── reasoning()      // 思考阶段
                    └── acting()         // 行动阶段
```

---

## 2. ToolCallingExample（工具调用）

### 功能说明
展示如何让 AI 调用自定义工具。

### 核心代码

```java
public class ToolCallingExample {
    public static void main(String[] args) {
        String apiKey = ExampleUtils.getDashScopeApiKey();

        // 创建工具箱并注册工具
        Toolkit toolkit = new Toolkit();
        toolkit.registerTool(new SimpleTools());

        // 创建 Agent
        ReActAgent agent = ReActAgent.builder()
                .name("ToolAgent")
                .sysPrompt("你是一个有工具的助手。需要时使用工具回答问题。")
                .model(DashScopeChatModel.builder()
                        .apiKey(apiKey)
                        .modelName("qwen-max")
                        .build())
                .toolkit(toolkit)          // 配置工具箱
                .memory(new InMemoryMemory())
                .build();

        ExampleUtils.startChat(agent);
    }

    // 工具定义
    public static class SimpleTools {
        
        @Tool(name = "get_current_time", description = "获取指定时区的当前时间")
        public String getCurrentTime(
                @ToolParam(name = "timezone", description = "时区，如 Asia/Tokyo")
                String timezone) {
            // 实现逻辑
        }

        @Tool(name = "calculate", description = "计算数学表达式")
        public String calculate(
                @ToolParam(name = "expression", description = "数学表达式")
                String expression) {
            // 实现逻辑
        }

        @Tool(name = "search", description = "搜索信息")
        public String search(
                @ToolParam(name = "query", description = "搜索关键词")
                String query) {
            // 实现逻辑
        }
    }
}
```

### 运行方式

```bash
mvn exec:java -Dexec.mainClass="io.agentscope.examples.quickstart.ToolCallingExample"
```

### 体验建议

输入以下问题，观察 AI 如何选择和调用工具：
- "东京现在几点？" → 调用 `get_current_time`
- "123 * 456 等于多少？" → 调用 `calculate`
- "搜索一下人工智能" → 调用 `search`

### 代码调用链

```
用户输入: "东京现在几点？"
    │
    ▼
ReActAgent.call()
    │
    ▼
reasoning() ──────────────────────────────────
    │                                         │
    │  AI 思考: "我需要查询东京时间"            │
    │  AI 决定: 调用 get_current_time 工具     │
    │                                         │
    ▼                                         │
acting()                                      │
    │                                         │
    ▼                                         │
ToolExecutor.execute()                        │
    │                                         │
    ├── 解析参数: timezone="Asia/Tokyo"       │
    │                                         │
    ▼                                         │
SimpleTools.getCurrentTime("Asia/Tokyo")      │
    │                                         │
    ├── 返回: "东京时间: 2024-01-15 14:30:00"  │
    │                                         │
    ▼                                         │
reasoning() (第二轮) ─────────────────────────┘
    │
    │  AI 思考: "我已经获得了东京时间"
    │  AI 决定: 直接回答
    │
    ▼
返回结果: "东京现在时间是 2024-01-15 14:30:00"
```

---

## 3. StructuredOutputExample（结构化输出）

### 功能说明
让 AI 返回结构化的 JSON 数据，方便程序处理。

### 核心代码

```java
public class StructuredOutputExample {
    public static void main(String[] args) {
        String apiKey = ExampleUtils.getDashScopeApiKey();

        ReActAgent agent = ReActAgent.builder()
                .name("Extractor")
                .model(DashScopeChatModel.builder()
                        .apiKey(apiKey)
                        .modelName("qwen-plus")
                        .build())
                .memory(new InMemoryMemory())
                .toolkit(new Toolkit())
                .build();

        // 定义输出结构
        @Data
        class ProductRequirements {
            private String productType;
            private String brand;
            private int minRam;
            private double maxBudget;
            private List<String> features;
        }

        // 提取结构化数据
        String input = "我想买一台苹果笔记本，至少16GB内存，预算2万以内，要轻便易携带";
        
        ProductRequirements result = agent.call(
            Msg.builder().textContent(input).build(),
            ProductRequirements.class  // 指定输出类型
        ).block().getStructuredData(ProductRequirements.class);

        System.out.println("产品类型: " + result.getProductType());
        System.out.println("品牌: " + result.getBrand());
        System.out.println("内存: " + result.getMinRam() + "GB");
        System.out.println("预算: " + result.getMaxBudget());
    }
}
```

### 运行方式

```bash
mvn exec:java -Dexec.mainClass="io.agentscope.examples.quickstart.StructuredOutputExample"
```

### 学习要点

1. **Schema 自动生成**：根据 Java 类自动生成 JSON Schema
2. **类型安全**：返回强类型对象，无需手动解析 JSON
3. **自动校验**：AI 输出不符合 Schema 时自动重试

---

## 4. HookExample（钩子系统）

### 功能说明
展示如何使用 Hook 监控和干预 Agent 执行过程。

### 核心代码

```java
public class HookExample {
    public static void main(String[] args) {
        String apiKey = ExampleUtils.getDashScopeApiKey();

        ReActAgent agent = ReActAgent.builder()
                .name("MonitoredAgent")
                .model(DashScopeChatModel.builder()
                        .apiKey(apiKey)
                        .modelName("qwen-plus")
                        .build())
                .memory(new InMemoryMemory())
                .toolkit(new Toolkit())
                .hook(new LoggingHook())     // 添加日志钩子
                .hook(new ProgressHook())    // 添加进度钩子
                .build();

        ExampleUtils.startChat(agent);
    }

    // 日志钩子
    static class LoggingHook implements Hook {
        @Override
        public Mono<PreReasoningEvent> onEvent(PreReasoningEvent event) {
            System.out.println("[LOG] 开始思考...");
            return Mono.just(event);
        }

        @Override
        public Mono<PostReasoningEvent> onEvent(PostReasoningEvent event) {
            System.out.println("[LOG] 思考完成");
            return Mono.just(event);
        }
    }

    // 进度钩子
    static class ProgressHook implements Hook {
        @Override
        public Mono<ReasoningChunkEvent> onEvent(ReasoningChunkEvent event) {
            System.out.print(".");  // 显示进度点
            return Mono.just(event);
        }
    }
}
```

### 运行方式

```bash
mvn exec:java -Dexec.mainClass="io.agentscope.examples.quickstart.HookExample"
```

### 学习要点

1. **Hook 接口**：实现 `Hook` 接口，重写需要的事件方法
2. **事件类型**：不同阶段触发不同事件
3. **事件修改**：可以修改事件内容，影响后续流程

---

## 5. SessionExample（会话管理）

### 功能说明
展示如何持久化对话历史，支持跨会话恢复。

### 核心代码

```java
public class SessionExample {
    public static void main(String[] args) {
        String apiKey = ExampleUtils.getDashScopeApiKey();

        // 创建或加载会话
        System.out.print("输入会话ID: ");
        String sessionId = new Scanner(System.in).nextLine();
        
        Session session = new JsonSession("sessions/" + sessionId + ".json");
        session.load();  // 加载历史

        // 创建 Agent
        ReActAgent agent = ReActAgent.builder()
                .name("PersistentAgent")
                .model(DashScopeChatModel.builder()
                        .apiKey(apiKey)
                        .modelName("qwen-plus")
                        .build())
                .memory(session.getMemory())  // 使用会话的 Memory
                .toolkit(new Toolkit())
                .build();

        // 对话
        ExampleUtils.startChat(agent);

        // 保存会话
        session.save();
    }
}
```

### 运行方式

```bash
mvn exec:java -Dexec.mainClass="io.agentscope.examples.quickstart.SessionExample"
```

### 体验流程

```
第一次运行:
输入会话ID: alice
You: 我叫小明，我喜欢吃披萨
AI: 你好小明！很高兴认识你。

第二次运行:
输入会话ID: alice
You: 我叫什么名字？我喜欢吃什么？
AI: 你叫小明，你喜欢吃披萨。  # AI 记住了之前的信息！
```

---

## 6. ToolGroupExample（工具组）

### 功能说明
展示如何将工具分组，让 AI 按需激活。

### 核心代码

```java
public class ToolGroupExample {
    public static void main(String[] args) {
        Toolkit toolkit = new Toolkit();

        // 创建工具组
        ToolGroup mathGroup = new ToolGroup("math_ops", "数学运算");
        mathGroup.addTool(new MathTools());

        ToolGroup networkGroup = new ToolGroup("network_ops", "网络操作");
        networkGroup.addTool(new NetworkTools());

        ToolGroup fileGroup = new ToolGroup("file_ops", "文件操作");
        fileGroup.addTool(new FileTools());

        // 注册工具组
        toolkit.registerGroup(mathGroup);
        toolkit.registerGroup(networkGroup);
        toolkit.registerGroup(fileGroup);

        // 默认所有组都是 INACTIVE
        // Agent 会通过 meta-tool 自动激活需要的组

        ReActAgent agent = ReActAgent.builder()
                .name("SmartAgent")
                .model(model)
                .toolkit(toolkit)
                .enableMetaTool(true)  // 启用 meta-tool
                .build();
    }
}
```

### 运行方式

```bash
mvn exec:java -Dexec.mainClass="io.agentscope.examples.quickstart.ToolGroupExample"
```

### 学习要点

1. **ToolGroup**：将相关工具组织在一起
2. **激活状态**：只有激活的组才能被调用
3. **Meta-tool**：让 AI 自动决定激活哪些组

---

## 7. McpToolExample（MCP 工具）

### 功能说明
展示如何使用 MCP（Model Context Protocol）协议集成外部工具。

### 核心代码

```java
public class McpToolExample {
    public static void main(String[] args) {
        Toolkit toolkit = new Toolkit();

        // 创建 MCP 客户端
        McpClient mcpClient = McpClient.builder()
                .command("npx", "-y", "@modelcontextprotocol/server-filesystem", "/tmp")
                .build();

        // 注册 MCP 工具
        toolkit.registerMcpTools(mcpClient);

        ReActAgent agent = ReActAgent.builder()
                .name("McpAgent")
                .model(model)
                .toolkit(toolkit)
                .build();

        // 现在 Agent 可以使用文件系统工具
        // 如: list_files, read_file, write_file 等
    }
}
```

### 运行方式

```bash
# 先安装 MCP 服务器
npm install -g @modelcontextprotocol/server-filesystem

# 运行示例
mvn exec:java -Dexec.mainClass="io.agentscope.examples.quickstart.McpToolExample"
```

---

## 8. RAG 示例

### 功能说明
展示如何使用 RAG（检索增强生成）让 AI 访问外部知识库。

### 支持的 RAG 实现

| 示例 | 说明 |
|------|------|
| PgVectorRAGExample | 使用 PostgreSQL + pgvector |
| HayStackRAGExample | 使用 Haystack 框架 |
| DifyRAGExample | 使用 Dify 平台 |
| RAGFlowRAGExample | 使用 RAGFlow |

### 核心代码

```java
public class PgVectorRAGExample {
    public static void main(String[] args) {
        // 创建向量存储
        VectorStore vectorStore = new PgVectorStore(
            "jdbc:postgresql://localhost:5432/agentscope",
            "username",
            "password"
        );

        // 创建知识库
        Knowledge knowledge = Knowledge.builder()
                .name("产品文档")
                .documents(loadDocuments())
                .embeddingModel(embeddingModel)
                .vectorStore(vectorStore)
                .build();

        // 创建 Agent
        ReActAgent agent = ReActAgent.builder()
                .name("KnowledgeAgent")
                .model(model)
                .knowledge(knowledge)
                .ragMode(RAGMode.GENERIC)
                .build();
    }
}
```

---

## 9. Pipeline 示例

### 功能说明
展示如何使用 Pipeline 组合多个 Agent。

### SequentialPipelineExample

顺序执行多个 Agent：

```java
SequentialPipeline pipeline = new SequentialPipeline();
pipeline.addAgent(researchAgent);   // 第一步：研究
pipeline.addAgent(writingAgent);    // 第二步：写作
pipeline.addAgent(reviewAgent);     // 第三步：审核

Msg result = pipeline.execute(userMsg).block();
```

### FanoutPipelineExample

并行执行多个 Agent：

```java
FanoutPipeline pipeline = new FanoutPipeline();
pipeline.addAgent(weatherAgent);    // 并行查天气
pipeline.addAgent(newsAgent);       // 并行查新闻
pipeline.addAgent(stockAgent);      // 并行查股票

List<Msg> results = pipeline.execute(userMsg).block();
```

---

## 10. 调试技巧

### 10.1 添加日志

```java
// 在代码中添加日志
private static final Logger log = LoggerFactory.getLogger(YourClass.class);

log.debug("当前消息数: {}", memory.getMessages().size());
log.debug("工具调用: {}", toolUseBlock.getName());
```

### 10.2 使用 IDEA 调试

推荐断点位置：

```java
// ReActAgent.java

// 思考阶段入口
private Mono<Msg> reasoning(int iter, boolean ignoreMaxIters) {
    // 断点：观察每次思考的输入
}

// 行动阶段入口
private Mono<Msg> acting(int iter) {
    // 断点：观察工具调用
}

// ToolExecutor.java

// 工具执行
public ToolResultBlock execute(ToolUseBlock toolUse) {
    // 断点：观察工具参数和返回值
}
```

### 10.3 查看消息内容

```java
// 打印消息详情
Msg msg = agent.call(userMsg).block();
System.out.println("消息ID: " + msg.getId());
System.out.println("角色: " + msg.getRole());
System.out.println("内容: " + msg.getTextContent());
System.out.println("元数据: " + msg.getMetadata());
```

---

## 下一步

- [开发指南](./05-development.md)：学习开发规范和最佳实践
- [常见问题](./06-faq.md)：排查常见问题