# AgentScope-Java 项目文档

> 面向新手的完整入门指南（对应仓库当前代码：`2.1.0-SNAPSHOT`，已发布版本 `2.0.x`）

## 文档导航

| 文档 | 内容 | 适合人群 |
|------|------|----------|
| [项目概述](./01-overview.md) | 项目定位、整体架构、模块划分、技术选型 | 所有人 |
| [快速开始](./02-quickstart.md) | 环境搭建、编译、运行第一个 Agent | 新手入门 |
| [核心概念](./03-concepts.md) | Agent、Model、State、Tool、Message、Middleware、Event 等 | 进阶学习 |
| [示例代码](./04-examples.md) | `agentscope-examples` 真实示例目录与说明 | 实战参考 |
| [开发指南](./05-development.md) | 代码规范、扩展开发、测试、调试 | 开发者 |
| [常见问题](./06-faq.md) | 常见问题排查 | 问题排查 |

## 这是什么项目？

**AgentScope-Java** 是一个用 Java 编写的 **AI Agent 开发框架**，面向生产环境：
既能几行代码搭出一个能对话的助手，也能支撑长时间运行、分布式部署的企业级 Agent。

简单来说，它帮你：
- 🤖 快速创建能对话的 AI 助手（`ReActAgent` / `HarnessAgent`）
- 🔧 让 AI 调用各种工具（`@Tool`、MCP、内置文件/搜索工具）
- 🧠 管理会话状态与记忆（`AgentState` + `AgentStateStore`）
- 🔄 编排多 Agent 协作（Subagent、Team、Channel）
- 🛡️ 控制风险（权限引擎、人机确认、沙箱执行）
- 🚀 支撑长任务（Durable 持久化运行时、断线重连、跨副本恢复）

## 核心能力

### 1. ReAct 智能体

采用 **ReAct（Reasoning + Acting）** 模式：AI 先"思考"再"行动"，循环直到任务完成。

```java
ReActAgent agent = ReActAgent.builder()
        .name("Assistant")
        .sysPrompt("You are a helpful AI assistant.")
        .model("dashscope:qwen-plus")   // 需要引入 dashscope 模型扩展
        .toolkit(new Toolkit())
        .build();

Msg reply = agent.call(new UserMessage("你好")).block();
```

### 2. 工具调用

用注解定义工具，AI 会自动决定何时调用：

```java
public class WeatherTools {
    @Tool(name = "get_weather", description = "获取城市天气")
    public String getWeather(@ToolParam(name = "city", description = "城市名") String city) {
        return city + " 今天晴天，25°C";
    }
}

Toolkit toolkit = new Toolkit();
toolkit.registerTool(new WeatherTools());
```

### 3. 事件流

`streamEvents()` 返回 **32 种类型化事件**，可直接驱动前端渲染、进度条与人机确认：

```java
agent.streamEvents(new UserMessage("介绍一下你自己"))
        .doOnNext(event -> {
            if (event instanceof TextBlockDeltaEvent e) {
                System.out.print(e.getDelta());
            }
        })
        .blockLast();
```

### 4. 多 Agent 协作

- **Subagent**：`agent_spawn` / `agent_send`，同步或后台委派
- **HarnessAgent**：在 ReActAgent 之上叠加工作区、技能、记忆压缩、上下文管理等工程能力
- **Channel**：接入钉钉、飞书、企业微信等 IM 渠道

## 技术栈

| 类别 | 技术 | 版本 | 说明 |
|------|------|------|------|
| 语言 | Java | 17 | 项目 `maven.compiler.release` 固定为 17 |
| 构建 | Maven | 3.6+ | 多模块聚合构建，`${revision}` 版本管理 |
| 响应式 | Project Reactor | BOM 2025.0.2 | `Mono` / `Flux` 异步流 |
| JSON | Jackson | BOM 2.21.1 | 消息、Schema、状态序列化 |
| 日志 | SLF4J | 2.0.17 | 统一日志门面 |
| 测试 | JUnit Jupiter + Mockito | 6.0.2 / 5.23.0 | 单元测试与打桩 |
| 覆盖率 | JaCoCo | 0.8.14 | `mvn test` 生成报告 |
| 代码格式 | Spotless + google-java-format | 3.4.0 / 1.28.0 | **AOSP 风格，4 空格缩进** |

