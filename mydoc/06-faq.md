# 常见问题 (FAQ)

本文档收集了使用 AgentScope-Java 过程中的常见问题和解决方案。

---

## 环境配置问题

### Q1: JDK 版本不对怎么办？

**问题现象**：
```
错误: 不支持发行版本 17
```

**解决方案**：
1. 检查当前 JDK 版本：
```bash
java -version
```

2. 安装 JDK 17 或更高版本：
- Windows: 下载 [Oracle JDK](https://www.oracle.com/java/technologies/downloads/) 或 [OpenJDK](https://adoptium.net/)
- macOS: `brew install openjdk@17`
- Linux: `sudo apt install openjdk-17-jdk`

3. 配置环境变量：
```bash
# macOS/Linux
export JAVA_HOME=/path/to/jdk-17
export PATH=$JAVA_HOME/bin:$PATH

# Windows
set JAVA_HOME=C:\Program Files\Java\jdk-17
set PATH=%JAVA_HOME%\bin;%PATH%
```

---

### Q2: Maven 下载依赖很慢怎么办？

**问题现象**：
依赖下载超时或速度很慢。

**解决方案**：
配置阿里云镜像，编辑 `~/.m2/settings.xml`：

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

**问题现象**：
```
Execution failed for task ':spotlessCheck'.
```

**解决方案**：
运行自动格式化：
```bash
mvn spotless:apply
```

---

## API Key 问题

### Q4: API Key 配置了还是报错？

**问题现象**：
```
错误: API Key 未设置
```

**解决方案**：

1. 检查环境变量是否设置：
```bash
# Linux/macOS
echo $DASHSCOPE_API_KEY

# Windows CMD
echo %DASHSCOPE_API_KEY%

# Windows PowerShell
echo $env:DASHSCOPE_API_KEY
```

2. 正确设置环境变量：
```bash
# Linux/macOS (临时)
export DASHSCOPE_API_KEY=sk-xxxxxxxx

# Linux/macOS (永久，添加到 ~/.bashrc 或 ~/.zshrc)
echo 'export DASHSCOPE_API_KEY=sk-xxxxxxxx' >> ~/.bashrc
source ~/.bashrc

# Windows CMD (临时)
set DASHSCOPE_API_KEY=sk-xxxxxxxx

# Windows PowerShell (临时)
$env:DASHSCOPE_API_KEY="sk-xxxxxxxx"
```

3. 在 IDEA 中配置：
   - 打开 `Run` → `Edit Configurations`
   - 在 `Environment variables` 中添加 `DASHSCOPE_API_KEY=sk-xxxxxxxx`

---

### Q5: 如何获取 DashScope API Key？

**解决方案**：

1. 访问 [阿里云 DashScope 控制台](https://dashscope.console.aliyun.com/)
2. 登录/注册阿里云账号
3. 点击左侧菜单 `API-KEY 管理`
4. 点击 `创建新的 API-KEY`
5. 复制并保存 API Key

**注意**：API Key 只会显示一次，请妥善保存！

---

## Agent 运行问题

### Q6: Agent 没有调用工具？

**问题现象**：
AI 直接回答，没有调用已注册的工具。

**可能原因和解决方案**：

1. **系统提示词没有说明有工具可用**
```java
.sysPrompt("你是一个助手，可以使用提供的工具来回答问题。")
```

2. **工具描述不够清晰**
```java
@Tool(
    name = "get_weather",
    description = "获取指定城市的当前天气信息。当用户询问天气时使用此工具。"
)
```

3. **工具没有正确注册**
```java
Toolkit toolkit = new Toolkit();
toolkit.registerTool(new MyTools());

// 检查工具是否注册成功
toolkit.getTools().forEach(t -> System.out.println("工具: " + t.getName()));
```

4. **模型不支持工具调用**
使用支持 Function Calling 的模型，如 `qwen-plus`、`qwen-max`、`gpt-4` 等。

---

### Q7: Agent 响应为空？

**问题现象**：
`response.getTextContent()` 返回空字符串。

**排查步骤**：

1. 检查消息内容块：
```java
Msg response = agent.call(input).block();
System.out.println("内容块数量: " + response.getContent().size());
response.getContent().forEach(block -> {
    System.out.println("块类型: " + block.getClass().getSimpleName());
});
```

2. 检查是否有工具调用：
```java
List<ToolUseBlock> tools = response.getContentBlocks(ToolUseBlock.class);
if (!tools.isEmpty()) {
    System.out.println("有工具调用，需要继续执行");
}
```

3. 检查 Agent 状态：
```java
System.out.println("生成原因: " + response.getGenerateReason());
// MODEL_STOP: 正常结束
// TOOL_SUSPENDED: 工具挂起
// MAX_ITERATIONS: 达到最大迭代
```

---

### Q8: Agent 进入无限循环？

**问题现象**：
Agent 一直循环，没有结束。

**解决方案**：

1. 设置最大迭代次数：
```java
ReActAgent.builder()
    .maxIters(10)  // 最多 10 轮
    .build();
```

2. 添加停止条件的 Hook：
```java
.hook(new Hook() {
    @Override
    public Mono<PostReasoningEvent> onEvent(PostReasoningEvent event) {
        Msg msg = event.getReasoningMessage();
        if (msg.getTextContent().contains("完成")) {
            event.requestStop();
        }
        return Mono.just(event);
    }
})
```

3. 检查系统提示词，引导 AI 正确结束。

---

### Q9: 内存占用越来越大？

**问题现象**：
长时间运行后内存占用持续增长。

**解决方案**：

1. 使用窗口记忆：
```java
// 自定义窗口记忆，只保留最近 N 条消息
public class WindowMemory implements Memory {
    private final int maxSize = 50;
    // ...
}
```

2. 定期清理记忆：
```java
if (memory.getMessages().size() > 100) {
    memory.clear();  // 或删除部分消息
}
```

3. 使用会话管理：
```java
Session session = new JsonSession("session.json");
// 每次新会话重新开始
session.clear();
```

---

## 工具调用问题

### Q10: 工具参数解析错误？

**问题现象**：
```
错误: 无法解析参数 xxx
```

**解决方案**：

1. 确保参数类型正确：
```java
// 正确
@Tool(name = "calculate")
public String calculate(
    @ToolParam(name = "a", description = "第一个数字") int a,
    @ToolParam(name = "b", description = "第二个数字") int b
) { }

// 错误 - AI 可能传入字符串
@Tool(name = "calculate")
public String calculate(int a, int b) { }  // 没有 @ToolParam
```

2. 添加参数验证：
```java
@Tool(name = "divide")
public String divide(
    @ToolParam(name = "a") double a,
    @ToolParam(name = "b") double b
) {
    if (b == 0) {
        return "错误: 除数不能为 0";
    }
    return String.valueOf(a / b);
}
```

---

### Q11: 工具执行超时？

**问题现象**：
工具执行时间过长，导致 Agent 卡住。

**解决方案**：

1. 设置超时配置：
```java
ExecutionConfig config = ExecutionConfig.builder()
    .timeout(Duration.ofSeconds(30))
    .build();

ReActAgent.builder()
    .toolExecutionConfig(config)
    .build();
```

2. 使用异步工具：
```java
@Tool(name = "long_running")
public Mono<String> longRunning(String input) {
    return Mono.fromCallable(() -> doWork(input))
        .timeout(Duration.ofSeconds(30))
        .onErrorReturn("操作超时");
}
```

---

## 模型问题

### Q12: 如何切换到 OpenAI GPT-4？

**解决方案**：

```java
// 添加依赖
// <dependency>
//     <groupId>io.agentscope</groupId>
//     <artifactId>agentscope-openai</artifactId>
// </dependency>

ReActAgent agent = ReActAgent.builder()
    .model(OpenAIChatModel.builder()
        .apiKey(System.getenv("OPENAI_API_KEY"))
        .modelName("gpt-4")
        .build())
    .build();
```

---

### Q13: 流式输出不显示？

**问题现象**：
设置 `stream(true)` 但没有实时输出。

**解决方案**：

使用 Hook 捕获流式输出：
```java
.hook(new Hook() {
    @Override
    public Mono<ReasoningChunkEvent> onEvent(ReasoningChunkEvent event) {
        System.out.print(event.getChunkMessage().getTextContent());
        return Mono.just(event);
    }
})
```

---

## 调试问题

### Q14: 如何查看 Agent 的思考过程？

**解决方案**：

1. 启用 thinking 模式：
```java
DashScopeChatModel.builder()
    .enableThinking(true)
    .build();
```

2. 使用 Hook 记录：
```java
.hook(new Hook() {
    @Override
    public Mono<PreReasoningEvent> onEvent(PreReasoningEvent event) {
        System.out.println("=== 思考输入 ===");
        event.getInputMessages().forEach(m -> 
            System.out.println(m.getTextContent()));
        return Mono.just(event);
    }
    
    @Override
    public Mono<PostReasoningEvent> onEvent(PostReasoningEvent event) {
        System.out.println("=== 思考输出 ===");
        System.out.println(event.getReasoningMessage().getTextContent());
        return Mono.just(event);
    }
})
```

---

### Q15: 如何调试工具调用？

**解决方案**：

在工具方法中添加日志：
```java
@Tool(name = "my_tool")
public String myTool(String param) {
    System.out.println("工具被调用，参数: " + param);
    
    String result = doWork(param);
    
    System.out.println("工具返回: " + result);
    return result;
}
```

或使用断点调试：
- 在工具方法第一行设置断点
- 使用 IDEA 的 Debug 模式运行

---

## 其他问题

### Q16: 如何处理敏感信息？

**解决方案**：

1. 使用环境变量存储 API Key：
```java
String apiKey = System.getenv("DASHSCOPE_API_KEY");
```

2. 不要在代码中硬编码：
```java
// 错误 ❌
.model(DashScopeChatModel.builder()
    .apiKey("sk-xxxxxxxx")  // 不要这样写！
    .build())

// 正确 ✅
.model(DashScopeChatModel.builder()
    .apiKey(System.getenv("DASHSCOPE_API_KEY"))
    .build())
```

3. 使用 `.gitignore` 排除敏感文件：
```
# .gitignore
.env
secrets.properties
*-local.properties
```

---

### Q17: 如何实现多轮对话？

**解决方案**：

使用 Memory 保存对话历史：

```java
// 创建 Agent（Memory 会自动保存历史）
ReActAgent agent = ReActAgent.builder()
    .memory(new InMemoryMemory())
    .build();

// 多次调用，历史会自动保留
agent.call(Msg.builder().textContent("我叫小明").build()).block();
agent.call(Msg.builder().textContent("我叫什么？").build()).block();
// AI 会回答"你叫小明"
```

---

### Q18: 如何在 Spring Boot 中使用？

**解决方案**：

1. 添加依赖：
```xml
<dependency>
    <groupId>io.agentscope</groupId>
    <artifactId>agentscope-spring-boot-starter</artifactId>
</dependency>
```

2. 配置 application.yml：
```yaml
agentscope:
  dashscope:
    api-key: ${DASHSCOPE_API_KEY}
    model-name: qwen-plus
```

3. 注入使用：
```java
@Service
public class AgentService {
    
    @Autowired
    private ReActAgent agent;
    
    public String chat(String input) {
        Msg response = agent.call(Msg.builder()
            .textContent(input)
            .build()).block();
        return response.getTextContent();
    }
}
```

---

### Q19: 项目可以商用吗？

**解答**：

AgentScope-Java 采用 **Apache 2.0 许可证**，可以免费用于商业用途。

主要条款：
- ✅ 可以商用
- ✅ 可以修改
- ✅ 可以分发
- ⚠️ 需要保留版权声明
- ⚠️ 需要说明代码的修改部分

---

### Q20: 如何获取帮助？

**资源链接**：

| 资源 | 链接 |
|------|------|
| 官方文档 | https://java.agentscope.io/ |
| GitHub 仓库 | https://github.com/agentscope-ai/agentscope-java |
| 问题反馈 | https://github.com/agentscope-ai/agentscope-java/issues |
| Discord 社区 | https://discord.gg/eYMpfnkG8h |

**提问建议**：
1. 先搜索 [Issues](https://github.com/agentscope-ai/agentscope-java/issues) 看是否有类似问题
2. 提供完整的环境信息（JDK 版本、Maven 版本、操作系统）
3. 提供可复现的代码示例
4. 提供完整的错误日志

---

## 问题未解决？

如果你的问题不在上述列表中：

1. 查阅 [官方文档](https://java.agentscope.io/)
2. 搜索 [GitHub Issues](https://github.com/agentscope-ai/agentscope-java/issues)
3. 在 [Discord](https://discord.gg/eYMpfnkG8h) 社区提问
4. 提交新的 [Issue](https://github.com/agentscope-ai/agentscope-java/issues/new)