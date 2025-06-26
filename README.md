# Hello Embabel

![Java](https://img.shields.io/badge/java-%23ED8B00.svg?style=for-the-badge&logo=openjdk&logoColor=white)
![Kotlin](https://img.shields.io/badge/kotlin-%237F52FF.svg?style=for-the-badge&logo=kotlin&logoColor=white)
![Spring](https://img.shields.io/badge/spring-%236DB33F.svg?style=for-the-badge&logo=spring&logoColor=white)
![Apache Maven](https://img.shields.io/badge/Apache%20Maven-C71A36?style=for-the-badge&logo=Apache%20Maven&logoColor=white)
![ChatGPT](https://img.shields.io/badge/chatGPT-74aa9c?style=for-the-badge&logo=openai&logoColor=white)

<img align="left" src="https://github.com/embabel/embabel-agent/blob/main/embabel-agent-api/images/315px-Meister_der_Weltenchronik_001.jpg?raw=true" width="180">

&nbsp;&nbsp;&nbsp;&nbsp;

&nbsp;&nbsp;&nbsp;&nbsp;

一个展示 [Embabel 框架](https://github.com/embabel/embabel-agent) AI 代理开发的多语言示例项目。

本项目包含 Java 和 Kotlin 两个版本的实现，演示如何使用 Embabel 框架构建智能代理。

## 📋 项目结构

```
hello-embabel/
├── java/           # Java 版本实现
│   ├── src/
│   ├── pom.xml
│   └── README.md
├── kotlin/         # Kotlin 版本实现
│   ├── src/
│   ├── pom.xml
│   └── README.md
├── .gitmodules     # Git 子模块配置
└── README.md       # 项目说明文档
```

## 🚀 快速开始
如果已经克隆了项目但没有子模块：
```bash
git submodule update --init --recursive
```

### 运行 Java 版本

```bash
cd java
mvn spring-boot:run
```

### 运行 Kotlin 版本

```bash
cd kotlin
mvn spring-boot:run
```

## 💡 使用示例

启动应用后，在 Embabel Shell 中使用以下命令：

```bash
# 创作一个故事
x "Tell me a story about a brave knight"

# 中文示例
x "给我讲一个关于勇敢骑士的故事"
```

## 🔄 完整工作流程

下图展示了从系统启动到故事生成的完整端到端工作流程，使用 GOAP 动态规划：

```mermaid
sequenceDiagram
    participant User as User
    participant Maven as Maven
    participant Spring as Spring Boot
    participant Embabel as Embabel Platform
    participant Ollama as Ollama Service
    participant GOAP as GOAP Planner
    participant Agent as WriteAndReviewAgent
    participant Shell as Spring Shell

    %% Build and Startup Phase
    User->>Maven: mvn spring-boot:run
    Maven->>Maven: Compile 2 source files + 1 test file
    Maven->>Spring: Start Spring Boot application
    
    Spring->>Spring: Activate profiles: "ollama", "starwars", "shell"
    Spring->>Embabel: Initialize AgentPlatformAutoConfiguration
    
    %% Model Discovery Phase
    Embabel->>Ollama: Connect to http://localhost:11434
    Ollama-->>Embabel: Return 16 available models
    Note over Embabel,Ollama: Models: qwen3 series, deepseek-r1, mistral-nemo,<br/>llama3 series, gemma2, codellama, etc.
    
    Embabel->>Embabel: Set default LLM: qwen2.5:latest
    Embabel->>Embabel: Set default embedding: nomic-embed-text:latest
    
    %% Tool and Agent Discovery Phase
    Embabel->>Embabel: Initialize MCP clients (0 found)
    Embabel->>Embabel: Register 6 tool groups (math, rag, etc.)
    
    Embabel->>Agent: Scan and deploy WriteAndReviewAgent
    Agent->>Agent: Analyze action bindings
    Note over Agent: craftStory: UserInput -> Story<br/>reviewStory: UserInput,Story -> ReviewedStory
    
    Embabel->>Shell: Initialize Spring Shell with Star Wars theme
    Shell-->>User: Ready to receive commands
    
    %% User Interaction and GOAP Planning Phase
    User->>Shell: x "给我讲个治愈的故事"
    Shell->>Embabel: Choose appropriate agent
    Embabel->>Embabel: Agent selection (confidence: 0.95)
    Embabel->>GOAP: Create process "hungry_chebyshev"
    
    %% Dynamic Planning Phase 1
    GOAP->>GOAP: Initial world state analysis
    Note over GOAP: {UserInput=TRUE, Story=FALSE,<br/>ReviewedStory=FALSE, hasRun_*=FALSE}
    GOAP->>GOAP: Formulate plan: craftStory -> reviewStory
    
    %% Action Execution Phase 1
    GOAP->>Agent: Execute craftStory action
    Agent->>Ollama: Generate story (temperature=0.9, creative persona)
    Ollama-->>Agent: Return generated story
    Agent-->>GOAP: Story object created
    
    %% Dynamic Replanning Phase
    GOAP->>GOAP: Update world state
    Note over GOAP: {UserInput=TRUE, Story=TRUE,<br/>hasRun_craftStory=TRUE, ReviewedStory=FALSE}
    GOAP->>GOAP: Replan: reviewStory only
    
    %% Action Execution Phase 2
    GOAP->>Agent: Execute reviewStory action
    Agent->>Ollama: Review story (temperature=0.5, reviewer persona)
    Ollama-->>Agent: Return review text
    Agent-->>GOAP: ReviewedStory object created
    
    %% Goal Achievement
    GOAP->>GOAP: Final world state verification
    Note over GOAP: {UserInput=TRUE, Story=TRUE,<br/>ReviewedStory=TRUE, hasRun_*=TRUE}
    GOAP->>GOAP: Goal achieved in 3.29 seconds
    
    GOAP-->>Shell: Return complete ReviewedStory
    Shell-->>User: Display story, review, and reviewer info

    %% Summary
    Note over User,Shell: Total execution: 3.29s<br/>LLM calls: 2 (qwen2.5:latest)<br/>Tokens: 453 prompt + 143 completion<br/>Cost: $0.0000
```

### 🔑 核心特性

- **GOAP 动态规划**: 基于当前世界状态规划和重新规划行动
- **多人格 LLM**: 针对创意和分析任务使用不同的人格和温度参数
- **自动代理选择**: 根据用户输入选择合适的代理
- **状态管理**: 在整个工作流程中跟踪执行状态和可用对象
- **工具集成**: 可扩展的工具系统，注册了 6 个工具组

## 🔧 配置

### LLM 模型配置

编辑 `src/main/resources/application.properties` 文件：

```properties
# 激活配置文件
spring.profiles.active=ollama,shell,starwars

# Ollama 配置
spring.ai.ollama.base-url=http://localhost:11434

# 模型配置
embabel.models.default-llm=qwen2.5
embabel.models.default-embedding-model=nomic-embed-text:latest

# 故事配置
storyWordCount=100
reviewWordCount=100
```