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
├── .github/        # GitHub 配置文件
│   ├── dependabot.yml              # Dependabot 依赖更新配置
│   └── workflows/
│       ├── ci.yml                  # CI/CD 流水线
│       └── update-submodules.yml   # 子模块自动更新
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

## ⚙️ 系统要求

- **Java**: JDK 21 或更高版本
- **Maven**: 3.6+ 
- **Git**: 支持子模块操作
- **操作系统**: Linux, macOS, Windows

## 🚀 快速开始

如果已经克隆了项目但没有子模块：

```bash
git submodule update --init --recursive
```

### 更新子模块

#### 手动更新

如果需要手动将子模块更新到最新版本：

```bash
# 更新所有子模块到最新提交
git submodule update --remote --recursive

# 或者手动进入子模块目录更新
cd java
git pull origin main
cd ../kotlin  
git pull origin main
cd ..

# 提交子模块更新
git add .
git commit -m "Update submodules to latest version"
```

#### 自动更新 🤖

项目已配置完全自动化的更新流程：

**📦 依赖更新流程**：
1. Dependabot 每周一自动检查Maven依赖更新
2. 创建PR并自动标记 `automerge`
3. CI流水线自动测试Java和Kotlin项目
4. 测试通过后自动合并到main分支

**🔄 子模块更新流程**：
1. GitHub Actions 每周一自动检查子模块更新
2. 检测到更新时自动创建PR
3. CI流水线验证两个子项目兼容性
4. 所有测试通过后自动合并

**✅ 完全自动化特性**：
- 🚀 零人工干预的更新流程
- 🔍 自动检测变更和兼容性
- 🛡️ 测试失败时停止自动合并
- 📝 详细的更新日志和说明

**配置文件位置**：
- `.github/dependabot.yml` - Dependabot配置
- `.github/workflows/ci.yml` - CI/CD流水线和自动合并
- `.github/workflows/update-submodules.yml` - 子模块自动更新

**手动触发更新**：
```bash
# 在GitHub仓库页面 Actions 标签页可手动触发工作流
# 或使用GitHub CLI
gh workflow run "Update Submodules"
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

启动应用后，您可以通过以下方式使用：

### Shell 模式 (传统方式)

在 Embabel Shell 中使用以下命令：

```bash
# 创作一个故事
x "Tell me a story about a brave knight"

# 中文示例
x "给我讲一个关于勇敢骑士的故事"
```

### Web API 模式 (开发中) 🚧

**注意**: Web API 当前处于开发阶段，缺少 Embabel Autonomy 集成，无法提供完整的 GOAP 工作流程。

#### 使用 Web 界面

启动应用后，打开浏览器访问：`http://localhost:8080`

#### 使用 REST API

```bash
curl -X POST http://localhost:8080/api/agent/execute \
  -H "Content-Type: application/json" \
  -d '{
    "userInput": "给我讲一个关于勇敢骑士的故事"
  }'
```

#### API 端点

- `POST /api/agent/execute` - 执行故事生成 (⚠️ 当前返回错误，等待 Embabel 集成)
- `GET /api/agent/health` - 检查服务状态
- `GET /api/agent/info` - 获取代理信息

#### 当前限制

Web API 版本当前无法执行完整的工作流程，因为：

1. 缺少 `Autonomy.chooseAndRunAgent()` API 集成
2. 无法进行动态 Agent 选择
3. 不支持 GOAP 规划和执行

要体验完整的 Embabel GOAP 功能，请使用 Shell 模式。

## 🔄 完整工作流程

下图展示了从系统启动到故事生成的完整端到端工作流程，揭示了真实的 GOAP 执行机制：

**注意**：当前 Web API 版本尚未完全实现此工作流程。Shell 版本完整支持此流程，Web API 版本需要 Embabel Autonomy 集成才能实现相同的执行路径。

