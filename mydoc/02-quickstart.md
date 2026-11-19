# 快速开始

本文档帮助你在 IDEA 或命令行中搭建开发环境，并运行第一个 AgentScope-Java 程序。

## 1. 环境准备

### 1.1 系统要求

| 软件 | 版本要求 | 检查命令 |
|------|----------|----------|
| JDK | **17**（编译目标固定为 17） | `java -version` |
| Maven | 3.6 或更高 | `mvn -version` |
| Git | 任意版本 | `git --version` |
| IDEA | 2023.x 或更高（推荐） | - |

### 1.2 安装 JDK 17

**Windows**：
1. 下载 [Eclipse Temurin JDK 17](https://adoptium.net/) 或 [Oracle JDK 17](https://www.oracle.com/java/technologies/downloads/#java17)
2. 运行安装程序
3. 配置环境变量 `JAVA_HOME`，并把 `%JAVA_HOME%\bin` 加入 `Path`

**macOS**：
```bash
brew install openjdk@17
echo 'export JAVA_HOME=$(/usr/libexec/java_home -v 17)' >> ~/.zshrc
source ~/.zshrc
```

**Linux**：
```bash
# Ubuntu/Debian
sudo apt update && sudo apt install openjdk-17-jdk
# CentOS/RHEL
sudo yum install java-17-openjdk-devel
```

**验证**：
```bash
java -version
# openjdk version "17.0.x" ...
```

### 1.3 安装 Maven

**macOS/Linux**：
```bash
brew install maven          # macOS
# 或手动安装后把 bin 目录加入 PATH
```

**Windows**：下载二进制包解压，把 `apache-maven-3.x/bin` 加入 `Path`。

**验证**：
```bash
mvn -version
```

## 2. 获取源码

```bash
git clone https://github.com/agentscope-ai/agentscope-java.git
cd agentscope-java
```

网络慢时可用浅克隆（本项目文件较多，建议这样起步）：

```bash
git clone --depth 1 https://github.com/agentscope-ai/agentscope-java.git
```

### 2.1 项目结构概览

```
agentscope-java/
├── pom.xml                          # 聚合父 POM
├── agentscope-core/                 # 核心（ReActAgent、消息、事件、工具、中间件、状态）
├── agentscope-harness/              # 工程层（HarnessAgent、工作区、记忆、技能、子 Agent）
├── agentscope-durable/              # 持久化运行时（长任务、检查点、恢复）
├── agentscope-extensions/           # 扩展（模型、存储、RAG、MCP、沙箱、Spring Boot Starter…）
├── agentscope-examples/             # 示例（documentation / jev / durable / agents/*）
├── agentscope-dependencies-bom/     # 第三方依赖版本管理
└── agentscope-distribution/         # 发布 BOM 与聚合分发
```

## 3. IDEA 环境配置

### 3.1 导入项目

1. `File` → `Open`
2. 选择根目录的 `pom.xml`
3. 选择 `Open as Project`，等待 Maven 导入完成

### 3.2 配置 JDK

1. `File` → `Project Structure`（`Ctrl+Alt+Shift+S`）
2. `Project` → `SDK` 选 JDK 17
3. `Language level` 选 `17`

### 3.3 配置 Maven

1. `Settings` → `Build, Execution, Deployment` → `Build Tools` → `Maven`
2. 设置 `Maven home path` 与 `User settings file`

### 3.4 代码格式化

项目用 **Spotless + google-java-format（AOSP 风格，4 空格缩进）**，`compile` 阶段会执行 `spotless:check`：

```bash
mvn spotless:apply     # 自动格式化
mvn spotless:check     # 仅检查
```

IDEA 中建议开启 `Settings` → `Tools` → `Actions on Save` → `Reformat code` + `Optimize imports`。

> 项目**没有** Checkstyle；Lombok 也**未使用**（示例中的 record / 普通 POJO 即可）。

### 3.5 推荐插件

| 插件 | 用途 |
|------|------|
| Maven Helper | 依赖冲突分析 |
| GitToolBox | Git 增强 |
| Rainbow Brackets | 括号着色 |

## 4. 获取 API Key

### 4.1 阿里云 DashScope（推荐）

1. 访问 [DashScope 控制台](https://dashscope.console.aliyun.com/)
2. 登录/注册阿里云账号
3. 左侧菜单 `API-KEY 管理` → `创建新的 API-KEY`
4. 复制保存

```bash
# Linux/macOS
export DASHSCOPE_API_KEY=sk-xxxxxxxxxxxxxxxx
# Windows CMD
set DASHSCOPE_API_KEY=sk-xxxxxxxxxxxxxxxx
# Windows PowerShell
$env:DASHSCOPE_API_KEY="sk-xxxxxxxxxxxxxxxx"
```

### 4.2 OpenAI / Anthropic（可选）

```bash
export OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxx
export ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxx
```

> Key 由各模型扩展的 `ModelProvider` 读取；`model("dashscope:qwen-plus")` 会自动从
> `DASHSCOPE_API_KEY` 取值。

## 5. 编译项目

### 5.1 命令行

```bash
# 全量构建（安装到本地仓库，示例模块依赖它）
mvn -B -T1 install -DskipTests

# 只构建核心模块
mvn -B install -pl agentscope-core -am -DskipTests

# 只编译不测试
mvn -B compile
```

### 5.2 IDEA 中编译

右侧 Maven 面板 → `agentscope-parent` → `Lifecycle` → 双击 `install`。

### 5.3 常见编译问题

**问题 1：依赖下载慢**

编辑 `~/.m2/settings.xml`：
```xml
<mirror>
    <id>aliyun</id>
    <mirrorOf>central</mirrorOf>
    <name>Aliyun Maven</name>
    <url>https://maven.aliyun.com/repository/public</url>
</mirror>
```

**问题 2：JDK 版本不对**

```
[ERROR] invalid target release: 17
```
确认 `JAVA_HOME` 指向 JDK 17，或 `mvn -v` 输出的 Java version 是 17。

**问题 3：Spotless 检查失败**

```
[ERROR] Failed to execute goal com.diffplug.spotless:spotless-maven-plugin ... spotless:check
```
执行 `mvn spotless:apply` 后重新编译。

**问题 4：`agentscope-examples` 找不到依赖**

示例模块依赖其他模块的 SNAPSHOT，需要先 `install`：
```bash
mvn -B -T1 install -DskipTests
# 或只安装示例所需的上游
mvn -B install -pl agentscope-examples/documentation -am -DskipTests
```

## 6. 运行第一个示例

`agentscope-examples` 的示例集中在 **`documentation`** 子模块，包名统一为
`io.agentscope.examples.documentation2.*`。

> 历史文档中的 `agentscope-examples/quickstart` 模块**已不存在**，对应示例已迁入
> `documentation` 子模块的 `quickstart` 包。

### 6.1 BasicChatExample（基础对话）

**命令行运行**：
```bash
mvn exec:java -pl agentscope-examples/documentation \
    -Dexec.mainClass=io.agentscope.examples.documentation2.quickstart.BasicChatExample
```

**IDEA 中运行**：
1. 打开 `agentscope-examples/documentation/src/main/java/io/agentscope/examples/documentation2/quickstart/BasicChatExample.java`
2. 点击 `main` 方法左侧的绿色箭头 → `Run`

### 6.2 预期输出

```
============================================================
Basic Chat Example
============================================================
A simple interactive chat with streaming output.
Type 'exit' to quit.

You: 你好，介绍一下你自己
Assistant: 你好！我是一个 AI 助手……
```

### 6.3 示例代码（真实源码）

```java
import io.agentscope.core.ReActAgent;
import io.agentscope.core.event.TextBlockDeltaEvent;
import io.agentscope.core.message.Msg;
import io.agentscope.core.message.UserMessage;
import io.agentscope.core.tool.Toolkit;

public class BasicChatExample {

    public static void main(String[] args) throws Exception {
        // 1. 创建 Agent —— model(...) 由 ModelRegistry 自动解析 provider 并读取环境变量
        ReActAgent agent =
                ReActAgent.builder()
                        .name("Assistant")
                        .sysPrompt("You are a helpful AI assistant. Be friendly and concise.")
                        .model("dashscope:qwen-plus")
                        .toolkit(new Toolkit())
                        .build();

        // 2. 流式对话：streamEvents() 返回类型化事件
        Msg userMsg = new UserMessage("你好，介绍一下你自己");
        agent.streamEvents(userMsg)
                .doOnNext(
                        event -> {
                            if (event instanceof TextBlockDeltaEvent e) {
                                System.out.print(e.getDelta());
                            }
                        })
                .blockLast();
    }
}
```

**与旧版文档的关键差异**：

| 旧写法（v1） | 新写法（v2） |
|---|---|
| `.memory(new InMemoryMemory())` | 不需要；会话历史在 `AgentState` 中，需要持久化时用 `.stateStore(...)` |
| `DashScopeChatModel.builder().apiKey(...)` | 可用 `.model("dashscope:qwen-plus")`，也可继续显式构造 Builder |
| `agent.stream(msg)` 返回 `Flux<Msg>` | `agent.streamEvents(msg)` 返回 `Flux<AgentEvent>` |
| `Msg.builder().textContent(...)` | `new UserMessage("...")` / `Msg.builder()...build()` |
| `agent.call(msg)` 返回 `Mono<Msg>` | 同上（仍然可用） |

## 7. 调试技巧

### 7.1 设置断点

行号左侧点击添加断点，右键 `Debug` 运行。

### 7.2 推荐断点位置

`ReActAgent` 的主循环位于内部类 `CallExecution` 中：

```java
// agentscope-core/src/main/java/io/agentscope/core/ReActAgent.java
final class CallExecution {                                   // line ~2372
    private Mono<Msg> doCallInner(List<Msg> msgs) { ... }     // line ~2577
    private Mono<Msg> reasoning(int iter, boolean ignoreMaxIters) { ... }  // line ~3207
    private Mono<Msg> acting(int iter) { ... }                // line ~3900
}
```

- `reasoning()`：观察每轮思考的输入消息、模型响应与工具决策
- `acting()`：观察工具调用参数与执行结果

工具执行可断在 `io.agentscope.core.tool.Toolkit#callTool(ToolCallParam)`。

### 7.3 查看日志

项目使用 SLF4J。测试/示例目录下放一个 `logback.xml`：

```xml
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder><pattern>%d{HH:mm:ss.SSS} %-5level %logger{36} - %msg%n</pattern></encoder>
    </appender>
    <logger name="io.agentscope" level="DEBUG"/>
    <root level="INFO"><appender-ref ref="STDOUT"/></root>
</configuration>
```

### 7.4 IDEA 调试快捷键

| 快捷键 | 功能 |
|--------|------|
| F8 | Step Over |
| F7 | Step Into |
| Shift+F8 | Step Out |
| F9 | Resume |
| Ctrl+F8 | 切换断点 |

## 8. 运行更多示例

> 所有命令都在**仓库根目录**执行；第一次运行前请先 `mvn -B -T1 install -DskipTests`。

### 8.1 工具调用

```bash
mvn exec:java -pl agentscope-examples/documentation \
    -Dexec.mainClass=io.agentscope.examples.documentation2.tool.ToolCallingExample
```
体验：`东京现在几点？`、`123 * 456 等于多少？`

### 8.2 结构化输出

```bash
mvn exec:java -pl agentscope-examples/documentation \
    -Dexec.mainClass=io.agentscope.examples.documentation2.structuredoutput.StructuredOutputExample
```

### 8.3 事件流

```bash
mvn exec:java -pl agentscope-examples/documentation \
    -Dexec.mainClass=io.agentscope.examples.documentation2.streaming.StreamingConsoleExample
```

### 8.4 状态持久化（会话恢复）

```bash
mvn exec:java -pl agentscope-examples/documentation \
    -Dexec.mainClass=io.agentscope.examples.documentation2.state.StateExample
```
体验：输入同一个 session id 两次运行，第二次会恢复上次对话。

### 8.5 中间件监控

```bash
mvn exec:java -pl agentscope-examples/documentation \
    -Dexec.mainClass=io.agentscope.examples.documentation2.middleware.CustomizedMiddlewareExample
```

### 8.6 MCP 工具

```bash
npm install -g @modelcontextprotocol/server-filesystem

mvn exec:java -pl agentscope-examples/documentation \
    -Dexec.mainClass=io.agentscope.examples.documentation2.mcp.McpStdioExample
```

### 8.7 完整应用（HarnessAgent + Durable）

```bash
mvn -B -T1 -pl agentscope-examples/agents/agentscope-chat -am package -DskipTests
java -jar agentscope-examples/agents/agentscope-chat/target/agentscope-chat.jar
# 打开 http://127.0.0.1:8087
```
该示例默认使用离线模型，**不需要 API Key / 数据库 / Node.js**。

## 9. 在自己的项目中使用

### 9.1 添加依赖

使用已发布版本（例如 2.0.3）：

```xml
<dependency>
    <groupId>io.agentscope</groupId>
    <artifactId>agentscope-harness</artifactId>
    <version>2.0.3</version>
</dependency>

<!-- 模型 provider 是独立模块，按需引入 -->
<dependency>
    <groupId>io.agentscope</groupId>
    <artifactId>agentscope-extensions-model-dashscope</artifactId>
    <version>2.0.3</version>
</dependency>
```

只需要 ReAct 推理能力、不要 Harness 工程层时，把 `agentscope-harness` 换成
`agentscope-core` + 所需的模型扩展即可。

用源码本地 SNAPSHOT 时版本写 `2.1.0-SNAPSHOT`（需先 `mvn install`）。

### 9.2 编写第一个 Agent

```java
import io.agentscope.core.ReActAgent;
import io.agentscope.core.message.Msg;
import io.agentscope.core.message.UserMessage;
import io.agentscope.core.tool.Toolkit;

public class MyFirstAgent {
    public static void main(String[] args) {
        ReActAgent agent = ReActAgent.builder()
                .name("小助手")
                .sysPrompt("你是一个热心的助手，用简洁的语言回答问题。")
                .model("dashscope:qwen-plus")   // 读取 DASHSCOPE_API_KEY
                .toolkit(new Toolkit())
                .build();

        Msg response = agent.call(new UserMessage("你好！")).block();
        System.out.println("Agent: " + response.getTextContent());
    }
}
```

### 9.3 添加自定义工具

```java
import io.agentscope.core.tool.Tool;
import io.agentscope.core.tool.ToolParam;

public class MyTools {

    @Tool(name = "get_joke", description = "讲一个笑话")
    public String getJoke() {
        return "为什么程序员总是分不清万圣节和圣诞节？因为 Oct 31 = Dec 25";
    }

    @Tool(name = "roll_dice", description = "掷骰子")
    public String rollDice(
            @ToolParam(name = "sides", description = "骰子面数") int sides) {
        return "掷出了 " + ((int) (Math.random() * sides) + 1) + " 点";
    }
}

Toolkit toolkit = new Toolkit();
toolkit.registerTool(new MyTools());
```

### 9.4 多轮会话与持久化

```java
import io.agentscope.core.state.AgentStateStore;
import io.agentscope.core.state.JsonFileAgentStateStore;
import java.nio.file.Paths;

AgentStateStore store = new JsonFileAgentStateStore(
        Paths.get(System.getProperty("user.home"), ".agentscope", "sessions"));

ReActAgent agent = ReActAgent.builder()
        .name("Assistant")
        .model("dashscope:qwen-plus")
        .stateStore(store)              // 自动加载/保存
        .defaultSessionId("alice")      // 会话隔离
        .build();
```

## 10. 常见问题

**Q：API Key 配置了还报错？**
确认环境变量在**启动进程的环境**中生效（IDEA 要在 Run Configuration 的
`Environment variables` 里配置），`echo $DASHSCOPE_API_KEY` 应有值。

**Q：Maven 下载依赖很慢？**
使用阿里云镜像，见 5.3 节。

**Q：`spotless:check` 失败？**
运行 `mvn spotless:apply`。

**Q：想用 GPT-4？**
引入 `agentscope-extensions-model-openai`，然后：

```java
.model("openai:gpt-4o")
```
或显式构造：
```java
import io.agentscope.extensions.model.openai.OpenAIChatModel;

.model(OpenAIChatModel.builder()
        .apiKey(System.getenv("OPENAI_API_KEY"))
        .modelName("gpt-4o")
        .build())
```

**Q：找不到 `quickstart` 模块 / `ExampleUtils`？**
v2 已移除。示例在 `agentscope-examples/documentation`，公共工具类已被各示例内联。

## 11. 下一步

- [核心概念](./03-concepts.md)：深入理解 Agent、Model、State、Tool、Middleware
- [示例代码](./04-examples.md)：全部示例清单与说明
- [开发指南](./05-development.md)：开发规范与扩展开发
