# 快速开始

本文档将帮助你在 IDEA 中搭建开发环境，并运行第一个 AgentScope-Java 程序。

## 1. 环境准备

### 1.1 系统要求

| 软件 | 版本要求 | 检查命令 |
|------|----------|----------|
| JDK | 17 或更高 | `java -version` |
| Maven | 3.6 或更高 | `mvn -version` |
| Git | 任意版本 | `git --version` |
| IDEA | 2022.1 或更高 | - |

### 1.2 安装 JDK 17

**Windows**:
1. 下载 [Oracle JDK 17](https://www.oracle.com/java/technologies/downloads/#java17) 或 [OpenJDK 17](https://adoptium.net/)
2. 运行安装程序
3. 配置环境变量 `JAVA_HOME`

**macOS**:
```bash
# 使用 Homebrew
brew install openjdk@17

# 配置环境变量
echo 'export PATH="/usr/local/opt/openjdk@17/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

**Linux**:
```bash
# Ubuntu/Debian
sudo apt update
sudo apt install openjdk-17-jdk

# CentOS/RHEL
sudo yum install java-17-openjdk-devel
```

**验证安装**:
```bash
java -version
# 输出应该类似：openjdk version "17.0.x"
```

### 1.3 安装 Maven

**Windows**:
1. 下载 [Maven](https://maven.apache.org/download.cgi)
2. 解压到 `C:\Program Files\Apache\maven`
3. 添加到环境变量 `PATH`

**macOS/Linux**:
```bash
# 使用 Homebrew (macOS)
brew install maven

# 或手动安装
wget https://dlcdn.apache.org/maven/maven-3/3.9.6/binaries/apache-maven-3.9.6-bin.tar.gz
tar -xzf apache-maven-3.9.6-bin.tar.gz
sudo mv apache-maven-3.9.6 /opt/maven
echo 'export PATH="/opt/maven/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

**验证安装**:
```bash
mvn -version
# 输出应该包含 Maven 版本信息
```

## 2. 获取源码

### 2.1 克隆项目

```bash
# 克隆仓库
git clone https://github.com/agentscope-ai/agentscope-java.git

# 进入项目目录
cd agentscope-java
```

### 2.2 项目结构概览

```
agentscope-java/
├── pom.xml                    # 父 POM，管理所有模块
├── agentscope-core/           # 核心模块
├── agentscope-extensions/     # 扩展模块
├── agentscope-examples/       # 示例代码
├── agentscope-dependencies-bom/ # 依赖版本管理
└── agentscope-distribution/   # 发布包
```

## 3. IDEA 环境配置

### 3.1 导入项目

1. **打开 IDEA**，选择 `File` → `Open`
2. 选择项目根目录的 `pom.xml` 文件
3. 选择 `Open as Project`
4. 等待 Maven 自动导入依赖（可能需要几分钟）

### 3.2 配置 JDK

1. 打开 `File` → `Project Structure` (快捷键 `Ctrl+Alt+Shift+S`)
2. 选择 `Project`
3. 设置 `SDK` 为 JDK 17
4. 设置 `Language level` 为 `17 - Sealed types, always-strict floating-point semantics`

### 3.3 配置 Maven

1. 打开 `File` → `Settings` (macOS: `Preferences`)
2. 导航到 `Build, Execution, Deployment` → `Build Tools` → `Maven`
3. 设置：
   - `Maven home path`: 选择你的 Maven 安装目录
   - `User settings file`: 选择你的 Maven settings.xml

### 3.4 配置代码格式化

项目使用 Spotless 进行代码格式化：

1. 打开 `Settings` → `Tools` → `Actions on Save`
2. 勾选 `Reformat code` 和 `Optimize imports`
3. 或者手动运行：`mvn spotless:apply`

### 3.5 安装推荐插件

| 插件 | 用途 |
|------|------|
| Lombok | 简化 Java 代码（本项目未使用，但推荐） |
| Maven Helper | Maven 依赖分析 |
| GitToolBox | Git 增强 |
| Rainbow Brackets | 彩虹括号 |

## 4. 获取 API Key

AgentScope-Java 需要大语言模型的 API Key。

### 4.1 阿里云 DashScope（推荐）

**获取步骤**：
1. 访问 [阿里云 DashScope](https://dashscope.console.aliyun.com/)
2. 登录/注册阿里云账号
3. 点击左侧菜单 `API-KEY 管理`
4. 点击 `创建新的 API-KEY`
5. 复制保存 API Key

**设置环境变量**：
```bash
# Linux/macOS
export DASHSCOPE_API_KEY=sk-xxxxxxxxxxxxxxxx

# Windows (CMD)
set DASHSCOPE_API_KEY=sk-xxxxxxxxxxxxxxxx

# Windows (PowerShell)
$env:DASHSCOPE_API_KEY="sk-xxxxxxxxxxxxxxxx"
```

### 4.2 OpenAI（可选）

```bash
export OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxx
```

### 4.3 Anthropic（可选）

```bash
export ANTHROPIC_API_KEY=sk-xxxxxxxxxxxxxxxx
```

## 5. 编译项目

### 5.1 命令行编译

```bash
# 编译整个项目（跳过测试，加快速度）
mvn clean install -DskipTests

# 只编译核心模块
mvn clean install -pl agentscope-core -DskipTests
```

### 5.2 IDEA 中编译

1. 打开右侧 Maven 面板
2. 展开 `agentscope-parent` → `Lifecycle`
3. 双击 `install`

### 5.3 常见编译问题

**问题 1：依赖下载失败**
```bash
# 使用阿里云镜像，编辑 ~/.m2/settings.xml
<mirror>
    <id>aliyun</id>
    <mirrorOf>central</mirrorOf>
    <name>Aliyun Maven</name>
    <url>https://maven.aliyun.com/repository/public</url>
</mirror>
```

**问题 2：JDK 版本不对**
```
# 确认 JAVA_HOME
echo $JAVA_HOME

# macOS 使用 jenv 管理多版本
brew install jenv
jenv add /path/to/jdk17
jenv global 17
```

**问题 3：Spotless 检查失败**
```bash
# 自动修复格式问题
mvn spotless:apply
```

## 6. 运行第一个示例

### 6.1 运行 BasicChatExample

这是最简单的示例，展示了如何创建一个能对话的 AI Agent。

**方式一：命令行运行**
```bash
cd agentscope-examples/quickstart
mvn exec:java -Dexec.mainClass="io.agentscope.examples.quickstart.BasicChatExample"
```

**方式二：IDEA 中运行**
1. 打开 `agentscope-examples/quickstart/src/main/java/io/agentscope/examples/quickstart/BasicChatExample.java`
2. 右键点击 `main` 方法
3. 选择 `Run 'BasicChatExample.main()'`

### 6.2 预期输出

```
========================================
  Basic Chat Example
  This example demonstrates the simplest Agent setup.
  You'll chat with an AI assistant powered by DashScope.
========================================

Starting chat with Assistant. Type 'exit' to quit.

You: 你好，介绍一下你自己
Assistant: 你好！我是一个 AI 助手，由阿里云通义千问模型驱动。
我可以回答问题、提供建议、进行对话。有什么我可以帮你的吗？

You: exit
Goodbye!
```

### 6.3 示例代码解读

```java
public class BasicChatExample {
    public static void main(String[] args) {
        // 1. 获取 API Key（从环境变量或交互式输入）
        String apiKey = ExampleUtils.getDashScopeApiKey();

        // 2. 创建 Agent
        ReActAgent agent = ReActAgent.builder()
                .name("Assistant")                    // 设置名称
                .sysPrompt("你是一个友好的 AI 助手")   // 设置系统提示
                .model(                               // 配置模型
                        DashScopeChatModel.builder()
                                .apiKey(apiKey)
                                .modelName("qwen-plus")  // 使用通义千问
                                .stream(true)            // 启用流式输出
                                .build())
                .memory(new InMemoryMemory())         // 使用内存记忆
                .toolkit(new Toolkit())               // 工具箱（本例为空）
                .build();

        // 3. 开始对话
        ExampleUtils.startChat(agent);
    }
}
```

## 7. 调试技巧

### 7.1 设置断点

1. 在代码行号左侧点击，添加红色断点
2. 右键选择 `Debug` 运行
3. 程序会在断点处暂停

### 7.2 推荐断点位置

```java
// ReActAgent.java - 理解 ReAct 循环
private Mono<Msg> reasoning(int iter, boolean ignoreMaxIters) {
    // 在这里打断点，观察每次思考
}

private Mono<Msg> acting(int iter) {
    // 在这里打断点，观察工具调用
}
```

### 7.3 查看日志

配置日志级别：

```java
// 在 resources 目录创建 logback.xml
<configuration>
    <logger name="io.agentscope" level="DEBUG"/>
</configuration>
```

或在代码中：
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

private static final Logger log = LoggerFactory.getLogger(YourClass.class);

log.debug("调试信息: {}", someVariable);
```

### 7.4 IDEA 调试快捷键

| 快捷键 | 功能 |
|--------|------|
| F8 | 单步执行（Step Over） |
| F7 | 进入方法（Step Into） |
| Shift+F8 | 跳出方法（Step Out） |
| F9 | 继续执行（Resume） |
| Ctrl+F8 | 切换断点 |

## 8. 运行更多示例

### 8.1 工具调用示例

```bash
mvn exec:java -Dexec.mainClass="io.agentscope.examples.quickstart.ToolCallingExample"
```

**体验内容**：
- 询问时间："东京现在几点？"
- 计算数学："123 * 456 等于多少？"
- 搜索信息："搜索一下人工智能"

### 8.2 结构化输出示例

```bash
mvn exec:java -Dexec.mainClass="io.agentscope.examples.quickstart.StructuredOutputExample"
```

**体验内容**：
- 从自然语言提取结构化数据
- 生成符合 Schema 的 JSON

### 8.3 会话持久化示例

```bash
mvn exec:java -Dexec.mainClass="io.agentscope.examples.quickstart.SessionExample"
```

**体验内容**：
- 保存对话历史到文件
- 下次启动时恢复对话

## 9. 开发你的第一个 Agent

### 9.1 创建新项目

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.agentscope</groupId>
    <artifactId>agentscope-core</artifactId>
    <version>1.0.9</version>
</dependency>
```

### 9.2 编写代码

```java
import io.agentscope.core.ReActAgent;
import io.agentscope.core.memory.InMemoryMemory;
import io.agentscope.core.model.DashScopeChatModel;
import io.agentscope.core.tool.Toolkit;
import io.agentscope.core.message.Msg;
import io.agentscope.core.message.TextBlock;

public class MyFirstAgent {
    public static void main(String[] args) {
        // 创建 Agent
        ReActAgent agent = ReActAgent.builder()
                .name("小助手")
                .sysPrompt("你是一个热心的助手，用简洁的语言回答问题。")
                .model(DashScopeChatModel.builder()
                        .apiKey(System.getenv("DASHSCOPE_API_KEY"))
                        .modelName("qwen-plus")
                        .build())
                .memory(new InMemoryMemory())
                .toolkit(new Toolkit())
                .build();

        // 发送消息
        Msg response = agent.call(Msg.builder()
                .textContent("你好！")
                .build()).block();

        // 打印响应
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
        return "为什么程序员总是分不清万圣节和圣诞节？\n" +
               "因为 Oct 31 = Dec 25";
    }
    
    @Tool(name = "roll_dice", description = "掷骰子")
    public String rollDice(
            @ToolParam(name = "sides", description = "骰子面数") int sides) {
        int result = (int) (Math.random() * sides) + 1;
        return "掷出了 " + result + " 点";
    }
}

// 使用
Toolkit toolkit = new Toolkit();
toolkit.registerTool(new MyTools());
```

## 10. 常见问题

### Q: API Key 配置了还是报错？
A: 检查环境变量是否生效：
```bash
echo $DASHSCOPE_API_KEY
```

### Q: Maven 下载依赖很慢？
A: 配置阿里云镜像，见 5.3 节。

### Q: 代码格式化报错？
A: 运行 `mvn spotless:apply` 自动修复。

### Q: 想用 GPT-4 怎么办？
A: 使用 `OpenAIChatModel`：
```java
.model(OpenAIChatModel.builder()
        .apiKey(System.getenv("OPENAI_API_KEY"))
        .modelName("gpt-4")
        .build())
```

## 11. 下一步

- [核心概念](./03-concepts.md)：深入学习 Agent、Model、Memory 等核心概念
- [示例代码](./04-examples.md)：了解各种示例的详细说明
- [开发指南](./05-development.md)：学习开发规范和最佳实践