```mermaid
sequenceDiagram
    participant User as User
    participant Shell as Spring Shell
    participant Autonomy as Autonomy
    participant Platform as AgentPlatform
    participant Process as SimpleAgentProcess
    participant Planner as AStarGoapPlanner
    participant Agent as WriteAndReviewAgent
    participant LLM as Ollama(qwen2.5)
    participant Blackboard as InMemoryBlackboard

    %% User Input and Agent Selection
    User->>Shell: x "给我讲个治愈的故事"
    Shell->>Autonomy: chooseAndRunAgent()
    Autonomy->>LLM: Agent selection request
    Note over LLM: Prompt tokens: 457, Temperature: default
    LLM-->>Autonomy: {"rankings":[{"name":"WriteAndReviewAgent","confidence":0.95}]}
    Note over Autonomy: Completion tokens: 20

    %% Process Creation and Agent Initialization
    Autonomy->>Platform: runAgent(WriteAndReviewAgent)
    Platform->>Platform: createAgentProcess()
    Platform->>Process: Create "confident_cori"
    Process->>Blackboard: Initialize blackboard
    Process->>Blackboard: Bind UserInput
    Note over Process: Agent Description: Generate story and review<br/>Goals: reviewStory with pre-conditions<br/>Actions: craftStory, reviewStory

    %% First GOAP Planning Cycle
    Process->>Process: tick() - First cycle
    Process->>Planner: bestValuePlanToAnyGoal()
    Note over Planner: **World State Analysis**<br/>UserInput=TRUE, Story=FALSE<br/>hasRun_craftStory=FALSE<br/>ReviewedStory=FALSE<br/>hasRun_reviewStory=FALSE

    Planner->>Planner: A* search algorithm
    Note over Planner: **GOAP Actions Available:**<br/>craftStory: UserInput → Story<br/>reviewStory: UserInput,Story → ReviewedStory

    Planner->>Planner: Calculate optimal path to goal
    Note over Planner: **Goal:** reviewStory satisfied<br/>**Path:** craftStory → reviewStory<br/>**Cost:** 0.0, **NetValue:** 0.0

    Planner-->>Process: Plan: craftStory → reviewStory

    %% Execute First Action
    Process->>Process: executeAction(craftStory)
    Process->>Agent: craftStory.execute()
    Agent->>LLM: Transform request (temperature=0.9)
    Note over LLM: Prompt tokens: 258<br/>Input: UserInput("给我讲个治愈的故事")
    LLM-->>Agent: Story text (2.63s)
    Note over LLM: Completion tokens: 82<br/>Output: 阿洛斯小男孩的治愈故事
    Agent-->>Process: Story object
    Process->>Blackboard: Update with Story
    Note over Process: **State Update:**<br/>Story=TRUE<br/>hasRun_craftStory=TRUE

    %% Second GOAP Planning Cycle
    Process->>Process: tick() - Second cycle
    Process->>Planner: bestValuePlanToAnyGoal()
    Note over Planner: **Updated World State:**<br/>UserInput=TRUE, Story=TRUE<br/>hasRun_craftStory=TRUE<br/>ReviewedStory=FALSE<br/>hasRun_reviewStory=FALSE

    Planner->>Planner: A* search with new state
    Note over Planner: **Re-planning Analysis:**<br/>craftStory preconditions no longer met<br/>reviewStory preconditions now satisfied<br/>**New Plan:** reviewStory only

    Planner-->>Process: Plan: reviewStory

    %% Execute Second Action
    Process->>Process: executeAction(reviewStory)
    Process->>Agent: reviewStory.execute()
    Agent->>LLM: Transform request (temperature=0.5)
    Note over LLM: Prompt tokens: 199<br/>Input: UserInput + Story
    LLM-->>Agent: Review text (2s)
    Note over LLM: Completion tokens: 66<br/>Output: 专业书评分析
    Agent->>Agent: Create ReviewedStory with persona
    Note over Agent: Reviewer: Media Book Review<br/>Persona: NY Times Book Reviewer
    Agent-->>Process: ReviewedStory object
    Process->>Blackboard: Update with ReviewedStory
    Note over Process: **Final State Update:**<br/>ReviewedStory=TRUE<br/>hasRun_reviewStory=TRUE

    %% Final GOAP Planning Cycle
    Process->>Process: tick() - Final cycle
    Process->>Planner: bestValuePlanToAnyGoal()
    Note over Planner: **Final World State:**<br/>UserInput=TRUE, Story=TRUE<br/>ReviewedStory=TRUE<br/>hasRun_craftStory=TRUE<br/>hasRun_reviewStory=TRUE

    Planner->>Planner: Goal satisfaction check
    Note over Planner: **Goal Achievement:**<br/>reviewStory goal fully satisfied<br/>All preconditions met<br/>**Status:** COMPLETED

    Planner-->>Process: Goal achieved - no further actions

    %% Process Completion
    Process->>Process: Set status to COMPLETED
    Process-->>Platform: AgentProcessExecution result
    Platform-->>Shell: Formatted output
    Shell-->>User: Display story, review, and reviewer

    %% Summary Statistics
    Note over User,Shell: **Execution Summary:**<br/>Total time: 6.65s<br/>LLM calls: 3 (selection + 2 actions)<br/>Prompt tokens: 914, Completion tokens: 168<br/>Cost: $0.0000<br/>**GOAP Cycles:** 3 planning iterations<br/>**Actions Executed:** 2 of 2 planned
```

## 🔑 核心特性

### 开发与维护

- **🤖 完全自动化更新**: 零人工干预的依赖和子模块更新流程
- **📦 智能依赖管理**: Dependabot自动检查Maven依赖兼容性
- **🔄 子模块自动同步**: 定期自动更新到最新子模块版本
- **✅ 安全的自动合并**: 仅在所有测试通过后自动合并
- **🚀 CI/CD流水线**: 完整的测试、构建和部署自动化

### Shell 模式（完整功能）

- **GOAP 动态规划**: 基于当前世界状态规划和重新规划行动
- **多人格 LLM**: 针对创意和分析任务使用不同的人格和温度参数
- **自动代理选择**: 根据用户输入选择合适的代理
- **状态管理**: 在整个工作流程中跟踪执行状态和可用对象
- **工具集成**: 可扩展的工具系统，注册了 6 个工具组

### Web API 模式（开发中）

- **Web API 支持**: 🆕 提供 RESTful API 接口
- **双模式运行**: 支持传统 Shell 模式和现代 Web API 模式
- **实时 Web 界面**: 🆕 包含用户友好的 Web 界面，无需命令行操作
- **⚠️ 限制**: 当前缺少 Embabel Autonomy 集成，无法实现完整的 GOAP 工作流程

### 实现状态对比

| 功能           | Shell 模式    | Web API 模式         |
| -------------- | ------------- | -------------------- |
| Agent 自动选择 | ✅ 完整支持   | ❌ 待实现            |
| GOAP 规划执行  | ✅ 完整支持   | ❌ 待实现            |
| LLM 集成       | ✅ 完整支持   | ❌ 待实现            |
| 用户接口       | ✅ Shell 命令 | ✅ REST API + Web UI |

**总结**: Shell 模式提供完整的 Embabel GOAP 体验，Web API 模式目前仅提供接口框架，等待 Embabel Autonomy API 集成。

## 📖 详细文档

- [Web API 使用指南](web-api-readme.md) - 详细的 Web API 功能说明
