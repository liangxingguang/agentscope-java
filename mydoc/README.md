# AgentScope-Java 项目文档

> 面向小白的完整入门指南

## 文档导航

| 文档 | 内容 | 适合人群 |
|------|------|----------|
| [项目概述](./01-overview.md) | 项目介绍、技术架构、模块说明 | 所有人 |
| [快速开始](./02-quickstart.md) | 环境搭建、运行调试、第一个程序 | 新手入门 |
| [核心概念](./03-concepts.md) | Agent、Model、Memory、Tool 等核心概念 | 进阶学习 |
| [示例代码](./04-examples.md) | 各类示例的详细说明 | 实战参考 |
| [开发指南](./05-development.md) | 代码开发规范、调试技巧 | 开发者 |
| [常见问题](./06-faq.md) | 常见问题解答 | 问题排查 |

## 这是什么项目？

**AgentScope-Java** 是一个用 Java 编写的 **AI Agent 开发框架**。

简单来说，它帮你：
- 🤖 快速创建能对话的 AI 助手
- 🔧 让 AI 调用各种工具（查天气、算数、搜索等）
- 🧠 管理 AI 的记忆（记住之前的对话）
- 🔄 支持多 AI 协作完成复杂任务

## 核心能力

### 1. ReAct 智能体
采用 **ReAct（Reasoning + Acting）** 模式，AI 会先"思考"再"行动"：
```
用户: 东京现在几点？
AI思考: 用户想知道东京时间，我需要调用 get_current_time 工具
AI行动: 调用 get_current_time("Asia/Tokyo")
AI回答: 东京现在时间是 2024-01-15 14:30:00
```

### 2. 工具调用
让 AI 拥有"双手"，可以执行实际操作：
```java
@Tool(name = "get_weather", description = "获取城市天气")
public String getWeather(@ToolParam(name = "city") String city) {
    // 调用天气 API
    return "北京今天晴天，温度 25°C";
}
```

### 3. 记忆管理
AI 能记住之前的对话：
```
第一次对话:
用户: 我叫小明
AI: 你好小明！

第二次对话:
用户: 我叫什么名字？
AI: 你叫小明！  // AI 记住了之前的信息
```

### 4. 多 Agent 协作
多个 AI 可以分工合作：
- **主管 Agent**: 分配任务
- **研究员 Agent**: 查找资料
- **程序员 Agent**: 写代码

## 技术栈

| 类别 | 技术 | 版本 | 说明 |
|------|------|------|------|
| 语言 | Java | 17+ | 需要 JDK 17 或更高版本 |
| 构建 | Maven | 3.6+ | 项目构建工具 |
| 响应式 | Project Reactor | - | 异步非阻塞编程 |
| JSON | Jackson | - | JSON 序列化 |
| 测试 | JUnit 5 + Mockito | - | 单元测试 |
| 代码格式 | Spotless | - | 自动格式化 |

## 项目结构

```
agentscope-java/
├── agentscope-core/              # 核心库（必须了解）
│   └── src/main/java/io/agentscope/core/
│       ├── ReActAgent.java       # 核心 Agent 实现
│       ├── model/                # 模型接口和实现
│       ├── memory/               # 记忆管理
│       ├── tool/                 # 工具系统
│       ├── message/              # 消息结构
│       ├── hook/                 # 钩子系统
│       └── ...
│
├── agentscope-extensions/        # 扩展模块
│   ├── agentscope-spring-boot-starters/  # Spring Boot 集成
│   ├── agentscope-quarkus-extensions/    # Quarkus 集成
│   ├── agentscope-extensions-rag-*/      # RAG 检索增强
│   └── ...
│
├── agentscope-examples/          # 示例代码（新手必看）
│   └── quickstart/               # 快速入门示例
│
├── agentscope-dependencies-bom/  # 依赖版本管理
│
└── agentscope-distribution/      # 发布包
```

## 五分钟快速体验

### 1. 准备环境
```bash
# 确保已安装 JDK 17+
java -version

# 克隆项目
git clone https://github.com/agentscope-ai/agentscope-java.git
cd agentscope-java
```

### 2. 设置 API Key
```bash
# 使用阿里云 DashScope（推荐）
export DASHSCOPE_API_KEY=你的API密钥
```

获取 API Key: https://dashscope.console.aliyun.com/apiKey

### 3. 运行第一个示例
```bash
# 编译项目
mvn clean install -DskipTests

# 运行基础对话示例
cd agentscope-examples/quickstart
mvn exec:java -Dexec.mainClass="io.agentscope.examples.quickstart.BasicChatExample"
```

## 学习路径

```
第1天: 阅读项目概述 → 运行 BasicChatExample
       ↓
第2天: 学习核心概念 → 理解 Agent、Model、Memory
       ↓
第3天: 运行 ToolCallingExample → 学习工具调用
       ↓
第4天: 运行 StructuredOutputExample → 学习结构化输出
       ↓
第5天: 阅读源码 → 理解 ReActAgent 实现
       ↓
第6天: 自己开发一个 Agent → 实践出真知
```

## 相关资源

- 📖 [官方文档](https://java.agentscope.io/)
- 💬 [Discord 社区](https://discord.gg/eYMpfnkG8h)
- 🐛 [问题反馈](https://github.com/agentscope-ai/agentscope-java/issues)
- 📝 [贡献指南](../CONTRIBUTING.md)