> 没有 Checkstyle；代码风格由 Spotless 强制（`mvn spotless:check` 绑定在 `compile` 阶段）。

## 项目结构

```
agentscope-java/
├── pom.xml                          # 聚合父 POM（7 个 module）
├── AGENTS.md                        # AI 编码规范 + 项目地图
├── CONTEXT.md                       # Channel / 会话术语表
├── SKILL.md                         # Agent Skill 说明
├── docs/                            # Mintlify 文档站（v1 / v2 / dev）
├── mydoc/                           # 本文档（中文入门）
│
├── agentscope-core/                 # 核心：ReAct 循环、消息、事件、工具、中间件、状态
├── agentscope-harness/              # 工程层：工作区、记忆、子 Agent、技能、沙箱
├── agentscope-durable/              # 持久化运行时：任务、检查点、恢复
│   ├── durable-runtime/
│   ├── durable-adapters/
│   ├── durable-worker/
│   └── protocol/
├── agentscope-extensions/           # 27 个扩展子模块
│   ├── agentscope-extensions-model/         # 模型提供方（dashscope/openai/anthropic/gemini/ollama/...）
│   ├── agentscope-extensions-rag-*/         # RAG（simple/dify/ragflow/haystack/bailian）
│   ├── agentscope-extensions-{redis,mysql,postgresql,mongodb,jdbc}/  # 存储
│   ├── agentscope-extensions-mem/{mem0,reme,memory-bailian}/        # 长期记忆
│   ├── agentscope-extensions-{oss,cos}/                              # 对象存储
│   ├── agentscope-extensions-{protocol,channel,sandbox,skills}/
│   ├── agentscope-extensions-durable-*/                             # Durable 适配器
│   ├── agentscope-extensions-{nacos,higress,studio,judge,jev,scheduler,training}/
│   └── agentscope-spring-boot-starters/                             # 14 个 Spring Boot Starter
├── agentscope-examples/             # 可运行示例
│   ├── documentation/               # 主线示例（quickstart/tool/middleware/mcp/rag/skill/...)
│   ├── jev/                         # JEV 评估示例
│   ├── durable/                     # Durable 集成示例
│   └── agents/                      # agentscope-chat / codingagent / dataagent / paw
├── agentscope-dependencies-bom/     # 第三方依赖版本管理
└── agentscope-distribution/         # agentscope-all / agentscope-bom
```

## 五分钟快速体验

### 1. 准备环境

```bash
java -version   # 需要 JDK 17
mvn -v
```

### 2. 克隆项目

```bash
git clone https://github.com/agentscope-ai/agentscope-java.git
cd agentscope-java
```

网络不稳定时可用浅克隆：`git clone --depth 1 <url>`。

### 3. 设置 API Key

```bash
# 使用阿里云 DashScope（推荐）
export DASHSCOPE_API_KEY=你的API密钥     # Windows CMD: set DASHSCOPE_API_KEY=...
```

获取地址：https://dashscope.console.aliyun.com/apiKey

### 4. 构建并运行第一个示例

```bash
# 先安装到本地仓库（示例模块依赖其他模块）
mvn -B -T1 install -DskipTests

# 运行基础对话示例
mvn exec:java -pl agentscope-examples/documentation \
    -Dexec.mainClass=io.agentscope.examples.documentation2.quickstart.BasicChatExample
```

## 学习路径

```
第 1 天: 读「项目概述」→ 跑通 BasicChatExample
         ↓
第 2 天: 读「核心概念」→ 理解 Agent / Model / State / Tool
         ↓
第 3 天: 跑 ToolCallingExample → 学会写工具
         ↓
第 4 天: 跑 StructuredOutputExample / StreamingConsoleExample → 结构化输出与事件流
         ↓
第 5 天: 跑 StateExample / CustomizedMiddlewareExample → 状态持久化与中间件
         ↓
第 6 天: 跑 McpStdioExample / SkillCompositionExample → 外部工具与技能
         ↓
第 7 天: 读 ReActAgent 源码 → 自己开发一个 Agent
```

## 相关资源

- 📖 [官方文档](https://java.agentscope.io/)
- 💬 [Discord 社区](https://discord.gg/eYMpfnkG8h)
- 🐛 [问题反馈](https://github.com/agentscope-ai/agentscope-java/issues)
- 📝 [贡献指南](../CONTRIBUTING.md)
- 🤖 [AI 编码规范](../AGENTS.md)
